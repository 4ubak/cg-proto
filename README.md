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
├── buf.yaml                  # Buf конфигурация
├── buf.gen.yaml              # Генерация кода
└── go.mod
```

## Установка

```bash
go get gitlab.com/xakpro/cg-proto@latest
```

## Использование

```go
import (
    authpb "gitlab.com/xakpro/cg-proto/gen/go/users/auth/v1"
    adpb "gitlab.com/xakpro/cg-proto/gen/go/marketplace/ad/v1"
)

// Создать клиент
conn, _ := grpc.Dial("auth-service:50051", grpc.WithInsecure())
client := authpb.NewAuthServiceClient(conn)

// Вызвать метод
resp, err := client.ValidateToken(ctx, &authpb.ValidateTokenRequest{
    AccessToken: token,
})
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
option go_package = "gitlab.com/xakpro/cg-proto/gen/go/{domain}/{service}/v1";
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

## Зависимости между proto

```protobuf
// Можно импортировать общие типы
import "google/protobuf/timestamp.proto";

// НЕ импортировать proto других доменов
// ❌ import "marketplace/ad.proto";
// Вместо этого дублировать нужные поля или использовать ID
```

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

## Миграция

При добавлении нового поля:

```protobuf
message User {
  int64 id = 1;
  string phone = 2;
  string name = 3;
  string avatar_url = 4;  // Новое поле - добавляем в конец
}
```

При deprecation:

```protobuf
message User {
  int64 id = 1;
  string phone = 2;
  string name = 3;
  
  // Deprecated: use avatar_url instead
  string avatar = 4 [deprecated = true];
  string avatar_url = 5;
}
```

