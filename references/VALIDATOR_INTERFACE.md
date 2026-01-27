# Request Validation Pattern

When a request has parameters that need validation, implement a `Validator` interface to decouple validation logic and allow for flexible, composable validation rules.

## Example Implementation

### Validator Interface (`validator.go`)

```go
package validator

import "context"

type Validator interface {
	Validate(context.Context, ...Validator) error
}
```

### All Pass Validator (`all_pass_validator.go`)

This validator aggregates multiple validators and ensures all of them pass.

```go
package validator

import "context"

type AllPassValidator struct{}

func NewAllPassValidator() *AllPassValidator {
	return &AllPassValidator{}
}

func (v *AllPassValidator) Validate(ctx context.Context, validators ...Validator) error {
	for _, v := range validators {
		if err := v.Validate(ctx); err != nil {
			return err
		}
	}
	return nil
}
```

### Service Usage (`service.go`)

Inject the root validator into the service.

```go
package service

import (
    "context"
    "yourproject/validator"
)

type Service struct {
	validator validator.Validator
}

func NewService(v validator.Validator) *Service {
	return &Service{
		validator: v,
	}
}

func (s *Service) CreateUser(ctx context.Context, req *CreateUserRequest) error {
    // Composable validation usage
	if err := s.validator.Validate(ctx,
			NewStringValidator(req.Name, 10, 30), // name must be between 10 and 30 characters
			NewNumberValidator(req.Age, 18, 120), // age must be between 18 and 120
			NewEmailValidator(req.Email),         // email must be valid
		); err != nil {
		return err
	}
    
    // ... business logic ...
    return nil
}
```

### Wiring (`main.go`)

Initialize the service with the `AllPassValidator` (or any other implementation).

```go
package main

import (
    "yourproject/service"
    "yourproject/validator"
)

func main() {
    // ...
    srv := service.NewService(validator.NewAllPassValidator())
    // ...
}
```
