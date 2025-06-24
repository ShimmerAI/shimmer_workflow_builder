# Workflow Builder API Design & Implementation Guide

## Table of Contents

1. Executive Summary
2. System Overview
3. Architecture Design
4. Database Schema
5. API Endpoints Specification
6. Core Services Design
7. Implementation Phases
8. Technical Requirements
9. Security Considerations
10. Performance Guidelines

---

## 1. Executive Summary

This document provides a comprehensive design guide for implementing a REST API backend for the Workflow Builder application. The API will enable programmatic creation, management, and execution of visual workflows through both traditional CRUD operations and AI-powered natural language generation.

### Key Objectives

- Enable full workflow lifecycle management (create, read, update, delete)
- Support AI-driven workflow generation from natural language descriptions
- Provide automatic node positioning and workflow validation
- Maintain compatibility with the existing React frontend
- Ensure scalability and performance for enterprise use

### Target Audience

This guide is designed for Python developers who will implement the backend API. No code snippets are included as the team will handle the actual implementation.

---

## 2. System Overview

### 2.1 High-Level Architecture

The system follows a three-tier architecture:

**Frontend Layer**
- React-based single-page application
- Uses ReactFlow library for workflow visualization
- Currently stores data in browser localStorage
- Expects specific data formats for nodes and edges

**API Layer** (To be implemented)
- RESTful API service
- Handles all business logic
- Integrates with AI services for workflow generation
- Manages data validation and transformation

**Data Layer**
- PostgreSQL database for persistent storage
- Redis cache for performance optimization
- File storage for import/export operations

### 2.2 Data Flow

1. User interacts with the React frontend
2. Frontend sends HTTP requests to API endpoints
3. API processes requests, validates data, and applies business logic
4. API interacts with database for persistence
5. API returns formatted responses to frontend
6. Frontend updates UI based on API responses

### 2.3 Integration Points

**External Services:**
- OpenAI API for natural language processing
- Email service for notifications
- File storage service for exports

**Internal Services:**
- Database service
- Cache service
- Background job processor

---

## 3. Architecture Design

### 3.1 Component Architecture

The API should be organized into the following logical components:

**API Gateway**
- Request routing
- Authentication/authorization
- Rate limiting
- Request/response logging

**Controllers**
- Handle HTTP requests/responses
- Input validation
- Error handling
- Response formatting

**Services**
- Business logic implementation
- Data processing
- Integration with external services
- Complex calculations

**Data Access Layer**
- Database operations
- Query optimization
- Transaction management
- Data mapping

**Utilities**
- Common helper functions
- Data converters
- Validators
- Graph algorithms

### 3.2 Service Architecture

**Core Services:**

1. **Workflow Service**
   - Manages workflow lifecycle
   - Coordinates node and edge operations
   - Handles workflow state transitions

2. **Validation Service**
   - Validates workflow structure
   - Checks node property schemas
   - Ensures edge connectivity rules
   - Identifies potential issues

3. **Layout Service**
   - Calculates optimal node positions
   - Implements multiple layout algorithms
   - Handles incremental positioning

4. **Schema Service**
   - Manages node type definitions
   - Provides default properties
   - Validates node configurations

5. **LLM Service**
   - Integrates with AI providers
   - Generates workflows from prompts
   - Enhances existing workflows
   - Fixes validation errors

6. **Import/Export Service**
   - Handles multiple file formats
   - Converts between formats
   - Validates imported data

### 3.3 Design Patterns

**Repository Pattern**
- Abstracts data access logic
- Enables easy testing
- Supports multiple data sources

**Service Layer Pattern**
- Encapsulates business logic
- Promotes reusability
- Simplifies testing

**Factory Pattern**
- Creates node instances
- Handles type-specific logic
- Manages default values

**Strategy Pattern**
- Multiple layout algorithms
- Different validation rules
- Various export formats

---

## 4. Database Schema

### 4.1 Core Tables

**workflows**
- id (UUID, primary key)
- name (string, required)
- description (text, optional)
- status (enum: draft, active, archived)
- created_at (timestamp)
- updated_at (timestamp)
- created_by (string, user identifier)
- metadata (JSON, additional properties)

**nodes**
- id (string, primary key)
- workflow_id (UUID, foreign key)
- type (enum: trigger, action, decision, delay, notification, conditional)
- label (string, required)
- position_x (float, required)
- position_y (float, required)
- properties (JSON, node-specific data)
- metadata (JSON, optional)

**edges**
- id (string, primary key)
- workflow_id (UUID, foreign key)
- source (string, node id)
- target (string, node id)
- type (enum: default, conditional)
- label (string, optional)
- data (JSON, edge-specific data)

### 4.2 Supporting Tables

**workflow_templates**
- id (UUID, primary key)
- name (string)
- description (text)
- category (string)
- template_data (JSON)
- is_public (boolean)

**workflow_executions**
- id (UUID, primary key)
- workflow_id (UUID, foreign key)
- status (enum: pending, running, completed, failed)
- started_at (timestamp)
- completed_at (timestamp)
- execution_data (JSON)

**audit_logs**
- id (UUID, primary key)
- workflow_id (UUID, foreign key)
- action (string)
- user_id (string)
- timestamp (timestamp)
- details (JSON)

### 4.3 Indexes

- workflow_id on nodes and edges tables
- status on workflows table
- created_at for sorting
- type on nodes table
- source and target on edges table

### 4.4 Relationships

- One workflow has many nodes
- One workflow has many edges
- Edges reference nodes through source/target
- Cascade delete for workflow deletion

---

## 5. API Endpoints Specification

### 5.1 Palette Management

**GET /api/diagram/palette**
- Purpose: Retrieve available node types for the workflow palette
- Response: Array of palette items with schemas and default properties
- Used by: Frontend palette component
- Notes: This endpoint must match the existing frontend expectation

### 5.2 Workflow Management

**POST /api/workflows**
- Purpose: Create a new workflow
- Request Body:
  - name (required): Workflow name
  - description (optional): Workflow description
  - nodes (array): Initial nodes
  - edges (array): Initial connections
- Response: Complete workflow object with generated IDs
- Validation: Must have at least one trigger node

**GET /api/workflows**
- Purpose: List all workflows
- Query Parameters:
  - skip: Pagination offset
  - limit: Number of results
  - status: Filter by status
  - sort: Sort field and direction
- Response: Paginated list of workflows

**GET /api/workflows/{workflow_id}**
- Purpose: Retrieve specific workflow
- Response: Complete workflow with nodes and edges
- Error Cases: 404 if workflow not found

**PUT /api/workflows/{workflow_id}**
- Purpose: Update workflow metadata
- Request Body: Fields to update (name, description, status)
- Response: Updated workflow object
- Notes: Does not update nodes/edges

**DELETE /api/workflows/{workflow_id}**
- Purpose: Delete workflow and all associated data
- Response: Success confirmation
- Notes: Cascades to nodes and edges

**GET /api/diagram/diagram**
- Purpose: Legacy endpoint for frontend compatibility
- Query Parameters: workflow_id (optional)
- Response: ReactFlow format data
- Notes: Returns empty diagram if no ID provided

### 5.3 Node Management (Priority 0)

**POST /api/workflows/{workflow_id}/nodes**
- Purpose: Add single node to workflow
- Request Body:
  - type (required): Node type
  - label (required): Display label
  - properties (required): Type-specific properties
  - position (optional): X,Y coordinates
- Response: Created node in ReactFlow format
- Validation: Properties must match node schema

**PUT /api/workflows/{workflow_id}/nodes/{node_id}**
- Purpose: Update node properties or position
- Request Body: Fields to update
- Response: Updated node
- Notes: Cannot change node type

**DELETE /api/workflows/{workflow_id}/nodes/{node_id}**
- Purpose: Remove node and connected edges
- Response: Success confirmation
- Side Effects: Deletes all edges connected to this node

**POST /api/workflows/{workflow_id}/nodes/batch**
- Purpose: Add multiple nodes efficiently
- Request Body: Array of node objects
- Response: Array of created nodes
- Use Case: Bulk operations, imports

### 5.4 Edge Management (priority 0)

**POST /api/workflows/{workflow_id}/edges**
- Purpose: Connect two nodes
- Request Body:
  - source (required): Source node ID
  - target (required): Target node ID
  - type (optional): Edge type
  - label (optional): Edge label
- Response: Created edge
- Validation: Both nodes must exist

**PUT /api/workflows/{workflow_id}/edges/{edge_id}**
- Purpose: Update edge properties
- Request Body: Fields to update
- Response: Updated edge
- Notes: Cannot change source/target

**DELETE /api/workflows/{workflow_id}/edges/{edge_id}**
- Purpose: Remove connection between nodes
- Response: Success confirmation

**POST /api/workflows/{workflow_id}/edges/batch**
- Purpose: Create multiple edges efficiently
- Request Body: Array of edge objects
- Response: Array of created edges

### 5.5 AI/Generation Endpoints

**GET /api/generator/llm/{prompt}**
- Purpose: Legacy endpoint for AI generation
- Path Parameter: Natural language prompt
- Response: Generated workflow in ReactFlow format
- Notes: Maintained for frontend compatibility

**POST /api/generator/generate**
- Purpose: Advanced AI workflow generation
- Request Body:
  - prompt (required): Natural language description
  - context (optional): Industry, complexity, etc.
  - options (optional): Generation preferences
- Response: Generated workflow with metadata
- Features: Context-aware generation

**GET /api/generator/templates**
- Purpose: Retrieve pre-built workflow templates
- Response: Array of template definitions
- Use Case: Quick-start options

**POST /api/generator/enhance/{workflow_id}**
- Purpose: AI-powered workflow enhancement
- Request Body: Enhancement type
- Enhancement Types:
  - add_error_handling
  - optimize_performance
  - add_notifications
  - add_logging
- Response: Enhanced workflow

### 5.6 Validation and Layout

**POST /api/workflows/validate**
- Purpose: Validate workflow structure
- Request Body: Workflow definition
- Response:
  - valid (boolean): Overall validity
  - errors (array): List of validation errors
  - warnings (array): Non-critical issues
- Use Case: Pre-save validation

**POST /api/workflows/layout**
- Purpose: Calculate optimal node positions
- Request Body:
  - nodes: Array of nodes
  - edges: Array of edges
  - algorithm: Layout algorithm choice
- Response: Nodes with calculated positions
- Algorithms: hierarchical, force, circular, grid

### 5.7 Import/Export

**POST /api/workflows/import**
- Purpose: Import workflow from file
- Request: Multipart form with file
- Query Parameter: format (json, yaml, bpmn, mermaid)
- Response: Created workflow
- Validation: Format-specific validation

**GET /api/workflows/{workflow_id}/export**
- Purpose: Export workflow to file
- Query Parameter: format
- Response: File download
- Formats: json, yaml, bpmn, mermaid, png, pdf

---

## 6. Core Services Design

### 6.1 Workflow Service

**Responsibilities:**
- Orchestrate workflow operations
- Manage workflow lifecycle
- Coordinate with other services
- Handle complex workflow operations

**Key Methods:**
- createWorkflow: Create with validation and layout
- updateWorkflow: Update metadata
- deleteWorkflow: Cascade deletion
- cloneWorkflow: Duplicate existing workflow
- getWorkflowStatistics: Analytics data

**Integration Points:**
- Validation Service for structure checking
- Layout Service for positioning
- Schema Service for node validation
- Database for persistence

### 6.2 Validation Service

**Responsibilities:**
- Validate workflow structure
- Check node properties
- Verify edge connections
- Identify potential issues

**Validation Rules:**
1. Every workflow must have at least one trigger
2. All nodes except triggers must be connected
3. No duplicate node IDs
4. Edge source and target must exist
5. Node properties must match schema
6. No orphaned nodes (except triggers)
7. Optional: No cycles in workflow

**Error Categories:**
- STRUCTURE_ERROR: Workflow structure issues
- PROPERTY_ERROR: Invalid node properties
- CONNECTION_ERROR: Edge connection issues
- SCHEMA_ERROR: Schema validation failures

### 6.3 Layout Service

**Responsibilities:**
- Calculate node positions
- Implement layout algorithms
- Handle incremental updates
- Optimize visual presentation

**Layout Algorithms:**

1. **Hierarchical Layout**
   - Best for: Sequential workflows
   - Approach: Top-down levels
   - Properties: Clear flow direction

2. **Force-Directed Layout**
   - Best for: Complex interconnected workflows
   - Approach: Physics simulation
   - Properties: Balanced spacing

3. **Grid Layout**
   - Best for: Simple workflows
   - Approach: Regular grid placement
   - Properties: Predictable positions

4. **Circular Layout**
   - Best for: Cyclic workflows
   - Approach: Nodes on circle
   - Properties: Equal importance

**Configuration:**
- Node dimensions: 180x80 pixels
- Horizontal spacing: 250 pixels
- Vertical spacing: 150 pixels
- Margin: 50 pixels

### 6.4 Schema Service

**Responsibilities:**
- Define node type schemas
- Provide default properties
- Validate node configurations
- Manage schema versions

**Node Types and Properties:**

1. **Trigger Node**
   - triggerType: manual, webhook, schedule, event
   - endpoint: URL for webhooks
   - schedule: Cron expression
   - event: Event identifier

2. **Action Node**
   - actionType: API call, email, database, transform
   - apiUrl: Endpoint URL
   - httpMethod: GET, POST, PUT, DELETE
   - headers: Request headers
   - body: Request payload

3. **Decision Node**
   - condition: Expression to evaluate
   - operator: equals, not_equals, greater_than, etc.
   - value: Comparison value

4. **Delay Node**
   - duration: Numeric value
   - unit: seconds, minutes, hours, days

5. **Notification Node**
   - notificationType: email, SMS, push, webhook
   - recipient: Target address
   - template: Message template
   - data: Template variables

### 6.5 LLM Service

**Responsibilities:**
- Generate workflows from prompts
- Fix validation errors
- Enhance existing workflows
- Manage AI provider integration

**Prompt Engineering:**
- System prompt defines available nodes
- User prompt describes desired workflow
- Context includes industry and complexity
- Output format matches schema

**Error Recovery:**
- Retry with refined prompts
- Fix common validation errors
- Fallback to template matching
- Log failures for improvement

**Enhancement Features:**
- Add error handling paths
- Optimize for performance
- Add logging and monitoring
- Include notifications

### 6.6 Import/Export Service

**Responsibilities:**
- Handle multiple file formats
- Convert between formats
- Validate imported data
- Generate export files

**Supported Formats:**

1. **JSON** (Native format)
   - Complete workflow definition
   - All properties preserved
   - Direct import/export

2. **YAML**
   - Human-readable format
   - Configuration-friendly
   - Preserves structure

3. **BPMN 2.0**
   - Industry standard
   - XML-based
   - Requires mapping

4. **Mermaid**
   - Text-based diagrams
   - Simple syntax
   - Limited properties

5. **Image Formats** (PNG, PDF)
   - Visual representation
   - Requires rendering service
   - Export only

---

## 7. Implementation Phases

### Phase 1: Core Infrastructure (Week 1-2)

**Objectives:**
- Set up project structure
- Configure development environment
- Implement database schema
- Create base models and schemas

**Deliverables:**
- Project skeleton
- Database migrations
- Basic data models
- Development environment

**Dependencies:**
- Database setup
- Development tools
- Team onboarding

### Phase 2: Basic CRUD Operations (Week 3-4)

**Objectives:**
- Implement workflow CRUD endpoints
- Add node management
- Create edge operations
- Basic validation

**Deliverables:**
- Workflow management API
- Node operations
- Edge operations
- Simple validation

**Testing Focus:**
- CRUD operations
- Data persistence
- Basic validation

### Phase 3: Advanced Services (Week 5-6)

**Objectives:**
- Implement validation service
- Add layout algorithms
- Create schema service
- Advanced operations

**Deliverables:**
- Complete validation
- Auto-layout functionality
- Schema management
- Batch operations

**Testing Focus:**
- Validation rules
- Layout algorithms
- Schema compliance

### Phase 4: AI Integration (Week 7-8)

**Objectives:**
- Integrate LLM service
- Implement generation endpoints
- Add enhancement features
- Template system

**Deliverables:**
- AI generation API
- Workflow enhancement
- Template management
- Error recovery

**Testing Focus:**
- AI integration
- Generation quality
- Error handling

### Phase 5: Import/Export & Polish (Week 9-10)

**Objectives:**
- Implement import/export
- Performance optimization
- Documentation
- Final testing

**Deliverables:**
- Import/export functionality
- Performance improvements
- API documentation
- Deployment ready

**Testing Focus:**
- File format handling
- Performance benchmarks
- Integration testing

---

## 8. Technical Requirements

### 8.1 Technology Stack

**Core Framework:**
- Python 3.11 or higher
- Async/await support
- Type hints throughout

**Web Framework:**
- FastAPI for API development
- Automatic API documentation
- Async request handling
- Dependency injection

**Data Validation:**
- Pydantic for schema validation
- Automatic serialization
- Type safety
- Error messages

**Database:**
- PostgreSQL 15+
- SQLAlchemy ORM
- Alembic migrations
- Connection pooling

**Caching:**
- Redis for caching
- Session storage
- Rate limiting
- Background jobs

**AI Integration:**
- LangChain framework
- OpenAI API
- Prompt management
- Response parsing

**Additional Libraries:**
- NetworkX for graph algorithms
- PyYAML for YAML support
- Pillow for image generation
- pytest for testing

### 8.2 Development Standards

**Code Organization:**
- Clear module separation
- Consistent naming conventions
- Comprehensive docstrings
- Type hints on all functions

**Error Handling:**
- Structured error responses
- Meaningful error codes
- Detailed error messages
- Proper HTTP status codes

**Logging:**
- Structured logging
- Request/response logging
- Error tracking
- Performance metrics

**Testing Requirements:**
- Unit tests for services
- Integration tests for APIs
- Mock external services
- 80% code coverage minimum

### 8.3 Performance Requirements

**Response Times:**
- Simple CRUD: < 100ms
- Complex operations: < 500ms
- AI generation: < 10 seconds
- File operations: < 30 seconds

**Scalability:**
- Support 1000+ workflows per user
- Handle 100+ concurrent users
- Process large workflows (100+ nodes)
- Efficient database queries

**Resource Limits:**
- Max nodes per workflow: 500
- Max edges per workflow: 1000
- Max file upload size: 10MB
- Request timeout: 30 seconds

---

## 9. Security Considerations

### 9.1 Authentication & Authorization

**Requirements:**
- API key authentication (initial phase)
- JWT tokens (future enhancement)
- Role-based access control
- Audit logging

**Security Headers:**
- CORS configuration
- Rate limiting
- Request size limits
- Content type validation

### 9.2 Data Security

**Validation:**
- Input sanitization
- SQL injection prevention
- XSS protection
- File type validation

**Sensitive Data:**
- Encrypt API keys
- Secure credential storage
- PII handling guidelines
- GDPR compliance

### 9.3 API Security

**Best Practices:**
- HTTPS only
- API versioning
- Deprecation notices
- Security headers

**Monitoring:**
- Failed request tracking
- Unusual pattern detection
- Rate limit violations
- Error rate monitoring

---

## 10. Performance Guidelines

### 10.1 Database Optimization

**Query Optimization:**
- Use appropriate indexes
- Avoid N+1 queries
- Batch operations
- Connection pooling

**Data Management:**
- Soft deletes for audit
- Archive old workflows
- Periodic cleanup
- Efficient JSON storage

### 10.2 Caching Strategy

**Cache Layers:**
- Database query cache
- API response cache
- Static data cache
- Session cache

**Cache Keys:**
- User-specific caching
- Workflow version tracking
- TTL configuration
- Cache invalidation

### 10.3 Async Operations

**Background Jobs:**
- AI generation
- File exports
- Bulk operations
- Email notifications

**Queue Management:**
- Priority queues
- Retry logic
- Dead letter queue
- Job monitoring

### 10.4 Monitoring & Metrics

**Key Metrics:**
- API response times
- Error rates
- Database performance
- Cache hit rates

**Alerting:**
- Performance degradation
- Error spikes
- Resource exhaustion
- Service availability

---

## Appendix A: Data Format Examples

### Workflow Creation Request

```
{
  "name": "Customer Onboarding",
  "description": "Automated customer onboarding process",
  "nodes": [
    {
      "type": "trigger",
      "label": "New Customer",
      "properties": {
        "triggerType": "webhook",
        "endpoint": "/api/customers/new"
      }
    },
    {
      "type": "action",
      "label": "Send Welcome Email",
      "properties": {
        "actionType": "email",
        "template": "welcome",
        "recipient": "{{customer.email}}"
      }
    }
  ],
  "edges": [
    {
      "source": "trigger-1",
      "target": "action-1",
      "type": "default"
    }
  ]
}
```

### Workflow Response Format

```
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Customer Onboarding",
  "description": "Automated customer onboarding process",
  "status": "active",
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-15T10:00:00Z",
  "data": {
    "nodes": [
      {
        "id": "trigger-1",
        "type": "node",
        "position": {"x": 100, "y": 100},
        "data": {
          "type": "trigger",
          "label": "New Customer",
          "icon": "Lightning",
          "properties": {
            "triggerType": "webhook",
            "endpoint": "/api/customers/new"
          }
        }
      }
    ],
    "edges": [
      {
        "id": "edge-1",
        "source": "trigger-1",
        "target": "action-1",
        "type": "default"
      }
    ],
    "viewport": {"x": 0, "y": 0, "zoom": 1}
  }
}
```

### Validation Error Response

```
{
  "valid": false,
  "errors": [
    {
      "code": "NO_TRIGGER_NODE",
      "message": "Workflow must have at least one trigger node"
    },
    {
      "code": "INVALID_NODE_TYPE",
      "node_id": "node-1",
      "field": "type",
      "message": "Invalid node type: unknown"
    }
  ],
  "warnings": [
    {
      "code": "NO_ERROR_HANDLING",
      "message": "Workflow has no error handling paths"
    }
  ]
}
```

---

## Appendix B: Node Property Schemas

### Trigger Node Properties

- **Manual Trigger**
  - No additional properties required
  
- **Webhook Trigger**
  - endpoint: string (required)
  - method: string (optional, default: POST)
  - headers: object (optional)
  
- **Schedule Trigger**
  - schedule: string (cron expression, required)
  - timezone: string (optional, default: UTC)
  
- **Event Trigger**
  - event: string (event name, required)
  - source: string (event source, optional)

### Action Node Properties

- **API Call Action**
  - apiUrl: string (required)
  - httpMethod: enum (GET, POST, PUT, DELETE, PATCH)
  - headers: object (optional)
  - body: object (optional)
  - timeout: number (seconds, optional)
  
- **Email Action**
  - recipient: string (required)
  - subject: string (required)
  - template: string (required)
  - data: object (template variables)
  
- **Database Action**
  - operation: enum (create, read, update, delete)
  - table: string (required)
  - filters: object (for read/update/delete)
  - data: object (for create/update)

### Decision Node Properties

- condition: string (expression, required)
- operator: enum (equals, not_equals, greater_than, less_than, contains, not_contains)
- value: any (comparison value, required)
- dataType: enum (string, number, boolean, date)

### Delay Node Properties

- duration: number (required, minimum: 1)
- unit: enum (seconds, minutes, hours, days)

### Notification Node Properties

- notificationType: enum (email, sms, push, webhook, inApp)
- recipient: string (required)
- template: string (optional)
- data: object (notification data)
- priority: enum (low, normal, high, urgent)

---

## Appendix C: Error Codes Reference

### Workflow Errors (WF_XXX)
- WF_001: Workflow not found
- WF_002: Workflow name required
- WF_003: Invalid workflow status
- WF_004: Workflow limit exceeded

### Node Errors (ND_XXX)
- ND_001: Node not found
- ND_002: Invalid node type
- ND_003: Node properties invalid
- ND_004: Duplicate node ID
- ND_005: Node position required

### Edge Errors (ED_XXX)
- ED_001: Edge not found
- ED_002: Invalid source node
- ED_003: Invalid target node
- ED_004: Duplicate edge
- ED_005: Self-connection not allowed

### Validation Errors (VL_XXX)
- VL_001: No trigger node
- VL_002: Orphaned node
- VL_003: Circular dependency
- VL_004: Schema validation failed
- VL_005: Required property missing

### System Errors (SY_XXX)
- SY_001: Database error
- SY_002: External service error
- SY_003: File operation error
- SY_004: Resource limit exceeded
- SY_005: Internal server error

---

## Document Version

- Version: 1.0
- Last Updated: January 2024
- Authors: Workflow Builder Team
- Review Status: Final

---

## Next Steps

1. Review this document with the development team
2. Clarify any questions or concerns
3. Set up development environment
4. Begin Phase 1 implementation
5. Schedule regular progress reviews

For questions or clarifications, please contact the project lead.
