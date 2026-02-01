# Application Development Standards

## Overview

This repository contains comprehensive development standards designed to inform agents (both human developers and AI agents) when performing application design and development. These standards are built around the [Twelve-Factor App methodology](https://12factor.net/) and modern cloud-native development practices.

## Standards Documents

### [Architecture Standards](./architecture-standards/architecture-standards.md)
Comprehensive architecture standards covering:
- **Agent-Parallel Design**: Support for multiple agents working simultaneously
- **Modular Architecture**: Component design and microservices patterns
- **Twelve-Factor App Compliance**: Complete coverage of all 12 factors
- **Testing Standards**: Test pyramid and quality assurance
- **Security Standards**: Authentication, authorization, and data protection
- **Monitoring and Observability**: Logging, metrics, and health checks

### [AWS Architecture Standards](./architecture-standards/platform-specific/aws.md)
AWS-specific architectural standards including:
- **Serverless-First Approach**: Prefer serverless services over managed services
- **Network Security**: Private subnet deployment for compute resources
- **Cost Optimization**: Right-sizing, reserved capacity, and lifecycle policies
- **High Availability**: Multi-AZ deployment and fault tolerance
- **Security Controls**: IAM, VPC, WAF, and encryption standards
- **Monitoring and Alerting**: CloudWatch alarms for 4XX/5XX errors
- **Infrastructure as Code**: Terraform configurations and examples

### [Development Standards](./development-standards/development-standards.md)
Code quality and development practices including:
- **Code Formatting**: Consistent formatting and encoding standards
- **Code Quality Analysis**: Framework for detecting code smells and bugs
- **Security Vulnerability Assessment**: Security hotspots and basic security guidance
- **Code Generation Standards**: Naming conventions and structure guidelines
- **Compliance and Standards**: Industry best practices and regulatory compliance

### [Language-Specific Standards](./development-standards/language-specific/)
Language-specific development guidelines:
- **[JavaScript Standards](./development-standards/language-specific/javascript.md)**: Client-side security, Node.js patterns, framework-specific guidelines
- **[Python Standards](./development-standards/language-specific/python.md)**: Injection prevention, serialization security, framework patterns

### [Deployment Standards](./deployment-standards/deployment-standards.md)
Deployment and infrastructure standards covering:
- **Infrastructure as Code**: Version-controlled infrastructure using Terraform
- **Continuous Deployment**: Automated deployment pipelines
- **Twelve-Factor Deployment**: Factors V, VII, IX, and XII
- **Deployable Package Organization**: Separate packages for frontend and backend
- **Container Deployment**: Docker and Kubernetes standards
- **Environment Configuration**: Configuration management and validation
- **Security Standards**: Container and network security
- **Independent Deployment**: Each package deployable independently

### [Operations Standards](./operations-standards/operations-standards.md)
Operational excellence standards including:
- **Observability**: Monitoring, logging, tracing, and alerting
- **Process Lifecycle Management**: Fast startup and graceful shutdown
- **Administrative Processes**: One-off process execution
- **Incident Response**: Classification, procedures, and runbooks
- **Performance Standards**: SLAs, availability targets, and scalability
- **Disaster Recovery**: Backup strategies and recovery procedures

### [Security Standards](./security-standards/security-standards.md)
Comprehensive security standards covering:
- **Defense in Depth**: Multiple security layers
- **Security by Design**: Built-in security architecture
- **OWASP Top 10**: Complete coverage of security vulnerabilities
- **Data Protection**: Encryption, access control, and audit trails
- **API Security**: Authentication, authorization, and rate limiting
- **Compliance**: GDPR, CCPA, HIPAA, and SOX compliance

### [Data Standards](./data-standards/data-standards.md)
Data management and governance standards:
- **Data as a Service**: Backing services and configuration
- **Data Governance**: Classification, lineage, and quality
- **Data Protection**: Encryption, access control, and compliance
- **Database Standards**: Design principles and migrations
- **Backup and Recovery**: Strategies and procedures
- **Performance Standards**: Optimization and monitoring

### [Documentation Standards](./documentation-standards/documentation-standards.md)
Comprehensive documentation standards including:
- **Project README Requirements**: Standardized README structure and content
- **API Documentation**: OpenAPI 3.1.0 specifications and standards
- **Code Documentation**: Inline comments, function, and class documentation
- **Infrastructure Documentation**: Infrastructure as Code documentation
- **Deployment Documentation**: Process documentation and checklists
- **Troubleshooting Documentation**: Common issues and solutions
- **Documentation Maintenance**: Update triggers and review processes

### [Integration Standards](./integration-standards/integration-standards.md)
Modern integration standards covering:
- **A2A Protocol**: Agent2Agent protocol compliance for AI agents
- **MCP Protocol**: Model Context Protocol support for services
- **API-Led Connectivity**: System, Process, and Experience API patterns
- **Protocol-First Approach**: A2A and MCP protocol requirements
- **Separation of Concerns**: Frontend, agent, and service API separation
- **Shared Logic Layer**: Avoiding duplication across API layers
- **Multi-Transport Support**: JSON-RPC, gRPC, HTTP+JSON protocols
- **Integration Security**: Protocol-specific authentication and authorization

### [Git Workflow Standards](./workflow-standards/git-workflow-standards.md)
Development workflow and release process standards including:
- **Branch Strategy**: Persistent and transient branch management
- **Tagging Conventions**: Semantic versioning and date-based versioning options
- **Workflow Triggers**: Branch-based validation and tag-based deployments
- **Environment Strategy**: Preview, development, QA, and production environments
- **Release Process**: Standard release flow from feature to production
- **Hotfix Process**: Emergency production fix procedures
- **Terraform Workspace Management**: Infrastructure state isolation
- **Resource Naming Standards**: Consistent resource naming across environments
- **ServiceNow Integration**: Change management and audit compliance

## AI IDE Rule Examples

This repository includes copy-ready example rule files for AI-enabled IDEs. Use them to instruct your IDE to load and apply these standards before generating code. Example files live under `templates/ai-rules/` and are meant to be copied into your project.

Intended behavior:
- Preflight: recursively read these standards before changes
- Apply relevant standards based on the application type
- Validate outputs against those standards
- Local project standards override remote standards

| IDE/Tool | Example file in this repo | Destination path in your project | Notes |
| --- | --- | --- | --- |
| Cursor | `templates/ai-rules/cursor/.cursor/rules/standards-bootstrap.mdc` | `.cursor/rules/standards-bootstrap.mdc` | Always-apply rule file |
| Claude Code | `templates/ai-rules/claude/CLAUDE.md` | `CLAUDE.md` | Claude Code reads this file when present |
| GitHub Copilot | `templates/ai-rules/github/.github/copilot-instructions.md` | `.github/copilot-instructions.md` | Copilot instructions file |
| Generic (AGENTS) | `templates/ai-rules/generic/AGENTS.md` | `AGENTS.md` | Common format for multiple tools |
| Windsurf | `templates/ai-rules/generic/AGENTS.md` | `AGENTS.md` | Use the generic agent guidance |

## Twelve-Factor App Coverage

All standards documents are designed to support the [Twelve-Factor App methodology](https://12factor.net/):

| Factor | Description | Primary Coverage |
|--------|-------------|------------------|
| I | Codebase | Architecture Standards |
| II | Dependencies | Development Standards |
| III | Config | Security Standards, Architecture Standards |
| IV | Backing Services | Data Standards, Architecture Standards |
| V | Build, Release, Run | Deployment Standards |
| VI | Processes | Architecture Standards, Data Standards |
| VII | Port Binding | Deployment Standards |
| VIII | Concurrency | Architecture Standards |
| IX | Disposability | Operations Standards, Deployment Standards |
| X | Dev/Prod Parity | Architecture Standards, Deployment Standards |
| XI | Logs | Operations Standards, Security Standards |
| XII | Admin Processes | Operations Standards, Deployment Standards |

## Usage for Agents

### For Human Developers
1. **Start with Architecture Standards**: Understand the overall system design principles
2. **Review Platform-Specific Standards**: Apply AWS or other platform-specific guidelines
3. **Review Development Standards**: Follow code quality and security guidelines
4. **Check Language-Specific Standards**: Apply language-specific best practices
5. **Follow Git Workflow Standards**: Implement branch strategy, tagging, and release processes
6. **Follow Deployment Standards**: Ensure proper deployment practices
7. **Implement Operations Standards**: Set up monitoring and operational procedures
8. **Apply Security Standards**: Implement comprehensive security measures
9. **Follow Data Standards**: Ensure proper data management and governance
10. **Create Documentation**: Follow documentation standards for all projects
11. **Implement Integration Standards**: Support A2A and MCP protocols

### For AI Agents (Claude Code)
1. **Interface Contracts**: Always implement defined interfaces
2. **Error Handling**: Include comprehensive error handling
3. **Testing**: Generate corresponding tests for all code
4. **Documentation**: Update relevant documentation files
5. **Validation**: Verify implementations match specifications
6. **Security**: Follow OWASP Top 10 guidelines
7. **Twelve-Factor Compliance**: Ensure all 12 factors are addressed
8. **Workflow Compliance**: Follow Git workflow standards for branches, tags, and releases
9. **Protocol Compliance**: Implement A2A and MCP protocols as required
10. **Package Organization**: Create separate deployable packages
11. **Integration Standards**: Follow API-led connectivity patterns

## Implementation Checklist

### New Project Setup
- [ ] Review all standards documents
- [ ] Set up development environment
- [ ] Configure version control and Git workflow
- [ ] Implement CI/CD pipeline
- [ ] Set up monitoring and logging
- [ ] Configure security measures
- [ ] Establish data governance
- [ ] Create comprehensive documentation
- [ ] Implement A2A and MCP protocols
- [ ] Organize into deployable packages

### Feature Development
- [ ] Follow architecture patterns
- [ ] Implement security controls
- [ ] Write comprehensive tests
- [ ] Update documentation
- [ ] Validate against standards
- [ ] Perform security review
- [ ] Deploy using proper procedures
- [ ] Ensure protocol compliance
- [ ] Update integration documentation
- [ ] Test cross-package dependencies

### Maintenance and Operations
- [ ] Monitor system health
- [ ] Review performance metrics
- [ ] Update dependencies
- [ ] Apply security patches
- [ ] Backup and recovery testing
- [ ] Compliance audits
- [ ] Standards review and updates
- [ ] Protocol compliance verification
- [ ] Integration health checks
- [ ] Documentation maintenance

## Contributing

When contributing to these standards:

1. **Follow Existing Patterns**: Maintain consistency with current structure
2. **Include Examples**: Provide practical code examples
3. **Update Cross-References**: Ensure documents reference each other appropriately
4. **Test Implementation**: Verify standards work in practice
5. **Document Changes**: Update relevant sections when making changes

## Standards Review

These standards should be reviewed and updated regularly to ensure they remain:

- **Current**: Aligned with latest industry practices
- **Comprehensive**: Cover all necessary aspects of development
- **Practical**: Provide actionable guidance
- **Consistent**: Maintain internal consistency across documents
- **Effective**: Support successful application development

## Support

For questions or suggestions about these standards:

1. Review the relevant standards document
2. Check for existing patterns or examples
3. Consider the Twelve-Factor App methodology
4. Ensure consistency across all standards
5. Update documentation as needed

---

*These standards are designed to support modern, cloud-native application development while ensuring security, reliability, and maintainability.*
