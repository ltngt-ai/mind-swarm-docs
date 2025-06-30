# MindSwarm Component Reference

This document provides detailed reference information for all components in the MindSwarm system.

## Table of Contents

1. [Agent Types](#agent-types)
2. [Tools System](#tools-system)
3. [Templates](#templates)
4. [Services](#services)
5. [Runtime Components](#runtime-components)
6. [Core Infrastructure](#core-infrastructure)

## Agent Types

Agent types define the behavior and capabilities of different kinds of agents in the system.

### Agent Type Schema

All agent types must conform to the following schema:

```yaml
# Required fields
agent_type: string          # Unique identifier (e.g., "ui", "project_manager")
description: string         # Human-readable description

# Optional configuration
color: string              # Hex color for UI (#888888)
icon: string               # Emoji or icon (🤖)
static_template: string    # Path to Jinja2 template
tool_sets: array           # Tool directories this agent can use
ai_config:
  model: string            # AI model (e.g., "google/gemini-2.0-flash-001")
  temperature: number      # Response variability (0-2)
  max_tokens: integer      # Maximum response tokens
context_management:
  strategy: string         # "entries", "timed", or "adaptive"
  parameters: object       # Strategy-specific parameters
```

### Built-in Agent Types

#### UI Agent (`ui.yaml`)
- **Purpose**: Interface between users and the system
- **Capabilities**: Route user requests, manage conversations
- **Tool Sets**: `communication`, `ui`, `agent_creation`
- **Context Strategy**: Entry-based with 5 max entries

#### Project Creator (`project_creator.yaml`)
- **Purpose**: Create and initialize new projects
- **Capabilities**: Project setup, initial agent creation
- **Tool Sets**: `project_creation`, `project_management`
- **Context Strategy**: Time-based with 30-minute limit

#### Consensus Adjudicator (`consensus_adjudicator.yaml`)
- **Purpose**: Resolve conflicts and make decisions
- **Capabilities**: Multi-agent consensus, conflict resolution
- **Tool Sets**: `consensus`, `communication`
- **Context Strategy**: Adaptive based on conversation complexity

#### Reference Librarian (`reference_librarian.yaml`)
- **Purpose**: Knowledge management and retrieval
- **Capabilities**: Information lookup, documentation
- **Tool Sets**: `knowledge`, `librarian`
- **Context Strategy**: Entry-based with 10 max entries

#### Archivist (`archivist.yaml`)
- **Purpose**: Data archiving and historical analysis
- **Capabilities**: Archive conversations, analyze patterns
- **Tool Sets**: `archivist`, `analysis`
- **Context Strategy**: Time-based with 1-day retention

### Agent Type Templates

Agent types use Jinja2 templates to generate their system prompts:

**Template Location**: `templates/static/{agent_type}.j2`

**Available Variables**:
```jinja2
{{ agent_id }}              # Unique agent identifier
{{ agent_type }}            # Agent type name
{{ project_id }}            # Project scope
{{ agent_email }}           # Agent's email address
{{ tools }}                 # Available tools array
{{ tool_instructions }}     # Tool usage instructions
{{ knowledge_packs }}       # Knowledge pack content
{{ context_vars }}          # Additional context variables
```

**Example Template Structure**:
```jinja2
# Agent Identity
You are {{ agent_id }}, a {{ agent_type }} agent.
Your email: {{ agent_email }}

# Capabilities
{% if tools %}
## Available Tools
{% for tool in tools %}
- **{{ tool.name }}**: {{ tool.description }}
{% endfor %}
{% endif %}

# Instructions
{{ tool_instructions }}

# Knowledge
{% for pack in knowledge_packs %}
{{ pack }}
{% endfor %}
```

## Tools System

Tools provide agents with specific capabilities through a YAML + Python architecture.

### Tool Schema

Every tool must have both a YAML definition and Python implementation:

**YAML Schema** (`schema_version: "1.0.0"`):
```yaml
schema_version: "1.0.0"
description: string                    # What the tool does
parameters_schema:                     # JSON Schema for parameters
  type: object
  properties:
    param_name:
      type: string|number|boolean|array
      description: string
      default: any                     # Optional default value
  required: [string]                   # Required parameter names
ai_prompt: string                      # Instructions for AI usage
examples: array                        # Usage examples (optional)
  - description: string
    code: string
tags: [string]                        # Categorization tags (optional)
tips: [string]                        # Usage tips (optional)
```

**Python Interface**:
```python
async def execute_async(
    # Parameters match YAML schema
    param1: str,
    param2: int = 0,
    # Always included
    context: ToolContext
) -> ToolResult:
    """Tool implementation."""
    # Implementation here
    pass
```

### Tool Categories

#### Communication Tools (`tools/communication/`)

**send_mail.py/yaml**
- Send emails to agents or users
- Parameters: `to_address`, `subject`, `body`
- Returns: Delivery confirmation

**broadcast_message.py/yaml**
- Send message to multiple recipients
- Parameters: `recipients`, `subject`, `body`
- Returns: Delivery status for each recipient

**get_mailbox_status.py/yaml**
- Check mailbox status and unread count
- Parameters: `email_address` (optional)
- Returns: Mailbox statistics

#### Project Management Tools (`tools/project_management/`)

**create_project.py/yaml**
- Create new project with initial structure
- Parameters: `project_name`, `description`, `initial_agents`
- Returns: Project ID and setup status

**get_project_info.py/yaml**
- Retrieve project information and status
- Parameters: `project_id`
- Returns: Project metadata and agent list

**update_project_status.py/yaml**
- Update project status and metadata
- Parameters: `project_id`, `status`, `metadata`
- Returns: Update confirmation

#### Task Management Tools (`tools/task_management/`)

**create_task.py/yaml**
- Create new task in project
- Parameters: `title`, `description`, `assignee`, `priority`
- Returns: Task ID and creation status

**assign_task.py/yaml**
- Assign task to agent or user
- Parameters: `task_id`, `assignee`, `due_date`
- Returns: Assignment confirmation

**update_task_status.py/yaml**
- Update task progress and status
- Parameters: `task_id`, `status`, `notes`
- Returns: Update confirmation

#### Knowledge Tools (`tools/knowledge/`)

**search_knowledge.py/yaml**
- Search knowledge base for information
- Parameters: `query`, `knowledge_pack_ids`, `max_results`
- Returns: Search results with relevance scores

**create_knowledge_entry.py/yaml**
- Add new knowledge entry
- Parameters: `title`, `content`, `tags`, `knowledge_pack_id`
- Returns: Entry ID and creation status

#### Git Tools (`tools/git/`)

**git_status.py/yaml**
- Get repository status
- Parameters: `repo_path`
- Returns: Git status information

**git_commit.py/yaml**
- Commit changes to repository
- Parameters: `repo_path`, `message`, `files`
- Returns: Commit hash and status

**git_branch.py/yaml**
- Manage git branches
- Parameters: `repo_path`, `action`, `branch_name`
- Returns: Branch operation result

#### UI Tools (`tools/ui/`)

**send_ui_message.py/yaml**
- Send message to user interface
- Parameters: `message`, `message_type`, `metadata`
- Returns: UI delivery confirmation

**request_user_input.py/yaml**
- Request input from user
- Parameters: `prompt`, `input_type`, `validation`
- Returns: User response

### Tool Context

The `ToolContext` object provides tools with access to runtime information:

```python
@dataclass
class ToolContext:
    agent_id: str              # Executing agent ID
    project_id: str            # Project scope
    agent_email: str           # Agent's email address
    runtime_paths: List[Path]  # Runtime directories
    metadata: Dict[str, Any]   # Additional context
    
    # Services (injected)
    mailbox: MailboxService
    project_service: ProjectService
    tool_registry: ToolRegistry
```

### Tool Results

Tools return structured results:

```python
@dataclass
class ToolResult:
    success: bool
    message: str
    data: Dict[str, Any] = field(default_factory=dict)
    error: Optional[str] = None
    metadata: Dict[str, Any] = field(default_factory=dict)
```

## Templates

Templates define agent prompts using Jinja2 templating.

### Template Structure

```
templates/
├── static/                 # Static agent templates
│   ├── ui.j2
│   ├── project_creator.j2
│   └── consensus_adjudicator.j2
├── shared/                 # Shared template components
│   ├── agent_identity.j2
│   ├── tool_usage.j2
│   └── communication_agent.j2
└── dynamic/                # Dynamic templates (future)
    └── adaptive_prompt.j2
```

### Template Variables

**Core Variables** (always available):
- `agent_id`: Unique agent identifier
- `agent_type`: Agent type name
- `project_id`: Project scope
- `agent_email`: Agent's email address
- `model_name`: AI model being used

**Tool Variables**:
- `tools`: Array of available tool objects
- `tool_instructions`: Generated tool usage instructions
- `tool_sets`: List of tool directories available

**Knowledge Variables**:
- `knowledge_packs`: Array of knowledge pack content
- `project_rules`: Project-specific rules
- `task_rules`: Task-specific rules

**Context Variables**:
- `context_vars`: Additional context string
- `user_email`: User email (for UI agents)
- `shared_components`: List of shared template includes

### Shared Components

Shared components provide reusable template blocks:

**agent_identity.j2**:
```jinja2
# Agent Identity
You are {{ agent_id }}, a {{ agent_type }} agent operating in project {{ project_id }}.
Your email address is {{ agent_email }}.
```

**tool_usage.j2**:
```jinja2
# Tool Usage
{% if tools %}
You have access to the following tools:
{% for tool in tools %}
## {{ tool.name }}
{{ tool.description }}
{% endfor %}
{% endif %}
```

**communication_agent.j2**:
```jinja2
# Communication Protocol
- Use send_mail tool to communicate with other agents
- Always include clear subject lines
- Respond to emails in your mailbox
- Use proper email etiquette
```

## Services

Services provide core functionality and are registered in the service registry.

### Core Services

#### AgentManager (`services/agents/agent_manager.py`)
- **Purpose**: Manage agent lifecycle and state
- **Key Methods**:
  - `create_agent()`: Create new agent
  - `stop_agent()`: Stop and cleanup agent
  - `pause_agent()`: Pause for maintenance
  - `resume_agent()`: Resume from pause
- **State Management**: IDLE, ACTIVE, PAUSED, RESUMING, STOPPED

#### ProjectService (`services/project_service.py`)
- **Purpose**: Manage projects and resources
- **Key Methods**:
  - `create_project()`: Initialize new project
  - `get_project()`: Retrieve project information
  - `update_agent_registry()`: Sync agent state
- **Features**: Project isolation, agent registry persistence

#### UIAgent (`services/agents/ui_agent.py`)
- **Purpose**: Bridge between users and agent system
- **Key Methods**:
  - `create_ui_agent_for_session()`: Create UI agent for user
  - `handle_frontend_mail()`: Process user messages
  - `register_websocket()`: Connect WebSocket to agent
- **Features**: Real-time communication, user session management

#### ToolRegistry (`tools/tool_registry.py`)
- **Purpose**: Manage available tools and their execution
- **Key Methods**:
  - `register_tool()`: Add tool to registry
  - `get_tool()`: Retrieve tool by name
  - `execute_tool()`: Execute tool with parameters
- **Features**: Dynamic tool loading, validation, hot reload

### Service Registry

The service registry provides dependency injection:

```python
from mind-swarm.services.service_registry import get_service_registry

registry = get_service_registry()
registry.register(ProjectService, project_service_instance)

# Later, retrieve service
project_service = registry.get(ProjectService)
```

### Authentication Services

#### AuthService (`auth/auth_service.py`)
- **Purpose**: User authentication and session management
- **Key Methods**:
  - `authenticate()`: Validate user credentials
  - `create_session()`: Create user session
  - `validate_token()`: Check token validity
- **Features**: Token-based auth, session persistence

#### WebSocketAuthMiddleware (`server/auth_middleware.py`)
- **Purpose**: WebSocket authentication
- **Key Methods**:
  - `authenticate_websocket()`: Validate WebSocket connection
- **Features**: Bearer token validation, session restoration

## Runtime Components

### Runtime Loader (`runtime/loader.py`)

The runtime loader discovers and loads components from the runtime repository:

**Key Components**:
- `ComponentDiscovery`: Find components in filesystem
- `ComponentValidator`: Validate component schemas
- `RuntimeLoader`: Load and cache components

**Loading Process**:
1. **Discovery**: Scan runtime directories for components
2. **Validation**: Check schemas and compatibility
3. **Loading**: Load Python modules and YAML data
4. **Caching**: Store in memory for fast access
5. **Hot Reload**: Monitor for changes

### Hot Reload System (`runtime/hot_reload.py`)

**Components**:
- `RuntimeWatcher`: Monitor filesystem changes
- `ComponentReloadManager`: Handle reload strategies
- `HotReloader`: Orchestrate reload process

**Change Detection**:
- File system events (create, modify, delete)
- Debouncing to prevent excessive reloads
- Component type identification
- Validation before reload

**Reload Strategies**:
- **Tools**: Reload Python module, re-register in tool registry
- **Agent Types**: Clear cache, reload template registry
- **Templates**: Clear Jinja2 cache, re-render prompts

### Component Validator (`runtime/validator.py`)

**Validation Types**:
- **Schema Validation**: JSON Schema conformance
- **Version Compatibility**: Check schema versions
- **Implementation Validation**: Verify Python interfaces
- **Dependency Validation**: Ensure required components exist

**Validation Results**:
```python
@dataclass
class ValidationResult:
    valid: bool
    errors: List[str]
    warnings: List[str]
    component_type: str
    component_id: str
```

## Core Infrastructure

### Communication System

#### Mailbox (`core/communication/mailbox.py`)

The mailbox provides the core communication infrastructure:

**Key Features**:
- RFC2822-compliant message format
- Agent registration and discovery
- Message routing and delivery
- Unread message tracking
- Wake-on-mail for IDLE agents

**Mail Object**:
```python
@dataclass
class Mail:
    from_address: str
    to_address: str
    subject: str
    body: str
    headers: Dict[str, str] = field(default_factory=dict)
    priority: MessagePriority = MessagePriority.NORMAL
    metadata: Dict[str, Any] = field(default_factory=dict)
```

#### Message Flow Tracer (`core/message_flow_tracer.py`)

Tracks message flow for debugging and analytics:
- Message routing paths
- Delivery confirmations
- Processing times
- Error tracking

### Context Management

#### AgentContext (`context/agent_context.py`)

**Features**:
- Conversation history management
- Metadata storage
- Context size tracking
- Automatic cleanup policies

**Context Strategies**:
- **Entry-based**: Limit by message count
- **Time-based**: Limit by age
- **Token-based**: Limit by token usage
- **Adaptive**: Dynamic based on content

#### Context Size Manager (`context/context_size_manager.py`)

**Capabilities**:
- Token counting for different models
- Context utilization monitoring
- Automatic context compression
- Model-specific optimizations

### AI Integration

#### AI Loop (`services/ai_execution/ai_loop.py`)

**Responsibilities**:
- Manage AI model interactions
- Handle token usage tracking
- Process continuation requests
- Manage model-specific quirks

**AI Loop Manager** (`services/ai_execution/ai_loop_manager.py`):
- Create and manage AI loops per agent
- Configure model parameters
- Handle model switching
- Monitor resource usage

### Logging and Observability

#### Core Logging (`core/logging.py`)
- Structured logging with context
- Log levels and filtering
- Performance metrics

#### Narrative Logging (`core/logging.py`)
- Human-readable agent activity logs
- Agent state changes
- Decision tracking
- User-friendly error messages

### Model Registry (`core/model_registry.py`)

**Features**:
- Model capability tracking
- Context length limits
- Provider integration
- Performance characteristics

**Model Information**:
```python
@dataclass
class ModelInfo:
    model_id: str
    provider: str
    context_length: int
    supports_tools: bool
    supports_streaming: bool
    cost_per_token: float
    capabilities: List[str]
```

This component reference provides the detailed information needed to understand, extend, and maintain the MindSwarm system.
