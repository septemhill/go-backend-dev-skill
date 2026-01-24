# Error Handling Examples

## ✅ Correct: Typed errors and proper JSON tags

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


## ❌ Incorrect: String errors and missing JSON tags

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
