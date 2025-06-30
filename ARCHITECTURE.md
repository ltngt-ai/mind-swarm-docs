# MindSwarm Architecture Documentation

This document provides a comprehensive overview of the MindSwarm agent-first AI system architecture, based on analysis of the `mind-swarm-core` and `mind-swarm-runtime` implementations.

## Table of Contents

1. [System Overview](#system-overview)
2. [Core Principles](#core-principles)
3. [Architecture Components](#architecture-components)
4. [Runtime System](#runtime-system)
5. [Agent Lifecycle](#agent-lifecycle)
6. [Communication System](#communication-system)
7. [Tool System](#tool-system)
8. [Knowledge System](#knowledge-system)
9. [Hot Reload System](#hot-reload-system)
10. [Context Management](#context-management)
11. [Security & Authentication](#security--authentication)
12. [Data Flow](#data-flow)
13. [Deployment Architecture](#deployment-architecture)

## System Overview

MindSwarm is an agent-first AI execution engine that enables dynamic, mail-based communication between AI agents and users. The system is built around the concept of autonomous agents that communicate through a structured email-like protocol, providing a more natural and asynchronous interaction model compared to traditional request-response APIs.

### Key Features

- **Agent-First Architecture**: Agents are first-class entities with their own mailboxes, contexts, and lifecycles
- **Mail-Based Communication**: All interactions use RFC2822-style email messages for consistency and traceability
- **Hot Reload Capability**: Runtime components can be updated without system restart
- **Dynamic Tool Loading**: Tools are loaded dynamically from YAML+Python definitions
- **Context Management**: Sophisticated context handling with automatic cleanup policies
- **Project-Scoped Agents**: Agents operate within project boundaries with proper isolation
- **WebSocket Real-Time Interface**: Real-time communication through WebSocket connections
- **Comprehensive Tool System**: 67+ tools across 15+ categories providing agent capabilities

## Core Principles

### 1. Agent-First Design
- Agents are autonomous entities, not request handlers
- All agents start in IDLE state and are activated by mail
- Agents decide when to continue processing or return to IDLE
- No direct function calls - all communication through mailbox

### 2. Mail-Based Communication
- All interactions use structured email messages
- Email addresses provide unique agent identification
- Messages support RFC2822 headers and metadata
- Asynchronous by design, supporting complex workflows

### 3. Runtime Flexibility
- Components can be hot-reloaded without restart
- Agent types, tools, and templates defined in external files
- Schema validation ensures compatibility
- Version management for gradual upgrades

### 4. Project Isolation
- Agents belong to specific projects
- Project-scoped resources and permissions
- Cross-project communication through explicit routing

## Architecture Components

### Core Infrastructure (`mind-swarm-core`)

The core system provides the foundational infrastructure:

```
mind-swarm-core/src/mind-swarm/
├── __init__.py              # Package initialization
├── py.typed                 # Type information marker
├── core/                    # Core infrastructure
│   ├── exceptions.py        # Exception hierarchy
│   ├── logging.py           # Structured logging & narrative system
│   └── communication/       # RFC2822 mail system
├── runtime/                 # Runtime loading and discovery
│   ├── discovery.py         # Component auto-discovery
│   ├── loader.py            # Component loading
│   └── registry.py          # Component registry
├── server/                  # FastAPI server with WebSocket support
│   ├── main.py              # Server implementation
│   └── websocket.py         # WebSocket handlers
├── services/                # Core services (agents, projects, etc.)
│   ├── agent_manager.py     # Agent lifecycle management
│   └── service_registry.py  # Service discovery
├── schemas/                 # Pydantic data models and validation
├── auth/                    # Authentication and authorization
├── context/                 # Agent context management
├── integrations/            # External service integrations
├── storage/                 # Data persistence layer
├── tools/                   # Tool infrastructure (base classes)
│   ├── tool_registry.py     # Tool registration system
│   └── CODEMAP.md           # Tool infrastructure docs
└── utils/                   # Core utilities and helpers
```

### Runtime Components (`mind-swarm-runtime`)

The runtime system provides dynamically loadable components:

```
mind-swarm-runtime/
├── agent_types/             # Agent type definitions (10 types)
│   ├── schema.yaml          # Agent type validation schema
│   ├── ui.yaml              # UI agent configuration
│   ├── reference_librarian.yaml
│   ├── archivist.yaml
│   ├── consensus_adjudicator.yaml
│   └── [6 other agent types]
├── tools/                   # Tool implementations (67+ tools)
│   ├── schema.yaml          # Tool validation schema
│   ├── README.md            # Tool system documentation
│   ├── communication/       # Mail and messaging (2 tools)
│   ├── agent_creation/      # Agent management (5 tools)
│   ├── git/                 # Git operations (12 tools)
│   ├── github/              # GitHub integration (8 tools)
│   ├── knowledge/           # Knowledge management (4 tools)
│   ├── project_management/  # Project operations (3 tools)
│   ├── readonly_filesystem/ # File reading (4 tools)
│   ├── write_filesystem/    # File writing (8 tools)
│   ├── analysis/            # Code analysis (5 tools)
│   ├── ai/                  # AI model capabilities (2 tools)
│   ├── scheduling/          # Task scheduling (3 tools)
│   ├── consensus/           # Consensus voting (6 tools)
│   ├── context_management/  # Context handling (3 tools)
│   ├── self_management/     # Agent lifecycle (2 tools)
│   └── [5 other categories]
├── templates/               # Jinja2 templates for agent prompts
│   ├── base/                # Base templates
│   ├── dynamic/             # Dynamic templates
│   └── static/              # Static templates
├── knowledge/               # Knowledge packs system
│   ├── schema.yaml          # Knowledge validation schema
│   ├── core/                # Trusted knowledge packs
│   ├── community/           # Community contributions
│   ├── examples/            # Example packs
│   └── project/             # Project-specific knowledge
└── tests/                   # Runtime component tests
```

## Runtime System

### Component Discovery

The runtime system automatically discovers components in the runtime repository:

1. **Agent Types** (`agent_types/*.yaml`): Define agent behaviors and capabilities
2. **Tools** (`tools/**/*.yaml` + `tools/**/*.py`): YAML definitions with Python implementations
3. **Templates** (`templates/**/*.j2`): Jinja2 templates for agent prompts
4. **Knowledge Packs** (`knowledge/**/*.yaml`): Structured knowledge for agents

### Hot Reload Architecture

```python
# Hot reload watches file system changes
RuntimeWatcher -> ComponentReloadManager -> RuntimeLoader
    |                       |                    |
    v                       v                    v
FileSystem              Reload Logic        Component Cache
Changes                 (by type)           (in memory)
```

Key components:

- **RuntimeWatcher**: Monitors filesystem for changes with debouncing
- **ComponentReloadManager**: Handles component-specific reload strategies
- **RuntimeLoader**: Loads and validates components from disk
- **HotReloader**: Orchestrates the entire hot reload process

### Component Validation

All runtime components go through strict validation:

1. **Schema Validation**: Components must conform to JSON schemas
2. **Compatibility Checking**: Version compatibility verification  
3. **Implementation Validation**: Python modules must have required functions
4. **Dependency Resolution**: Ensures all dependencies are available

## Agent Lifecycle

### Agent States

Agents progress through well-defined states:

```
IDLE ←→ ACTIVE ←→ PAUSED
  ↓      ↓         ↓
STOPPED ← STOPPED ← RESUMING
```

- **IDLE**: Waiting for mail, minimal resource usage
- **ACTIVE**: Processing mail and executing tasks
- **PAUSED**: Temporarily suspended for maintenance
- **RESUMING**: Transitioning back to IDLE after pause
- **STOPPED**: Permanently terminated

### Agent Creation Process

1. **Type Resolution**: Determine agent type from configuration
2. **Template Loading**: Load and render agent prompt templates
3. **Context Creation**: Initialize agent context with metadata
4. **AI Loop Setup**: Configure AI model and processing loop
5. **Mailbox Registration**: Register agent's email address
6. **Background Processor**: Start monitoring task

### Agent Processing Loop

```python
while agent.state != STOPPED:
    if agent.state == IDLE:
        # Wait for mail to arrive
        await sleep(1.0)
    elif agent.state == ACTIVE:
        # Process with wake prompt: "You have mail"
        result = await agent.process_message("You have mail")
        
        # Agent decides next action
        if result.finish_reason == "stop":
            agent.state = IDLE
        elif result.agent_state == "CONTINUE":
            # Continue with next iteration
            continue
```

## Communication System

### Mailbox Architecture

The mailbox system provides the core communication infrastructure:

```python
# Email address format
agent_id@project_id.local.mind-swarm.ltngt.ai

# Example addresses
ui-agent@project-123.local.mind-swarm.ltngt.ai
task-manager-5@project-123.local.mind-swarm.ltngt.ai
user@users.local.mind-swarm.ltngt.ai
```

### Mail Message Structure

Messages follow RFC2822 format with MindSwarm extensions:

```python
Mail(
    from_address="sender@project.local.mind-swarm.ltngt.ai",
    to_address="recipient@project.local.mind-swarm.ltngt.ai", 
    subject="Task Assignment",
    body="Please analyze the codebase for potential improvements.",
    priority="normal",
    headers={
        "Message-ID": "<unique-id@mind-swarm.ltngt.ai>",
        "Date": "Mon, 30 Jun 2025 14:30:00 +0000",
        "X-MindSwarm-Project": "project-123",
        "X-MindSwarm-Agent-Type": "task_manager"
    }
)
```

### Message Routing

1. **Address Parsing**: Extract agent_id and project_id from email
2. **Agent Resolution**: Locate target agent session
3. **Delivery**: Place message in agent's mailbox
4. **Wake Trigger**: Activate IDLE agents when mail arrives
5. **Processing**: Agent processes mail and may respond

## Tool System

**The Tool System is the "hands" of agents** - providing all the capabilities agents need to interact with their environment, execute tasks, and collaborate with other agents. Tools are the primary interface between AI agents and the external world.

### Architecture Overview

MindSwarm's tool system follows a revolutionary dual-file architecture that separates metadata from implementation:

```
tools/communication/
├── send_mail.yaml    # Tool metadata, schema, AI instructions
└── send_mail.py      # Implementation with execute_async function
```

This separation enables:
- **AI-First Design**: YAML files contain rich AI prompts and examples
- **Hot Reload**: Tools can be updated without system restart
- **Schema Validation**: Strict validation ensures tool quality
- **Dynamic Discovery**: Tools are automatically discovered and loaded

### Tool Categories (67+ Tools Across 15+ Categories)

#### 🗣️ **Communication Tools** (2 tools)
The foundation of agent interaction:
- **`send_mail`**: Send RFC2822 compliant email messages between agents
- **`check_mail`**: Check for new mail in agent's mailbox

```yaml
# Example: send_mail.yaml
schema_version: "1.0.0"
description: Send an RFC 2822 compliant email message to another agent
ai_prompt: |
  Use this tool to send messages to other agents. Always include clear 
  subject lines and well-structured body content. Use priority levels 
  appropriately (low, normal, high, urgent).
parameters_schema:
  type: object
  properties:
    to: {type: string, description: "Recipient email address"}
    subject: {type: string, description: "Subject line"}
    body: {type: string, description: "Message content"}
    priority: {type: string, enum: [low, normal, high, urgent], default: normal}
```

#### 🤖 **Agent Management Tools** (5 tools)
Control agent lifecycle and coordination:
- **`create_agent`**: Spawn new agents with specified capabilities
- **`list_agents`**: View all agents in project with status
- **`pause_agent`**: Temporarily suspend agent processing
- **`resume_agent`**: Reactivate paused agents
- **`terminate_agent`**: Permanently stop agents

#### 📁 **File System Tools** (12 tools)
Essential file operations split into read-only and write categories:

**Read-Only Operations** (4 tools):
- **`get_file_content`**: Read file contents with encoding detection
- **`list_directory`**: List directory contents with metadata
- **`search_files`**: Search for files by name patterns
- **`find_pattern`**: Search for text patterns within files

**Write Operations** (8 tools):
- **`write_file`**: Create or update files with content
- **`create_directory`**: Create directory structures
- **`delete_file`**: Remove files safely
- **`move_file`**: Move/rename files and directories

#### 🔧 **Git Tools** (12 tools)
Complete Git workflow support:
- **`git_init`**: Initialize new Git repositories
- **`git_status`**: Check repository status
- **`git_add`**: Stage changes for commit
- **`git_commit`**: Create commits with messages
- **`git_push`**: Push changes to remote repositories
- **`git_pull`**: Pull changes from remote repositories
- **`git_branch`**: Create and manage branches
- **`git_checkout`**: Switch branches or restore files
- **`git_merge`**: Merge branches
- **`git_log`**: View commit history
- **`git_diff`**: Show changes between commits
- **`git_stash`**: Temporarily store changes

#### 🌐 **GitHub Integration Tools** (8 tools)
Advanced GitHub API operations:
- **`create_pull_request`**: Create PRs with automation
- **`get_pull_request`**: Retrieve PR information
- **`create_issue`**: Create GitHub issues
- **`get_repository_info`**: Fetch repository metadata
- **`list_branches`**: List repository branches
- **`create_branch`**: Create new branches via API

#### 🧠 **Knowledge Management Tools** (4 tools)
Navigate and manage knowledge packs:
- **`list_knowledge_packs`**: Discover available knowledge
- **`explore_knowledge_pack`**: Navigate hyperlinked content
- **`search_knowledge`**: Find relevant knowledge sections
- **`validate_knowledge`**: Ensure knowledge pack integrity

#### 🏗️ **Project Management Tools** (3 tools)
Create and manage projects:
- **`create_project`**: Initialize new MindSwarm projects
- **`import_project`**: Import existing codebases
- **`get_project_structure`**: Analyze project architecture

#### 🔍 **Analysis Tools** (5 tools)
Code analysis and understanding:
- **`analyze_languages`**: Detect programming languages
- **`analyze_dependencies`**: Map project dependencies
- **`find_similar_code`**: Identify code patterns
- **`get_project_structure`**: Generate project overviews

#### 🤖 **AI Model Tools** (2 tools)
AI capabilities management:
- **`get_model_capabilities`**: Query model features
- **`list_available_models`**: Show available AI models

#### ⏰ **Scheduling Tools** (3 tools)
Time-based task management:
- **`schedule_wakeup`**: Schedule future agent activations
- **`cancel_scheduled_wakeup`**: Cancel scheduled tasks
- **`list_scheduled_wakeups`**: View pending scheduled tasks

#### 🗳️ **Consensus Tools** (6 tools)
Multi-agent decision making:
- **`create_consensus_vote`**: Start voting processes
- **`cast_vote`**: Submit votes on proposals
- **`get_vote_status`**: Check voting progress
- **`get_vote_results`**: Retrieve final results

#### 📚 **Archivist Tools** (4 tools)
Knowledge curation and promotion:
- **`promote_knowledge_pack`**: Elevate knowledge status
- **`analyze_knowledge_usage`**: Track knowledge utilization
- **`suggest_knowledge_improvements`**: Recommend enhancements

### Tool Implementation Architecture

#### YAML Metadata Structure

Each tool requires comprehensive YAML metadata:

```yaml
# yaml-language-server: $schema=../schema.yaml
schema_version: "1.0.0"
description: "Brief description of what the tool does"

# Parameter validation schema
parameters_schema:
  type: object
  properties:
    param_name:
      type: string
      description: "Parameter description for both AI and validation"
      minLength: 1
  required: ["param_name"]

# AI-specific instructions (CRITICAL for agent understanding)
ai_prompt: |
  Detailed instructions for AI agents on how to use this tool.
  
  Include:
  - When to use this tool
  - Parameter guidelines
  - Expected outputs
  - Error handling
  - Usage tips

# Usage examples (help AI learn patterns)
examples:
  - description: "Basic usage"
    code: "tool_name(param='value')"
    expected_output: "Success message"
  - description: "With options"
    code: "tool_name(param='value', option=true)"

# Categorization and discovery
tags:
  - filesystem
  - read-only

# Additional guidance
tips:
  - "Use relative paths when possible"
  - "Always check file existence first"
```

#### Python Implementation Pattern

```python
# tools/category/tool_name.py
import asyncio
from pathlib import Path
from typing import Dict, Any

async def execute_async(context: Dict[str, Any], **kwargs) -> Dict[str, Any]:
    """
    Main tool implementation.
    
    Args:
        context: Execution context (agent_id, project_path, etc.)
        **kwargs: Tool parameters from YAML schema
    
    Returns:
        Dict with status, message, and optional data
    """
    try:
        # Extract and validate parameters
        param = kwargs.get('param_name')
        if not param:
            return {
                "status": "error",
                "message": "Parameter 'param_name' is required"
            }
        
        # Tool logic here
        result = perform_operation(param)
        
        return {
            "status": "success",
            "message": f"Operation completed successfully",
            "data": result
        }
        
    except Exception as e:
        return {
            "status": "error",
            "message": f"Tool execution failed: {str(e)}"
        }
```

### Tool Discovery and Loading

#### Automatic Discovery Process

1. **Scan Runtime Directories**: Recursively find all `*.yaml` files in `tools/`
2. **Schema Validation**: Validate each YAML against `tools/schema.yaml`
3. **Implementation Check**: Verify corresponding `.py` file exists
4. **Function Validation**: Ensure `execute_async` function is present
5. **Registration**: Add to tool registry with metadata
6. **Hot Reload**: Monitor files for changes

#### Tool Registry System

```python
# Core tool registry (mind-swarm-core)
from mind-swarm.tools import get_tool_registry

registry = get_tool_registry()

# Get available tools
tools = registry.get_all_tools()
communication_tools = registry.get_tools_by_category("communication")

# Get AI instructions for tools
instructions = registry.get_ai_prompt_instructions(["send_mail", "check_mail"])

# Execute tool
result = await registry.execute_tool("send_mail", context, **params)
```

### Tool Context System

Every tool execution receives rich context:

```python
context = {
    # Agent identity
    "agent_id": "ui-agent-123",
    "agent_name": "UI Agent",
    "agent_type": "ui",
    
    # Project information
    "project_id": "project-456",
    "project_path": "/path/to/project",
    "project_name": "My Project",
    
    # User context
    "user_id": "user-789",
    "user_email": "user@example.com",
    
    # Execution metadata
    "timestamp": "2025-06-30T14:30:00Z",
    "correlation_id": "exec-abc123"
}
```

### Tool Validation and Quality

#### Schema Validation

All tools must pass strict validation:

```yaml
# tools/schema.yaml enforces:
- schema_version: Must be "1.0.0"
- description: Non-empty string
- parameters_schema: Valid JSON Schema
- ai_prompt: Non-empty instructions
- examples: Array with code and description
- tags: Array of strings for categorization
```

#### Implementation Validation

- **Function Signature**: Must have `execute_async(context, **kwargs)`
- **Return Format**: Must return `{"status": "success|error", "message": "..."}`
- **Error Handling**: Must catch and return errors gracefully
- **Type Safety**: Parameters validated against schema

#### Quality Standards

- **AI-First Documentation**: `ai_prompt` field is comprehensive
- **Rich Examples**: Multiple usage examples with expected outputs
- **Consistent Patterns**: All tools follow same architectural patterns
- **Performance**: Async execution for non-blocking operations

### Tool Security Model

1. **Context Validation**: Tools validate execution context
2. **Parameter Sanitization**: All inputs validated against schemas
3. **Path Restrictions**: File operations restricted to project scope
4. **Permission Checks**: Tools respect project-level permissions
5. **Audit Logging**: All tool executions logged for traceability

### For AI Agents: Tool Usage Guidelines

When using tools, agents should:

1. **Read AI Prompts**: Each tool's `ai_prompt` contains specific usage guidance
2. **Follow Examples**: Use provided examples as templates
3. **Validate Inputs**: Ensure parameters match schema requirements
4. **Handle Errors**: Check `status` field in responses
5. **Chain Operations**: Use output from one tool as input to another
6. **Respect Context**: Consider project scope and permissions

### For Developers: Extending the Tool System

To add new tools:

1. **Create YAML File**: Define metadata, schema, and AI instructions
2. **Implement Function**: Create `execute_async` in corresponding `.py` file
3. **Test Thoroughly**: Ensure tool works in isolation and with agents
4. **Document Examples**: Provide clear usage examples
5. **Hot Reload**: Tools automatically discovered and loaded

The tool system's dual-file architecture enables rapid development while maintaining quality and AI-friendliness, making it the perfect foundation for agent capabilities.

## Knowledge System

### Knowledge Pack Architecture

The knowledge system provides structured, hyperlinked knowledge that agents can navigate dynamically:

```
knowledge/
├── schema.yaml          # Knowledge validation schema
├── core/               # Trusted knowledge packs
│   ├── mind-swarm-basics.yaml
│   ├── agent-types.yaml
│   └── git-workflows.yaml
├── community/          # Community contributions
├── examples/           # Example knowledge packs
└── project/             # Project-specific knowledge
```

### Knowledge Features

- **Hyperlinked Navigation**: Wiki-style `[[link]]` syntax
- **Trust Levels**: core, verified, community, experimental, untrusted
- **Dynamic Discovery**: Agents can explore knowledge on-demand
- **Validation System**: Comprehensive validation of knowledge integrity

## Hot Reload System

### File System Monitoring

```python
# Hot reload implementation
class RuntimeWatcher:
    async def watch_changes(self):
        # Monitor YAML and Python files
        for file_change in file_watcher:
            if file_change.path.suffix in ['.yaml', '.py']:
                await self.reload_component(file_change.path)
```

### Reload Strategies

- **Tools**: Reload both YAML metadata and Python implementation
- **Agent Types**: Update agent configuration and restart if needed
- **Templates**: Reload template cache immediately
- **Knowledge**: Validate and update knowledge graph

## Context Management

### Agent Context Scope

Each agent maintains isolated context containing:

- **Identity**: Agent ID, type, and metadata
- **Project Scope**: Current project and permissions
- **Communication**: Mailbox state and conversation history
- **Working Memory**: Short-term context for current tasks
- **Tool State**: Available tools and their configurations

### Context Cleanup Policies

- **Automatic Cleanup**: Old context cleaned based on configurable policies
- **Memory Management**: Context size limits prevent memory leaks
- **State Persistence**: Critical context persisted across agent restarts

## Security & Authentication

### Agent Security Model

- **Project Isolation**: Agents cannot access other projects' data
- **Tool Permissions**: Tools validated against agent permissions
- **Communication Security**: Mail routing validates sender permissions
- **Resource Limits**: CPU, memory, and storage limits enforced

### Authentication Flow

1. **API Key Validation**: Initial request authentication
2. **Project Authorization**: Verify project access permissions
3. **Agent Registration**: Secure agent identity establishment
4. **Tool Access Control**: Per-tool permission validation

## Data Flow

### Request Processing Flow

```
User/Client → WebSocket → Server → AgentManager → Agent
     ↓             ↓          ↓         ↓          ↓
  Auth Check → Route → Mailbox → Process → Tools
     ↓             ↓          ↓         ↓          ↓
  Response ← Format ← Mail ← Result ← Execute
```

### Component Interaction

- **Server**: Manages WebSocket connections and routing
- **AgentManager**: Handles agent lifecycle and message delivery
- **ToolRegistry**: Provides tool discovery and execution
- **RuntimeLoader**: Manages hot reload of components

## Deployment Architecture

### Container Structure

```yaml
# Docker deployment
services:
  mind-swarm-core:
    image: mind-swarm/core:latest
    volumes:
      - ./mind-swarm-runtime:/runtime
    environment:
      - MIND_SWARM_RUNTIME_PATH=/runtime
  
  mind-swarm-ui:
    image: mind-swarm/ui:latest
    depends_on:
      - mind-swarm-core
```

### Scaling Considerations

- **Horizontal Scaling**: Multiple core instances with shared runtime
- **Load Balancing**: WebSocket session affinity required
- **State Management**: Agent state persisted for failover
- **Runtime Sync**: Shared runtime directory for consistency

This architecture enables MindSwarm to provide a powerful, flexible platform for agent-first AI applications with comprehensive tool capabilities, making agents truly autonomous and capable.
