# Microservices Proto

Este repositório contém as definições de **Protocol Buffers (protobuf)** utilizadas como contratos de comunicação entre os microsserviços via **gRPC**.

O repositório tem como objetivo centralizar, versionar e documentar os contratos de comunicação, mantendo total desacoplamento entre os serviços e suas implementações.

---

## Estrutura

```
pf-microservices-proto/
├── order/
│   └── order.proto              # Definições de mensagens e serviço Order
├── payment/
│   └── payment.proto            # Definições de mensagens e serviço Payment
├── shipping/
│   └── shipping.proto           # Definições de mensagens e serviço Shipping
├── golang/
│   ├── order/                   # Código Go gerado para Order
│   ├── payment/                 # Código Go gerado para Payment
│   └── shipping/                # Código Go gerado para Shipping
├── buf.yaml                     # Configuração do Buf (lint e breaking changes)
├── buf.gen.yaml                 # Configuração de geração de código
├── go.mod                       # Módulo Go para distribuição dos stubs
├── go.sum
└── README.md
```

---

## Serviços Definidos

### Order Service

**Arquivo:** `order/order.proto`

```protobuf
service Order {
  rpc Create (CreateOrderRequest) returns (CreateOrderResponse) {}
}

message CreateOrderRequest {
  int32 customer_id = 1;
  repeated OrderItem order_items = 2;
}

message OrderItem {
  string product_code = 1;
  float unit_price = 2;
  int32 quantity = 3;
}

message CreateOrderResponse {
  int32 order_id = 1;
}
```

O serviço **Order** é responsável por receber pedidos, orquestrar chamadas para os serviços de **Payment** e **Shipping**, e retornar o identificador do pedido criado.

---

### Payment Service

**Arquivo:** `payment/payment.proto`

```protobuf
service Payment {
  rpc Create (CreatePaymentRequest) returns (CreatePaymentResponse) {}
}

message CreatePaymentRequest {
  int64 user_id = 1;
  int64 order_id = 2;
  float total_price = 3;
}

message CreatePaymentResponse {
  int64 payment_id = 1;
}
```

O serviço **Payment** é responsável por validar regras de pagamento, rejeitar valores inválidos e persistir informações relacionadas ao pagamento.

---

### Shipping Service

**Arquivo:** `shipping/shipping.proto`

```protobuf
service Shipping {
  rpc Create (CreateShippingRequest) returns (CreateShippingResponse) {}
}

message ShippingItem {
  string product_code = 1;
  int32 quantity = 2;
}

message CreateShippingRequest {
  int64 order_id = 1;
  repeated ShippingItem items = 2;
}

message CreateShippingResponse {
  int32 delivery_days = 1;
}
```

O serviço **Shipping** é responsável por calcular o prazo de entrega e registrar informações de envio associadas ao pedido.

---

## Como Compilar os Protos

Os arquivos `.proto` podem ser compilados para gerar **stubs Go**, que ficam disponíveis na pasta `golang/` e são utilizados diretamente pelos microsserviços.

---

### Pré-requisitos

- Protocol Buffer Compiler (`protoc`)
- Plugins Go:
  - `protoc-gen-go`
  - `protoc-gen-go-grpc`

---

### Instalação dos Plugins Go

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

---

### Compilação Manual com protoc

```bash
protoc --go_out=./golang --go-grpc_out=./golang order/order.proto
protoc --go_out=./golang --go-grpc_out=./golang payment/payment.proto
protoc --go_out=./golang --go-grpc_out=./golang shipping/shipping.proto
```

---

### Compilação Usando Buf (Recomendado)

Este repositório utiliza **Buf** para padronizar validação e geração dos protos.

```bash
buf generate
```

---

### Compilação Usando Docker (Alternativa)

```bash
docker run --rm -v $(pwd):/workspace -w /workspace bufbuild/buf:latest generate
```

---

## Estrutura dos Arquivos Gerados

Para cada serviço são gerados dois arquivos Go:

- `{service}.pb.go` — definições das mensagens
- `{service}_grpc.pb.go` — código cliente e servidor gRPC

Exemplo para o serviço Order:

```
golang/order/
├── order.pb.go
└── order_grpc.pb.go
```

---

## Uso em Microsserviços (Go)

Este repositório é publicado como um **módulo Go versionado** e pode ser consumido diretamente pelos microsserviços.

### Importação

```go
import (
	orderpb "github.com/phenrique-sousa/pf-microservices-proto/golang/order"
	"google.golang.org/grpc"
)
```

### Cliente gRPC

```go
conn, _ := grpc.Dial("order:3000", grpc.WithInsecure())
client := orderpb.NewOrderClient(conn)

response, _ := client.Create(ctx, &orderpb.CreateOrderRequest{...})
```

### Servidor gRPC

```go
s := grpc.NewServer()
orderpb.RegisterOrderServer(s, &OrderServer{})
```

---

## Versionamento

As definições de proto seguem **Semantic Versioning**.

- `v1.0.0` representa a versão inicial estável dos contratos
- Mudanças compatíveis não quebram versões anteriores
- Mudanças incompatíveis exigem incremento de versão maior

Os microsserviços dependem explicitamente de uma versão:

```go
require github.com/phenrique-sousa/pf-microservices-proto v1.0.0
```

---

## Boas Práticas

1. Nunca reutilizar números de campo já utilizados
2. Manter compatibilidade com versões anteriores
3. Separar contratos de implementação
4. Utilizar `snake_case` nos arquivos `.proto`
5. Versionar contratos independentemente dos microsserviços

---

## Observação Arquitetural

Este repositório **não possui Dockerfile**, pois não executa serviços nem participa do runtime da aplicação.  
A containerização é aplicada exclusivamente aos microsserviços que consomem estes contratos.

---

## Referências

- https://protobuf.dev/
- https://grpc.io/docs/languages/go/
- https://buf.build/docs
- https://pkg.go.dev/google.golang.org/protobuf
