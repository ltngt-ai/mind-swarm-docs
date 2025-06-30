# MindSwarm Documentation

Comprehensive documentation for the MindSwarm agent-first AI system.

## Overview

MindSwarm is a sophisticated agent-first AI execution engine that enables dynamic, mail-based communication between AI agents and users. The system provides a natural and asynchronous interaction model through structured email-like protocols, hot-reload capabilities, and robust agent lifecycle management.

## Documentation Structure

### 📋 [Architecture Overview](ARCHITECTURE.md)
Comprehensive high-level architecture documentation covering:
- Core principles and design philosophy
- System components and their interactions
- Runtime system and hot reload architecture
- Agent lifecycle and communication patterns
- Tool system and context management
- Security and authentication model

### 🔧 [Component Reference](COMPONENT_REFERENCE.md)
Detailed reference for all system components:
- Agent types and configuration schemas
- Tool system with YAML + Python architecture
- Template system using Jinja2
- Core services and their APIs
- Runtime components and validation
- Infrastructure components

### 👨‍💻 [Developer Guide](DEVELOPER_GUIDE.md)
Practical guide for developers working with MindSwarm:
- Development environment setup
- Creating custom agent types
- Building tools with YAML + Python
- Working with templates and shared components
- Agent communication patterns
- Testing strategies and debugging
- Hot reload development workflow

### 🚀 [Deployment Guide](DEPLOYMENT_GUIDE.md)
Production deployment and operations:
- System requirements and setup
- Container and Kubernetes deployment
- Configuration management and secrets
- Monitoring, logging, and alerting
- Security hardening procedures
- Backup and disaster recovery
- Scaling strategies and performance tuning

## Quick Start

### Prerequisites
- Python 3.11+
- Docker and Docker Compose
- Basic understanding of AI agents and email protocols

### Development Setup

1. **Clone the repositories**:
```bash
git clone https://github.com/ltngt-ai/mindswarm-core.git
git clone https://github.com/ltngt-ai/mindswarm-runtime.git
```

2. **Set up the environment**:
```bash
cd mindswarm-core
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

3. **Configure runtime path**:
```bash
export MINDSWARM_RUNTIME_PATH="../mindswarm-runtime"
```

4. **Start the development server**:
```bash
python -m mindswarm.server.main
```

The server will start with hot reload enabled, automatically picking up changes to runtime components.

### Basic Usage

1. **Connect via WebSocket** to `ws://localhost:8000/ws`

2. **Declare identity**:
```json
{
  "type": "set_identity",
  "email_address": "user@users.local.mindswarm.ltngt.ai"
}
```

3. **Send mail to agents**:
```json
{
  "type": "mail",
  "mail": {
    "to_address": "ui-agent@project.local.mindswarm.ltngt.ai",
    "subject": "Hello",
    "body": "Create a new project for me"
  }
}
```

## Key Concepts

### Agent-First Architecture
- **Autonomous Agents**: AI agents are first-class entities with their own mailboxes and lifecycles
- **Mail-Based Communication**: All interactions use RFC2822-style email messages
- **State Management**: Agents transition between IDLE, ACTIVE, PAUSED, and STOPPED states
- **Project Isolation**: Agents operate within project boundaries with proper security

### Hot Reload System
- **Runtime Flexibility**: Components can be updated without system restart
- **File System Monitoring**: Automatic detection of changes to YAML, Python, and template files
- **Validation**: Schema validation ensures compatibility before applying changes
- **Component Types**: Support for agent types, tools, templates, and knowledge packs

### Tool Architecture
- **YAML + Python**: Tools combine declarative YAML definitions with Python implementations
- **Schema Validation**: Strict parameter validation using JSON Schema
- **Hot Reload**: Tools can be updated and reloaded without restarting agents
- **Async Execution**: All tools use async/await for non-blocking execution

### Context Management
- **Intelligent Cleanup**: Automatic context management based on configurable policies
- **Multiple Strategies**: Entry-based, time-based, and adaptive context management
- **Memory Optimization**: Efficient token usage and context size tracking
- **Preservation**: Important metadata survives context resets

## System Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend UI   │    │   WebSocket     │    │   Agent         │
│                 │◄──►│   Gateway       │◄──►│   Manager       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                ▲                       ▲
                                │                       │
                       ┌─────────────────┐    ┌─────────────────┐
                       │   Mailbox       │    │   Runtime       │
                       │   System        │◄──►│   Loader        │
                       └─────────────────┘    └─────────────────┘
                                ▲                       ▲
                                │                       │
                       ┌─────────────────┐    ┌─────────────────┐
                       │   Tool          │    │   Hot Reload    │
                       │   Registry      │    │   System        │
                       └─────────────────┘    └─────────────────┘
```

## Component Overview

### Core System (`mindswarm-core`)
- **Server**: FastAPI application with WebSocket support
- **Agent Manager**: Lifecycle management for agents
- **Mailbox System**: RFC2822-based message routing
- **Context Management**: Intelligent conversation context handling
- **AI Integration**: Support for multiple AI models and providers
- **Authentication**: JWT-based user authentication and sessions

### Runtime System (`mindswarm-runtime`)
- **Agent Types**: YAML definitions for different agent behaviors
- **Tools**: YAML + Python implementations for agent capabilities
- **Templates**: Jinja2 templates for agent prompts and responses
- **Knowledge Packs**: Structured knowledge for specialized agents
- **Hot Reload**: File system monitoring and automatic updates

## Agent Communication Flow

```
User Message → WebSocket → UI Agent → Mailbox → Target Agent
     ▲                                              ▼
User Interface ◄─── WebSocket ◄─── UI Agent ◄─── Response
```

1. User sends message via WebSocket
2. UI Agent receives and processes the message
3. UI Agent routes request to appropriate specialized agent
4. Target agent processes request using available tools
5. Response flows back through the same path

## Development Workflow

### Creating a New Agent Type

1. **Define agent behavior** in `mindswarm-runtime/agent_types/my_agent.yaml`
2. **Create prompt template** in `mindswarm-runtime/templates/static/my_agent.j2`
3. **Test the agent** - hot reload will pick up changes automatically
4. **Deploy** - changes are automatically validated and applied

### Building Custom Tools

1. **Create YAML definition** with schema and AI instructions
2. **Implement Python function** with `execute_async` interface
3. **Test execution** - hot reload enables immediate testing
4. **Documentation** - AI prompt provides usage instructions

### Template Development

1. **Create Jinja2 templates** with shared components
2. **Use template variables** for dynamic content
3. **Test rendering** with different contexts
4. **Optimize for different models** and use cases

## Production Considerations

### Deployment Options
- **Docker Compose**: Single-node deployment with all services
- **Kubernetes**: Multi-node deployment with auto-scaling
- **Cloud Platforms**: AWS, GCP, Azure with managed services

### Monitoring & Observability
- **Metrics**: Prometheus metrics for agents, tools, and system health
- **Logging**: Structured logging with correlation IDs
- **Alerting**: Proactive alerts for system issues
- **Tracing**: Request tracing across agent interactions

### Security Features
- **Authentication**: JWT-based user authentication
- **Authorization**: Project-based access control
- **Network Security**: TLS encryption and firewall configuration
- **Input Validation**: Comprehensive input sanitization
- **Audit Logging**: Complete audit trail of all interactions

### Scalability
- **Horizontal Scaling**: Multiple server instances with load balancing
- **Agent Distribution**: Agents can run on different instances
- **Database Scaling**: Read replicas and connection pooling
- **Caching**: Redis for sessions and frequently accessed data

## Contributing

### Code Structure
- Follow the existing patterns for agent types, tools, and templates
- Use hot reload for rapid development and testing
- Include comprehensive tests for new components
- Document all public APIs and configuration options

### Testing Strategy
- **Unit Tests**: Individual component testing
- **Integration Tests**: Agent communication testing
- **End-to-End Tests**: Complete workflow testing
- **Performance Tests**: Load testing for scalability

### Documentation
- Update relevant documentation for new features
- Include examples and usage patterns
- Maintain API documentation
- Update deployment guides for new requirements

## Community and Support

### Resources
- **GitHub Issues**: Bug reports and feature requests
- **Documentation**: Comprehensive guides and references
- **Examples**: Sample implementations and patterns
- **Community**: Developer discussions and support

### Getting Help
1. **Check Documentation**: Start with the relevant guide
2. **Search Issues**: Look for existing solutions
3. **Create Issue**: Provide detailed reproduction steps
4. **Discussion**: Join community discussions for guidance

## License

MindSwarm is released under the MIT License. See LICENSE file for details.

## Acknowledgments

MindSwarm builds upon the work of many open-source projects and the broader AI and software development communities. We're grateful for the foundation provided by FastAPI, Jinja2, Redis, PostgreSQL, and the many other tools that make this system possible.
