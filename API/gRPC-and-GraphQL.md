# RPC, gRPC и GraphQL — обзор для собеседования

**RPC** описывает вызов удалённого метода, хотя сеть добавляет таймауты, частичные сбои и повторы. `JSON-RPC` уже упоминается в [[API/REST-and-Protocols]]. **gRPC** определяет сервис и сообщения в `.proto`, обычно использует Protocol Buffers и HTTP/2; предоставляет unary и streaming вызовы. Контракт важен для совместимости, deadline — для ограничения ожидания, а retry допустим только с учётом идемпотентности операции.

```proto
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}
message GetUserRequest { int64 id = 1; }
message User { int64 id = 1; string name = 2; }
```

**GraphQL** — язык запросов и схема API: клиент указывает нужные поля, сервер резолвит их. Удобен для сложных клиентских представлений, но надо следить за глубиной/сложностью запросов, N+1 при resolvers, авторизацией полей и кешированием. Это **два разных подхода к контракту**, прямого «лучше REST» нет.

Тебе нужны только уверенные базовые ответы: ты не заявлял опыта работы с gRPC/GraphQL. Если спросят, честно назови механизм, сценарий применения и важный недостаток.

Источники: [gRPC introduction](https://grpc.io/docs/what-is-grpc/introduction/), [GraphQL Learn](https://graphql.org/learn/).
