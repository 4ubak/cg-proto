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
│   │   ├── auth.pb.go        # Protobuf типы
│   │   ├── auth_grpc.pb.go   # Стандартный gRPC сервисы
│   │   ├── authv1connect/     # Connect handlers & clients
│   │   │   └── auth.connect.go
│   │   └── ...
├── buf.yaml                  # Buf конфигурация
├── buf.gen.yaml              # Генерация кода (Connect + gRPC)
└── go.mod
```

## Установка

```bash
go get gitlab.com/xakpro/cg-proto@latest
```

## API протоколы: Connect-go и стандартный gRPC

В платформе используются **два подхода** в зависимости от требований сервиса:

### 1. Connect-go (HTTP + gRPC) — для публичных API

**Когда использовать:**
- Сервисы, которым нужен доступ из фронтенда (HTTP/JSON)
- Сервисы, которые должны поддерживать и HTTP/JSON, и gRPC одновременно
- Примеры: `auth`, `user`, `organization`, `counter`, `nsi`

**Connect-go** даёт:
- **gRPC** для сервис-к-сервис коммуникации
- **HTTP/JSON** для фронтенда (без прокси)
- **gRPC-Web** для браузеров

### 2. Стандартный gRPC — для внутренних сервисов

**Когда использовать:**
- Сервисы, которые используются только для сервис-к-сервис коммуникации
- Сервисы, которым не нужен HTTP/JSON доступ
- Примеры: `search`, `catalog`, `notification`, `chat`

### buf.gen.yaml

```yaml
version: v1
plugins:
  # Standard protobuf Go code
  - plugin: go
    out: gen/go
    opt:
      - paths=source_relative

  # Connect-go (HTTP/JSON + gRPC)
  - plugin: buf.build/connectrpc/go
    out: gen/go
    opt:
      - paths=source_relative

  # Standard gRPC (для внутренних сервисов)
  - plugin: buf.build/protocolbuffers/go
    out: gen/go
    opt:
      - paths=source_relative
```

**Генерируемые файлы:**
- `.pb.go` — protobuf типы (общие для обоих)
- `*_grpc.pb.go` — стандартный gRPC сервисы (для стандартного gRPC)
- `*connect/*.connect.go` — Connect handlers & clients (для Connect)

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

---

## Стандартный gRPC (для внутренних сервисов)

### Server (gRPC Server)

```go
import (
    pb "gitlab.com/xakpro/cg-proto/gen/go/services/search"
    sharedGRPC "gitlab.com/xakpro/cg-shared-libs/grpc"
)

// Implement the gRPC service interface
type SearchHandler struct {
    pb.UnimplementedSearchServiceServer
    searchService SearchServiceI
}

func (h *SearchHandler) Search(
    ctx context.Context,
    req *pb.SearchRequest,
) (*pb.SearchResponse, error) {
    // Your logic here
    return &pb.SearchResponse{
        Results: []*pb.SearchResult{},
    }, nil
}

// Register handler
func main() {
    grpcServer, err := sharedGRPC.NewServer(cfg.GRPC)
    if err != nil {
        log.Fatal(err)
    }
    
    pb.RegisterSearchServiceServer(grpcServer.Server(), &SearchHandler{})
    
    if err := grpcServer.Start(); err != nil {
        log.Fatal(err)
    }
}
```

### Client (gRPC Client)

```go
import (
    pb "gitlab.com/xakpro/cg-proto/gen/go/services/search"
    sharedGRPC "gitlab.com/xakpro/cg-shared-libs/grpc"
)

// Create client
conn, err := sharedGRPC.NewClient(ctx, cfg)
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

client := pb.NewSearchServiceClient(conn.Conn())

// Call method
resp, err := client.Search(ctx, &pb.SearchRequest{
    Query: "test",
})
if err != nil {
    // Handle error (status.Code(err) returns error code)
}
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

## Interceptors

### Connect Interceptors

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

### Стандартный gRPC Interceptors

```go
import (
    "google.golang.org/grpc"
    sharedGRPC "gitlab.com/xakpro/cg-shared-libs/grpc"
)

// Используйте interceptors из cg-shared-libs/grpc
grpcServer, err := sharedGRPC.NewServer(cfg.GRPC, 
    grpc.ChainUnaryInterceptor(
        customInterceptor1,
        customInterceptor2,
    ),
)

// Или используйте встроенные interceptors:
// - recoveryInterceptor (автоматически)
// - loggingInterceptor (автоматически)
// - timeoutInterceptor (автоматически)
// - AuthInterceptor (через sharedGRPC.AuthInterceptor)
```

## Error Handling

### Connect Error Handling

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

### Стандартный gRPC Error Handling

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Return error
return nil, status.Error(codes.NotFound, "user not found")

// Check error code
if status.Code(err) == codes.NotFound {
    // Handle not found
}

// Common codes:
// - codes.InvalidArgument
// - codes.NotFound
// - codes.Unauthenticated
// - codes.PermissionDenied
// - codes.Internal
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

# Commit и создание тега
git add .
git commit -m "feat: add new method to AuthService"
git tag -a v1.1.0 -m "Release v1.1.0: описание изменений"
git push origin main
git push origin v1.1.0
git push github v1.1.0  # если используете GitHub mirror
```

## Обновление зависимостей в других проектах

После создания нового тега версии, обновите зависимость в проектах:

### Для проектов с GitHub replace

```bash
# В каждом сервисе выполните:
cd cg-users/services/auth
go get gitlab.com/xakpro/cg-proto@v1.1.0
go mod tidy

# Или используйте скрипт:
./cg-proto/update-dependencies.sh v1.1.0
```

### Для проектов с локальным replace

Проекты с `replace gitlab.com/xakpro/cg-proto => ../cg-proto` автоматически используют последнюю версию из локальной директории. Просто выполните:

```bash
cd cg-services
go mod tidy
```

### Автоматическое обновление

Используйте скрипт для автоматического обновления всех проектов:

```bash
cd cg-proto
./update-dependencies.sh v1.1.0
```
