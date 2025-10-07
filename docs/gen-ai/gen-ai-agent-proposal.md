# Proposal: Workflow and Task Semantic Conventions for GenAI Observability

## Summary

This proposal extends the existing [GenAI Agent semantic conventions](https://github.com/open-telemetry/semantic-conventions/blob/main/docs/gen-ai/gen-ai-agent-spans.md) with **Workflow** and **Task** span types to support complex multi-agent systems. It also adds agent context to existing LLM and embedding metrics for better attribution.

## Motivation

### Current State
The existing GenAI semantic conventions (as of v1.28.0) include:
- ✅ **Agent spans**: `create_agent` and `invoke_agent` operations ([docs](https://github.com/open-telemetry/semantic-conventions/blob/main/docs/gen-ai/gen-ai-agent-spans.md))
- ✅ **LLM spans**: `gen_ai.client.chat` and `gen_ai.client.completion`
- ✅ **Embedding spans**: `gen_ai.client.embeddings`
- ✅ **Metrics**: Token usage, duration, etc.

### Gap
Modern agentic AI systems involve **orchestration** that isn't captured:
- **Workflows**: Top-level orchestration coordinating multiple agents (e.g., LangGraph graphs, CrewAI crews)
- **Tasks**: Discrete units of work assigned to or decomposed by agents
- **Hierarchical relationships**: Workflow → Agents → Tasks → LLM calls
- **Attribution**: Which agent/task generated which LLM calls and costs

Without workflow and task conventions, we can observe individual agents but not their orchestration or task decomposition.

## Proposed Changes

### 1. New Span Types

#### `gen_ai.workflow` Span
Represents a workflow orchestrating multiple agents and tasks.

**Span Name Format:**
- `gen_ai.workflow {workflow_name}`
- Examples: `gen_ai.workflow multi_agent_rag`, `gen_ai.workflow customer_support_pipeline`

**Required Attributes:**
- `gen_ai.workflow.name` (string): Name/identifier of the workflow
- `gen_ai.operation.name` (string): `"workflow"`

**Optional Attributes:**
- `gen_ai.workflow.type` (string): Orchestration type (e.g., "sequential", "parallel", "graph", "dynamic")
- `gen_ai.workflow.description` (string): Human-readable description of workflow's purpose
- `gen_ai.framework` (string): Framework implementing the workflow (e.g., "langgraph", "crewai", "autogen")

**Event Attributes (for content capture):**
- `gen_ai.workflow.initial_input` (string): User's initial query/request that triggered the workflow
- `gen_ai.workflow.final_output` (string): Final response/result produced by the workflow

**Example:**
```json
{
  "span_name": "gen_ai.workflow multi_agent_rag",
  "attributes": {
    "gen_ai.operation.name": "workflow",
    "gen_ai.workflow.name": "multi_agent_rag",
    "gen_ai.workflow.type": "sequential",
    "gen_ai.workflow.description": "Multi-agent RAG with research, memory, and synthesis",
    "gen_ai.framework": "langgraph"
  },
  "events": [
    {
      "name": "gen_ai.content.prompt",
      "attributes": {
        "gen_ai.workflow.initial_input": "What are the latest AI developments?"
      }
    },
    {
      "name": "gen_ai.content.completion",
      "attributes": {
        "gen_ai.workflow.final_output": "Based on recent research..."
      }
    }
  ]
}
```

#### `gen_ai.task` Span
Represents a discrete unit of work in an agentic AI system.

**Span Name Format:**
- `gen_ai.task {task_name}`
- Examples: `gen_ai.task research_task`, `gen_ai.task synthesis_task`

**Required Attributes:**
- `gen_ai.task.name` (string): Name/identifier of the task
- `gen_ai.operation.name` (string): `"task"`

**Optional Attributes:**
- `gen_ai.task.type` (string): Task type (e.g., "research", "planning", "execution", "reflection", "tool_use")
- `gen_ai.task.objective` (string): What the task aims to achieve
- `gen_ai.task.source` (string): Where task originated - `"workflow"` or `"agent"`
- `gen_ai.task.assigned_agent` (string): Name of agent assigned to execute the task (for workflow-assigned tasks)
- `gen_ai.task.status` (string): Task status (e.g., "pending", "in_progress", "completed", "failed")

**Event Attributes (for content capture):**
- `gen_ai.task.input_data` (string): Input data/context for the task
- `gen_ai.task.output_data` (string): Output data/result from the task

**Example:**
```json
{
  "span_name": "gen_ai.task research_task",
  "attributes": {
    "gen_ai.operation.name": "task",
    "gen_ai.task.name": "research_task",
    "gen_ai.task.type": "research",
    "gen_ai.task.objective": "Search and analyze current information",
    "gen_ai.task.source": "agent",
    "gen_ai.task.status": "completed"
  },
  "events": [
    {
      "name": "gen_ai.content.prompt",
      "attributes": {
        "gen_ai.task.input_data": "What are the latest AI developments?"
      }
    },
    {
      "name": "gen_ai.content.completion",
      "attributes": {
        "gen_ai.task.output_data": "Recent AI breakthroughs include..."
      }
    }
  ]
}
```

**Span Hierarchy:**
```
gen_ai.workflow multi_agent_rag
├── gen_ai.agent invoke_agent research_agent
│   ├── gen_ai.task research_task
│   │   └── gen_ai.client chat
│   └── gen_ai.client chat
├── gen_ai.agent invoke_agent memory_agent
│   ├── gen_ai.task memory_retrieval_task
│   │   └── gen_ai.client chat
│   └── gen_ai.client embeddings
└── gen_ai.agent invoke_agent synthesizer_agent
    ├── gen_ai.task synthesis_task
    │   └── gen_ai.client chat
    └── gen_ai.client chat
```

### 2. New Metrics

#### `gen_ai.workflow.duration` (Histogram)
Measures the duration of workflow orchestration.

**Unit:** `s` (seconds)

**Attributes:**
- `gen_ai.operation.name` (string): `"workflow"`
- `gen_ai.workflow.name` (string): Workflow name
- `gen_ai.workflow.type` (string, optional): Workflow type
- `gen_ai.framework` (string, optional): Framework name

#### `gen_ai.agent.duration` (Histogram)
Measures the duration of agent operations (extends existing agent spans with metrics).

**Unit:** `s` (seconds)

**Attributes:**
- `gen_ai.operation.name` (string): `"create_agent"` or `"invoke_agent"`
- `gen_ai.agent.name` (string): Agent name
- `gen_ai.agent.id` (string): Agent execution ID
- `gen_ai.agent.type` (string, optional): Agent type
- `gen_ai.framework` (string, optional): Framework name

#### `gen_ai.task.duration` (Histogram)
Measures the duration of task execution.

**Unit:** `s` (seconds)

**Attributes:**
- `gen_ai.operation.name` (string): `"task"`
- `gen_ai.task.name` (string): Task name
- `gen_ai.task.type` (string, optional): Task type
- `gen_ai.task.source` (string, optional): Task source
- `gen_ai.agent.name` (string, optional): Assigned agent name

### 3. Agent Context in Existing Metrics

To enable cost attribution and performance analysis, add agent context to existing LLM and embedding metrics:

#### `gen_ai.client.operation.duration` Histogram
**New Optional Attributes:**
- `gen_ai.agent.name` (string): Name of the agent making the LLM/embedding call
- `gen_ai.agent.id` (string): ID of the agent execution

#### `gen_ai.client.token.usage` Histogram  
**New Optional Attributes:**
- `gen_ai.agent.name` (string): Name of the agent making the LLM call
- `gen_ai.agent.id` (string): ID of the agent execution

**Use Cases:**
- **Cost attribution**: Track token usage and costs per agent
- **Performance monitoring**: Identify which agents make expensive LLM calls
- **Optimization**: Find agents that could benefit from caching or smaller models

**Example Metrics:**
```
# LLM duration with agent context
gen_ai.client.operation.duration{
  gen_ai.operation.name="chat",
  gen_ai.request.model="gpt-4",
  gen_ai.agent.name="research_agent",
  gen_ai.agent.id="550e8400-e29b-41d4-a716-446655440000"
} = 2.1s

# Token usage with agent context
gen_ai.client.token.usage{
  gen_ai.operation.name="chat",
  gen_ai.request.model="gpt-4",
  gen_ai.token.type="input",
  gen_ai.agent.name="research_agent"
} = 1500 tokens
```

## Implementation Examples

### Example 1: Multi-Agent Workflow (Complete Hierarchy)

This example shows how workflow, agent, task, and LLM spans work together:

```
gen_ai.workflow multi_agent_rag
├── gen_ai.agent invoke_agent research_agent
│   ├── gen_ai.task research_task
│   │   └── gen_ai.client chat gpt-4
│   └── gen_ai.client chat gpt-4
├── gen_ai.agent invoke_agent memory_agent
│   ├── gen_ai.task memory_retrieval_task
│   │   └── gen_ai.client chat gpt-4
│   └── gen_ai.client embeddings text-embedding-3
└── gen_ai.agent invoke_agent synthesizer_agent
    ├── gen_ai.task synthesis_task
    │   └── gen_ai.client chat gpt-4
    └── gen_ai.client chat gpt-4
```

**Key Relationships:**
- **Workflow** orchestrates multiple agents sequentially
- Each **Agent** (research, memory, synthesizer) performs its specialized role
- Each agent creates **Tasks** to organize its work
- **LLM calls** are made within task or agent context

### Example 2: Complete Metrics for Cost Attribution

```
# Workflow duration
gen_ai.workflow.duration{
  gen_ai.operation.name="workflow",
  gen_ai.workflow.name="multi_agent_rag",
  gen_ai.workflow.type="sequential",
  gen_ai.framework="langgraph"
} = 45.2s

# Agent duration
gen_ai.agent.duration{
  gen_ai.operation.name="invoke_agent",
  gen_ai.agent.name="research_agent",
  gen_ai.agent.type="researcher",
  gen_ai.framework="langgraph"
} = 15.3s

# Task duration
gen_ai.task.duration{
  gen_ai.operation.name="task",
  gen_ai.task.name="research_task",
  gen_ai.task.type="research",
  gen_ai.task.source="agent"
} = 12.1s

# LLM calls with agent context (for cost attribution)
gen_ai.client.operation.duration{
  gen_ai.operation.name="chat",
  gen_ai.request.model="gpt-4",
  gen_ai.agent.name="research_agent"
} = 2.1s

gen_ai.client.token.usage{
  gen_ai.operation.name="chat",
  gen_ai.request.model="gpt-4",
  gen_ai.token.type="input",
  gen_ai.agent.name="research_agent"
} = 1500 tokens
```

**Analysis Enabled:**
- Total workflow cost = sum of all agent token costs
- Research agent cost = sum of tokens where `gen_ai.agent.name="research_agent"`
- Identify most expensive agent in the workflow

## Attribute Registry

### New Workflow Attributes

| Attribute | Type | Description | Requirement Level | Examples |
|-----------|------|-------------|-------------------|----------|
| `gen_ai.workflow.name` | string | Name/identifier of the workflow | Required | `"multi_agent_rag"`, `"customer_support"` |
| `gen_ai.workflow.type` | string | Orchestration type | Recommended | `"sequential"`, `"parallel"`, `"graph"`, `"dynamic"` |
| `gen_ai.workflow.description` | string | Human-readable description | Optional | `"Multi-agent RAG with research and synthesis"` |
| `gen_ai.workflow.initial_input` | string | User's initial query/request | Optional (event) | `"What are the latest AI developments?"` |
| `gen_ai.workflow.final_output` | string | Final response/result | Optional (event) | `"Based on recent research..."` |

### New Task Attributes

| Attribute | Type | Description | Requirement Level | Examples |
|-----------|------|-------------|-------------------|----------|
| `gen_ai.task.name` | string | Name/identifier of the task | Required | `"research_task"`, `"synthesis_task"` |
| `gen_ai.task.type` | string | Task type | Recommended | `"research"`, `"planning"`, `"execution"`, `"reflection"` |
| `gen_ai.task.objective` | string | What the task aims to achieve | Optional | `"Search and analyze current information"` |
| `gen_ai.task.source` | string | Where task originated | Optional | `"workflow"`, `"agent"` |
| `gen_ai.task.assigned_agent` | string | Agent assigned to execute | Optional | `"research_agent"` |
| `gen_ai.task.status` | string | Task status | Optional | `"pending"`, `"in_progress"`, `"completed"`, `"failed"` |
| `gen_ai.task.input_data` | string | Input data/context | Optional (event) | `"What are the latest AI developments?"` |
| `gen_ai.task.output_data` | string | Output data/result | Optional (event) | `"Recent AI breakthroughs include..."` |

### Modified Attributes (Agent Context in Metrics)

| Attribute | Type | Added To | Description |
|-----------|------|----------|-------------|
| `gen_ai.agent.name` | string | `gen_ai.client.*` metrics | Agent making the LLM/embedding call |
| `gen_ai.agent.id` | string | `gen_ai.client.*` metrics | Agent execution ID |

## Benefits

1. **Complete Observability**: Full visibility into workflow orchestration, not just individual agents
2. **Hierarchical Tracing**: Clear lineage from workflow → agents → tasks → LLM calls
3. **Cost Attribution**: Track token usage and costs per agent in multi-agent systems
4. **Performance Analysis**: Identify bottlenecks at workflow, agent, or task level
5. **Debugging**: Pinpoint which component (workflow/agent/task) is causing issues
6. **Framework Agnostic**: Works across LangGraph, CrewAI, AutoGen, and custom frameworks

## Backward Compatibility

- **Fully backward compatible**: All new span types and attributes are additive
- **Existing agent spans unchanged**: `create_agent` and `invoke_agent` continue to work as defined
- **Optional agent context**: Adding `gen_ai.agent.name` to metrics is optional
- **Incremental adoption**: Frameworks can add workflow/task spans independently

## Open Questions

1. **Workflow State**: Should we capture intermediate workflow state as events?
2. **Task Dependencies**: How to represent task dependencies in parallel workflows?
3. **Dynamic Workflows**: How to handle workflows that create agents/tasks dynamically?
4. **Tool Invocations**: Should tool calls have their own span type or remain as events within tasks?
5. **Agent Communication**: How to represent inter-agent messages in collaborative workflows?

## References

- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [CrewAI Documentation](https://docs.crewai.com/)
- [AutoGen Documentation](https://microsoft.github.io/autogen/)

## Implementation Status

This proposal has been implemented and validated in:
- **opentelemetry-python-contrib**: `opentelemetry-util-genai-dev` package
- **Frameworks tested**: LangGraph (multi-agent RAG, single-agent MCP)
- **Production usage**: Running in Kubernetes with Splunk Observability Cloud

### Example Applications
1. **Multi-Agent RAG System**: Research agent + Memory agent + Synthesizer agent
2. **Single-Agent Weather Assistant**: ReAct agent with MCP tool integration

Both applications successfully emit agent spans, metrics, and maintain proper trace hierarchy.
