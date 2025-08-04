# Backend Development Prompt for PBX Voice System

## Project Context
You are building a complete PBX (Private Branch Exchange) backend system using Go with Fiber framework. This backend will integrate with Freeswitch to provide a full-featured voice communication system similar to GoToConnect. The system must support multitenancy, user management, call routing, and all telephony features.

## Technology Stack
- **Backend**: Go 1.22+ with Fiber framework
- **Database**: PostgreSQL with GORM
- **Cache**: Redis for sessions and real-time data
- **Telephony**: Freeswitch with mod_xml_curl and ESL integration
- **Authentication**: JWT tokens
- **Logging**: Structured logging with OpenSearch integration
- **Containerization**: Docker with docker-compose

## Core Requirements

### 1. Database Schema Design
Create a comprehensive database schema that includes:

#### Users & Authentication
- Users table (id, tenant_id, username, email, password_hash, role, status, created_at, updated_at)
- User profiles (user_id, first_name, last_name, phone_number, extension, department, avatar_url)
- User sessions (id, user_id, token, expires_at, created_at)
- User permissions (id, user_id, permission, created_at)

#### Tenants & Multitenancy
- Tenants table (id, name, domain, status, plan, settings, created_at, updated_at)
- Tenant configurations (tenant_id, key, value, created_at)

#### Extensions & SIP Configuration
- Extensions table (id, tenant_id, user_id, extension_number, sip_username, sip_password, status, created_at, updated_at)
- Extension settings (extension_id, setting_key, setting_value, created_at)
- SIP domains (id, tenant_id, domain_name, status, created_at)

#### Call Management
- Calls table (id, tenant_id, call_uuid, caller_extension, callee_extension, direction, status, duration, recording_url, created_at, updated_at)
- Call logs (id, call_id, event_type, event_data, timestamp)
- Call recordings (id, call_id, file_path, duration, file_size, created_at)

#### Dialplan & Routing
- Dialplan rules (id, tenant_id, name, pattern, destination, priority, enabled, created_at)
- Outbound routes (id, tenant_id, name, pattern, gateway_id, enabled, created_at)
- Inbound routes (id, tenant_id, name, did_number, destination, enabled, created_at)
- Gateways (id, tenant_id, name, gateway_type, host, port, username, password, enabled, created_at)

#### Voicemail & Features
- Voicemail boxes (id, tenant_id, extension_id, pin, greeting_url, created_at)
- Voicemail messages (id, voicemail_id, caller_id, duration, file_path, read, created_at)
- IVR menus (id, tenant_id, name, greeting, options, created_at)
- Conference rooms (id, tenant_id, name, pin, max_participants, created_at)

#### System Configuration
- System settings (id, tenant_id, key, value, created_at, updated_at)
- API keys (id, tenant_id, name, key_hash, permissions, expires_at, created_at)
- Audit logs (id, tenant_id, user_id, action, resource, details, ip_address, created_at)

### 2. Core Backend Structure

#### Project Structure
```
backend/
├── cmd/
│   └── main.go                 # Application entry point
├── config/
│   ├── config.go               # Configuration management
│   ├── database.go             # Database configuration
│   └── freeswitch.go           # Freeswitch configuration
├── controllers/
│   ├── auth_controller.go      # Authentication endpoints
│   ├── user_controller.go      # User management
│   ├── extension_controller.go # Extension management
│   ├── call_controller.go      # Call management
│   ├── dialplan_controller.go  # Dialplan management
│   ├── voicemail_controller.go # Voicemail management
│   ├── conference_controller.go # Conference management
│   └── system_controller.go    # System management
├── services/
│   ├── auth_service.go         # Authentication logic
│   ├── user_service.go         # User management logic
│   ├── extension_service.go    # Extension management logic
│   ├── call_service.go         # Call management logic
│   ├── freeswitch_service.go   # Freeswitch integration
│   ├── dialplan_service.go     # Dialplan logic
│   ├── voicemail_service.go    # Voicemail logic
│   ├── conference_service.go   # Conference logic
│   └── notification_service.go # Notification logic
├── models/
│   ├── user.go                 # User model
│   ├── tenant.go               # Tenant model
│   ├── extension.go            # Extension model
│   ├── call.go                 # Call model
│   ├── dialplan.go             # Dialplan model
│   ├── voicemail.go            # Voicemail model
│   └── conference.go           # Conference model
├── middleware/
│   ├── auth.go                 # JWT authentication
│   ├── cors.go                 # CORS handling
│   ├── logging.go              # Request logging
│   ├── rate_limit.go           # Rate limiting
│   └── tenant.go               # Tenant isolation
├── routes/
│   ├── auth_routes.go          # Authentication routes
│   ├── user_routes.go          # User routes
│   ├── extension_routes.go     # Extension routes
│   ├── call_routes.go          # Call routes
│   ├── dialplan_routes.go      # Dialplan routes
│   ├── voicemail_routes.go     # Voicemail routes
│   ├── conference_routes.go    # Conference routes
│   └── system_routes.go        # System routes
├── database/
│   ├── connection.go           # Database connection
│   ├── migrations.go           # Database migrations
│   └── seeds.go                # Database seeds
├── utils/
│   ├── jwt.go                  # JWT utilities
│   ├── password.go             # Password hashing
│   ├── validation.go           # Input validation
│   ├── logger.go               # Logging utilities
│   └── helpers.go              # General utilities
├── freeswitch/
│   ├── client.go               # Freeswitch client
│   ├── esl.go                  # Event Socket Library
│   ├── xml_curl.go             # XML curl integration
│   ├── dialplan.go             # Dialplan generation
│   └── events.go               # Event handling
├── websocket/
│   ├── manager.go              # WebSocket manager
│   ├── handlers.go             # WebSocket handlers
│   └── events.go               # Real-time events
└── tests/
    ├── unit/                   # Unit tests
    ├── integration/            # Integration tests
    └── fixtures/               # Test data
```

### 3. Freeswitch Integration Requirements

#### mod_xml_curl Integration
- Dynamic user registration and authentication
- Real-time extension status updates
- Dynamic dialplan generation
- Call routing and management
- Voicemail integration
- Conference room management

#### ESL (Event Socket Library) Integration
- Real-time call events (start, answer, hangup, transfer)
- Extension registration status
- Call quality monitoring
- System health monitoring
- Custom event handling

#### Core Freeswitch Features to Implement
- **User Registration**: SIP user registration and authentication
- **Call Routing**: Inbound and outbound call routing
- **Dialplan**: Dynamic dialplan generation for each tenant
- **Voicemail**: Voicemail system with greetings and messages
- **Conference**: Conference room creation and management
- **Recording**: Call recording and storage
- **IVR**: Interactive Voice Response system
- **Call Transfer**: Blind and attended transfer
- **Call Forwarding**: Forward calls to different destinations
- **Call Queuing**: Queue management for incoming calls
- **Call Monitoring**: Real-time call monitoring and statistics
- **Gateway Management**: SIP gateway configuration
- **DID Management**: Direct Inward Dialing
- **Call History**: Complete call history and logs
- **Call Analytics**: Call statistics and reporting

### 4. API Endpoints Required

#### Authentication
- `POST /api/v1/auth/login` - User login
- `POST /api/v1/auth/logout` - User logout
- `POST /api/v1/auth/refresh` - Refresh token
- `POST /api/v1/auth/register` - User registration

#### User Management
- `GET /api/v1/users` - List users
- `POST /api/v1/users` - Create user
- `GET /api/v1/users/:id` - Get user
- `PUT /api/v1/users/:id` - Update user
- `DELETE /api/v1/users/:id` - Delete user
- `PUT /api/v1/users/:id/password` - Change password

#### Extension Management
- `GET /api/v1/extensions` - List extensions
- `POST /api/v1/extensions` - Create extension
- `GET /api/v1/extensions/:id` - Get extension
- `PUT /api/v1/extensions/:id` - Update extension
- `DELETE /api/v1/extensions/:id` - Delete extension
- `POST /api/v1/extensions/:id/register` - Register extension
- `POST /api/v1/extensions/:id/unregister` - Unregister extension

#### Call Management
- `GET /api/v1/calls` - List calls
- `GET /api/v1/calls/:id` - Get call details
- `POST /api/v1/calls` - Initiate call
- `PUT /api/v1/calls/:id/hangup` - Hangup call
- `PUT /api/v1/calls/:id/transfer` - Transfer call
- `PUT /api/v1/calls/:id/hold` - Hold call
- `PUT /api/v1/calls/:id/unhold` - Unhold call
- `GET /api/v1/calls/:id/recording` - Get call recording

#### Dialplan Management
- `GET /api/v1/dialplan` - List dialplan rules
- `POST /api/v1/dialplan` - Create dialplan rule
- `GET /api/v1/dialplan/:id` - Get dialplan rule
- `PUT /api/v1/dialplan/:id` - Update dialplan rule
- `DELETE /api/v1/dialplan/:id` - Delete dialplan rule
- `POST /api/v1/dialplan/reload` - Reload dialplan

#### Voicemail Management
- `GET /api/v1/voicemail` - List voicemail boxes
- `POST /api/v1/voicemail` - Create voicemail box
- `GET /api/v1/voicemail/:id` - Get voicemail box
- `PUT /api/v1/voicemail/:id` - Update voicemail box
- `DELETE /api/v1/voicemail/:id` - Delete voicemail box
- `GET /api/v1/voicemail/:id/messages` - List messages
- `GET /api/v1/voicemail/messages/:id` - Get message
- `DELETE /api/v1/voicemail/messages/:id` - Delete message

#### Conference Management
- `GET /api/v1/conferences` - List conferences
- `POST /api/v1/conferences` - Create conference
- `GET /api/v1/conferences/:id` - Get conference
- `PUT /api/v1/conferences/:id` - Update conference
- `DELETE /api/v1/conferences/:id` - Delete conference
- `POST /api/v1/conferences/:id/join` - Join conference
- `POST /api/v1/conferences/:id/leave` - Leave conference

#### System Management
- `GET /api/v1/system/status` - System status
- `GET /api/v1/system/health` - Health check
- `GET /api/v1/system/logs` - System logs
- `POST /api/v1/system/restart` - Restart system
- `GET /api/v1/system/statistics` - System statistics

#### WebSocket Endpoints
- `GET /ws` - WebSocket connection for real-time events
- Events: call_start, call_end, extension_status, system_alert

### 5. Core Features Implementation

#### User Registration & Authentication
- JWT-based authentication
- Role-based access control
- Session management
- Password hashing with bcrypt
- Multi-factor authentication support

#### Extension Management
- SIP user registration
- Extension number assignment
- Extension status monitoring
- Extension settings management
- Bulk extension operations

#### Call Routing & Management
- Inbound call routing based on DID
- Outbound call routing through gateways
- Call transfer (blind and attended)
- Call forwarding
- Call recording
- Call monitoring

#### Dialplan System
- Dynamic dialplan generation
- Pattern matching for call routing
- Priority-based routing
- Tenant-specific dialplans
- Real-time dialplan updates

#### Voicemail System
- Voicemail box creation
- Greeting management
- Message storage and retrieval
- Email notifications
- Web interface for voicemail

#### Conference System
- Conference room creation
- PIN-based access
- Participant management
- Recording capabilities
- Conference statistics

#### Real-time Features
- WebSocket connections for real-time updates
- Call status updates
- Extension status updates
- System notifications
- Live call monitoring

### 6. Database Migrations

Create comprehensive database migrations that include:
- All table structures
- Proper indexes for performance
- Foreign key relationships
- Default data seeding
- Tenant isolation constraints

### 7. Configuration Management

#### Environment Variables
```env
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=pbx_voice_db
DB_USER=pbx_user
DB_PASSWORD=secure_password
DB_SSL_MODE=disable

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# Freeswitch
FREESWITCH_HOST=localhost
FREESWITCH_PORT=5060
FREESWITCH_ESL_PORT=8021
FREESWITCH_PASSWORD=ClueCon
FREESWITCH_XML_CURL_URL=http://localhost:8080/api/v1/freeswitch/xml_curl

# JWT
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRY=24h
JWT_REFRESH_EXPIRY=168h

# Server
PORT=8080
ENV=development
LOG_LEVEL=debug

# File Storage
UPLOAD_PATH=./uploads
MAX_FILE_SIZE=10485760

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

### 8. Security Requirements

#### Authentication & Authorization
- JWT token-based authentication
- Role-based access control (RBAC)
- API key authentication for external integrations
- Session management with Redis
- Password complexity requirements

#### Data Protection
- Input validation and sanitization
- SQL injection prevention
- XSS protection
- CSRF protection
- Rate limiting
- Request logging and monitoring

#### Network Security
- HTTPS enforcement
- CORS configuration
- API rate limiting
- IP whitelisting for admin access
- Audit logging

### 9. Performance Requirements

#### Database Optimization
- Proper indexing strategy
- Connection pooling
- Query optimization
- Database partitioning for large datasets
- Read replicas for scaling

#### Caching Strategy
- Redis caching for frequently accessed data
- Session storage in Redis
- Real-time data caching
- API response caching
- Call statistics caching

#### Monitoring & Logging
- Structured logging with correlation IDs
- Performance monitoring
- Error tracking and alerting
- Call quality metrics
- System health monitoring

### 10. Testing Requirements

#### Unit Tests
- All service layer functions
- All utility functions
- All model validations
- All middleware functions

#### Integration Tests
- API endpoint testing
- Database integration testing
- Freeswitch integration testing
- WebSocket testing

#### End-to-End Tests
- Complete call flow testing
- User registration and authentication
- Extension management workflows
- Voicemail system testing

### 11. Docker Configuration

Create a complete Docker setup with:
- Multi-stage builds for optimization
- Health checks for all services
- Proper networking between containers
- Volume mounts for persistent data
- Environment-specific configurations

### 12. Documentation Requirements

#### API Documentation
- OpenAPI/Swagger specification
- Detailed endpoint documentation
- Request/response examples
- Error code documentation

#### Code Documentation
- Comprehensive code comments
- Function documentation
- Architecture documentation
- Deployment guides

### 13. Error Handling

#### Standardized Error Responses
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": {
      "field": "email",
      "reason": "Invalid email format"
    }
  }
}
```

#### Error Categories
- Validation errors (400)
- Authentication errors (401)
- Authorization errors (403)
- Not found errors (404)
- Conflict errors (409)
- Internal server errors (500)

### 14. Real-time Features

#### WebSocket Events
- Call events (start, answer, hangup, transfer)
- Extension status changes
- System notifications
- Call quality updates
- Conference events

#### Event Types
```json
{
  "type": "call_start",
  "data": {
    "call_uuid": "uuid",
    "caller": "1001",
    "callee": "1002",
    "direction": "outbound",
    "timestamp": "2024-01-01T12:00:00Z"
  }
}
```

### 15. Multitenancy Implementation

#### Tenant Isolation
- Database-level tenant isolation
- API-level tenant filtering
- Resource quota management
- Tenant-specific configurations
- Billing and usage tracking

#### Tenant Features
- Tenant-specific extensions
- Tenant-specific dialplans
- Tenant-specific gateways
- Tenant-specific recordings
- Tenant-specific statistics

## Implementation Priority

### Phase 1: Core Infrastructure
1. Database schema and migrations
2. Basic authentication system
3. User management
4. Extension management
5. Basic Freeswitch integration

### Phase 2: Call Management
1. Call routing system
2. Dialplan management
3. Call recording
4. Call transfer and forwarding
5. Real-time call events

### Phase 3: Advanced Features
1. Voicemail system
2. Conference system
3. IVR system
4. Call queuing
5. Advanced reporting

### Phase 4: Optimization
1. Performance optimization
2. Security hardening
3. Monitoring and alerting
4. Documentation
5. Testing coverage

## Success Criteria

The backend should be able to:
- Handle multiple concurrent calls
- Support real-time call management
- Provide comprehensive API for frontend integration
- Support multitenancy with proper isolation
- Integrate seamlessly with Freeswitch
- Provide detailed call analytics and reporting
- Support all standard PBX features
- Be scalable and maintainable
- Have comprehensive test coverage
- Be production-ready with proper security

Create this backend system following the Go best practices, using the Fiber framework, with proper error handling, logging, and security measures. Ensure all code is well-documented, tested, and follows the established patterns.