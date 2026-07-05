# Complete AI Agent Building Guide — For Embedded Engineers

> **Context:** You are an embedded Linux engineer who understands C, Python, kernel internals, and hardware. This guide shows you how to build AI agents that automate your daily work — and then sell those agents as products.

---

## What is an AI Agent? (Beginner Explanation)

A traditional program:
```
Input → Fixed logic → Output
```

An AI agent:
```
Goal → LLM reasoning → Decides which tools to call → Tools return results → 
LLM reasons again → Decides next action → ... → Final answer
```

**Analogy:** A traditional program is like a **vending machine** — press button, get item. An AI agent is like a **junior engineer** — tell them a goal, they figure out the steps, ask clarifying questions, use their tools, and deliver results.

For embedded engineers, AI agents can:
- Analyze kernel logs and explain crash causes
- Review device tree files and find errors
- Search kernel git history for specific bugs
- Generate driver code from specifications
- Write datasheets into test plans

---

## Technology Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| LLM backbone | Claude/GPT-4/local LLaMA | Reasoning and text generation |
| Agent framework | LangGraph (preferred) or LangChain | State machine for agent logic |
| Tool framework | Python functions + MCP | What the agent can "do" |
| Storage | ChromaDB / FAISS | RAG — agent's long-term memory |
| API | FastAPI | Expose agent as HTTP service |
| UI | Streamlit | Quick web interface |

---

## Part 1: Setup — Install Everything

```bash
# Create dedicated Python environment
python3 -m venv ~/ai_agents
source ~/ai_agents/bin/activate

# Core dependencies
pip install langgraph langchain langchain-anthropic langchain-openai
pip install chromadb faiss-cpu sentence-transformers
pip install fastapi uvicorn streamlit
pip install GitPython requests beautifulsoup4 pygments
pip install python-dotenv pydantic

# Create project structure
mkdir -p ~/ai_agents_project/{agents,tools,knowledge,api,ui}
cd ~/ai_agents_project
```

```bash
# .env file — API keys
cat > .env << 'EOF'
ANTHROPIC_API_KEY=sk-ant-...     # Get from console.anthropic.com
OPENAI_API_KEY=sk-...            # Get from platform.openai.com

# OR use local model (Ollama):
OLLAMA_BASE_URL=http://localhost:11434
LOCAL_MODEL=llama3.2:3b
EOF
```

---

## Part 2: Your First Agent — Kernel Log Analyzer

This agent takes a kernel log (dmesg output), analyzes it, and explains what went wrong.

### Step 1: Define the Tools

```python
# tools/kernel_tools.py
import subprocess
import re
from langchain_core.tools import tool

@tool
def run_dmesg() -> str:
    """Get current kernel log messages"""
    result = subprocess.run(['dmesg', '--level=err,warn,crit', '-T'],
                           capture_output=True, text=True)
    return result.stdout[-4000:] if len(result.stdout) > 4000 else result.stdout

@tool
def search_kernel_source(function_name: str) -> str:
    """Search Linux kernel source for a function definition.
    
    Args:
        function_name: The kernel function to look up
    """
    # Search in local kernel source
    try:
        result = subprocess.run(
            ['grep', '-r', '-n', f'int {function_name}', 
             '/usr/src/linux-headers-$(uname -r)/'],
            capture_output=True, text=True, timeout=10
        )
        if result.stdout:
            return result.stdout[:2000]
        return f"Function {function_name} not found in local headers"
    except Exception as e:
        return f"Search error: {e}"

@tool
def explain_error_code(code: str) -> str:
    """Look up what a Linux error code means.
    
    Args:
        code: Error code like EINVAL, EPROBE_DEFER, or number like -22
    """
    error_codes = {
        "EPERM":  ("1",  "Operation not permitted"),
        "ENOENT": ("2",  "No such file or directory"),
        "EIO":    ("5",  "I/O error"),
        "ENOMEM": ("12", "Out of memory"),
        "EBUSY":  ("16", "Device or resource busy"),
        "EINVAL": ("22", "Invalid argument"),
        "ENOTTY": ("25", "Inappropriate ioctl for device"),
        "EPROBE_DEFER": ("517", "Driver probe deferred — dependency not ready"),
        "ENODEV": ("19", "No such device"),
    }
    
    code_upper = code.upper().lstrip('-')
    if code_upper in error_codes:
        num, desc = error_codes[code_upper]
        return f"Error {code}: {desc} (errno={num})"
    return f"Unknown error code: {code}. Check /usr/include/asm/errno.h"

@tool
def check_driver_loaded(driver_name: str) -> str:
    """Check if a kernel driver/module is currently loaded.
    
    Args:
        driver_name: Module name like 'counter_driver' or 'ufs'
    """
    result = subprocess.run(['lsmod'], capture_output=True, text=True)
    lines = [l for l in result.stdout.split('\n') if driver_name.lower() in l.lower()]
    
    if lines:
        return f"Module '{driver_name}' IS loaded:\n" + "\n".join(lines)
    else:
        return f"Module '{driver_name}' is NOT loaded"
```

### Step 2: Build the Agent with LangGraph

```python
# agents/kernel_log_agent.py
import os
from typing import Annotated, TypedDict
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from dotenv import load_dotenv
from tools.kernel_tools import (run_dmesg, search_kernel_source, 
                                  explain_error_code, check_driver_loaded)

load_dotenv()

# State: what the agent remembers between steps
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]

# System prompt: defines agent's personality and capabilities
SYSTEM_PROMPT = """You are an expert Linux kernel engineer with 20 years experience.
Your specialty is analyzing kernel logs, diagnosing driver issues, and explaining 
complex kernel behaviors in simple terms.

When given a kernel log or error, you:
1. Identify the root cause (not just symptoms)
2. Explain what each error message means
3. Suggest specific fixes with code examples
4. Reference the relevant kernel subsystem or driver

You have access to tools:
- run_dmesg: get current kernel log
- search_kernel_source: look up kernel functions
- explain_error_code: decode Linux error codes
- check_driver_loaded: check if a module is loaded

Always use tools when you need current system state. Don't guess."""

# Initialize LLM with tools
tools = [run_dmesg, search_kernel_source, explain_error_code, check_driver_loaded]
llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
llm_with_tools = llm.bind_tools(tools)

# Agent node: LLM decides what to do next
def agent_node(state: AgentState):
    messages = [SystemMessage(content=SYSTEM_PROMPT)] + state["messages"]
    response = llm_with_tools.invoke(messages)
    return {"messages": [response]}

# Build the graph
workflow = StateGraph(AgentState)
workflow.add_node("agent", agent_node)
workflow.add_node("tools", ToolNode(tools))

workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    tools_condition,    # if agent wants tools → tools; if done → END
)
workflow.add_edge("tools", "agent")  # after tools, go back to agent

app = workflow.compile()

# Run the agent
def analyze_kernel_log(user_query: str) -> str:
    result = app.invoke({
        "messages": [HumanMessage(content=user_query)]
    })
    return result["messages"][-1].content

if __name__ == "__main__":
    # Example queries
    print("=== Kernel Log Analyzer ===\n")
    
    answer = analyze_kernel_log(
        "My driver failed to probe with -EPROBE_DEFER. What does this mean "
        "and how do I fix it?"
    )
    print(f"Answer:\n{answer}\n")
    
    answer = analyze_kernel_log(
        "I see 'BUG: scheduling while atomic' in dmesg. Can you check the "
        "current kernel log and explain what's happening?"
    )
    print(f"Answer:\n{answer}")
```

```bash
# Run the agent
cd ~/ai_agents_project
source ~/ai_agents/bin/activate
python3 agents/kernel_log_agent.py
```

---

## Part 3: RAG Agent — Kernel Documentation Assistant

RAG = Retrieval Augmented Generation. The agent retrieves relevant documents from a knowledge base before answering.

Use case: Ask questions about your company's proprietary BSP, datasheets, or internal documentation.

### Step 1: Build the Knowledge Base

```python
# knowledge/build_kb.py
import os
import subprocess
from pathlib import Path
from langchain_community.document_loaders import (
    TextLoader, DirectoryLoader
)
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings

# Download kernel documentation
def prepare_kernel_docs():
    """Download and prepare Linux kernel documentation"""
    docs_dir = Path("knowledge/kernel_docs")
    docs_dir.mkdir(parents=True, exist_ok=True)
    
    # Copy kernel docs from local kernel source
    subprocess.run([
        'cp', '-r', 
        '/usr/src/linux-headers-$(uname -r)/Documentation/driver-api/',
        str(docs_dir)
    ])
    
    # Also add your own notes/files
    # Copy your Embedded-Linux-Mastery files for Ravi's personal KB!
    ravi_docs = Path("/home/devasri/Documents/next-career/Embedded-Linux-Mastery")
    if ravi_docs.exists():
        subprocess.run(['cp', '-r', str(ravi_docs), str(docs_dir / "ravi_notes")])

# Load documents
def build_vector_store():
    # Load all .md and .txt files from knowledge base
    loader = DirectoryLoader(
        "knowledge/kernel_docs",
        glob="**/*.md",
        loader_cls=TextLoader,
        loader_kwargs={"encoding": "utf-8"},
        show_progress=True
    )
    documents = loader.load()
    print(f"Loaded {len(documents)} documents")
    
    # Split into chunks (LLM context window is limited)
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=1000,         # characters per chunk
        chunk_overlap=200,       # overlap prevents cutting at bad points
        separators=["\n## ", "\n### ", "\n\n", "\n", " "]
    )
    chunks = splitter.split_documents(documents)
    print(f"Split into {len(chunks)} chunks")
    
    # Create embeddings (local, no API needed)
    embeddings = HuggingFaceEmbeddings(
        model_name="all-MiniLM-L6-v2",    # small, fast, good quality
        model_kwargs={"device": "cpu"}
    )
    
    # Build vector store
    vectorstore = Chroma.from_documents(
        documents=chunks,
        embedding=embeddings,
        persist_directory="knowledge/chroma_db"
    )
    vectorstore.persist()
    print(f"Vector store built: {len(chunks)} chunks indexed")
    return vectorstore

if __name__ == "__main__":
    prepare_kernel_docs()
    build_vector_store()
```

### Step 2: RAG Agent

```python
# agents/rag_agent.py
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# Load existing vector store
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2",
                                    model_kwargs={"device": "cpu"})
vectorstore = Chroma(
    persist_directory="knowledge/chroma_db",
    embedding_function=embeddings
)
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})  # top 5 results

# RAG prompt
prompt = ChatPromptTemplate.from_template("""
You are an expert embedded Linux engineer. Answer the question using ONLY 
the provided context. If the context doesn't contain enough information, 
say so clearly.

Context from knowledge base:
{context}

Question: {question}

Provide a detailed technical answer with code examples where relevant.
""")

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")

def format_docs(docs):
    return "\n\n---\n\n".join([
        f"[Source: {doc.metadata.get('source', 'unknown')}]\n{doc.page_content}"
        for doc in docs
    ])

# Build RAG chain
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

def ask_kb(question: str) -> str:
    return rag_chain.invoke(question)

if __name__ == "__main__":
    print(ask_kb("How do I implement DMA in a platform driver?"))
    print(ask_kb("What is EPROBE_DEFER and when does it happen?"))
    print(ask_kb("Show me a complete platform driver example with PM ops"))
```

---

## Part 4: Device Tree Validator Agent

This is a **practical agent you can sell as a service** — it analyzes device tree files and finds bugs.

```python
# agents/dt_validator_agent.py
"""
Device Tree Validator Agent
- Takes DT source (.dts) file as input
- Validates syntax and common mistakes
- Checks compatible strings, phandle references, address cells
- Suggests fixes with explanations
"""

from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
import subprocess
import re

@tool
def compile_device_tree(dts_content: str) -> str:
    """Try to compile a device tree source. Returns compiler errors if any.
    
    Args:
        dts_content: The full .dts file content as a string
    """
    # Write to temp file
    with open("/tmp/test_check.dts", "w") as f:
        f.write(dts_content)
    
    result = subprocess.run(
        ['dtc', '-I', 'dts', '-O', 'dtb', '-o', '/dev/null', '/tmp/test_check.dts'],
        capture_output=True, text=True
    )
    
    if result.returncode == 0:
        return "DTS compiled successfully — no syntax errors"
    else:
        return f"DTC errors:\n{result.stderr}"

@tool  
def check_compatible_string(compatible: str) -> str:
    """Check if a compatible string follows Linux kernel conventions.
    
    Args:
        compatible: The compatible string like 'qcom,sc7180-ufs' or 'myvendor,my-device'
    """
    # Correct format: vendor,device-name (all lowercase, hyphens not underscores)
    issues = []
    
    if not re.match(r'^[a-z0-9]+,[a-z0-9-]+$', compatible):
        if '_' in compatible:
            issues.append(f"Use hyphens not underscores: {compatible.replace('_', '-')}")
        if any(c.isupper() for c in compatible):
            issues.append(f"Must be lowercase: {compatible.lower()}")
        if ',' not in compatible:
            issues.append(f"Must have vendor prefix: 'yourvendor,{compatible}'")
    
    if not issues:
        return f"'{compatible}' is valid"
    return f"Issues with '{compatible}':\n" + "\n".join(f"  - {i}" for i in issues)

@tool
def analyze_reg_property(reg_value: str, address_cells: int, size_cells: int) -> str:
    """Analyze a DT reg property and explain what it means.
    
    Args:
        reg_value: The reg property value like '<0x0 0xfe650000 0x0 0x1000>'
        address_cells: Value of #address-cells in parent node
        size_cells: Value of #size-cells in parent node
    """
    values = [int(v, 16) if v.startswith('0x') else int(v) 
              for v in reg_value.strip('<>').split()]
    
    if len(values) != (address_cells + size_cells):
        return (f"ERROR: Expected {address_cells + size_cells} values "
                f"(#address-cells={address_cells} + #size-cells={size_cells}), "
                f"got {len(values)}")
    
    if address_cells == 2:
        addr_high, addr_low = values[0], values[1]
        addr = (addr_high << 32) | addr_low
    else:
        addr = values[0]
    
    if size_cells == 2:
        size_high, size_low = values[address_cells], values[address_cells+1]
        size = (size_high << 32) | size_low
    else:
        size = values[address_cells]
    
    return (f"Register mapping:\n"
            f"  Base address: 0x{addr:016x}\n"
            f"  Size: 0x{size:x} ({size} bytes = {size/1024:.1f} KB)")

# Build agent
from langgraph.prebuilt import create_react_agent

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
tools = [compile_device_tree, check_compatible_string, analyze_reg_property]

agent = create_react_agent(llm, tools, 
    state_modifier="""You are a Device Tree expert. When given DTS content or questions,
    use the available tools to validate and analyze it. Always compile the DTS first, 
    then check specific properties. Explain issues clearly with the correct fix.""")

def validate_dts(dts_content_or_question: str) -> str:
    result = agent.invoke({
        "messages": [{"role": "user", "content": dts_content_or_question}]
    })
    return result["messages"][-1].content

# Example usage
if __name__ == "__main__":
    sample_dts = """
/ {
    my_ufs: ufs@fc580000 {
        compatible = "qcom,SC7180-ufs";   /* bug: uppercase */
        reg = <0xfc580000 0x2500>;        /* bug: wrong address-cells */
        clocks = <&gcc GCC_UFS_PHY_AHB_CLK>;
        clock_names = "core_clk";         /* bug: should be clock-names */
        status = "okay";
    };
};
"""
    print(validate_dts(f"Please validate this DTS snippet:\n{sample_dts}"))
```

---

## Part 5: AI Code Review Agent for Driver PRs

```python
# agents/driver_review_agent.py
"""
Kernel Driver Code Reviewer
- Takes kernel driver C code as input  
- Reviews for: memory safety, locking, error handling, style
- Returns structured review with severity levels
"""

from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field
from typing import List

class ReviewIssue(BaseModel):
    severity: str = Field(description="critical/major/minor/style")
    line: int = Field(description="line number where issue occurs")
    issue: str = Field(description="description of the problem")
    fix: str = Field(description="concrete suggested fix")
    explanation: str = Field(description="why this matters")

class CodeReview(BaseModel):
    summary: str = Field(description="overall assessment")
    score: int = Field(description="score 1-10")
    issues: List[ReviewIssue] = Field(description="list of issues found")
    positive_aspects: List[str] = Field(description="good things about the code")

REVIEW_PROMPT = ChatPromptTemplate.from_template("""
You are a senior kernel engineer doing code review. Review this Linux kernel driver code
for the following categories:

1. CRITICAL: Memory safety (missing kfree, use-after-free, buffer overflow)
2. CRITICAL: Locking (missing lock, sleeping in atomic, double lock)
3. MAJOR: Error handling (unchecked return values, missing devm_, no dev_err_probe)
4. MAJOR: Resource management (IRQ not freed, clock not disabled, ioremap not unmapped)
5. MINOR: Coding style (kernel style violations, naming conventions)
6. MINOR: Comments and documentation

Return a JSON object matching this schema:
{format_instructions}

Code to review:
```c
{code}
```
""")

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
parser = JsonOutputParser(pydantic_object=CodeReview)

review_chain = REVIEW_PROMPT | llm | parser

def review_driver_code(code: str) -> CodeReview:
    return review_chain.invoke({
        "code": code,
        "format_instructions": parser.get_format_instructions()
    })

def print_review(review: CodeReview):
    print(f"\n{'='*60}")
    print(f"CODE REVIEW SCORE: {review.score}/10")
    print(f"Summary: {review.summary}")
    print(f"{'='*60}\n")
    
    if review.issues:
        print("ISSUES FOUND:")
        for i, issue in enumerate(review.issues, 1):
            emoji = {"critical": "🔴", "major": "🟡", "minor": "🔵", "style": "⚪"}.get(
                issue.severity, "•")
            print(f"\n{i}. [{issue.severity.upper()}] Line {issue.line}: {emoji} {issue.issue}")
            print(f"   Fix: {issue.fix}")
            print(f"   Why: {issue.explanation}")
    
    if review.positive_aspects:
        print(f"\nGOOD PRACTICES:")
        for aspect in review.positive_aspects:
            print(f"  ✓ {aspect}")

if __name__ == "__main__":
    test_code = """
static int my_probe(struct platform_device *pdev)
{
    struct my_priv *priv;
    void __iomem *base;
    int irq, ret;
    
    priv = kzalloc(sizeof(*priv), GFP_KERNEL);  /* not devm_ */
    if (!priv) return -ENOMEM;
    
    base = ioremap(0xfe650000, 0x1000);          /* not devm_ */
    
    irq = platform_get_irq(pdev, 0);
    request_irq(irq, my_handler, 0, "drv", priv);  /* not devm_, unchecked */
    
    ret = some_init();
    if (ret) return ret;   /* BUG: kfree/iounmap/free_irq leaked! */
    
    return 0;
}
"""
    review = review_driver_code(test_code)
    print_review(review)
```

---

## Part 6: Streamlit UI for Your Agents

```python
# ui/agent_dashboard.py
import streamlit as st
import sys
sys.path.insert(0, "..")
from agents.kernel_log_agent import analyze_kernel_log
from agents.dt_validator_agent import validate_dts
from agents.driver_review_agent import review_driver_code, print_review

st.set_page_config(
    page_title="Embedded AI Assistant",
    page_icon="🔧",
    layout="wide"
)

st.title("🔧 Embedded Linux AI Assistant")
st.caption("Powered by Claude + LangGraph | For embedded engineers")

tab1, tab2, tab3 = st.tabs(["Kernel Log Analyzer", "DTS Validator", "Driver Code Review"])

with tab1:
    st.header("Kernel Log Analyzer")
    query = st.text_area(
        "Ask about kernel logs or describe an issue:",
        height=100,
        placeholder="e.g., 'My driver fails with EPROBE_DEFER. Check my current dmesg.'"
    )
    if st.button("Analyze", key="btn1") and query:
        with st.spinner("Analyzing..."):
            answer = analyze_kernel_log(query)
        st.markdown(answer)

with tab2:
    st.header("Device Tree Validator")
    dts_input = st.text_area(
        "Paste your .dts snippet:",
        height=300,
        placeholder="/ {\n    my_dev: device@fe650000 {\n        compatible = ...\n    };\n};"
    )
    if st.button("Validate DTS", key="btn2") and dts_input:
        with st.spinner("Validating..."):
            result = validate_dts(f"Validate this DTS:\n{dts_input}")
        st.markdown(result)

with tab3:
    st.header("Driver Code Reviewer")
    code_input = st.text_area(
        "Paste kernel driver code:",
        height=400,
        placeholder="static int my_probe(struct platform_device *pdev) {...}"
    )
    if st.button("Review Code", key="btn3") and code_input:
        with st.spinner("Reviewing..."):
            review = review_driver_code(code_input)
        
        col1, col2 = st.columns([1, 3])
        with col1:
            st.metric("Score", f"{review.score}/10")
        with col2:
            st.write(review.summary)
        
        for issue in review.issues:
            color = {"critical": "🔴", "major": "🟡", "minor": "🔵", "style": "⚪"}.get(
                issue.severity, "•")
            with st.expander(f"{color} Line {issue.line}: {issue.issue}"):
                st.write(f"**Fix:** {issue.fix}")
                st.write(f"**Why:** {issue.explanation}")
```

```bash
# Run the dashboard
cd ~/ai_agents_project
streamlit run ui/agent_dashboard.py
# Opens at http://localhost:8501
```

---

## Part 7: Connect to MCP (Model Context Protocol)

MCP lets your agents connect to VS Code, GitHub, filesystem, and other tools:

```python
# mcp_server.py — expose your tools as MCP server
# Install: pip install mcp

from mcp.server import Server
from mcp.server.models import InitializationOptions
import mcp.types as types
from tools.kernel_tools import run_dmesg, check_driver_loaded

server = Server("embedded-linux-tools")

@server.list_tools()
async def list_tools():
    return [
        types.Tool(
            name="dmesg_analyze",
            description="Get and analyze kernel log messages",
            inputSchema={"type": "object", "properties": {
                "filter": {"type": "string", "description": "grep filter"}
            }}
        ),
        types.Tool(
            name="check_module",
            description="Check if a kernel module is loaded",
            inputSchema={"type": "object", "properties": {
                "name": {"type": "string", "description": "module name"}
            }, "required": ["name"]}
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "dmesg_analyze":
        result = run_dmesg.invoke({})
        return [types.TextContent(type="text", text=result)]
    elif name == "check_module":
        result = check_driver_loaded.invoke({"driver_name": arguments["name"]})
        return [types.TextContent(type="text", text=result)]
    raise ValueError(f"Unknown tool: {name}")

if __name__ == "__main__":
    import asyncio, mcp.server.stdio
    asyncio.run(mcp.server.stdio.stdio_server(server))
```

---

## Your 5-Agent Portfolio

After building these, your GitHub shows:

| Agent | What It Does | Sellable As |
|-------|-------------|-------------|
| `kernel-log-agent` | Analyzes dmesg, explains crashes | $500-2000 one-time tool |
| `dts-validator-agent` | Reviews DT files, catches bugs | $1000/month SaaS |
| `driver-review-agent` | Code review for kernel drivers | Part of consulting |
| `rag-bsp-assistant` | Answers questions about your BSP | $5000+ enterprise tool |
| `dt-to-driver-generator` | Generates driver skeleton from DT | Consulting demo |
