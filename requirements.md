### **Real-Time TODO Application: Comprehensive Technical Requirements & Design Context**

**Version 1.0**

### **1.0 System Overview & Core Strategy**

This document provides the complete technical requirements and architectural context for building a real-time collaborative TODO application with React frontend, Go backend, and PostgreSQL database. The system enables users to create, manage, and share TODO lists with real-time synchronization across multiple devices.

*   **1.1. Core Messaging Strategy**: The system **will use WebSocket connections for real-time updates**, with REST API fallback for offline changes. The backend service will be responsible for all state management and synchronization; clients will maintain local caches for offline support.
*   **1.2. Rationale**: WebSockets provide low-latency bidirectional communication essential for real-time collaboration. The REST fallback ensures reliability when WebSocket connections are unavailable.
*   **1.3. High-Level Goal**: To create a scalable, responsive, and reliable TODO management system that supports real-time collaboration, offline functionality, and seamless synchronization across web and mobile clients.

### **2.0 Core Architecture & Components**

*   **2.1. Backend API Service (New Component)**
    *   **Description**: A secure, high-performance backend service (Go + Gin/Echo framework) must be created to act as the central authority for all TODO data and real-time synchronization. This service will expose both REST and WebSocket APIs.
    *   **Data Model**: The service will maintain the following core entities:
        - **User**: user_id (UUID), email, username, created_at, last_seen
        - **List**: list_id (UUID), owner_id, name, description, is_shared, created_at, updated_at
        - **Todo**: todo_id (UUID), list_id, title, description, status, priority, due_date, assigned_to, created_by, created_at, updated_at
        - **Share**: share_id (UUID), list_id, user_id, permission_level, created_at
        - **Activity**: activity_id (UUID), entity_type, entity_id, user_id, action, metadata (JSONB), created_at
    *   **API Endpoints**: All endpoints must be secured via JWT authentication
        - `POST /api/auth/register`: User registration
        - `POST /api/auth/login`: User authentication
        - `GET /api/lists`: Get user's lists (owned and shared)
        - `POST /api/lists`: Create new list
        - `PUT /api/lists/:id`: Update list
        - `DELETE /api/lists/:id`: Delete list
        - `GET /api/lists/:id/todos`: Get todos in list
        - `POST /api/lists/:id/todos`: Create todo
        - `PUT /api/todos/:id`: Update todo
        - `DELETE /api/todos/:id`: Delete todo
        - `POST /api/lists/:id/share`: Share list with user
        - `WS /api/ws`: WebSocket endpoint for real-time updates
*   **2.2. Client-Side Architecture**
    *   **Description**: React web application with Redux for state management and Socket.io for real-time communication.
    *   **States**:
        - INITIALIZING: App loading, not authenticated
        - ONLINE: Connected to backend, real-time sync active
        - OFFLINE: No connection, operating with local cache
        - SYNCING: Reconnected, synchronizing offline changes
    *   **State Transition Matrix**:
        - **From INITIALIZING**:
            - → ONLINE: On successful authentication and WebSocket connection
            - → OFFLINE: On connection failure (operate with cached data)
        - **From ONLINE**:
            - → OFFLINE: On WebSocket disconnect or network error
            - → ONLINE: Maintain state on successful operations
        - **From OFFLINE**:
            - → SYNCING: On network restoration
            - → ONLINE: After successful sync completion
        - **From SYNCING**:
            - → ONLINE: On successful sync
            - → OFFLINE: On sync failure
    *   **Offline Support**: Client apps **must** use IndexedDB to cache lists and todos for offline access
    *   **Conflict Resolution**: Last-write-wins with activity history for audit
*   **2.3. Real-Time Synchronization Architecture**
    *   **Description**: The system employs WebSocket connections for instant updates with automatic reconnection.
    *   **Update Types**:
        - **List Updates**: Creation, modification, deletion, sharing
        - **Todo Updates**: All CRUD operations on todos
        - **Presence Updates**: Show active users in shared lists
        - **Notification Events**: Due date reminders, mentions, shares
    *   **Sync Intervals**:
        - **WebSocket Heartbeat**: 30 seconds (keep-alive)
        - **Reconnection Backoff**: 1s, 2s, 4s, 8s, 16s, 30s (max)
        - **Offline Sync Check**: On app focus or network change
    *   **Design Principles**:
        - **Optimistic Updates**: Apply changes locally immediately
        - **Eventual Consistency**: Sync when connection available
        - **Conflict Visibility**: Show conflicts in activity feed

### **3.0 Functional Requirements (FR)**

*   **FR-1: User Management**
    *   **FR-1.1**: Users **must** register with email and password
    *   **FR-1.2**: System **must** support JWT-based authentication with refresh tokens
    *   **FR-1.3**: Tokens **must** expire after 24 hours with 7-day refresh token validity
*   **FR-2: List Management**
    *   **FR-2.1**: Users **must** be able to create multiple TODO lists
    *   **FR-2.2**: Lists **must** support title, description, and color coding
    *   **FR-2.3**: Users **must** be able to share lists with view/edit permissions
    *   **FR-2.4**: Shared list changes **must** propagate in real-time to all participants
*   **FR-3: TODO Operations**
    *   **FR-3.1**: TODOs **must** support title, description, status (pending/completed), priority (low/medium/high)
    *   **FR-3.2**: TODOs **must** support optional due dates with timezone handling
    *   **FR-3.3**: TODOs **must** be assignable to shared list participants
    *   **FR-3.4**: Completed TODOs **must** remain visible with strikethrough
*   **FR-4: Real-Time Synchronization**
    *   **FR-4.1**: All changes **must** be broadcast via WebSocket to connected clients within 100ms
    *   **FR-4.2**: Clients **must** implement optimistic updates with rollback on failure
    *   **FR-4.3**: System **must** queue offline changes and sync on reconnection
    *   **FR-4.4**: Sync conflicts **must** be resolved using last-write-wins with full history
*   **FR-5: Search & Filtering**
    *   **FR-5.1**: Users **must** be able to search todos by title and description
    *   **FR-5.2**: System **must** support filtering by status, priority, assignee, and due date
    *   **FR-5.3**: Search **must** work across all accessible lists (owned and shared)
*   **FR-6: Activity Tracking**
    *   **FR-6.1**: System **must** log all todo and list modifications
    *   **FR-6.2**: Activity feed **must** show who made changes and when
    *   **FR-6.3**: Activity history **must** be retained for 30 days
*   **FR-7: Data Persistence**
    *   **FR-7.1**: All data **must** be persisted to PostgreSQL with ACID guarantees
    *   **FR-7.2**: Database **must** use UUID primary keys for all entities
    *   **FR-7.3**: Soft deletes **must** be implemented for lists and todos
*   **FR-8: API Design**
    *   **FR-8.1**: REST API **must** follow RESTful conventions
    *   **FR-8.2**: All responses **must** use consistent JSON structure
    *   **FR-8.3**: Errors **must** include error codes and user-friendly messages
    *   **FR-8.4**: API **must** support pagination for list operations

### **4.0 Non-Functional Requirements (NFR)**

*   **NFR-1: Performance (PERF)**
    *   **PERF-1.1**: P95 API response time **must be < 200ms** for CRUD operations
    *   **PERF-1.2**: WebSocket message delivery **must be < 100ms** P95
    *   **PERF-1.3**: Initial page load **must be < 2 seconds** on 3G networks
*   **NFR-2: Reliability (REL)**
    *   **REL-2.1**: 99.9% uptime during business hours
    *   **REL-2.2**: Zero data loss for committed transactions
    *   **REL-2.3**: Automatic reconnection with exponential backoff
    *   **REL-2.4**: Graceful degradation to read-only mode during outages
*   **NFR-3: Scale**
    *   **NFR-3.1**: Support 100,000 active users
    *   **NFR-3.2**: Support 10 million todos
    *   **NFR-3.3**: Handle 1,000 concurrent WebSocket connections per server
*   **NFR-4: Security & Privacy**
    *   **NFR-4.1**: All API endpoints **must** require authentication (except auth endpoints)
    *   **NFR-4.2**: Users **must** only access their own and explicitly shared data
    *   **NFR-4.3**: Passwords **must** be hashed with bcrypt (cost factor 12)
    *   **NFR-4.4**: HTTPS **must** be enforced for all connections
    *   **NFR-4.5**: Rate limiting **must** be implemented (100 requests/minute per user)
*   **NFR-5: Monitoring & Operations**
    *   **NFR-5.1**: Application **must** export Prometheus metrics
    *   **NFR-5.2**: Structured logging **must** use JSON format
    *   **NFR-5.3**: Distributed tracing **must** be implemented with OpenTelemetry
    *   **NFR-5.4**: Database migrations **must** be versioned and reversible
    *   **NFR-5.5**: Health check endpoints **must** be provided

### **5.0 Risk Assessment & Mitigation Plan**

*   **Risk 1: Network Partition**
    *   **Scenario**: Client loses connection during collaborative editing
    *   **Impact**: Potential data conflicts or loss
    *   **Mitigation**: Offline queue with conflict resolution on reconnect
*   **Risk 2: Scaling Bottleneck**
    *   **Scenario**: WebSocket server reaches connection limit
    *   **Impact**: New users cannot connect for real-time updates
    *   **Mitigation**: Horizontal scaling with Redis PubSub for cross-server communication
*   **Risk 3: Data Conflicts**
    *   **Scenario**: Multiple users edit same todo simultaneously offline
    *   **Impact**: One user's changes may be overwritten
    *   **Mitigation**: Activity history shows all changes; future: operational transforms

### **6.0 Technology Stack Summary**

| Component | Technology | Rationale |
|-----------|------------|-----------|
| Frontend | React + TypeScript | Type safety, component reusability |
| State Mgmt | Redux Toolkit | Predictable state, DevTools support |
| UI Framework | Material-UI | Consistent design, accessibility |
| Backend | Go + Gin | Performance, simplicity, concurrency |
| Database | PostgreSQL 15 | ACID, JSON support, proven reliability |
| Cache | Redis | Session storage, PubSub for scaling |
| Real-time | WebSockets | Low latency, bidirectional |
| Auth | JWT | Stateless, scalable |
| Monitoring | Prometheus + Grafana | Industry standard, Go integration |

### **7.0 Database Schema Overview**

```sql
-- Core tables with essential fields
users (user_id UUID PK, email, username, password_hash, created_at, updated_at)
lists (list_id UUID PK, owner_id FK, name, description, color, is_archived, created_at, updated_at)
todos (todo_id UUID PK, list_id FK, title, description, status, priority, due_date, assigned_to FK, created_by FK, position, created_at, updated_at, completed_at)
shares (share_id UUID PK, list_id FK, user_id FK, permission, created_at)
activities (activity_id UUID PK, entity_type, entity_id, user_id FK, action, metadata JSONB, created_at)
sessions (session_id UUID PK, user_id FK, token_hash, expires_at, created_at)
```

### **8.0 API Response Format**

```json
// Success Response
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2025-01-07T10:00:00Z",
    "version": "1.0"
  }
}

// Error Response
{
  "success": false,
  "error": {
    "code": "LIST_NOT_FOUND",
    "message": "The requested list does not exist",
    "details": { "list_id": "..." }
  },
  "meta": { ... }
}
```

### **9.0 WebSocket Message Format**

```json
{
  "type": "TODO_UPDATED",
  "payload": {
    "list_id": "...",
    "todo": { ... },
    "updated_by": "user_id",
    "timestamp": "2025-01-07T10:00:00Z"
  },
  "correlation_id": "..."
}
```

### **10.0 Implementation Priorities**

1. **Phase 1**: Core CRUD operations, authentication, basic web UI
2. **Phase 2**: Real-time sync, sharing, activity tracking
3. **Phase 3**: Offline support, mobile apps, advanced search
4. **Phase 4**: Reminders, recurring todos, team features