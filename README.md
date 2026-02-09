# Pipeline Agent

A flexible agent framework for building LLM-powered automation pipelines with tool calling, resource management, and state machine orchestration.

## Overview

Pipeline Agent is a Python framework that enables building sophisticated AI agents with:

- **Resource Injection** - Declarative dependency injection for LLMs, embeddings, and other resources
- **Tool System** - Expose Python methods as LLM-callable tools with automatic schema generation
- **State Machines** - FSM-based workflow orchestration with error recovery
- **Agent Abstractions** - Pre-built agents for code generation, testing, and review
- **Robust Tool Calling** - Alignment system for correcting misspelled tool names/arguments

## Architecture

```mermaid
graph TD
    Config[config.yaml] --> RP[ResourceProvider]
    RP --> Agent[Agent]
    Agent --> Tools[ToolProvider]
    Agent --> FSM[State Machine]
    FSM --> Memory[MemoryLedger]
```

## Modules

### Core Framework

- **[core/](core/README.md)** - Resource management, agent abstractions, FSM, and memory
- **[chat/](chat/README.md)** - Chat model implementations (Ollama) and tool alignment
- **[embeddings/](embeddings/README.md)** - Semantic and lexical text matching for tool alignment

### Agent Implementations

- **[agent/](agent/README.md)** - Pre-built agents (Simple, Reviewer, PythonCoder, etc.)

### Tools & Utilities

- **[coding/](coding/README.md)** - Python workspace management and script execution
- **[cmd_line/](cmd_line/README.md)** - Command execution and process monitoring
- **[directory/](directory/README.md)** - File system access with lazy loading

### Deprecated

- **jenkins_utils/** - Deprecated, scheduled for refactoring

## Quick Start

### Basic Agent

```python
from pipiline_agent.agent.simple import Simple
from pipiline_agent.core.resources import ResourceProvider

# Initialize resources from config
provider = ResourceProvider("config/config.yaml")
agent = provider.initialize_user("my_agent")

# Execute agent
result = agent.execute_agent("What is 2+2?")
print(result)
```

### Agent with Tools

```python
from pipiline_agent.agent.simple import PythonCoder

# Agent with Python workspace tools
coder = PythonCoder(
    workspace_path="./workspace",
    use_venv=True
)

# Agent can create and run scripts
result = coder.execute_agent(
    "Create a script that processes CSV files"
)
```

### Custom Agent

```python
class PythonCoder(PlainSimpleAgent):
    model: Annotated[LLMFactory, resource(category="llm", rid="llm")]
    tool_aligner: Annotated[ToolAlignerFactory, resource(category="tool_aligner", rid="tool_aligner")]
    python_coder_prompt: Annotated[SysPromptFactory, resource(category="sysprompt", rid="python_coder_prompt")]
    python_workspace: Annotated[PythonWorkSpaceFactory, ToolsDefinition(name="python_workspace", bind_to="model")] = PythonWorkSpaceFactory()

    def __init__(self, sys_prompt: str = None, workspace_path: str = "./workspace", use_venv: bool = False):
        tool_args = {
            "python_workspace": {
                "path": workspace_path,
                "create_venv": use_venv
            }
        }
        super().__init__(model_name="model", sys_prompt=sys_prompt or "", tool_args=tool_args)
        self.define_output_schema(schema_validator={
            "type": "object",
            "properties": {
                "script_path" : {"type": "string"},
                "script_args" : {"type": "array", "items": {"type": "string"}},
                "script_output" : {"type": "string"},
                "is_interactive" : {"type": "boolean"},
                "summarization" : {"type": "string"}
            }
        },
        schema={
            "script_path": "path to the created script",
            "script_args": "arguments for the created script",
            "script_output": "output of the created script",
            "is_interactive": "indicates if the created script is interactive",
            "summarization": "summarization of the created script"
        })
        if hasattr(self, "python_coder_prompt") and self.python_coder_prompt:
             self.add_sysprompt(self.python_coder_prompt)
```

## Key Features

### Resource Injection

Define dependencies declaratively with automatic injection:

```python
class MyAgent(ResourceUser):
    model: Annotated[LLMFactory, resource(category="llm", rid="llm")]
    embeddings: Annotated[Any, resource(category="embeddings", rid="text-embedding")]
```

### Tool Calling

Expose methods as LLM-callable tools:

```python
class MyTools(ToolProvider):
    @toolmethod(name="search")
    def search(self, query: str) -> str:
        """Search for information"""
        return search_api(query)
```

### Tool Alignment

Automatically correct misspelled tool calls using lexical and semantic matching:

```python
# LLM calls "creat_script" instead of "create_script"
# ToolAligner automatically corrects it
aligner.align_tool_call(ToolCall(name="creat_script", args={...}))
# Returns: ToolCall(name="create_script", args={...})
```

### State Machine Workflows

Build complex workflows with FSM orchestration:

```python
fsm = FSM()
fsm.add_state("planning", PlanningAgent())
fsm.add_state("coding", CodingAgent())
fsm.add_state("testing", TestingAgent())

fsm.add_transition("planning", StateResult.NEXT, "coding")
fsm.add_transition("coding", StateResult.NEXT, "testing")

fsm.run(initial_state="planning")
```

## Configuration

Example:

```yaml
rresources:
  main_llm:
    category: llm
    type: ollama
    host: "http://localhost:11434"
    model: "granite4:3b"
    induced_tools: True

  gemma12b:
    category: llm
    type: ollama
    host: "http://localhost:11434"
    model: "gemma3:12b"
    induced_tools: True

  deepseek:
    category: llm
    type: ollama
    host: "http://localhost:11434"
    model: "deepseek-r1:7b"
    induced_tools: True
    thinking: True

  lfm_thinking:
    category: llm
    type: ollama
    host: "http://localhost:11434"
    model: "lfm2.5-thinking:1.2b"
    induced_tools: True
    thinking: True

  granite:
    category: llm
    type: ollama
    host: "http://localhost:11434"
    model: "granite4:3b"
    induced_tools: True
    
  verification_pre_sysprompt:
    category: sysprompt
    type: sysprompt
    source: "verifier.prompt"
    
  python_coder_prompt:
    category: sysprompt
    type: sysprompt
    source: "python_coder.prompt"

  python_tester_prompt:
    category: sysprompt
    type: sysprompt
    source: "tester.prompt"

  main_tool_aligner:
    category: tool_aligner
    type: tool_aligner
    model_name: "BAAI/bge-small-en-v1.5"
    threads: 2
    tool_name_lexical_threshold: 0.8
    tool_name_semantic_threshold: 0.5
    tool_args_lexical_threshold: 0.8
    tool_args_semantic_threshold: 0.6


users:
  python_coder:
    module: pipiline_agent.agent.simple
    class: PythonCoder
    resources:
      llm: gemma12b
      python_coder_prompt: python_coder_prompt
      tool_aligner: main_tool_aligner
```

## Requirements

- Python 3.10+
- Ollama (for LLM support)
- FastEmbed (for embeddings)

## Installation

```bash
pip install -r requirements.txt
```

## Documentation

Each module has comprehensive documentation:

- [Core Framework](core/README.md) - Architecture and abstractions
- [Agent Module](agent/README.md) - Pre-built agents
- [Chat Module](chat/README.md) - Chat models and tool alignment
- [Coding Module](coding/README.md) - Code generation tools
- [Command Line Module](cmd_line/README.md) - Process management
- [Directory Module](directory/README.md) - File system utilities
- [Embeddings Module](embeddings/README.md) - Text matching

## License

See LICENSE file for details.
