# Context Usage Examples

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
