# Coding Standards & Tooling Examples

## ✅ Can Do: Standard library solutions

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


## ❌ Can't Do: Unnecessary external dependencies

```go
// DON'T: Use gin/echo when standard library suffices for simple APIs
router := gin.Default()
router.GET("/users", handleUsers)

// DON'T: Use custom JSON library without good reason
import "github.com/json-iterator/go"

// DON'T: Use third-party time library for basic operations
import "github.com/jinzhu/now"
```


## ✅ Correct: Context as first parameter

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


## ❌ Incorrect: Missing context

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


## ✅ Correct: Raw SQL with pgx

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


## ❌ Incorrect: Using ORM

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


## ✅ Correct: Middleware implementation

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


## ✅ Correct: Functional Options Pattern

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


## ✅ Correct: Options Struct Pattern

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


## ❌ Incorrect: Multiple positional parameters

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


## ✅ Correct: Simple and focused

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


## ❌ Incorrect: Over-engineered

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


## ✅ Correct: Testable code with mocks

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


## ❌ Incorrect: Untestable code

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


## ✅ Example .golangci.yml

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
