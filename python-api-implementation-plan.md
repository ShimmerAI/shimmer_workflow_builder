# Python REST API Implementation Plan for Workflow Builder

## Executive Summary

This document provides a detailed implementation plan for building REST API endpoints in Python to support the Workflow Builder application. The plan is designed for a skilled Python development team and covers all necessary endpoints, data models, request/response formats, and implementation phases.

## Current State Analysis

### Frontend Expectations
Based on the frontend code analysis:
1. The frontend currently uses **localStorage** for persistence (no backend integration)
2. API endpoints are defined in environment files but **not implemented**:
   - `api/diagram/palette` - Get available node types
   - `api/diagram/diagram` - Load/save diagrams
   - `api/generator/llm` - Generate workflows from prompts
3. The frontend expects to work with ReactFlow format for nodes and edges

### Data Flow
```
Frontend (React) → API Gateway → Python Backend → Database
                                      ↓
                                 LLM Service
```

## Phase 1: Core Infrastructure

### 1.1 Technology Stack
```python
# Recommended stack
- FastAPI (async web framework)
- Pydantic (data validation)
- SQLAlchemy (ORM)
- PostgreSQL (database)
- Redis (caching)
- NetworkX (graph algorithms)
- LangChain (LLM integration)
```

### 1.2 Project Structure
```
workflow-builder-api/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app entry point
│   ├── config.py               # Configuration management
│   ├── database.py             # Database connection
│   ├── models/                 # SQLAlchemy models
│   │   ├── __init__.py
│   │   ├── workflow.py
│   │   ├── node.py
│   │   └── edge.py
│   ├── schemas/                # Pydantic schemas
│   │   ├── __init__.py
│   │   ├── workflow.py
│   │   ├── node.py
│   │   ├── edge.py
│   │   └── validation.py
│   ├── api/                    # API endpoints
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── workflows.py
│   │   │   ├── nodes.py
│   │   │   ├── edges.py
│   │   │   ├── palette.py
│   │   │   └── generator.py
│   ├── services/               # Business logic
│   │   ├── __init__.py
│   │   ├── workflow_service.py
│   │   ├── validation_service.py
│   │   ├── layout_service.py
│   │   ├── schema_service.py
│   │   └── llm_service.py
│   ├── core/                   # Core utilities
│   │   ├── __init__.py
│   │   ├── exceptions.py
│   │   ├── security.py
│   │   └── dependencies.py
│   └── utils/                  # Helper functions
│       ├── __init__.py
│       ├── graph_algorithms.py
│       └── converters.py
├── tests/
├── alembic/                    # Database migrations
├── requirements.txt
├── docker-compose.yml
└── Dockerfile
```

### 1.3 Data Models (SQLAlchemy)

```python
# app/models/workflow.py
from sqlalchemy import Column, String, JSON, DateTime, Enum
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
import uuid
from datetime import datetime

class Workflow(Base):
    __tablename__ = "workflows"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String, nullable=False)
    description = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    created_by = Column(String)  # User ID when auth is implemented
    status = Column(Enum('draft', 'active', 'archived'), default='draft')
    metadata = Column(JSON)  # Store additional properties
    
    # Relationships
    nodes = relationship("Node", back_populates="workflow", cascade="all, delete-orphan")
    edges = relationship("Edge", back_populates="workflow", cascade="all, delete-orphan")

# app/models/node.py
class Node(Base):
    __tablename__ = "nodes"
    
    id = Column(String, primary_key=True)  # Frontend-generated ID
    workflow_id = Column(UUID(as_uuid=True), ForeignKey("workflows.id"))
    type = Column(Enum('trigger', 'action', 'decision', 'delay', 'notification', 'conditional'))
    label = Column(String, nullable=False)
    position_x = Column(Float, nullable=False)
    position_y = Column(Float, nullable=False)
    properties = Column(JSON, nullable=False)  # Node-specific properties
    metadata = Column(JSON)  # Additional metadata
    
    # Relationships
    workflow = relationship("Workflow", back_populates="nodes")

# app/models/edge.py
class Edge(Base):
    __tablename__ = "edges"
    
    id = Column(String, primary_key=True)  # Frontend-generated ID
    workflow_id = Column(UUID(as_uuid=True), ForeignKey("workflows.id"))
    source = Column(String, nullable=False)  # Node ID
    target = Column(String, nullable=False)  # Node ID
    type = Column(Enum('default', 'conditional'), default='default')
    label = Column(String)
    data = Column(JSON)  # Edge-specific data (conditions, etc.)
    
    # Relationships
    workflow = relationship("Workflow", back_populates="edges")
```

### 1.4 Pydantic Schemas

```python
# app/schemas/node.py
from pydantic import BaseModel, Field, validator
from typing import Optional, Dict, Any, Literal
from enum import Enum

class NodeType(str, Enum):
    TRIGGER = "trigger"
    ACTION = "action"
    DECISION = "decision"
    DELAY = "delay"
    NOTIFICATION = "notification"
    CONDITIONAL = "conditional"

class Position(BaseModel):
    x: float
    y: float

class NodeBase(BaseModel):
    type: NodeType
    label: str
    properties: Dict[str, Any]
    metadata: Optional[Dict[str, Any]] = None

class NodeCreate(NodeBase):
    id: Optional[str] = None
    position: Optional[Position] = None

class NodeUpdate(BaseModel):
    label: Optional[str] = None
    properties: Optional[Dict[str, Any]] = None
    position: Optional[Position] = None
    metadata: Optional[Dict[str, Any]] = None

class NodeInDB(NodeBase):
    id: str
    workflow_id: str
    position: Position
    
    class Config:
        orm_mode = True

class ReactFlowNode(BaseModel):
    """Node format expected by ReactFlow frontend"""
    id: str
    type: Literal["node"] = "node"  # ReactFlow node type
    position: Position
    data: Dict[str, Any]
    selected: Optional[bool] = False

# app/schemas/edge.py
class EdgeType(str, Enum):
    DEFAULT = "default"
    CONDITIONAL = "conditional"

class EdgeBase(BaseModel):
    source: str
    target: str
    type: EdgeType = EdgeType.DEFAULT
    label: Optional[str] = None
    data: Optional[Dict[str, Any]] = None

class EdgeCreate(EdgeBase):
    id: Optional[str] = None

class EdgeInDB(EdgeBase):
    id: str
    workflow_id: str
    
    class Config:
        orm_mode = True

class ReactFlowEdge(BaseModel):
    """Edge format expected by ReactFlow frontend"""
    id: str
    source: str
    target: str
    type: str = "default"
    data: Optional[Dict[str, Any]] = None

# app/schemas/workflow.py
class WorkflowBase(BaseModel):
    name: str
    description: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = None

class WorkflowCreate(WorkflowBase):
    nodes: List[NodeCreate] = []
    edges: List[EdgeCreate] = []

class WorkflowUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    status: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = None

class WorkflowInDB(WorkflowBase):
    id: str
    created_at: datetime
    updated_at: datetime
    status: str
    nodes: List[NodeInDB] = []
    edges: List[EdgeInDB] = []
    
    class Config:
        orm_mode = True

class ReactFlowData(BaseModel):
    """Complete workflow data in ReactFlow format"""
    nodes: List[ReactFlowNode]
    edges: List[ReactFlowEdge]
    viewport: Optional[Dict[str, float]] = {"x": 0, "y": 0, "zoom": 1}

class WorkflowResponse(BaseModel):
    """API response format"""
    id: str
    name: str
    description: Optional[str]
    created_at: datetime
    updated_at: datetime
    status: str
    data: ReactFlowData
    metadata: Optional[Dict[str, Any]]
```

### 1.5 Validation Service

```python
# app/services/validation_service.py
from typing import List, Dict, Set, Tuple
from app.schemas.workflow import WorkflowCreate
from app.schemas.validation import ValidationResult, ValidationError
from app.services.schema_service import SchemaService

class ValidationService:
    def __init__(self, schema_service: SchemaService):
        self.schema_service = schema_service
    
    async def validate_workflow(self, workflow: WorkflowCreate) -> ValidationResult:
        errors = []
        warnings = []
        
        # 1. Validate nodes
        node_ids = set()
        for node in workflow.nodes:
            node_errors = await self._validate_node(node)
            errors.extend(node_errors)
            
            # Check for duplicate IDs
            if node.id:
                if node.id in node_ids:
                    errors.append(ValidationError(
                        node_id=node.id,
                        message=f"Duplicate node ID: {node.id}",
                        code="DUPLICATE_NODE_ID"
                    ))
                node_ids.add(node.id)
        
        # 2. Validate edges
        for edge in workflow.edges:
            edge_errors = self._validate_edge(edge, node_ids)
            errors.extend(edge_errors)
        
        # 3. Validate workflow structure
        structure_errors = self._validate_structure(workflow)
        errors.extend(structure_errors)
        
        # 4. Check for warnings
        warnings = self._check_warnings(workflow)
        
        return ValidationResult(
            valid=len(errors) == 0,
            errors=errors,
            warnings=warnings
        )
    
    async def _validate_node(self, node: NodeCreate) -> List[ValidationError]:
        errors = []
        
        # Validate node type
        valid_types = await self.schema_service.get_valid_node_types()
        if node.type not in valid_types:
            errors.append(ValidationError(
                node_id=node.id,
                field="type",
                message=f"Invalid node type: {node.type}",
                code="INVALID_NODE_TYPE"
            ))
        
        # Validate properties against schema
        schema = await self.schema_service.get_node_schema(node.type)
        if schema:
            property_errors = self._validate_properties(node.properties, schema)
            errors.extend(property_errors)
        
        return errors
    
    def _validate_edge(self, edge: EdgeCreate, node_ids: Set[str]) -> List[ValidationError]:
        errors = []
        
        if edge.source not in node_ids:
            errors.append(ValidationError(
                edge_id=edge.id,
                field="source",
                message=f"Source node not found: {edge.source}",
                code="INVALID_SOURCE_NODE"
            ))
        
        if edge.target not in node_ids:
            errors.append(ValidationError(
                edge_id=edge.id,
                field="target",
                message=f"Target node not found: {edge.target}",
                code="INVALID_TARGET_NODE"
            ))
        
        return errors
    
    def _validate_structure(self, workflow: WorkflowCreate) -> List[ValidationError]:
        errors = []
        
        # Must have at least one trigger node
        trigger_nodes = [n for n in workflow.nodes if n.type == "trigger"]
        if not trigger_nodes:
            errors.append(ValidationError(
                message="Workflow must have at least one trigger node",
                code="NO_TRIGGER_NODE"
            ))
        
        # Check for orphaned nodes (except triggers)
        connected_nodes = set()
        for edge in workflow.edges:
            connected_nodes.add(edge.source)
            connected_nodes.add(edge.target)
        
        for node in workflow.nodes:
            if node.id and node.id not in connected_nodes and node.type != "trigger":
                errors.append(ValidationError(
                    node_id=node.id,
                    message="Node is not connected to any other node",
                    code="ORPHANED_NODE"
                ))
        
        # Check for cycles (optional, depending on requirements)
        if self._has_cycles(workflow):
            errors.append(ValidationError(
                message="Workflow contains cycles",
                code="WORKFLOW_HAS_CYCLES"
            ))
        
        return errors
    
    def _has_cycles(self, workflow: WorkflowCreate) -> bool:
        # Implement cycle detection using DFS
        # This is optional depending on whether cycles are allowed
        pass
```

### 1.6 Layout Service

```python
# app/services/layout_service.py
import networkx as nx
from typing import List, Dict, Tuple
from app.schemas.node import NodeCreate, Position
from app.schemas.edge import EdgeCreate

class LayoutService:
    def __init__(self):
        self.node_width = 180
        self.node_height = 80
        self.h_spacing = 250
        self.v_spacing = 150
    
    async def calculate_positions(
        self, 
        nodes: List[NodeCreate], 
        edges: List[EdgeCreate]
    ) -> List[NodeCreate]:
        """Calculate positions for nodes that don't have positions"""
        
        # If all nodes have positions, return as-is
        if all(node.position for node in nodes):
            return nodes
        
        # Create NetworkX graph
        G = nx.DiGraph()
        
        # Add nodes
        node_map = {}
        for i, node in enumerate(nodes):
            node_id = node.id or f"node_{i}"
            G.add_node(node_id)
            node_map[node_id] = node
        
        # Add edges
        for edge in edges:
            if edge.source in G and edge.target in G:
                G.add_edge(edge.source, edge.target)
        
        # Determine layout algorithm
        layout_algo = self._determine_layout_algorithm(G)
        
        # Calculate positions
        pos = layout_algo(G)
        
        # Apply positions to nodes
        positioned_nodes = []
        for node in nodes:
            node_id = node.id or f"node_{nodes.index(node)}"
            if node_id in pos:
                x, y = pos[node_id]
                node.position = Position(
                    x=x * self.h_spacing,
                    y=y * self.v_spacing
                )
            positioned_nodes.append(node)
        
        return positioned_nodes
    
    def _determine_layout_algorithm(self, G: nx.DiGraph):
        """Choose appropriate layout algorithm based on graph properties"""
        
        # For DAGs, use hierarchical layout
        if nx.is_directed_acyclic_graph(G):
            return self._hierarchical_layout
        
        # For small graphs, use spring layout
        if len(G.nodes()) <= 10:
            return lambda g: nx.spring_layout(g, k=2, iterations=50)
        
        # For larger graphs, use Kamada-Kawai
        return nx.kamada_kawai_layout
    
    def _hierarchical_layout(self, G: nx.DiGraph) -> Dict[str, Tuple[float, float]]:
        """Custom hierarchical layout for DAGs"""
        
        # Calculate levels using topological sort
        try:
            topo_order = list(nx.topological_sort(G))
        except nx.NetworkXError:
            # Fall back to spring layout if not a DAG
            return nx.spring_layout(G)
        
        # Assign levels
        levels = {}
        for node in topo_order:
            predecessors = list(G.predecessors(node))
            if not predecessors:
                levels[node] = 0
            else:
                levels[node] = max(levels[pred] for pred in predecessors) + 1
        
        # Group nodes by level
        level_nodes = {}
        for node, level in levels.items():
            if level not in level_nodes:
                level_nodes[level] = []
            level_nodes[level].append(node)
        
        # Calculate positions
        pos = {}
        for level, nodes in level_nodes.items():
            for i, node in enumerate(nodes):
                x = level
                y = i - (len(nodes) - 1) / 2
                pos[node] = (x, y)
        
        return pos
```

### 1.7 Schema Service

```python
# app/services/schema_service.py
from typing import Dict, List, Any
import json

class SchemaService:
    def __init__(self):
        self.schemas = self._load_schemas()
        self.default_properties = self._load_default_properties()
    
    def _load_schemas(self) -> Dict[str, Dict]:
        """Load node schemas - in production, these could come from a database"""
        return {
            "trigger": {
                "type": "object",
                "properties": {
                    "triggerType": {
                        "type": "string",
                        "enum": ["manual", "webhook", "schedule", "event", "recordChange", "apiCall", "userAction"]
                    },
                    "endpoint": {"type": "string"},
                    "schedule": {"type": "string"},
                    "event": {"type": "string"}
                },
                "required": ["triggerType"]
            },
            "action": {
                "type": "object",
                "properties": {
                    "actionType": {
                        "type": "string",
                        "enum": ["makeApiCall", "createRecord", "updateRecord", "deleteRecord", "sendEmail", "transform"]
                    },
                    "makeAPICall": {
                        "type": "object",
                        "properties": {
                            "apiUrl": {"type": "string"},
                            "httpMethod": {"type": "string", "enum": ["GET", "POST", "PUT", "DELETE", "PATCH"]},
                            "headers": {"type": "object"},
                            "body": {"type": "object"},
                            "responseFormat": {"type": "string", "enum": ["json", "xml", "text"]}
                        }
                    }
                },
                "required": ["actionType"]
            },
            "decision": {
                "type": "object",
                "properties": {
                    "condition": {"type": "string"},
                    "operator": {
                        "type": "string",
                        "enum": ["equals", "not_equals", "greater_than", "less_than", "contains", "not_contains"]
                    },
                    "value": {"type": ["string", "number", "boolean"]}
                },
                "required": ["condition", "operator", "value"]
            },
            "delay": {
                "type": "object",
                "properties": {
                    "duration": {"type": "number", "minimum": 1},
                    "unit": {"type": "string", "enum": ["seconds", "minutes", "hours", "days"]}
                },
                "required": ["duration", "unit"]
            },
            "notification": {
                "type": "object",
                "properties": {
                    "notificationType": {"type": "string", "enum": ["email", "sms", "push", "webhook", "inApp"]},
                    "recipient": {"type": "string"},
                    "template": {"type": "string"},
                    "data": {"type": "object"}
                },
                "required": ["notificationType", "recipient"]
            }
        }
    
    def _load_default_properties(self) -> Dict[str, Dict]:
        """Load default properties for each node type"""
        return {
            "trigger": {
                "triggerType": "manual",
                "label": "Start Workflow"
            },
            "action": {
                "actionType": "makeApiCall",
                "makeAPICall": {
                    "apiUrl": "https://api.example.com/endpoint",
                    "httpMethod": "GET",
                    "responseFormat": "json"
                }
            },
            "decision": {
                "operator": "equals",
                "label": "Check Condition"
            },
            "delay": {
                "duration": 5,
                "unit": "minutes",
                "label": "Wait"
            },
            "notification": {
                "notificationType": "email",
                "label": "Send Notification"
            }
        }
    
    async def get_node_schema(self, node_type: str) -> Dict:
        return self.schemas.get(node_type, {})
    
    async def get_default_properties(self, node_type: str) -> Dict:
        return self.default_properties.get(node_type, {})
    
    async def get_valid_node_types(self) -> List[str]:
        return list(self.schemas.keys())
    
    def get_node_icon(self, node_type: str) -> str:
        """Get icon name for node type (matching frontend expectations)"""
        icon_map = {
            "trigger": "Lightning",
            "action": "Gear",
            "decision": "GitBranch",
            "delay": "Clock",
            "notification": "Bell",
            "conditional": "GitMerge"
        }
        return icon_map.get(node_type, "Circle")
```

## Phase 2: API Development

### 2.1 API Endpoints

#### Palette Endpoints
```python
# app/api/v1/palette.py
from fastapi import APIRouter, Depends
from typing import List
from app.schemas.palette import PaletteItem
from app.services.schema_service import SchemaService

router = APIRouter(prefix="/api/diagram", tags=["palette"])

@router.get("/palette", response_model=List[PaletteItem])
async def get_palette(schema_service: SchemaService = Depends()):
    """
    Get available node types for the palette
    
    Expected by frontend at: api/diagram/palette
    """
    node_types = await schema_service.get_valid_node_types()
    palette_items = []
    
    for node_type in node_types:
        schema = await schema_service.get_node_schema(node_type)
        default_props = await schema_service.get_default_properties(node_type)
        icon = schema_service.get_node_icon(node_type)
        
        palette_items.append(PaletteItem(
            type=node_type,
            icon=icon,
            label=node_type.title(),
            description=f"Add a {node_type} node",
            schema=schema,
            defaultPropertiesData=default_props
        ))
    
    return palette_items
```

#### Workflow Endpoints
```python
# app/api/v1/workflows.py
from fastapi import APIRouter, Depends, HTTPException, Query
from typing import List, Optional
from uuid import UUID
from app.schemas.workflow import (
    WorkflowCreate, WorkflowUpdate, WorkflowResponse, ReactFlowData
)
from app.services.workflow_service import WorkflowService
from app.core.dependencies import get_workflow_service

router = APIRouter(prefix="/api", tags=["workflows"])

@router.post("/workflows", response_model=WorkflowResponse)
async def create_workflow(
    workflow: WorkflowCreate,
    service: WorkflowService = Depends(get_workflow_service)
):
    """
    Create a new workflow
    
    Request body:
    {
        "name": "Customer Onboarding",
        "description": "Automated customer onboarding process",
        "nodes": [
            {
                "type": "trigger",
                "label": "Start",
                "properties": {"triggerType": "manual"}
            }
        ],
        "edges": []
    }
    
    Response:
    {
        "id": "uuid",
        "name": "Customer Onboarding",
        "created_at": "2024-01-15T10:00:00Z",
        "updated_at": "2024-01-15T10:00:00Z",
        "status": "draft",
        "data": {
            "nodes": [...],
            "edges": [...],
            "viewport": {"x": 0, "y": 0, "zoom": 1}
        }
    }
    """
    return await service.create_workflow(workflow)

@router.get("/workflows", response_model=List[WorkflowResponse])
async def list_workflows(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    status: Optional[str] = None,
    service: WorkflowService = Depends(get_workflow_service)
):
    """List all workflows with pagination"""
    return await service.list_workflows(skip=skip, limit=limit, status=status)

@router.get("/workflows/{workflow_id}", response_model=WorkflowResponse)
async def get_workflow(
    workflow_id: UUID,
    service: WorkflowService = Depends(get_workflow_service)
):
    """Get a specific workflow by ID"""
    workflow = await service.get_workflow(workflow_id)
    if not workflow:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return workflow

@router.put("/workflows/{workflow_id}", response_model=WorkflowResponse)
async def update_workflow(
    workflow_id: UUID,
    workflow_update: WorkflowUpdate,
    service: WorkflowService = Depends(get_workflow_service)
):
    """Update workflow metadata (name, description, status)"""
    workflow = await service.update_workflow(workflow_id, workflow_update)
    if not workflow:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return workflow

@router.delete("/workflows/{workflow_id}")
async def delete_workflow(
    workflow_id: UUID,
    service: WorkflowService = Depends(get_workflow_service)
):
    """Delete a workflow"""
    success = await service.delete_workflow(workflow_id)
    if not success:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return {"message": "Workflow deleted successfully"}

@router.get("/diagram/diagram", response_model=Optional[ReactFlowData])
async def get_diagram(
    workflow_id: Optional[UUID] = Query(None),
    service: WorkflowService = Depends(get_workflow_service)
):
    """
    Get diagram data (for compatibility with existing frontend)
    
    Expected by frontend at: api/diagram/diagram
    """
    if not workflow_id:
        # Return empty diagram if no workflow specified
        return ReactFlowData(nodes=[], edges=[], viewport={"x": 0, "y": 0, "zoom": 1})
    
    workflow = await service.get_workflow(workflow_id)
    if not workflow:
        return None
    
    return workflow.data

@router.post("/workflows/{workflow_id}/save")
async def save_workflow_state(
    workflow_id: UUID,
    data: ReactFlowData,
    service: WorkflowService = Depends(get_workflow_service)
):
    """Save the current state of a workflow (nodes, edges, viewport)"""
    success = await service.save_workflow_state(workflow_id, data)
    if not success:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return {"message": "Workflow saved successfully"}
```

#### Node Management Endpoints
```python
# app/api/v1/nodes.py
from fastapi import APIRouter, Depends, HTTPException
from typing import List
from uuid import UUID
from app.schemas.node import NodeCreate, NodeUpdate, ReactFlowNode
from app.services.workflow_service import WorkflowService

router = APIRouter(prefix="/api/workflows/{workflow_id}", tags=["nodes"])

@router.post("/nodes", response_model=ReactFlowNode)
async def add_node(
    workflow_id: UUID,
    node: NodeCreate,
    service: WorkflowService = Depends(get_workflow_service)
):
    """
    Add a node to a workflow
    
    Request body:
    {
        "type": "action",
        "label": "Send Email",
        "properties": {
            "actionType": "sendEmail",
            "recipient": "{{customer.email}}"
        },
        "position": {"x": 250, "y": 100}
    }
    """
    result = await service.add_node(workflow_id, node)
    if not result:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return result

@router.put("/nodes/{node_id}", response_model=ReactFlowNode)
async def update_node(
    workflow_id: UUID,
    node_id: str,
    node_update: NodeUpdate,
    service: WorkflowService = Depends(get_workflow_service)
):
    """Update a node's properties or position"""
    result = await service.update_node(workflow_id, node_id, node_update)
    if not result:
        raise HTTPException(status_code=404, detail="Node not found")
    return result

@router.delete("/nodes/{node_id}")
async def delete_node(
    workflow_id: UUID,
    node_id: str,
    service: WorkflowService = Depends(get_workflow_service)
):
    """Delete a node and its connected edges"""
    success = await service.delete_node(workflow_id, node_id)
    if not success:
        raise HTTPException(status_code=404, detail="Node not found")
    return {"message": "Node deleted successfully"}

@router.post("/nodes/batch", response_model=List[ReactFlowNode])
async def add_nodes_batch(
    workflow_id: UUID,
    nodes: List[NodeCreate],
    service: WorkflowService = Depends(get_workflow_service)
):
    """Add multiple nodes at once"""
    results = await service.add_nodes_batch(workflow_id, nodes)
    if not results:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return results
```

#### Edge Management Endpoints
```python
# app/api/v1/edges.py
from fastapi import APIRouter, Depends, HTTPException
from typing import List
from uuid import UUID
from app.schemas.edge import EdgeCreate, EdgeUpdate, ReactFlowEdge
from app.services.workflow_service import WorkflowService

router = APIRouter(prefix="/api/workflows/{workflow_id}", tags=["edges"])

@router.post("/edges", response_model=ReactFlowEdge)
async def add_edge(
    workflow_id: UUID,

# Python REST API Implementation Plan - Part 2

## Continuation of API Endpoints

### Edge Management Endpoints (continued)

```python
# app/api/v1/edges.py
from fastapi import APIRouter, Depends, HTTPException
from typing import List
from uuid import UUID
from app.schemas.edge import EdgeCreate, EdgeUpdate, ReactFlowEdge
from app.services.workflow_service import WorkflowService

router = APIRouter(prefix="/api/workflows/{workflow_id}", tags=["edges"])

@router.post("/edges", response_model=ReactFlowEdge)
async def add_edge(
    workflow_id: UUID,
    edge: EdgeCreate,
    service: WorkflowService = Depends(get_workflow_service)
):
    """
    Add an edge between two nodes
    
    Request body:
    {
        "source": "trigger-1",
        "target": "action-1",
        "type": "default",
        "label": "On Success"
    }
    """
    result = await service.add_edge(workflow_id, edge)
    if not result:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return result

@router.put("/edges/{edge_id}", response_model=ReactFlowEdge)
async def update_edge(
    workflow_id: UUID,
    edge_id: str,
    edge_update: EdgeUpdate,
    service: WorkflowService = Depends(get_workflow_service)
):
    """Update an edge's properties"""
    result = await service.update_edge(workflow_id, edge_id, edge_update)
    if not result:
        raise HTTPException(status_code=404, detail="Edge not found")
    return result

@router.delete("/edges/{edge_id}")
async def delete_edge(
    workflow_id: UUID,
    edge_id: str,
    service: WorkflowService = Depends(get_workflow_service)
):
    """Delete an edge"""
    success = await service.delete_edge(workflow_id, edge_id)
    if not success:
        raise HTTPException(status_code=404, detail="Edge not found")
    return {"message": "Edge deleted successfully"}

@router.post("/edges/batch", response_model=List[ReactFlowEdge])
async def add_edges_batch(
    workflow_id: UUID,
    edges: List[EdgeCreate],
    service: WorkflowService = Depends(get_workflow_service)
):
    """Add multiple edges at once"""
    results = await service.add_edges_batch(workflow_id, edges)
    if not results:
        raise HTTPException(status_code=404, detail="Workflow not found")
    return results
```

### Generator/LLM Endpoints

```python
# app/api/v1/generator.py
from fastapi import APIRouter, Depends, HTTPException
from app.schemas.generator import (
    GenerateFromPromptRequest,
    GenerateFromPromptResponse,
    WorkflowTemplate
)
from app.services.llm_service import LLMService
from app.services.workflow_service import WorkflowService

router = APIRouter(prefix="/api/generator", tags=["generator"])

@router.get("/llm/{prompt}")
async def generate_from_prompt_legacy(
    prompt: str,
    llm_service: LLMService = Depends(),
    workflow_service: WorkflowService = Depends()
):
    """
    Legacy endpoint for compatibility with existing frontend
    Expected by frontend at: api/generator/llm/{prompt}
    """
    # Generate workflow using LLM
    workflow_desc = await llm_service.generate_workflow(prompt)
    
    # Create and return workflow in ReactFlow format
    workflow = await workflow_service.create_workflow(workflow_desc)
    return workflow.data

@router.post("/generate", response_model=GenerateFromPromptResponse)
async def generate_from_prompt(
    request: GenerateFromPromptRequest,
    llm_service: LLMService = Depends(),
    workflow_service: WorkflowService = Depends()
):
    """
    Generate a workflow from a natural language prompt
    
    Request body:
    {
        "prompt": "Create a customer onboarding workflow with email verification",
        "context": {
            "industry": "e-commerce",
            "complexity": "medium"
        },
        "options": {
            "include_error_handling": true,
            "optimize_for_performance": false
        }
    }
    """
    # Generate workflow using LLM with context
    workflow_desc = await llm_service.generate_workflow_with_context(
        prompt=request.prompt,
        context=request.context,
        options=request.options
    )
    
    # Validate the generated workflow
    validation_result = await workflow_service.validate_workflow(workflow_desc)
    
    if not validation_result.valid:
        # Try to fix common issues
        workflow_desc = await llm_service.fix_workflow_errors(
            workflow_desc, 
            validation_result.errors
        )
    
    # Create workflow
    workflow = await workflow_service.create_workflow(workflow_desc)
    
    return GenerateFromPromptResponse(
        workflow=workflow,
        generation_metadata={
            "model": "gpt-4",
            "prompt_tokens": request.prompt,
            "validation_attempts": 1,
            "success": True
        }
    )

@router.get("/templates", response_model=List[WorkflowTemplate])
async def get_workflow_templates():
    """Get pre-defined workflow templates"""
    return [
        WorkflowTemplate(
            id="customer-onboarding",
            name="Customer Onboarding",
            description="Complete customer onboarding with verification",
            category="sales",
            prompt="Create a customer onboarding workflow with email verification and data validation"
        ),
        WorkflowTemplate(
            id="order-processing",
            name="Order Processing",
            description="E-commerce order processing workflow",
            category="operations",
            prompt="Create an order processing workflow with inventory check and payment processing"
        ),
        # Add more templates
    ]

@router.post("/enhance/{workflow_id}")
async def enhance_workflow(
    workflow_id: UUID,
    enhancement_type: str,
    llm_service: LLMService = Depends(),
    workflow_service: WorkflowService = Depends()
):
    """
    Enhance an existing workflow using AI
    
    Enhancement types:
    - add_error_handling: Add error handling nodes
    - optimize_performance: Optimize workflow for performance
    - add_notifications: Add notification nodes
    - add_logging: Add logging/audit nodes
    """
    # Get existing workflow
    workflow = await workflow_service.get_workflow(workflow_id)
    if not workflow:
        raise HTTPException(status_code=404, detail="Workflow not found")
    
    # Enhance workflow using LLM
    enhanced_workflow = await llm_service.enhance_workflow(
        workflow=workflow,
        enhancement_type=enhancement_type
    )
    
    # Update workflow
    return await workflow_service.update_workflow_data(workflow_id, enhanced_workflow)
```

### Validation and Layout Endpoints

```python
# app/api/v1/validation.py
from fastapi import APIRouter, Depends
from app.schemas.workflow import WorkflowCreate
from app.schemas.validation import ValidationResult
from app.services.validation_service import ValidationService

router = APIRouter(prefix="/api", tags=["validation"])

@router.post("/workflows/validate", response_model=ValidationResult)
async def validate_workflow(
    workflow: WorkflowCreate,
    validation_service: ValidationService = Depends()
):
    """
    Validate a workflow structure
    
    Request body: WorkflowCreate schema
    
    Response:
    {
        "valid": true/false,
        "errors": [
            {
                "node_id": "action-1",
                "field": "properties.apiUrl",
                "message": "Invalid URL format",
                "code": "INVALID_URL"
            }
        ],
        "warnings": [
            {
                "message": "Workflow has no error handling",
                "code": "NO_ERROR_HANDLING"
            }
        ]
    }
    """
    return await validation_service.validate_workflow(workflow)

@router.post("/workflows/layout", response_model=List[NodeWithPosition])
async def calculate_layout(
    nodes: List[NodeCreate],
    edges: List[EdgeCreate],
    layout_algorithm: str = "hierarchical",
    layout_service: LayoutService = Depends()
):
    """
    Calculate optimal positions for nodes
    
    Layout algorithms:
    - hierarchical: Top-down hierarchical layout
    - force: Force-directed layout
    - circular: Circular layout
    - grid: Grid-based layout
    """
    return await layout_service.calculate_positions(
        nodes=nodes,
        edges=edges,
        algorithm=layout_algorithm
    )
```

### Import/Export Endpoints

```python
# app/api/v1/import_export.py
from fastapi import APIRouter, Depends, File, UploadFile, HTTPException
from fastapi.responses import FileResponse
from app.schemas.workflow import WorkflowResponse
from app.services.import_export_service import ImportExportService

router = APIRouter(prefix="/api/workflows", tags=["import-export"])

@router.post("/import", response_model=WorkflowResponse)
async def import_workflow(
    file: UploadFile = File(...),
    format: str = "json",
    service: ImportExportService = Depends()
):
    """
    Import a workflow from file
    
    Supported formats:
    - json: Workflow Builder JSON format
    - yaml: YAML format
    - bpmn: BPMN 2.0 XML format
    - mermaid: Mermaid diagram format
    """
    content = await file.read()
    
    try:
        workflow = await service.import_workflow(
            content=content,
            format=format,
            filename=file.filename
        )
        return workflow
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/{workflow_id}/export")
async def export_workflow(
    workflow_id: UUID,
    format: str = "json",
    service: ImportExportService = Depends()
):
    """
    Export a workflow to various formats
    
    Supported formats:
    - json: Workflow Builder JSON format
    - yaml: YAML format
    - bpmn: BPMN 2.0 XML format
    - mermaid: Mermaid diagram format
    - png: PNG image (requires additional setup)
    - pdf: PDF document (requires additional setup)
    """
    file_path, filename = await service.export_workflow(
        workflow_id=workflow_id,
        format=format
    )
    
    return FileResponse(
        path=file_path,
        filename=filename,
        media_type=service.get_media_type(format)
    )
```

## Complete API Endpoint Summary

### Base URL: `http://localhost:8000`

## API Endpoints with Time Estimates

| Method | Endpoint | Description | Request Body | Response | Dev Hours | Notes |
|--------|----------|-------------|--------------|----------|-----------|-------|
| **Palette** |
| GET | `/api/diagram/palette` | Get available node types | - | List of PaletteItem | **4 hours** | Simple endpoint returning static node type definitions |
| **Workflows** |
| POST | `/api/workflows` | Create new workflow | WorkflowCreate | WorkflowResponse | **6 hours** | Includes validation, ID generation, database persistence |
| GET | `/api/workflows` | List all workflows | - | List of WorkflowResponse | **4 hours** | Pagination, filtering, sorting support |
| GET | `/api/workflows/{id}` | Get specific workflow | - | WorkflowResponse | **3 hours** | Simple retrieval with error handling |
| PUT | `/api/workflows/{id}` | Update workflow metadata | WorkflowUpdate | WorkflowResponse | **5 hours** | Partial updates, validation, versioning |
| DELETE | `/api/workflows/{id}` | Delete workflow | - | Success message | **4 hours** | Cascade deletion, soft delete option |
| GET | `/api/diagram/diagram` | Get diagram (legacy) | - | ReactFlowData | **3 hours** | Legacy support, data transformation |
| POST | `/api/workflows/{id}/save` | Save workflow state | ReactFlowData | Success message | **8 hours** | Complex validation, state management, versioning |
| **Nodes** |
| POST | `/api/workflows/{id}/nodes` | Add node | NodeCreate | ReactFlowNode | **8 hours** | Schema validation, position calculation, ID generation |
| PUT | `/api/workflows/{id}/nodes/{nodeId}` | Update node | NodeUpdate | ReactFlowNode | **6 hours** | Property validation, connection updates |
| DELETE | `/api/workflows/{id}/nodes/{nodeId}` | Delete node | - | Success message | **5 hours** | Edge cleanup, validation of workflow integrity |
| POST | `/api/workflows/{id}/nodes/batch` | Add multiple nodes | List[NodeCreate] | List[ReactFlowNode] | **10 hours** | Batch processing, transaction handling, position optimization |
| **Edges** |
| POST | `/api/workflows/{id}/edges` | Add edge | EdgeCreate | ReactFlowEdge | **6 hours** | Connection validation, cycle detection |
| PUT | `/api/workflows/{id}/edges/{edgeId}` | Update edge | EdgeUpdate | ReactFlowEdge | **5 hours** | Reconnection logic, validation |
| DELETE | `/api/workflows/{id}/edges/{edgeId}` | Delete edge | - | Success message | **4 hours** | Workflow integrity checks |
| POST | `/api/workflows/{id}/edges/batch` | Add multiple edges | List[EdgeCreate] | List[ReactFlowEdge] | **8 hours** | Batch validation, complex graph operations |
| **Generator/LLM** |
| GET | `/api/generator/llm/{prompt}` | Generate from prompt (legacy) | - | ReactFlowData | **12 hours** | LLM integration, prompt engineering, response parsing |
| POST | `/api/generator/generate` | Generate from prompt | GenerateRequest | GenerateResponse | **16 hours** | Advanced LLM integration, template matching, validation |
| GET | `/api/generator/templates` | Get workflow templates | - | List[WorkflowTemplate] | **4 hours** | Template storage and retrieval |
| POST | `/api/generator/enhance/{id}` | Enhance workflow with AI | Enhancement type | WorkflowResponse | **20 hours** | Complex AI logic, workflow analysis, optimization |
| **Validation & Layout** |
| POST | `/api/workflows/validate` | Validate workflow | WorkflowCreate | ValidationResult | **10 hours** | Comprehensive validation rules, error reporting |
| POST | `/api/workflows/layout` | Calculate node positions | Nodes & Edges | List[NodeWithPosition] | **12 hours** | Graph layout algorithms, optimization |
| **Import/Export** |
| POST | `/api/workflows/import` | Import workflow | File upload | WorkflowResponse | **8 hours** | File parsing, format validation, data transformation |
| GET | `/api/workflows/{id}/export` | Export workflow | - | File download | **6 hours** | Multiple export formats, file generation |

## LLM Service Implementation

```python
# app/services/llm_service.py
from langchain import PromptTemplate, LLMChain
from langchain.chat_models import ChatOpenAI
from langchain.output_parsers import PydanticOutputParser
from app.schemas.workflow import WorkflowCreate
import json

class LLMService:
    def __init__(self, openai_api_key: str):
        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0.7,
            openai_api_key=openai_api_key
        )
        self.parser = PydanticOutputParser(pydantic_object=WorkflowCreate)
    
    async def generate_workflow(self, prompt: str) -> WorkflowCreate:
        """Generate a workflow from a natural language prompt"""
        
        template = """
        You are an expert workflow designer. Create a workflow based on this description:
        {prompt}
        
        Available node types:
        - trigger: Initiates the workflow (manual, webhook, schedule, event)
        - action: Performs operations (api call, email, database operations)
        - decision: Conditional branching
        - delay: Time-based delays
        - notification: Send notifications
        
        {format_instructions}
        
        Important rules:
        1. Every workflow must start with at least one trigger node
        2. All nodes except triggers must be connected
        3. Use meaningful labels for nodes
        4. Include appropriate properties for each node type
        5. Generate unique IDs for nodes (format: nodeType-timestamp)
        
        Create a complete, valid workflow.
        """
        
        prompt_template = PromptTemplate(
            template=template,
            input_variables=["prompt"],
            partial_variables={"format_instructions": self.parser.get_format_instructions()}
        )
        
        chain = LLMChain(llm=self.llm, prompt=prompt_template)
        result = await chain.arun(prompt=prompt)
        
        # Parse the result
        try:
            workflow = self.parser.parse(result)
            return workflow
        except Exception as e:
            # Fallback: try to extract JSON from the response
            import re
            json_match = re.search(r'\{.*\}', result, re.DOTALL)
            if json_match:
                workflow_dict = json.loads(json_match.group())
                return WorkflowCreate(**workflow_dict)
            raise ValueError(f"Failed to parse LLM response: {e}")
    
    async def generate_workflow_with_context(
        self, 
        prompt: str, 
        context: dict, 
        options: dict
    ) -> WorkflowCreate:
        """Generate workflow with additional context and options"""
        
        enhanced_prompt = f"""
        {prompt}
        
        Context:
        - Industry: {context.get('industry', 'general')}
        - Complexity: {context.get('complexity', 'medium')}
        - Users: {context.get('target_users', 'general users')}
        
        Requirements:
        - Include error handling: {options.get('include_error_handling', False)}
        - Optimize for performance: {options.get('optimize_for_performance', False)}
        - Add logging: {options.get('add_logging', False)}
        """
        
        return await self.generate_workflow(enhanced_prompt)
    
    async def fix_workflow_errors(
        self, 
        workflow: WorkflowCreate, 
        errors: List[ValidationError]
    ) -> WorkflowCreate:
        """Attempt to fix validation errors in a workflow"""
        
        error_summary = "\n".join([f"- {e.message}" for e in errors])
        
        fix_prompt = f"""
        The following workflow has validation errors:
        
        {json.dumps(workflow.dict(), indent=2)}
        
        Errors:
        {error_summary}
        
        Please fix these errors and return a valid workflow.
        Maintain the original intent but ensure all validation rules are met.
        """
        
        return await self.generate_workflow(fix_prompt)
    
    async def enhance_workflow(
        self, 
        workflow: WorkflowResponse, 
        enhancement_type: str
    ) -> ReactFlowData:
        """Enhance an existing workflow"""
        
        enhancement_prompts = {
            "add_error_handling": """
            Add comprehensive error handling to this workflow:
            - Add decision nodes to check for errors after each action
            - Add notification nodes for error alerts
            - Add retry logic where appropriate
            """,
            "optimize_performance": """
            Optimize this workflow for performance:
            - Add parallel execution where possible
            - Remove unnecessary delays
            - Combine redundant actions
            """,
            "add_notifications": """
            Add appropriate notifications to this workflow:
            - Add status notifications at key milestones
            - Add completion notifications
            - Add failure notifications
            """,
            "add_logging": """
            Add logging and audit trail to this workflow:
            - Add action nodes to log important events
            - Add nodes to track user actions
            - Add nodes for compliance logging
            """
        }
        
        prompt = enhancement_prompts.get(enhancement_type, "Enhance this workflow")
        
        enhance_prompt = f"""
        {prompt}
        
        Current workflow:
        {json.dumps(workflow.data.dict(), indent=2)}
        
        Return the enhanced workflow maintaining all existing functionality.
        """
        
        enhanced = await self.generate_workflow(enhance_prompt)
        
        # Merge with existing workflow
        return self._merge_workflows(workflow.data, enhanced)
```

## Workflow Service Implementation

```python
# app/services/workflow_service.py
from typing import List, Optional
from uuid import UUID
from sqlalchemy.orm import Session
from app.models import Workflow, Node, Edge
from app.schemas.workflow import WorkflowCreate, WorkflowUpdate, WorkflowResponse
from app.services.validation_service import ValidationService
from app.services.layout_service import LayoutService
from app.utils.converters import convert_to_reactflow

class WorkflowService:
    def __init__(
        self, 
        db: Session,
        validation_service: ValidationService,
        layout_service: LayoutService
    ):
        self.db = db
        self.validation_service = validation_service
        self.layout_service = layout_service
    
    async def create_workflow(self, workflow_data: WorkflowCreate) -> WorkflowResponse:
        """Create a new workflow with validation and layout"""
        
        # Validate workflow
        validation_result = await self.validation_service.validate_workflow(workflow_data)
        if not validation_result.valid:
            raise ValueError(f"Invalid workflow: {validation_result.errors}")
        
        # Calculate positions for nodes without positions
        positioned_nodes = await self.layout_service.calculate_positions(
            workflow_data.nodes,
            workflow_data.edges
        )
        
        # Create workflow in database
        workflow = Workflow(
            name=workflow_data.name,
            description=workflow_data.description,
            metadata=workflow_data.metadata
        )
        self.db.add(workflow)
        self.db.flush()
        
        # Create nodes
        for node_data in positioned_nodes:
            node = Node(
                id=node_data.id or f"{node_data.type}-{workflow.id}-{positioned_nodes.index(node_data)}",
                workflow_id=workflow.id,
                type=node_data.type,
                label=node_data.label,
                position_x=node_data.position.x,
                position_y=node_data.position.y,
                properties=node_data.properties,
                metadata=node_data.metadata
            )
            self.db.add(node)
        
        # Create edges
        for edge_data in workflow_data.edges:
            edge = Edge(
                id=edge_data.id or f"edge-{workflow.id}-{workflow_data.edges.index(edge_data)}",
                workflow_id=workflow.id,
                source=edge_data.source,
                target=edge_data.target,
                type=edge_data.type,
                label=edge_data.label,
                data=edge_data.data
            )
            self.db.add(edge)
        
        self.db.commit()
        self.db.refresh(workflow)
        
        # Convert to response format
        return self._workflow_to_response(workflow)
    
    async def get_workflow(self, workflow_id: UUID) -> Optional[WorkflowResponse]:
        """Get a workflow by ID"""
        workflow = self.db.query(Workflow).filter(Workflow.id == workflow_id).first()
        if not workflow:
            return None
        return self._workflow_to_response(workflow)
    
    async def list_workflows(
        self, 
        skip: int = 0, 
        limit: int = 100,
        status: Optional[str] = None
    ) -> List[WorkflowResponse]:
        """List workflows with pagination and filtering"""
        query = self.db.query(Workflow)
        
        if status:
            query = query.filter(Workflow.status == status)
        
        workflows = query.offset(skip).limit(limit).all()
        return [self._workflow_to_response(w) for w in workflows]
    
    async def update_workflow(
        self, 
        workflow_id: UUID, 
        update_data: WorkflowUpdate
    ) -> Optional[WorkflowResponse]:
        """Update workflow metadata"""
        workflow = self.db.query(Workflow).filter(Workflow.id == workflow_id).first()
        if not workflow:
            return None
        
        # Update fields
        if update_data.name is not None:
            workflow.name = update_data.name
        if update_data.description is not None:
            workflow.description = update_data.description
        if update_data.status is not None:
            workflow.status = update_data.status
        if update_data.metadata is not None:
            workflow.metadata = update_data.metadata
        
        self.db.commit()
        self.db.refresh(workflow)
        
        return self._workflow_to_response(workflow)
    
    async def delete_workflow(self, workflow_id: UUID) -> bool:
        """Delete a workflow and all its nodes/edges"""
        workflow = self.db.query(Workflow).filter(Workflow.id == workflow_id).first()
        if not workflow:
            return False
        
        self.db.delete(workflow)
        self.db.commit()
        return True
    
    async def add_node(self, workflow_id: UUID, node_data: NodeCreate) -> Optional[ReactFlowNode]:
        """Add a node to an existing workflow"""
        workflow = self.db.query(Workflow).filter(Workflow.id == workflow_id).first()
        if not workflow:
            return None
        
        # Validate node
        errors = await self.validation_service._validate_node(node_data)
        if errors:
            raise ValueError(f"Invalid node: {errors}")
        
        # Calculate position if not provided
        if not node_data.position:
            existing_nodes = [
                NodeCreate(
                    id=n.id,
                    type=n.type,
                    label=n.label,
                    position=Position(x=n.position_x, y=n.position_y),
                    properties=n.properties
                ) for n in workflow.nodes
            ]
            existing_nodes.append(node_data)
            
            positioned = await self.layout_service.calculate_positions(
                existing_nodes,
                []  # Empty edges for position calculation
            )
            node_data = positioned[-1]
        
        # Create node
        node = Node(
            id=node_data.id or f"{node_data.type}-{workflow_id}-{len(workflow.nodes)}",
            workflow_id=workflow_id,
            type=node_data.type,
            label=node_data.label,
            position_x=node_data.position.x,
            position_y=node_data.position.y,
            properties=node_data.properties,
            metadata=node_data.metadata
        )
        
        self.db.add(node)
        self.db.commit()
        self.db.refresh(node)
        
        return convert_to_reactflow_node(node)
    
    def _workflow_to_response(self, workflow: Workflow) -> WorkflowResponse:
        """Convert database workflow to response format"""
        nodes = [convert_to_reactflow_node(n) for n in workflow.nodes]
        edges = [convert_to_reactflow_edge(e) for e in workflow.edges]
        
        return WorkflowResponse(
            id=str(workflow.id),
            name=workflow.name,
            description=workflow.description,
            created_at=workflow.created_at,
            updated_at=workflow.updated_at,
            status=workflow.status,
            data=ReactFlowData(
                nodes=nodes,
                edges=edges,
                viewport={"x": 0, "y": 0, "zoom": 1}
            ),
            metadata=workflow.metadata
        )
```

## Deployment Configuration

### Docker Setup

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Run migrations and start server
CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: workflow_user
      POSTGRES_PASSWORD: workflow_pass
      POSTGRES_DB: workflow_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://workflow_user:workflow_pass@postgres:5432/workflow_db
      REDIS_URL: redis://redis:6379
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    depends_on:
      - postgres
      - redis
    volumes:
      - ./app:/app/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

volumes:
  postgres_data:
```

### Requirements.txt

```txt
# requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
sqlalchemy==2.0.23
alembic==1.12.1
psycopg2-binary==2.9.9
redis==5.0.1
python-multipart==0.0.6
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
langchain==0.0.340
openai==1.3.5
networkx==3.2.1
pyyaml==6.0.1
httpx==0.25.2
pytest==7.4.3
pytest-asyncio==0.21.1
```

## Testing Strategy

### Unit Tests Example

```python
# tests/test_workflow_service.py
import pytest
from app.services.workflow_service import WorkflowService
from app.schemas.workflow import WorkflowCreate, NodeCreate

@pytest.mark.asyncio
async def test_create_workflow(workflow_service: WorkflowService):
    """Test workflow creation"""
    workflow_data = WorkflowCreate(
        name="Test Workflow",
        description="Test description",
        nodes=[
            NodeCreate(
                type="trigger",
                label="Start",
                properties={"triggerType": "manual"}
            ),
            NodeCreate(
                type="action",
                label="Process",
                properties={"actionType": "transform"}
            )
        ],
        edges=[
            EdgeCreate(
                source="trigger-0",
                target="action-1"
            )
        ]
    )
    
    result = await workflow_service.create_workflow(workflow_data)
    
    assert result.name == "Test Workflow"
    assert len(result.data.nodes) == 2
    assert len(result.data.edges) == 1
    assert result.data.nodes[0].position.x is not None

@pytest.mark.asyncio
async def test_validate_workflow_missing_trigger(validation_service):
    """Test validation catches missing trigger"""
    workflow_data = WorkflowCreate(
        name="Invalid Workflow",
        nodes=[
            NodeCreate(
                type="action",
                label="Action Only",
                properties={"actionType": "transform"}
            )
        ],
        edges=[]
    )
    
    result = await validation_service.validate_workflow(workflow_data)
    
    assert not result.valid
    assert any(e.code == "NO_TRIGGER_NODE" for e in result.errors)
```

## Conclusion

This implementation plan provides a complete Python-based REST API for the Workflow Builder application. The API includes:

1. **Complete CRUD operations** for workflows, nodes, and edges
2. **LLM integration** for natural language workflow generation
3. **Validation and layout services** for ensuring workflow integrity
4. **Import/export capabilities** for various formats
5. **Docker deployment** configuration
6. **Comprehensive testing** strategy

The Python implementation leverages:
- FastAPI for high-performance async APIs
- Pydantic for robust data validation
- SQLAlchemy for database operations
- NetworkX for graph algorithms
- LangChain for LLM integration

This provides a solid foundation for the development team to build upon.
