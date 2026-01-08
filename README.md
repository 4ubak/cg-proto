# CG Proto

Централизованное хранилище Proto контрактов для всех микросервисов CG Platform.

## Структура

```
cg-proto/
├── users/                    # Домен пользователей
│   ├── auth.proto           # Авторизация
│   ├── user.proto           # Профили
│   └── organization.proto   # Организации
│
├── marketplace/              # Домен маркетплейса
│   └── ad.proto             # Объявления
│
├── communication/            # Домен коммуникаций
│   ├── chat.proto           # Чаты
│   └── notification.proto   # Уведомления
│
├── jobs/                     # Домен вакансий
│   └── job.proto            # Вакансии и резюме
│
├── platform/                 # Инфраструктура
│   ├── counter.proto        # Счётчики
│   └── nsi.proto            # Справочники
│
├── gen/go/                   # Сгенерированный Go код
│   ├── users/
│   │   ├── auth.pb.go
│   │   ├── authv1connect/    # Connect handlers & clients
│   │   │   └── auth.connect.go
│   │   └── ...
├── buf.yaml                  # Buf конфигурация
├── buf.gen.yaml              # Генерация кода (Connect-go)
└── go.mod
```

## Установка

```bash
go get gitlab.com/xakpro/cg-proto@latest
```

## Connect-go

Мы используем **Connect-go** вместо стандартного gRPC — это даёт:
- **gRPC** для сервис-к-сервис коммуникации
- **HTTP/JSON** для фронтенда (без прокси)
- **gRPC-Web** для браузеров

### buf.gen.yaml

```yaml
version: v1
plugins:
  # Standard protobuf Go code
  - plugin: go
    out: gen/go
    opt:
      - paths=source_relative

  # Connect-go (replaces go-grpc)
  - plugin: buf.build/connectrpc/go
    out: gen/go
    opt:
      - paths=source_relative
```

## Использование

### Server (Connect Handler)

```go
import (
    "connectrpc.com/connect"
    "net/http"
    
    pb "gitlab.com/xakpro/cg-proto/gen/go/users/auth/v1"
    "gitlab.com/xakpro/cg-proto/gen/go/users/authv1connect"
)

// Implement the handler interface
type AuthHandler struct {
    authv1connect.UnimplementedAuthServiceHandler
    authService AuthServiceI
}

func (h *AuthHandler) SendCode(
    ctx context.Context,
    req *connect.Request[pb.SendCodeRequest],
) (*connect.Response[pb.SendCodeResponse], error) {
    // Your logic here
    return connect.NewResponse(&pb.SendCodeResponse{
        Success: true,
    }), nil
}

// Register handler
func main() {
    mux := http.NewServeMux()
    path, handler := authv1connect.NewAuthServiceHandler(&AuthHandler{})
    mux.Handle(path, handler)
    
    http.ListenAndServe(":8080", h2c.NewHandler(mux, &http2.Server{}))
}
```

### Client (Connect Client)

```go
import (
    "net/http"
    "connectrpc.com/connect"
    
    pb "gitlab.com/xakpro/cg-proto/gen/go/users/auth/v1"
    "gitlab.com/xakpro/cg-proto/gen/go/users/authv1connect"
)

// Create client
client := authv1connect.NewAuthServiceClient(
    http.DefaultClient,
    "http://auth-service:8080",
    connect.WithGRPC(), // Use gRPC protocol for service-to-service
)

// Call method
resp, err := client.ValidateToken(ctx, connect.NewRequest(&pb.ValidateTokenRequest{
    AccessToken: token,
}))
if err != nil {
    // Handle error (connect.CodeOf(err) returns error code)
}
userID := resp.Msg.UserId
```

### HTTP/JSON (для фронтенда)

```bash
# POST запрос с JSON
curl -X POST http://localhost:8080/users.auth.v1.AuthService/SendCode \
  -H "Content-Type: application/json" \
  -d '{"phone": "+77001234567", "device_id": "abc123"}'

# Response
{
  "success": true,
  "retryAfter": 60,
  "message": "Code sent"
}
```

### gRPC (для сервис-к-сервис)

```bash
grpcurl -plaintext localhost:8080 \
  users.auth.v1.AuthService/SendCode \
  -d '{"phone": "+77001234567", "device_id": "abc123"}'
```

## Генерация кода

```bash
# Установить buf
brew install bufbuild/buf/buf

# Сгенерировать Go код
buf generate

# Проверить линтером
buf lint

# Проверить breaking changes
buf breaking --against '.git#branch=main'
```

## Правила именования

### Пакеты

```protobuf
package {domain}.{service}.v1;

// Примеры:
package users.auth.v1;
package marketplace.ad.v1;
package communication.chat.v1;
```

### Go Package

```protobuf
// Формат: путь;alias
option go_package = "gitlab.com/xakpro/cg-proto/gen/go/users/auth/v1;authv1";
```

### Сервисы

```protobuf
// Имя сервиса = Имя файла + Service
service AuthService { ... }
service AdService { ... }
service ChatService { ... }
```

### Методы

```protobuf
// CRUD
rpc Create{Entity}(...) returns (...);
rpc Get{Entity}(...) returns (...);
rpc Update{Entity}(...) returns (...);
rpc Delete{Entity}(...) returns (...);
rpc List{Entities}(...) returns (...);

// Бизнес-действия
rpc SendCode(...) returns (...);
rpc VerifyCode(...) returns (...);
rpc SearchAds(...) returns (...);
```

### Сообщения

```protobuf
// Request/Response
message Get{Entity}Request { ... }
message Get{Entity}Response { ... }

// Entity
message User { ... }
message Ad { ... }
message Message { ... }
```

## Interceptors (Connect)

```go
import "connectrpc.com/connect"

// Logging interceptor
type LoggingInterceptor struct{}

func (i *LoggingInterceptor) WrapUnary(next connect.UnaryFunc) connect.UnaryFunc {
    return func(ctx context.Context, req connect.AnyRequest) (connect.AnyResponse, error) {
        start := time.Now()
        resp, err := next(ctx, req)
        log.Printf("%s took %v", req.Spec().Procedure, time.Since(start))
        return resp, err
    }
}

// Implement other methods: WrapStreamingClient, WrapStreamingHandler

// Use interceptors
path, handler := authv1connect.NewAuthServiceHandler(
    &AuthHandler{},
    connect.WithInterceptors(
        &LoggingInterceptor{},
        &AuthInterceptor{},
    ),
)
```

## Error Handling (Connect)

```go
import "connectrpc.com/connect"

// Return error
return nil, connect.NewError(connect.CodeNotFound, errors.New("user not found"))

// Check error code
if connect.CodeOf(err) == connect.CodeNotFound {
    // Handle not found
}

// Common codes:
// - connect.CodeInvalidArgument
// - connect.CodeNotFound
// - connect.CodeUnauthenticated
// - connect.CodePermissionDenied
// - connect.CodeInternal
```

## Версионирование

### Semver

```
v1.0.0 - первый стабильный релиз
v1.1.0 - добавлены новые методы (backward compatible)
v1.1.1 - исправления
v2.0.0 - breaking changes
```

### Breaking Changes

Что считается breaking change:
- Удаление поля
- Изменение типа поля
- Изменение номера поля
- Удаление метода
- Изменение сигнатуры метода

Что НЕ breaking change:
- Добавление нового поля
- Добавление нового метода
- Добавление нового enum value

## Проверка перед релизом

```bash
# Lint
buf lint

# Breaking changes против main
buf breaking --against '.git#branch=main'

# Генерация
buf generate

# Commit
git add .
git commit -m "feat: add new method to AuthService"
git tag v1.1.0
git push --tags
```
