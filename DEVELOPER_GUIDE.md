# MindSwarm Developer Guide

This guide provides practical information for developers working with the MindSwarm system.

## Table of Contents

1. [Development Setup](#development-setup)
2. [Creating Agent Types](#creating-agent-types)
3. [Building Tools](#building-tools)
4. [Working with Templates](#working-with-templates)
5. [Agent Communication Patterns](#agent-communication-patterns)
6. [Testing Strategies](#testing-strategies)
7. [Debugging and Troubleshooting](#debugging-and-troubleshooting)
8. [Hot Reload Development](#hot-reload-development)
9. [Performance Optimization](#performance-optimization)
10. [Deployment Patterns](#deployment-patterns)

## Development Setup

### Environment Prerequisites

```bash
# Python 3.11+
python3.11 -m venv .venv
source .venv/bin/activate

# Core dependencies
pip install -r mind-swarm-core/requirements.txt
pip install -r mind-swarm-runtime/requirements.txt

# Development tools
pip install pytest pytest-asyncio black ruff mypy
```

### Project Structure

```
workspace/
├── mind-swarm-core/          # Core infrastructure
├── mind-swarm-runtime/       # Runtime components
├── mind-swarm-docs/          # Documentation
└── projects/                # User projects
    └── example-project/
        ├── project.json
        └── data/
```

### Configuration

**Environment Variables**:
```bash
export MIND_SWARM_RUNTIME_PATH="./mind-swarm-runtime"
export MIND_SWARM_DATA_DIR="./data"
export LOG_LEVEL="DEBUG"
export MIND_SWARM_HOST="0.0.0.0"
export MIND_SWARM_PORT="8000"
```

**Server Configuration**:
```python
config = {
    "runtime_paths": [Path("./mind-swarm-runtime")],
    "data_dir": "./data",
    "cors_origins": ["http://localhost:3000"],
    "hot_reload_enabled": True,
    "log_level": "INFO"
}
```

### Running the Development Server

```bash
cd mind-swarm-core
python -m mind-swarm.server.main
```

The server will start with hot reload enabled, automatically picking up changes to runtime components.

## Creating Agent Types

### Basic Agent Type

Create `mind-swarm-runtime/agent_types/my_agent.yaml`:

```yaml
agent_type: my_agent
description: "A custom agent that performs specific tasks"

# Visual identification
color: "#4A90E2"
icon: "🔧"

# Template configuration
static_template: "agent_types/my_agent.j2"
shared_components:
  - agent_identity
  - tool_usage
  - communication_agent

# Tool access
tool_sets:
  - communication
  - project_management
  - my_custom_tools

# AI configuration
ai_config:
  model: "google/gemini-2.0-flash-001"
  temperature: 0.3
  max_tokens: 2000

# Context management
context_management:
  strategy: entries
  parameters:
    max_entries: 8

# Metadata
metadata:
  version: "1.0"
  author: "Your Name"
  tags: ["custom", "utility"]
```

### Agent Template

Create `mind-swarm-runtime/templates/static/my_agent.j2`:

```jinja2
{% include "shared/agent_identity.j2" %}

# Role and Responsibilities
You are a specialized agent designed to handle specific tasks within the MindSwarm system.

## Your Capabilities
- Task analysis and breakdown
- Resource coordination
- Progress tracking
- Team communication

{% include "shared/tool_usage.j2" %}

## Working Style
- Be systematic and thorough
- Ask clarifying questions when needed
- Provide regular status updates
- Collaborate effectively with other agents

{% include "shared/communication_agent.j2" %}

## Context Information
{% if project_rules %}
### Project Rules
{{ project_rules }}
{% endif %}

{% if task_rules %}
### Task Rules
{{ task_rules }}
{% endif %}

{% if knowledge_packs %}
## Knowledge Base
{% for pack in knowledge_packs %}
{{ pack }}
{% endfor %}
{% endif %}

Remember: Always stay in character as {{ agent_id }} and use your tools effectively to accomplish your goals.
```

### Testing the Agent Type

```python
# Test agent creation
from mind-swarm.services.agents.agent_manager import AgentManager

async def test_my_agent():
    config = {"runtime_paths": [Path("./mind-swarm-runtime")]}
    manager = AgentManager(config)
    
    session = await manager.create_agent(
        project_id="test-project",
        agent_type="my_agent",
        agent_id="test-agent-1"
    )
    
    assert session.agent_type == "my_agent"
    assert session.state == AgentState.IDLE
    
    # Test agent can receive mail
    from mind-swarm.core.communication.mailbox import Mail, get_mailbox
    
    mail = Mail(
        from_address="test@system.local.mind-swarm.ltngt.ai",
        to_address=session.metadata["email"],
        subject="Test Message",
        body="Hello, agent!"
    )
    
    mailbox = get_mailbox()
    result = mailbox.send_mail(mail)
    assert result.success
```

## Building Tools

### Tool Structure

Every tool requires two files:

1. **YAML Definition** (`tools/category/tool_name.yaml`)
2. **Python Implementation** (`tools/category/tool_name.py`)

### Example Tool: File Operations

**tools/filesystem/read_file.yaml**:
```yaml
schema_version: "1.0.0"
description: "Read contents of a file from the filesystem"

parameters_schema:
  type: object
  properties:
    file_path:
      type: string
      description: "Path to the file to read"
    encoding:
      type: string
      description: "Text encoding to use"
      default: "utf-8"
    max_size:
      type: integer
      description: "Maximum file size in bytes"
      default: 1048576
  required: ["file_path"]

ai_prompt: |
  Use this tool to read the contents of files. Provide the file path relative
  to the project directory. The tool will return the file contents as text.
  
  Be careful with large files - the default maximum size is 1MB.

examples:
  - description: "Read a configuration file"
    code: |
      read_file(file_path="config/settings.json")
  
  - description: "Read with specific encoding"
    code: |
      read_file(file_path="data/utf16_file.txt", encoding="utf-16")

tags: ["filesystem", "io", "utility"]

tips:
  - "Use relative paths from the project directory"
  - "Check file size before reading large files"
  - "Handle encoding errors gracefully"
```

**tools/filesystem/read_file.py**:
```python
import os
from pathlib import Path
from typing import Optional

from mind-swarm.tools.tool_context import ToolContext
from mind-swarm.tools.tool_result import ToolResult


async def execute_async(
    file_path: str,
    encoding: str = "utf-8",
    max_size: int = 1048576,
    context: ToolContext
) -> ToolResult:
    """Read contents of a file from the filesystem."""
    
    try:
        # Resolve path relative to project
        if hasattr(context, 'project_service') and context.project_service:
            project = context.project_service.get_project(context.project_id)
            if project:
                base_path = Path(project.get("path", "."))
            else:
                base_path = Path(".")
        else:
            base_path = Path(".")
        
        full_path = base_path / file_path
        
        # Security check - ensure path is within project
        try:
            full_path.resolve().relative_to(base_path.resolve())
        except ValueError:
            return ToolResult(
                success=False,
                message="Access denied: path outside project directory",
                error="Invalid file path"
            )
        
        # Check if file exists
        if not full_path.exists():
            return ToolResult(
                success=False,
                message=f"File not found: {file_path}",
                error="File does not exist"
            )
        
        # Check file size
        file_size = full_path.stat().st_size
        if file_size > max_size:
            return ToolResult(
                success=False,
                message=f"File too large: {file_size} bytes (max: {max_size})",
                error="File size exceeds limit"
            )
        
        # Read file content
        content = full_path.read_text(encoding=encoding)
        
        return ToolResult(
            success=True,
            message=f"Successfully read {len(content)} characters from {file_path}",
            data={
                "content": content,
                "file_path": str(file_path),
                "file_size": file_size,
                "encoding": encoding
            }
        )
        
    except UnicodeDecodeError as e:
        return ToolResult(
            success=False,
            message=f"Encoding error reading {file_path}",
            error=f"Failed to decode with {encoding}: {str(e)}"
        )
    
    except Exception as e:
        return ToolResult(
            success=False,
            message=f"Failed to read {file_path}",
            error=str(e)
        )
```

### Tool Testing

```python
import pytest
from mind-swarm.tools.tool_context import ToolContext
from tools.filesystem.read_file import execute_async


@pytest.mark.asyncio
async def test_read_file():
    # Create test file
    test_file = Path("test_data.txt")
    test_file.write_text("Hello, world!")
    
    try:
        # Create mock context
        context = ToolContext(
            agent_id="test-agent",
            project_id="test-project",
            agent_email="test@example.com",
            runtime_paths=[Path(".")],
            metadata={}
        )
        
        # Test successful read
        result = await execute_async("test_data.txt", context=context)
        
        assert result.success
        assert result.data["content"] == "Hello, world!"
        assert result.data["file_size"] == 13
        
        # Test file not found
        result = await execute_async("nonexistent.txt", context=context)
        assert not result.success
        assert "not found" in result.message
        
    finally:
        # Cleanup
        if test_file.exists():
            test_file.unlink()
```

### Advanced Tool Features

**Async Tool with External API**:
```python
import aiohttp
from mind-swarm.tools.tool_context import ToolContext
from mind-swarm.tools.tool_result import ToolResult


async def execute_async(
    url: str,
    method: str = "GET",
    headers: dict = None,
    timeout: int = 30,
    context: ToolContext
) -> ToolResult:
    """Make HTTP request to external API."""
    
    headers = headers or {}
    
    try:
        async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=timeout)) as session:
            async with session.request(method, url, headers=headers) as response:
                content = await response.text()
                
                return ToolResult(
                    success=response.status < 400,
                    message=f"HTTP {response.status} from {url}",
                    data={
                        "status_code": response.status,
                        "headers": dict(response.headers),
                        "content": content,
                        "url": url
                    }
                )
    
    except aiohttp.ClientTimeout:
        return ToolResult(
            success=False,
            message=f"Request to {url} timed out",
            error="Timeout"
        )
    
    except Exception as e:
        return ToolResult(
            success=False,
            message=f"Request to {url} failed",
            error=str(e)
        )
```

## Working with Templates

### Template Hierarchy

```
templates/
├── static/                    # Static agent templates
│   ├── ui.j2
│   └── my_agent.j2
├── shared/                    # Reusable components
│   ├── agent_identity.j2
│   ├── tool_usage.j2
│   └── project_context.j2
└── macros/                    # Jinja2 macros
    └── formatting.j2
```

### Creating Shared Components

**templates/shared/project_context.j2**:
```jinja2
{% if project_rules or task_rules %}
## Project Context

{% if project_rules %}
### Project Rules
{{ project_rules }}
{% endif %}

{% if task_rules %}
### Current Task Rules
{{ task_rules }}
{% endif %}

{% endif %}
```

**templates/macros/formatting.j2**:
```jinja2
{% macro format_tool_list(tools) %}
{% if tools %}
## Available Tools

{% for tool in tools %}
### {{ tool.name }}
**Description**: {{ tool.description }}

**Parameters**:
{% for param_name, param_info in tool.parameters.items() %}
- `{{ param_name }}` ({{ param_info.type }}){% if param_info.required %} **required**{% endif %}: {{ param_info.description }}
{% endfor %}

{% endfor %}
{% endif %}
{% endmacro %}
```

### Using Macros in Templates

**templates/static/enhanced_agent.j2**:
```jinja2
{% from "macros/formatting.j2" import format_tool_list %}

{% include "shared/agent_identity.j2" %}

# Enhanced Agent

You are an enhanced agent with sophisticated capabilities.

{{ format_tool_list(tools) }}

{% include "shared/project_context.j2" %}

## Instructions
- Be systematic in your approach
- Use tools effectively
- Communicate clearly with other agents
- Ask for clarification when needed
```

### Template Testing

```python
from mind-swarm.services.agents.template_engine import TemplateEngine
from pathlib import Path


def test_template_rendering():
    template_dirs = [Path("mind-swarm-runtime/templates")]
    engine = TemplateEngine(template_dirs)
    
    context = {
        "agent_id": "test-agent",
        "agent_type": "enhanced_agent",
        "project_id": "test-project",
        "agent_email": "test@example.com",
        "tools": [
            {
                "name": "send_mail",
                "description": "Send email to another agent",
                "parameters": {
                    "to_address": {"type": "string", "required": True, "description": "Recipient email"},
                    "subject": {"type": "string", "required": True, "description": "Email subject"},
                    "body": {"type": "string", "required": True, "description": "Email body"}
                }
            }
        ],
        "shared_components": ["agent_identity", "project_context"]
    }
    
    prompt = engine.render_static_prompt(
        template_name="static/enhanced_agent.j2",
        context=context
    )
    
    assert "test-agent" in prompt
    assert "send_mail" in prompt
    assert "Available Tools" in prompt
```

## Agent Communication Patterns

### Basic Mail Exchange

```python
# Agent A sends mail to Agent B
from mind-swarm.core.communication.mailbox import Mail, get_mailbox

mail = Mail(
    from_address="agent-a@project.local.mind-swarm.ltngt.ai",
    to_address="agent-b@project.local.mind-swarm.ltngt.ai",
    subject="Task Assignment",
    body="Please handle task #123: Update documentation",
    headers={
        "X-Task-ID": "123",
        "X-Priority": "HIGH"
    }
)

mailbox = get_mailbox()
result = mailbox.send_mail(mail)
```

### Request-Response Pattern

**Requestor Agent**:
```python
# Send request with correlation ID
request_mail = Mail(
    from_address=context.agent_email,
    to_address="service-agent@project.local.mind-swarm.ltngt.ai",
    subject="Data Request",
    body="Please provide user statistics for project",
    headers={
        "X-Correlation-ID": "req-12345",
        "X-Response-Expected": "true"
    }
)

mailbox.send_mail(request_mail)
```

**Service Agent**:
```python
# Process request and send response
response_mail = Mail(
    from_address=context.agent_email,
    to_address=original_mail.from_address,
    subject=f"Re: {original_mail.subject}",
    body="Statistics: 100 users, 50 active sessions",
    headers={
        "X-Correlation-ID": original_mail.headers.get("X-Correlation-ID"),
        "In-Reply-To": original_mail.headers.get("Message-ID")
    }
)

mailbox.send_mail(response_mail)
```

### Broadcast Pattern

```python
# Send to multiple agents
recipients = [
    "agent-1@project.local.mind-swarm.ltngt.ai",
    "agent-2@project.local.mind-swarm.ltngt.ai",
    "agent-3@project.local.mind-swarm.ltngt.ai"
]

for recipient in recipients:
    mail = Mail(
        from_address=context.agent_email,
        to_address=recipient,
        subject="Project Update",
        body="New requirements have been added to the project",
        headers={"X-Broadcast": "true"}
    )
    mailbox.send_mail(mail)
```

### Coordination Pattern

```python
# Coordinator sends tasks
task_assignments = [
    ("agent-1@project.local.mind-swarm.ltngt.ai", "Implement feature A"),
    ("agent-2@project.local.mind-swarm.ltngt.ai", "Write tests for feature A"),
    ("agent-3@project.local.mind-swarm.ltngt.ai", "Update documentation")
]

for agent_email, task in task_assignments:
    mail = Mail(
        from_address=context.agent_email,
        to_address=agent_email,
        subject=f"Task Assignment: {task}",
        body=f"You have been assigned: {task}",
        headers={
            "X-Task-Group": "feature-a-implementation",
            "X-Coordinator": context.agent_email
        }
    )
    mailbox.send_mail(mail)
```

## Testing Strategies

### Unit Testing Tools

```python
import pytest
from unittest.mock import MagicMock, AsyncMock
from mind-swarm.tools.tool_context import ToolContext


@pytest.fixture
def mock_context():
    context = MagicMock(spec=ToolContext)
    context.agent_id = "test-agent"
    context.project_id = "test-project"
    context.agent_email = "test@example.com"
    context.runtime_paths = [Path(".")]
    context.metadata = {}
    return context


@pytest.mark.asyncio
async def test_tool_execution(mock_context):
    from tools.my_category.my_tool import execute_async
    
    result = await execute_async(
        param1="value1",
        param2=42,
        context=mock_context
    )
    
    assert result.success
    assert "expected value" in result.data
```

### Integration Testing

```python
@pytest.mark.asyncio
async def test_agent_communication():
    # Create test environment
    config = {"runtime_paths": [Path("./mind-swarm-runtime")]}
    manager = AgentManager(config)
    
    # Create two agents
    agent_a = await manager.create_agent(
        project_id="test",
        agent_type="ui",
        agent_id="agent-a"
    )
    
    agent_b = await manager.create_agent(
        project_id="test", 
        agent_type="ui",
        agent_id="agent-b"
    )
    
    # Send mail from A to B
    mail = Mail(
        from_address=agent_a.metadata["email"],
        to_address=agent_b.metadata["email"],
        subject="Test Communication",
        body="Hello from agent A"
    )
    
    mailbox = get_mailbox()
    result = mailbox.send_mail(mail)
    
    assert result.success
    
    # Verify B received the mail
    assert mailbox.has_unread_mail(agent_b.metadata["email"])
```

### Hot Reload Testing

```python
@pytest.mark.asyncio
async def test_hot_reload():
    # Create runtime loader with hot reload
    runtime_paths = [Path("./test_runtime")]
    loader = RuntimeLoader(runtime_paths)
    hot_reloader = loader.enable_hot_reload()
    
    # Create test tool
    tool_yaml = Path("test_runtime/tools/test_tool.yaml")
    tool_py = Path("test_runtime/tools/test_tool.py")
    
    # Write initial tool
    tool_yaml.write_text("""
schema_version: "1.0.0"
description: "Test tool"
parameters_schema:
  type: object
  properties:
    message:
      type: string
  required: ["message"]
ai_prompt: "Use this tool for testing"
""")
    
    tool_py.write_text("""
async def execute_async(message: str, context):
    return {"success": True, "message": f"Hello {message}"}
""")
    
    # Wait for hot reload
    await asyncio.sleep(1.0)
    
    # Verify tool was loaded
    tools = loader.load_tools()
    assert "test_tool" in tools
    
    # Modify tool
    tool_py.write_text("""
async def execute_async(message: str, context):
    return {"success": True, "message": f"Hi {message}!"}
""")
    
    # Wait for reload
    await asyncio.sleep(1.0)
    
    # Verify tool was reloaded
    tools = loader.load_tools()
    # Test execution to verify change
    result = await tools["test_tool"]["execute_async"]("world", None)
    assert "Hi world!" in result["message"]
```

## Debugging and Troubleshooting

### Logging Configuration

```python
import logging
from mind-swarm.core.logging import setup_logging, get_logger, get_narrative_logger

# Configure logging
setup_logging(level="DEBUG")

logger = get_logger(__name__)
narrative_logger = get_narrative_logger()

# Use in code
logger.info("System event", extra={"agent_id": "agent-1", "project_id": "test"})
narrative_logger.log_agent_thinking("agent-1", "I'm processing the request")
```

### Common Issues and Solutions

**Issue: Agent not waking up**
```python
# Check mailbox registration
mailbox = get_mailbox()
registered = mailbox.list_registered_mailboxes()
print(f"Registered mailboxes: {registered}")

# Check for unread mail
email = "agent@project.local.mind-swarm.ltngt.ai"
unread_count = mailbox.get_unread_count(email)
print(f"Unread messages for {email}: {unread_count}")

# Check agent state
manager = get_agent_manager()
states = manager.get_agent_states()
print(f"Agent states: {states}")
```

**Issue: Tool not loading**
```python
# Check tool validation
from mind-swarm.runtime.loader import RuntimeLoader

loader = RuntimeLoader([Path("./mind-swarm-runtime")])
validation_results = loader.validate_runtime_data()

for component_key, result in validation_results["validation_results"].items():
    if not result["valid"]:
        print(f"Invalid component {component_key}: {result['errors']}")
```

**Issue: Template rendering errors**
```python
# Test template rendering
from mind-swarm.services.agents.template_engine import TemplateEngine

try:
    engine = TemplateEngine([Path("./mind-swarm-runtime/templates")])
    prompt = engine.render_static_prompt(
        template_name="static/my_agent.j2",
        context={"agent_id": "test", "agent_type": "test"}
    )
    print("Template rendered successfully")
except Exception as e:
    print(f"Template error: {e}")
    import traceback
    traceback.print_exc()
```

### Performance Monitoring

```python
# Monitor context usage
from mind-swarm.context.context_size_manager import get_context_size_manager

size_manager = get_context_size_manager()
context_info = size_manager.get_context_size_info(
    context,
    model_name="google/gemini-2.0-flash-001",
    provider="openrouter"
)

print(f"Context usage: {context_info.usage_percentage:.1%}")
print(f"Estimated tokens: {context_info.estimated_tokens}")
print(f"Model limit: {context_info.model_context_limit}")
```

### Memory Debugging

```python
import psutil
import gc

def log_memory_usage():
    process = psutil.Process()
    memory_info = process.memory_info()
    
    print(f"RSS: {memory_info.rss / 1024 / 1024:.1f} MB")
    print(f"VMS: {memory_info.vms / 1024 / 1024:.1f} MB")
    print(f"Objects: {len(gc.get_objects())}")
    
    # Force garbage collection
    collected = gc.collect()
    print(f"Collected: {collected} objects")
```

## Hot Reload Development

### Development Workflow

1. **Start Server with Hot Reload**:
```bash
cd mind-swarm-core
MIND_SWARM_RUNTIME_PATH="../mind-swarm-runtime" python -m mind-swarm.server.main
```

2. **Edit Runtime Components**:
   - Modify YAML files for configuration changes
   - Edit Python files for implementation changes
   - Update Jinja2 templates for prompt changes

3. **Observe Automatic Reload**:
   - Watch server logs for reload messages
   - Test changes immediately without restart

### Hot Reload Best Practices

**File Organization**:
```
mind-swarm-runtime/
├── tools/
│   ├── category1/
│   │   ├── tool1.yaml
│   │   ├── tool1.py
│   │   └── __init__.py
│   └── category2/
├── agent_types/
│   ├── agent1.yaml
│   └── agent2.yaml
└── templates/
    └── static/
        ├── agent1.j2
        └── agent2.j2
```

**Atomic Changes**:
- Save related files (YAML + Python) quickly
- Use editor features to save multiple files at once
- Test one change at a time

**Error Recovery**:
- Hot reload validates before applying changes
- Invalid components won't break existing functionality
- Check logs for validation errors

### Development Helpers

**Runtime Component Validator**:
```python
# Validate before committing
from mind-swarm.runtime.validator import ComponentValidator
from pathlib import Path

validator = ComponentValidator([Path("./mind-swarm-runtime")])

# Validate a specific tool
with open("mind-swarm-runtime/tools/my_tool.yaml") as f:
    tool_data = yaml.safe_load(f)

result = validator.validate_component(tool_data, "tool")
if not result.valid:
    print(f"Validation errors: {result.errors}")
```

**Component Linter**:
```python
def lint_runtime_components():
    """Check all runtime components for common issues."""
    runtime_path = Path("./mind-swarm-runtime")
    
    # Check for orphaned files
    yaml_files = set(runtime_path.glob("tools/**/*.yaml"))
    py_files = set(runtime_path.glob("tools/**/*.py"))
    
    yaml_stems = {f.stem for f in yaml_files}
    py_stems = {f.stem for f in py_files if not f.name.startswith("__")}
    
    orphaned_yaml = yaml_stems - py_stems
    orphaned_py = py_stems - yaml_stems
    
    if orphaned_yaml:
        print(f"YAML files without Python: {orphaned_yaml}")
    if orphaned_py:
        print(f"Python files without YAML: {orphaned_py}")
```

## Performance Optimization

### Context Management

**Efficient Context Strategies**:
```yaml
# For chat agents - limit conversation length
context_management:
  strategy: entries
  parameters:
    max_entries: 5

# For long-running agents - time-based cleanup  
context_management:
  strategy: timed
  parameters:
    max_age: "1h"

# For resource-intensive agents - token limiting
context_management:
  strategy: adaptive
  parameters:
    max_tokens: 3000
    target_usage: 0.7
```

### Agent Pool Management

```python
# Limit concurrent agents per project
class ProjectAgentLimiter:
    def __init__(self, max_agents_per_project: int = 10):
        self.max_agents = max_agents_per_project
        self.project_counts = {}
    
    def can_create_agent(self, project_id: str) -> bool:
        current_count = self.project_counts.get(project_id, 0)
        return current_count < self.max_agents
    
    def agent_created(self, project_id: str):
        self.project_counts[project_id] = self.project_counts.get(project_id, 0) + 1
    
    def agent_destroyed(self, project_id: str):
        if project_id in self.project_counts:
            self.project_counts[project_id] -= 1
            if self.project_counts[project_id] <= 0:
                del self.project_counts[project_id]
```

### Tool Optimization

**Efficient Tool Implementation**:
```python
# Use connection pooling for external APIs
import aiohttp

class HTTPToolMixin:
    _session_pool = {}
    
    @classmethod
    async def get_session(cls, timeout: int = 30) -> aiohttp.ClientSession:
        if timeout not in cls._session_pool:
            cls._session_pool[timeout] = aiohttp.ClientSession(
                timeout=aiohttp.ClientTimeout(total=timeout),
                connector=aiohttp.TCPConnector(limit=100, limit_per_host=30)
            )
        return cls._session_pool[timeout]
    
    @classmethod
    async def cleanup_sessions(cls):
        for session in cls._session_pool.values():
            await session.close()
        cls._session_pool.clear()
```

## Deployment Patterns

### Production Configuration

```python
# Production config
production_config = {
    "runtime_paths": [Path("/opt/mind-swarm/runtime")],
    "data_dir": "/var/lib/mind-swarm",
    "cors_origins": ["https://app.company.com"],
    "hot_reload_enabled": False,  # Disable in production
    "log_level": "INFO",
    "max_agents_per_project": 20,
    "context_cleanup_interval": 3600  # 1 hour
}
```

### Container Deployment

**Dockerfile**:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy application
COPY mind-swarm-core/ ./mind-swarm-core/
COPY mind-swarm-runtime/ ./mind-swarm-runtime/

# Set environment
ENV MIND_SWARM_RUNTIME_PATH="/app/mind-swarm-runtime"
ENV MIND_SWARM_DATA_DIR="/data"
ENV PYTHONPATH="/app/mind-swarm-core/src"

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# Run application
CMD ["python", "-m", "mind-swarm.server.main"]
```

**Docker Compose**:
```yaml
version: '3.8'

services:
  mind-swarm:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./data:/data
      - ./runtime:/app/mind-swarm-runtime:ro
    environment:
      - LOG_LEVEL=INFO
      - MIND_SWARM_DATA_DIR=/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - mind-swarm
    restart: unless-stopped
```

### Load Balancing

```nginx
# nginx.conf
upstream mind-swarm_backend {
    server mind-swarm:8000;
    # Add more instances for scaling
    # server mind-swarm2:8000;
    # server mind-swarm3:8000;
}

server {
    listen 80;
    server_name api.company.com;
    
    location / {
        proxy_pass http://mind-swarm_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # WebSocket support
    location /ws {
        proxy_pass http://mind-swarm_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }
}
```

This developer guide provides the practical knowledge needed to effectively work with and extend the MindSwarm system.
