# Authentication Strategy for PBX Voice System

## Overview
Instead of JWT, we'll implement a robust session-based authentication system similar to GoToConnect, with separate authentication mechanisms for different components.

## Authentication Architecture

### 1. Web Application Authentication (Session-based)

#### Session Management
```go
type Session struct {
    ID           string    `json:"id"`
    UserID       uint      `json:"user_id"`
    TenantID     uint      `json:"tenant_id"`
    Token        string    `json:"token"`
    RefreshToken string    `json:"refresh_token"`
    IPAddress    string    `json:"ip_address"`
    UserAgent    string    `json:"user_agent"`
    ExpiresAt    time.Time `json:"expires_at"`
    CreatedAt    time.Time `json:"created_at"`
    LastActivity time.Time `json:"last_activity"`
    IsActive     bool      `json:"is_active"`
}
```

#### Authentication Flow
1. **Login**: Username/password → Session token + Refresh token
2. **API Calls**: Session token in Authorization header
3. **Token Refresh**: Automatic refresh using refresh token
4. **Logout**: Invalidate session on server

#### Security Features
- **Session Encryption**: Tokens criptografados
- **IP Binding**: Sessões vinculadas ao IP
- **Device Tracking**: Controle de dispositivos
- **Session Timeout**: Expiração automática
- **Concurrent Sessions**: Múltiplas sessões por usuário

### 2. SIP Authentication (Separate System)

#### SIP Credentials
```go
type SIPCredentials struct {
    ID           uint      `json:"id"`
    ExtensionID  uint      `json:"extension_id"`
    Username     string    `json:"username"`
    Password     string    `json:"password"`
    Domain       string    `json:"domain"`
    Realm        string    `json:"realm"`
    Nonce        string    `json:"nonce"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}
```

#### SIP Authentication Flow
1. **Registration**: SIP REGISTER com digest authentication
2. **Call Authentication**: Cada chamada autenticada
3. **Dynamic Credentials**: Credenciais geradas dinamicamente
4. **Certificate Support**: Certificados para dispositivos físicos

### 3. WebRTC Authentication

#### WebRTC Session
```go
type WebRTCSession struct {
    ID           string    `json:"id"`
    UserID       uint      `json:"user_id"`
    ExtensionID  uint      `json:"extension_id"`
    SessionToken string    `json:"session_token"`
    STUNServers  []string  `json:"stun_servers"`
    TURNServers  []string  `json:"turn_servers"`
    ExpiresAt    time.Time `json:"expires_at"`
    CreatedAt    time.Time `json:"created_at"`
}
```

#### WebRTC Authentication Flow
1. **Session Creation**: Criar sessão WebRTC específica
2. **STUN/TURN Auth**: Autenticação para servidores de mídia
3. **SIP Integration**: Integração com credenciais SIP
4. **Session Management**: Controle de sessões ativas

## Implementation Strategy

### 1. Session-based Authentication

#### Login Endpoint
```go
// POST /api/v1/auth/login
type LoginRequest struct {
    Username string `json:"username" validate:"required"`
    Password string `json:"password" validate:"required"`
    DeviceID string `json:"device_id"`
    Remember bool   `json:"remember"`
}

type LoginResponse struct {
    SessionToken  string `json:"session_token"`
    RefreshToken  string `json:"refresh_token"`
    ExpiresIn     int    `json:"expires_in"`
    User          User   `json:"user"`
    Permissions   []string `json:"permissions"`
}
```

#### Session Middleware
```go
func SessionAuthMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        token := c.Get("Authorization")
        if token == "" {
            return c.Status(401).JSON(fiber.Map{
                "error": "No session token provided",
            })
        }

        session, err := sessionService.ValidateSession(token)
        if err != nil {
            return c.Status(401).JSON(fiber.Map{
                "error": "Invalid session token",
            })
        }

        // Add session to context
        c.Locals("session", session)
        c.Locals("user_id", session.UserID)
        c.Locals("tenant_id", session.TenantID)

        return c.Next()
    }
}
```

### 2. SIP Authentication Service

#### SIP Registration
```go
type SIPRegistrationService struct {
    db *gorm.DB
    fs *FreeswitchClient
}

func (s *SIPRegistrationService) RegisterExtension(extension *Extension) error {
    // Generate SIP credentials
    credentials := &SIPCredentials{
        ExtensionID: extension.ID,
        Username:    extension.SIPUsername,
        Password:    generateSecurePassword(),
        Domain:      extension.Domain,
        Realm:       extension.Realm,
    }

    // Store in database
    if err := s.db.Create(credentials).Error; err != nil {
        return err
    }

    // Register with Freeswitch via mod_xml_curl
    return s.fs.RegisterUser(credentials)
}

func (s *SIPRegistrationService) AuthenticateSIP(username, password, realm string) (*SIPCredentials, error) {
    var credentials SIPCredentials
    err := s.db.Where("username = ? AND realm = ?", username, realm).First(&credentials).Error
    if err != nil {
        return nil, err
    }

    // Verify password using digest authentication
    if !verifySIPPassword(password, credentials.Password) {
        return nil, errors.New("invalid SIP credentials")
    }

    return &credentials, nil
}
```

### 3. WebRTC Authentication Service

#### WebRTC Session Management
```go
type WebRTCService struct {
    db *gorm.DB
    fs *FreeswitchClient
}

func (w *WebRTCService) CreateWebRTCSession(userID, extensionID uint) (*WebRTCSession, error) {
    // Generate session token
    sessionToken := generateSecureToken()
    
    // Get STUN/TURN servers
    stunServers := []string{"stun:stun.l.google.com:19302"}
    turnServers := []string{"turn:your-turn-server.com:3478"}
    
    session := &WebRTCSession{
        UserID:       userID,
        ExtensionID:  extensionID,
        SessionToken: sessionToken,
        STUNServers:  stunServers,
        TURNServers:  turnServers,
        ExpiresAt:    time.Now().Add(24 * time.Hour),
    }

    if err := w.db.Create(session).Error; err != nil {
        return nil, err
    }

    return session, nil
}

func (w *WebRTCService) ValidateWebRTCSession(sessionToken string) (*WebRTCSession, error) {
    var session WebRTCSession
    err := w.db.Where("session_token = ? AND expires_at > ?", sessionToken, time.Now()).First(&session).Error
    if err != nil {
        return nil, err
    }

    return &session, nil
}
```

### 4. Database Schema Updates

#### Sessions Table
```sql
CREATE TABLE sessions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    tenant_id BIGINT NOT NULL,
    token VARCHAR(255) NOT NULL UNIQUE,
    refresh_token VARCHAR(255) NOT NULL UNIQUE,
    ip_address INET,
    user_agent TEXT,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE
);

CREATE INDEX idx_sessions_token ON sessions(token);
CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

#### SIP Credentials Table
```sql
CREATE TABLE sip_credentials (
    id BIGSERIAL PRIMARY KEY,
    extension_id BIGINT NOT NULL,
    username VARCHAR(100) NOT NULL,
    password VARCHAR(255) NOT NULL,
    domain VARCHAR(100) NOT NULL,
    realm VARCHAR(100) NOT NULL,
    nonce VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (extension_id) REFERENCES extensions(id) ON DELETE CASCADE
);

CREATE INDEX idx_sip_credentials_username ON sip_credentials(username);
CREATE INDEX idx_sip_credentials_extension_id ON sip_credentials(extension_id);
```

#### WebRTC Sessions Table
```sql
CREATE TABLE webrtc_sessions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    extension_id BIGINT NOT NULL,
    session_token VARCHAR(255) NOT NULL UNIQUE,
    stun_servers JSONB,
    turn_servers JSONB,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (extension_id) REFERENCES extensions(id) ON DELETE CASCADE
);

CREATE INDEX idx_webrtc_sessions_token ON webrtc_sessions(session_token);
CREATE INDEX idx_webrtc_sessions_user_id ON webrtc_sessions(user_id);
```

### 5. API Endpoints

#### Authentication Endpoints
```go
// POST /api/v1/auth/login
func (a *AuthController) Login(c *fiber.Ctx) error {
    var req LoginRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
    }

    // Validate credentials
    user, err := a.authService.ValidateCredentials(req.Username, req.Password)
    if err != nil {
        return c.Status(401).JSON(fiber.Map{"error": "Invalid credentials"})
    }

    // Create session
    session, err := a.authService.CreateSession(user.ID, user.TenantID, c.IP(), c.Get("User-Agent"))
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to create session"})
    }

    return c.JSON(LoginResponse{
        SessionToken: session.Token,
        RefreshToken: session.RefreshToken,
        ExpiresIn:    int(session.ExpiresAt.Sub(time.Now()).Seconds()),
        User:         *user,
        Permissions:  a.authService.GetUserPermissions(user.ID),
    })
}

// POST /api/v1/auth/logout
func (a *AuthController) Logout(c *fiber.Ctx) error {
    session := c.Locals("session").(*Session)
    
    if err := a.authService.InvalidateSession(session.Token); err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to logout"})
    }

    return c.JSON(fiber.Map{"message": "Logged out successfully"})
}

// POST /api/v1/auth/refresh
func (a *AuthController) Refresh(c *fiber.Ctx) error {
    refreshToken := c.Get("X-Refresh-Token")
    if refreshToken == "" {
        return c.Status(401).JSON(fiber.Map{"error": "No refresh token provided"})
    }

    session, err := a.authService.RefreshSession(refreshToken)
    if err != nil {
        return c.Status(401).JSON(fiber.Map{"error": "Invalid refresh token"})
    }

    return c.JSON(fiber.Map{
        "session_token": session.Token,
        "expires_in":   int(session.ExpiresAt.Sub(time.Now()).Seconds()),
    })
}
```

#### WebRTC Endpoints
```go
// POST /api/v1/webrtc/session
func (w *WebRTCController) CreateSession(c *fiber.Ctx) error {
    session := c.Locals("session").(*Session)
    extensionID := c.Query("extension_id")
    
    extID, err := strconv.ParseUint(extensionID, 10, 32)
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid extension ID"})
    }

    webrtcSession, err := w.webrtcService.CreateWebRTCSession(session.UserID, uint(extID))
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to create WebRTC session"})
    }

    return c.JSON(webrtcSession)
}

// GET /api/v1/webrtc/session/:token
func (w *WebRTCController) ValidateSession(c *fiber.Ctx) error {
    token := c.Params("token")
    
    session, err := w.webrtcService.ValidateWebRTCSession(token)
    if err != nil {
        return c.Status(401).JSON(fiber.Map{"error": "Invalid WebRTC session"})
    }

    return c.JSON(session)
}
```

### 6. Security Features

#### Session Security
- **Token Encryption**: Tokens criptografados com AES-256
- **IP Binding**: Sessões vinculadas ao IP do usuário
- **Device Tracking**: Controle de dispositivos ativos
- **Session Rotation**: Renovação automática de tokens
- **Brute Force Protection**: Rate limiting para login

#### SIP Security
- **Digest Authentication**: Autenticação SIP padrão
- **Dynamic Passwords**: Senhas geradas dinamicamente
- **Certificate Support**: Suporte a certificados
- **Registration Limits**: Limite de registros por extensão

#### WebRTC Security
- **Session Tokens**: Tokens específicos para WebRTC
- **STUN/TURN Auth**: Autenticação para servidores de mídia
- **Session Expiration**: Expiração automática de sessões
- **Device Validation**: Validação de dispositivos

### 7. Benefits Over JWT

#### Session-based Advantages
- **Server-side Control**: Controle total das sessões
- **Immediate Invalidation**: Logout instantâneo
- **Device Management**: Controle de dispositivos
- **Security Monitoring**: Monitoramento de segurança
- **Compliance**: Melhor para compliance (GDPR, etc.)

#### SIP Integration
- **Native SIP Auth**: Autenticação SIP nativa
- **Dynamic Credentials**: Credenciais geradas dinamicamente
- **Certificate Support**: Suporte a certificados
- **Multi-device**: Suporte a múltiplos dispositivos

#### WebRTC Integration
- **Session Management**: Gerenciamento de sessões WebRTC
- **Media Security**: Segurança para mídia
- **STUN/TURN Auth**: Autenticação para NAT traversal
- **Real-time Control**: Controle em tempo real

This authentication strategy provides enterprise-grade security similar to GoToConnect, with proper session management, SIP integration, and WebRTC support.