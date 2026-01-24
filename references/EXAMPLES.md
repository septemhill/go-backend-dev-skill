# Golang Backend Development Examples

This file contains code examples and patterns extracted from the main SKILL.md specification.

## 1. Project Organization

### ✅ Correct: Slim main.go

```go
// backend/cmd/api/main.go
package main

import (
    "context"
    "log"
    "os"

    "github.com/jackc/pgx/v5/pgxpool"
    "yourproject/internal/handler"
    "yourproject/internal/repository"
    "yourproject/internal/service"
)

func main() {
    dbPool, err := pgxpool.New(context.Background(), os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer dbPool.Close()

    // Dependency injection
    repo := repository.NewRepository(dbPool)
    svc := service.NewService(repo, os.Getenv("JWT_SECRET"))
    h := handler.NewHandler(svc)

    // Start server
    if err := h.Start(":8080"); err != nil {
        log.Fatal(err)
    }
}
```


### ❌ Incorrect: Business logic in main.go

```go
// backend/cmd/api/main.go
package main

import (
    "encoding/json"
    "net/http"
)

func main() {
    // DON'T: Business logic in main.go
    http.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPost {
            w.WriteHeader(http.StatusMethodNotAllowed)
            return
        }
        
        var user struct {
            Email string `json:"email"`
        }
        json.NewDecoder(r.Body).Decode(&user)
        
        // Business logic should be in service layer
        if user.Email == "" {
            w.WriteHeader(http.StatusBadRequest)
            return
        }
        // ... more logic
    })
    
    http.ListenAndServe(":8080", nil)
}
```


## 2. Core Architectural Patterns

### ✅ Correct: Proper Layer Separation

```go
// internal/handler/user_handler.go
type UserHandler struct {
    userService service.UserService // Accept interface
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req models.CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, err)
        return
    }

    // Defer to service - no business logic here
    user, err := h.userService.CreateUser(r.Context(), req)
    if err != nil {
        respondError(w, http.StatusInternalServerError, err)
        return
    }

    respondJSON(w, http.StatusCreated, user)
}

// internal/service/user_service.go
type UserService interface {
    CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error)
}

type userService struct {
    repo repository.UserRepository
}

// Return concrete type, accept interface
func NewUserService(repo repository.UserRepository) *userService {
    return &userService{repo: repo}
}

func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error) {
    // Business logic here
    if err := validateEmail(req.Email); err != nil {
        return nil, ErrInvalidEmail
    }

    hashedPassword, err := hashPassword(req.Password)
    if err != nil {
        return nil, err
    }

    user := &models.User{
        ID:       uuid.New(),
        Email:    req.Email,
        Password: hashedPassword,
    }

    return s.repo.Create(ctx, user)
}
```


### ❌ Incorrect: Business logic in handler

```go
// internal/handler/user_handler.go
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req models.CreateUserRequest
    json.NewDecoder(r.Body).Decode(&req)

    // DON'T: Validation and business logic in handler
    if !strings.Contains(req.Email, "@") {
        respondError(w, http.StatusBadRequest, errors.New("invalid email"))
        return
    }

    if len(req.Password) < 8 {
        respondError(w, http.StatusBadRequest, errors.New("password too short"))
        return
    }

    // DON'T: Direct database access from handler
    hashedPassword := bcrypt.GenerateFromPassword([]byte(req.Password), 10)
    _, err := h.db.Exec("INSERT INTO users (email, password) VALUES ($1, $2)", 
        req.Email, hashedPassword)
}
```


### ✅ Correct: Using Handler Wrapper

```go
// internal/handler/base.go
type Handler[REQ, RESP any] struct {
    HandleFunc    func(ctx context.Context, req REQ) (RESP, error)
    DecodeFunc    func(r *http.Request) (REQ, error)
    EncodeFunc    func(w http.ResponseWriter, resp RESP) error
    ErrorHandler  func(w http.ResponseWriter, err error)
}

func (h Handler[REQ, RESP]) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    req, err := h.DecodeFunc(r)
    if err != nil {
        h.ErrorHandler(w, err)
        return
    }

    resp, err := h.HandleFunc(r.Context(), req)
    if err != nil {
        h.ErrorHandler(w, err)
        return
    }

    if err := h.EncodeFunc(w, resp); err != nil {
        h.ErrorHandler(w, err)
    }
}

// internal/handler/user_handler.go
func NewCreateUserHandler(svc service.UserService) http.Handler {
    return Handler[models.CreateUserRequest, models.User]{
        HandleFunc: func(ctx context.Context, req models.CreateUserRequest) (models.User, error) {
            return svc.CreateUser(ctx, req)
        },
        DecodeFunc:   DefaultDecoder[models.CreateUserRequest],
        EncodeFunc:   DefaultEncoder[models.User],
        ErrorHandler: DefaultErrorHandler,
    }
}
```


### ❌ Incorrect: Manual request/response handling

```go
// DON'T: Reimplementing the same logic in every handler
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req models.CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
        return
    }

    user, err := h.svc.CreateUser(r.Context(), req)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}
```


### ✅ Correct: Constructor-based DI

```go
// internal/repository/user_repository.go
type UserRepository interface {
    Create(ctx context.Context, user *models.User) (*models.User, error)
    FindByEmail(ctx context.Context, email string) (*models.User, error)
}

type userRepository struct {
    db *pgxpool.Pool
}

func NewUserRepository(db *pgxpool.Pool) *userRepository {
    return &userRepository{db: db}
}

// internal/service/user_service.go
type userService struct {
    repo      repository.UserRepository
    jwtSecret string
}

func NewUserService(repo repository.UserRepository, jwtSecret string) *userService {
    return &userService{
        repo:      repo,
        jwtSecret: jwtSecret,
    }
}

// main.go
repo := repository.NewUserRepository(dbPool)
svc := service.NewUserService(repo, config.JWTSecret)
handler := handler.NewUserHandler(svc)
```


### ❌ Incorrect: Global variables or direct instantiation

```go
// DON'T: Global database connection
var globalDB *pgxpool.Pool

type userService struct{}

func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest) error {
    // DON'T: Using global variable
    _, err := globalDB.Exec(ctx, "INSERT INTO users ...")
    return err
}

// DON'T: Creating dependencies inside
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    // DON'T: Direct instantiation
    db, _ := pgxpool.New(context.Background(), "connection-string")
    repo := &userRepository{db: db}
    svc := &userService{repo: repo}
}
```


## 3. Coding Standards & Tooling

### ✅ Can Do: Standard library solutions

```go
// Use standard library for HTTP routing
mux := http.NewServeMux()
mux.HandleFunc("/users", handleUsers)
mux.HandleFunc("/health", handleHealth)

// Use standard library for JSON
json.Marshal(data)
json.Unmarshal(bytes, &result)

// Use standard library for time
time.Now()
time.Sleep(5 * time.Second)

// Use standard library for context
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```


### ❌ Can't Do: Unnecessary external dependencies

```go
// DON'T: Use gin/echo when standard library suffices for simple APIs
router := gin.Default()
router.GET("/users", handleUsers)

// DON'T: Use custom JSON library without good reason
import "github.com/json-iterator/go"

// DON'T: Use third-party time library for basic operations
import "github.com/jinzhu/now"
```


### ✅ Correct: Context as first parameter

```go
// internal/service/user_service.go
func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error) {
    // Can propagate cancellation
    user, err := s.repo.Create(ctx, user)
    if err != nil {
        return nil, err
    }
    return user, nil
}

// internal/repository/user_repository.go
func (r *userRepository) Create(ctx context.Context, user *models.User) (*models.User, error) {
    query := `INSERT INTO users (id, email, password) VALUES ($1, $2, $3) RETURNING *`
    
    // Context enables timeout and cancellation
    err := r.db.QueryRow(ctx, query, user.ID, user.Email, user.Password).Scan(
        &user.ID, &user.Email, &user.CreatedAt,
    )
    return user, err
}
```


### ❌ Incorrect: Missing context

```go
// DON'T: No context parameter
func (s *userService) CreateUser(req models.CreateUserRequest) (*models.User, error) {
    // Can't handle cancellation or timeouts
    user, err := s.repo.Create(user)
    return user, err
}

// DON'T: Using background context instead of propagating
func (r *userRepository) Create(user *models.User) (*models.User, error) {
    // DON'T: Creating new context instead of accepting one
    ctx := context.Background()
    err := r.db.QueryRow(ctx, query, user.ID, user.Email).Scan(&user.ID)
    return user, err
}
```


### ✅ Correct: Raw SQL with pgx

```go
// internal/repository/user_repository.go
func (r *userRepository) FindByEmail(ctx context.Context, email string) (*models.User, error) {
    query := `
        SELECT id, email, password, created_at, updated_at 
        FROM users 
        WHERE email = $1
    `
    
    var user models.User
    err := r.db.QueryRow(ctx, query, email).Scan(
        &user.ID,
        &user.Email,
        &user.Password,
        &user.CreatedAt,
        &user.UpdatedAt,
    )
    
    if err == pgx.ErrNoRows {
        return nil, ErrUserNotFound
    }
    
    return &user, err
}

func (r *userRepository) List(ctx context.Context, limit, offset int) ([]*models.User, error) {
    query := `
        SELECT id, email, created_at 
        FROM users 
        ORDER BY created_at DESC 
        LIMIT $1 OFFSET $2
    `
    
    rows, err := r.db.Query(ctx, query, limit, offset)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var users []*models.User
    for rows.Next() {
        var user models.User
        if err := rows.Scan(&user.ID, &user.Email, &user.CreatedAt); err != nil {
            return nil, err
        }
        users = append(users, &user)
    }
    
    return users, rows.Err()
}
```


### ❌ Incorrect: Using ORM

```go
// DON'T: Using GORM or other ORM
import "gorm.io/gorm"

func (r *userRepository) FindByEmail(email string) (*models.User, error) {
    var user models.User
    result := r.db.Where("email = ?", email).First(&user)
    return &user, result.Error
}

// DON'T: Active Record pattern
func (u *User) Save() error {
    return db.Save(u).Error
}
```


### ✅ Correct: Middleware implementation

```go
// internal/middleware/logging.go
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Wrap response writer to capture status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        
        next.ServeHTTP(wrapped, r)
        
        log.Printf("%s %s %d %v", r.Method, r.URL.Path, wrapped.statusCode, time.Since(start))
    })
}

// internal/middleware/auth.go
func AuthMiddleware(jwtSecret string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, "missing authorization header", http.StatusUnauthorized)
                return
            }
            
            tokenString := strings.TrimPrefix(authHeader, "Bearer ")
            userID, err := validateJWT(tokenString, jwtSecret)
            if err != nil {
                http.Error(w, "invalid token", http.StatusUnauthorized)
                return
            }
            
            // Add userID to context
            ctx := context.WithValue(r.Context(), "userID", userID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// internal/middleware/cors.go
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

// Usage in main.go or router setup
mux := http.NewServeMux()
mux.Handle("/api/users", handler.NewUserHandler(svc))

handler := CORSMiddleware(LoggingMiddleware(AuthMiddleware(jwtSecret)(mux)))
http.ListenAndServe(":8080", handler)
```


### ✅ Correct: Functional Options Pattern

```go
// internal/service/user_service.go
type CreateUserOptions struct {
    SendWelcomeEmail bool
    AssignRole       string
    Metadata         map[string]string
}

type CreateUserOption func(*CreateUserOptions)

func WithWelcomeEmail() CreateUserOption {
    return func(o *CreateUserOptions) {
        o.SendWelcomeEmail = true
    }
}

func WithRole(role string) CreateUserOption {
    return func(o *CreateUserOptions) {
        o.AssignRole = role
    }
}

func WithMetadata(metadata map[string]string) CreateUserOption {
    return func(o *CreateUserOptions) {
        o.Metadata = metadata
    }
}

func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest, opts ...CreateUserOption) (*models.User, error) {
    options := &CreateUserOptions{
        SendWelcomeEmail: false,
        AssignRole:       "user",
    }
    
    for _, opt := range opts {
        opt(options)
    }
    
    // Use options...
    user, err := s.repo.Create(ctx, &models.User{
        Email: req.Email,
        Role:  options.AssignRole,
    })
    
    if err != nil {
        return nil, err
    }
    
    if options.SendWelcomeEmail {
        s.sendWelcomeEmail(ctx, user)
    }
    
    return user, nil
}

// Usage - backward compatible as new options are added
user, err := svc.CreateUser(ctx, req)
user, err := svc.CreateUser(ctx, req, WithWelcomeEmail())
user, err := svc.CreateUser(ctx, req, WithWelcomeEmail(), WithRole("admin"))
```


### ✅ Correct: Options Struct Pattern

```go
type ListUsersRequest struct {
    Limit      int
    Offset     int
    SortBy     string
    SortOrder  string
    FilterRole string
}

func (s *userService) ListUsers(ctx context.Context, req ListUsersRequest) ([]*models.User, error) {
    // Set defaults
    if req.Limit == 0 {
        req.Limit = 20
    }
    if req.SortBy == "" {
        req.SortBy = "created_at"
    }
    if req.SortOrder == "" {
        req.SortOrder = "DESC"
    }
    
    return s.repo.List(ctx, req)
}

// Easy to extend without breaking existing calls
// Before: ListUsers(ctx, 20, 0)
// After: ListUsers(ctx, ListUsersRequest{Limit: 20, Offset: 0})
// Later: ListUsers(ctx, ListUsersRequest{Limit: 20, Offset: 0, FilterRole: "admin"})
```


### ❌ Incorrect: Multiple positional parameters

```go
// DON'T: Too many positional parameters - not extensible
func (s *userService) CreateUser(
    ctx context.Context,
    email string,
    password string,
    sendWelcomeEmail bool,
    role string,
    metadata map[string]string,
) (*models.User, error) {
    // Adding new parameters breaks all existing calls
}

// DON'T: Changing function signature breaks compatibility
// Version 1
func (s *userService) ListUsers(ctx context.Context, limit, offset int) ([]*models.User, error)

// Version 2 - BREAKS existing code
func (s *userService) ListUsers(ctx context.Context, limit, offset int, sortBy string) ([]*models.User, error)
```


### ✅ Correct: Simple and focused

```go
// Single Responsibility - one clear purpose
type EmailSender struct {
    smtpHost string
    smtpPort int
}

func (e *EmailSender) Send(to, subject, body string) error {
    // Simple, focused implementation
    return smtp.SendMail(
        fmt.Sprintf("%s:%d", e.smtpHost, e.smtpPort),
        nil,
        "noreply@example.com",
        []string{to},
        []byte(fmt.Sprintf("Subject: %s\n\n%s", subject, body)),
    )
}

// Interface Segregation - small, focused interface
type UserCreator interface {
    CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error)
}

// Dependency Inversion - depend on abstraction
type NotificationService struct {
    emailSender EmailSender // Could be interface if needed
}
```


### ❌ Incorrect: Over-engineered

```go
// DON'T: Over-abstraction without clear benefit
type AbstractFactoryProvider interface {
    GetFactory() AbstractFactory
}

type AbstractFactory interface {
    CreateUserFactory() UserFactory
}

type UserFactory interface {
    CreateUser() User
}

// DON'T: God object with too many responsibilities
type UserManager struct {
    db            *pgxpool.Pool
    cache         *redis.Client
    emailSender   *EmailSender
    logger        *Logger
    metrics       *MetricsCollector
    eventBus      *EventBus
}

func (m *UserManager) CreateUser() error { /* ... */ }
func (m *UserManager) UpdateUser() error { /* ... */ }
func (m *UserManager) DeleteUser() error { /* ... */ }
func (m *UserManager) SendEmail() error { /* ... */ }
func (m *UserManager) LogActivity() error { /* ... */ }
func (m *UserManager) PublishEvent() error { /* ... */ }
```


### ✅ Correct: Testable code with mocks

```go
// internal/service/user_service_test.go
//go:generate mockery --name=UserRepository --output=./mocks --outpkg=mocks

func TestUserService_CreateUser(t *testing.T) {
    mockRepo := new(mocks.UserRepository)
    svc := NewUserService(mockRepo, "test-secret")
    
    req := models.CreateUserRequest{
        Email:    "test@example.com",
        Password: "password123",
    }
    
    expectedUser := &models.User{
        ID:    uuid.New(),
        Email: req.Email,
    }
    
    // Setup mock expectation
    mockRepo.On("Create", mock.Anything, mock.AnythingOfType("*models.User")).
        Return(expectedUser, nil)
    
    // Execute
    user, err := svc.CreateUser(context.Background(), req)
    
    // Assert
    assert.NoError(t, err)
    assert.Equal(t, expectedUser.Email, user.Email)
    mockRepo.AssertExpectations(t)
}

func TestUserService_CreateUser_ValidationError(t *testing.T) {
    mockRepo := new(mocks.UserRepository)
    svc := NewUserService(mockRepo, "test-secret")
    
    req := models.CreateUserRequest{
        Email:    "invalid-email",
        Password: "123",
    }
    
    // Execute
    user, err := svc.CreateUser(context.Background(), req)
    
    // Assert
    assert.Error(t, err)
    assert.Nil(t, user)
    assert.Equal(t, ErrInvalidEmail, err)
}

// internal/repository/user_repository_test.go
func TestUserRepository_Create(t *testing.T) {
    // Use testcontainers or pgx test pool
    ctx := context.Background()
    pool := setupTestDB(t)
    defer pool.Close()
    
    repo := NewUserRepository(pool)
    
    user := &models.User{
        ID:       uuid.New(),
        Email:    "test@example.com",
        Password: "hashed-password",
    }
    
    created, err := repo.Create(ctx, user)
    
    assert.NoError(t, err)
    assert.NotNil(t, created)
    assert.Equal(t, user.Email, created.Email)
}
```


### ❌ Incorrect: Untestable code

```go
// DON'T: Direct database access without interface
type userService struct {
    db *pgxpool.Pool // Can't mock this easily
}

func (s *userService) CreateUser(req models.CreateUserRequest) error {
    // DON'T: Hard to test - direct DB access
    _, err := s.db.Exec(context.Background(), "INSERT INTO users ...")
    return err
}

// DON'T: Global state makes testing difficult
var globalConfig = loadConfig()

func CreateUser(email string) error {
    // Uses global state - hard to test
    db := connectDB(globalConfig.DBUrl)
    // ...
}
```


### ✅ Example .golangci.yml

```yaml
run:
  timeout: 5m
  tests: true
  
linters:
  enable:
    - gofmt
    - govet
    - errcheck
    - staticcheck
    - unused
    - gosimple
    - ineffassign
    - typecheck
    - revive
    - gosec
    - gocyclo
    
linters-settings:
  gocyclo:
    min-complexity: 15
  govet:
    check-shadowing: true
  revive:
    rules:
      - name: exported
        disabled: false
```


## 4. Error Handling & Return Values

### ✅ Correct: Typed errors and proper JSON tags

```go
// internal/service/errors.go
var (
    ErrUserNotFound     = errors.New("user not found")
    ErrInvalidEmail     = errors.New("invalid email address")
    ErrUnauthorized     = errors.New("unauthorized")
    ErrEmailAlreadyUsed = errors.New("email already in use")
)

// internal/models/user.go
type User struct {
    ID        uuid.UUID  `json:"id"`
    Email     string     `json:"email"`
    Password  string     `json:"-"` // Never expose password in JSON
    CreatedAt time.Time  `json:"created_at"`
    UpdatedAt *time.Time `json:"updated_at,omitempty"`
}

type CreateUserRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type CreateUserResponse struct {
    ID        uuid.UUID `json:"id"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

// internal/service/user_service.go
func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error) {
    // Check if email already exists
    existing, err := s.repo.FindByEmail(ctx, req.Email)
    if err != nil && !errors.Is(err, ErrUserNotFound) {
        return nil, err
    }
    if existing != nil {
        return nil, ErrEmailAlreadyUsed
    }
    
    user := &models.User{
        ID:        uuid.New(), // Always use UUID
        Email:     req.Email,
        CreatedAt: time.Now(),
    }
    
    return s.repo.Create(ctx, user)
}

// internal/handler/user_handler.go
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req models.CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, err)
        return
    }
    
    user, err := h.svc.CreateUser(r.Context(), req)
    if err != nil {
        // Map errors to appropriate HTTP status codes
        switch {
        case errors.Is(err, ErrInvalidEmail):
            respondError(w, http.StatusBadRequest, err)
        case errors.Is(err, ErrEmailAlreadyUsed):
            respondError(w, http.StatusConflict, err)
        case errors.Is(err, ErrUnauthorized):
            respondError(w, http.StatusUnauthorized, err)
        default:
            respondError(w, http.StatusInternalServerError, err)
        }
        return
    }
    
    respondJSON(w, http.StatusCreated, user)
}
```


### ❌ Incorrect: String errors and missing JSON tags

```go
// DON'T: String-based errors
func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error) {
    existing, err := s.repo.FindByEmail(ctx, req.Email)
    if existing != nil {
        // DON'T: Return string errors
        return nil, errors.New("email already used")
    }
    
    user := &models.User{
        ID:    "user-123", // DON'T: Use string IDs instead of UUID
        Email: req.Email,
    }
    
    return s.repo.Create(ctx, user)
}

// DON'T: Missing JSON tags
type User struct {
    ID        uuid.UUID  // Missing json tag
    Email     string     `json:"email"`
    Password  string     `json:"password"` // DON'T: Exposing password
    CreatedAt time.Time  // Missing json tag
}

// DON'T: Generic error handling
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    user, err := h.svc.CreateUser(r.Context(), req)
    if err != nil {
        // DON'T: Always return 500 for any error
        respondError(w, http.StatusInternalServerError, err)
        return
    }
}
```


## 5. Extensibility Goal

### ✅ Correct: Event-driven extensibility

```go
// pkg/events/events.go
type Event struct {
    Type      string
    Timestamp time.Time
    Data      interface{}
}

type EventHandler func(Event) error

type EventBus struct {
    handlers map[string][]EventHandler
    mu       sync.RWMutex
}

func NewEventBus() *EventBus {
    return &EventBus{
        handlers: make(map[string][]EventHandler),
    }
}

func (eb *EventBus) Subscribe(eventType string, handler EventHandler) {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    eb.handlers[eventType] = append(eb.handlers[eventType], handler)
}

func (eb *EventBus) Publish(event Event) {
    eb.mu.RLock()
    handlers := eb.handlers[event.Type]
    eb.mu.RUnlock()
    
    for _, handler := range handlers {
        // Run async to not block main flow
        go func(h EventHandler) {
            if err := h(event); err != nil {
                log.Printf("event handler error: %v", err)
            }
        }(handler)
    }
}

// internal/service/user_service.go
type userService struct {
    repo     repository.UserRepository
    eventBus *events.EventBus
}

func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error) {
    user, err := s.repo.Create(ctx, &models.User{
        ID:    uuid.New(),
        Email: req.Email,
    })
    if err != nil {
        return nil, err
    }
    
    // Publish event for side effects
    s.eventBus.Publish(events.Event{
        Type:      "user.created",
        Timestamp: time.Now(),
        Data:      user,
    })
    
    return user, nil
}

// main.go - Wire up side effects without coupling
eventBus := events.NewEventBus()

// Subscribe notification handler
eventBus.Subscribe("user.created", func(e events.Event) error {
    user := e.Data.(*models.User)
    return notificationSvc.SendWelcomeEmail(user.Email)
})

// Subscribe analytics handler
eventBus.Subscribe("user.created", func(e events.Event) error {
    user := e.Data.(*models.User)
    return analyticsSvc.TrackSignup(user.ID)
})

// Subscribe audit log handler
eventBus.Subscribe("user.created", func(e events.Event) error {
    user := e.Data.(*models.User)
    return auditSvc.Log("user_created", user.ID)
})

svc := service.NewUserService(repo, eventBus)
```


### ✅ Correct: Hook pattern for extensibility

```go
// internal/service/hooks.go
type UserHooks struct {
    BeforeCreate []func(context.Context, *models.User) error
    AfterCreate  []func(context.Context, *models.User) error
}

type userService struct {
    repo  repository.UserRepository
    hooks UserHooks
}

func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error) {
    user := &models.User{
        ID:    uuid.New(),
        Email: req.Email,
    }
    
    // Run before hooks
    for _, hook := range s.hooks.BeforeCreate {
        if err := hook(ctx, user); err != nil {
            return nil, err
        }
    }
    
    // Core logic
    created, err := s.repo.Create(ctx, user)
    if err != nil {
        return nil, err
    }
    
    // Run after hooks (don't fail the operation if hooks fail)
    for _, hook := range s.hooks.AfterCreate {
        if err := hook(ctx, created); err != nil {
            log.Printf("after create hook error: %v", err)
        }
    }
    
    return created, nil
}

// main.go - Add hooks without modifying service logic
hooks := service.UserHooks{
    BeforeCreate: []func(context.Context, *models.User) error{
        validateUserEmail,
        checkUserQuota,
    },
    AfterCreate: []func(context.Context, *models.User) error{
        sendWelcomeEmail,
        trackAnalytics,
        logAudit,
    },
}

svc := service.NewUserService(repo)
svc.hooks = hooks
```


### ❌ Incorrect: Tightly coupled side effects

```go
// DON'T: Hardcoded side effects in business logic
func (s *userService) CreateUser(ctx context.Context, req models.CreateUserRequest) (*models.User, error) {
    user, err := s.repo.Create(ctx, &models.User{
        ID:    uuid.New(),
        Email: req.Email,
    })
    if err != nil {
        return nil, err
    }
    
    // DON'T: Tightly coupled notification logic
    if err := s.sendWelcomeEmail(user.Email); err != nil {
        log.Printf("failed to send email: %v", err)
    }
    
    // DON'T: Tightly coupled analytics
    if err := s.trackSignup(user.ID); err != nil {
        log.Printf("failed to track: %v", err)
    }
    
    // DON'T: Tightly coupled audit logging
    if err := s.auditLog("user_created", user.ID); err != nil {
        log.Printf("failed to log: %v", err)
    }
    
    // Adding new side effects requires modifying this function
    return user, nil
}

// DON'T: Service depends on too many concerns
type userService struct {
    repo             repository.UserRepository
    emailService     *EmailService
    analyticsService *AnalyticsService
    auditService     *AuditService
    notificationSvc  *NotificationService
    // This list keeps growing...
}
```


