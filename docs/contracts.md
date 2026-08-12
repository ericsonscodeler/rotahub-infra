# RotaHub — Contratos (MVP)

> Contrato entre os serviços antes de escrever código. Domínio: encomendas/entregas genéricas
> (estilo transportadora — pacotes de origem A para destino B, sem itens de catálogo).
>
> Convenção de idioma: nomes de domínio no código (entidades, rotas, campos, enums) em **inglês**;
> esta documentação em português.
>
> Repositórios: `orders-service`, `tracking-service`, `routing-service`, `notification-service`,
> `rotahub-bff`, `rotahub-web`, `rotahub-web-shell`, `rotahub-customer-web`, `rotahub-infra`.

---

## Entidades

### Order (dono: orders-service / Postgres)
- `id` (UUID)
- `trackingCode` (string)
- `sender` { name, address, email }
- `recipient` { name, address, email }
- `status`: `CREATED → IN_TRANSIT → DELIVERED` (ou `CANCELLED`)
- `createdAt`, `updatedAt`

### Tracking (dono: tracking-service / MongoDB)
- `id`
- `orderId` (referência lógica ao Order — nunca FK direta)
- `status`: `AWAITING_PICKUP → PICKED_UP → IN_TRANSIT → OUT_FOR_DELIVERY → DELIVERED` (ou `FAILED_ATTEMPT`)
- `position` opcional { lat, lng }
- `history`: lista de eventos (status, timestamp, note, position)
- `createdAt`

### Route (dono: routing-service / Postgres)
- `id` (UUID)
- `status`: `PLANNED → IN_PROGRESS → COMPLETED`
- `stops`: lista ordenada de { orderId, address, lat, lng } — a ordem já é a otimizada
- `totalDistanceKm` (calculado, não persistido)
- `createdAt`

### Notification (dono: notification-service / Postgres)
- `id` (UUID)
- `orderId` (referência lógica ao Order — nunca FK direta)
- `recipientEmail`, `recipientName`
- `trackingStatus` (string crua do evento recebido, ex: `PICKED_UP` — não é o enum
  `TrackingStatus` do tracking-service; ver nota de acoplamento abaixo)
- `subject`, `body`
- `createdAt`

---

## REST — orders-service (porta 8081)

| Método | Rota | Descrição |
|---|---|---|
| POST | `/orders` | Cria pedido → `201`, status inicial `CREATED` |
| GET | `/orders/{id}` | Busca por id |
| GET | `/orders/by-tracking-code/{trackingCode}` | Busca pelo código de rastreio (usado pelo Acompanhamento do Cliente, que não conhece o UUID interno) |
| GET | `/orders?status=&page=&size=` | Lista paginada |
| PATCH | `/orders/{id}/status` | Atualização manual (ex: cancelamento) — uso interno/admin |

## REST — tracking-service (porta 8082)

| Método | Rota | Descrição |
|---|---|---|
| POST | `/trackings` | `{ orderId }` → cria registro inicial (`AWAITING_PICKUP`) |
| POST | `/trackings/{orderId}/events` | `{ status, position?, timestamp, note? }` → registra evento na timeline. Se `status == DELIVERED`, publica o evento assíncrono |
| GET | `/trackings/{orderId}` | Status atual + histórico completo |

## REST — routing-service (porta 8083)

| Método | Rota | Descrição |
|---|---|---|
| POST | `/routes` | `{ stops: [{ orderId, address, lat, lng }] }` → otimiza a ordem (vizinho mais próximo, distância haversine) e cria a rota → `201` |
| GET | `/routes/{id}` | Busca por id, com `totalDistanceKm` recalculado |

**Nota honesta:** a "otimização" é uma heurística de vizinho mais próximo sobre distância em
linha reta (haversine) entre coordenadas — não é uma rota real de estrada. Não há integração com
Google Maps/OSRM/nenhuma API de roteamento real (exigiria chave paga, fora do escopo do projeto).
O objetivo é demonstrar um serviço de domínio com um algoritmo real, não simular uma integração
externa que não existe.

## REST — notification-service (porta 8084)

| Método | Rota | Descrição |
|---|---|---|
| GET | `/notifications/{orderId}` | Lista notificações do pedido, mais recente primeiro |

**Nota honesta:** não há envio de e-mail real (SMTP, SendGrid, etc.) — "notificar" aqui é montar
assunto/corpo a partir de um template por status, logar em `INFO` e persistir um registro
consultável. Ver `notification-service/README.md`.

## Eventos assíncronos

RabbitMQ, exchange `rotahub.events` (topic).

### `delivery.completed`

Routing key `delivery.completed`. Publicado por `tracking-service` quando o status vira
`DELIVERED`, consumido por `orders-service` (seta `Order.status = DELIVERED`).

```json
{
  "eventId": "uuid",
  "eventType": "delivery.completed",
  "occurredAt": "2026-08-10T14:30:00Z",
  "orderId": "uuid",
  "trackingId": "uuid",
  "deliveredAt": "2026-08-10T14:30:00Z",
  "finalPosition": { "lat": -23.55, "lng": -46.63 }
}
```

### `tracking.status-changed`

Routing key `tracking.status-changed`. Publicado por `tracking-service` em **toda** mudança de
status do rastreio (não só `DELIVERED`). Consumido por `notification-service` — segundo
consumidor independente do `tracking-service`, demonstrando fan-out (um evento publicado, dois
serviços reagindo sem se conhecerem).

```json
{
  "eventId": "uuid",
  "eventType": "tracking.status-changed",
  "occurredAt": "2026-08-10T14:30:00Z",
  "orderId": "uuid",
  "trackingId": "uuid",
  "status": "PICKED_UP",
  "position": { "lat": -23.55, "lng": -46.63 },
  "note": "string ou null"
}
```

## BFF — rotahub-bff (porta 3000)

Único ponto de entrada para o frontend. Nunca fala com Postgres/Mongo/RabbitMQ diretamente — só REST síncrono com `orders-service`, `tracking-service` e `routing-service`.

| Método | Rota | Descrição |
|---|---|---|
| POST | `/api/auth/login` | JWT simples |
| POST | `/api/orders` | Orquestra: cria no orders-service, depois inicializa tracking no tracking-service; retorna visão combinada |
| GET | `/api/orders/{id}` | Combina Orders + Tracking num único DTO (status do pedido + timeline) |
| GET | `/api/orders/by-tracking-code/{trackingCode}` | Mesma combinação acima, busca pelo código de rastreio — usado pelo `rotahub-customer-web` |
| GET | `/api/orders` | Lista paginada (sem tracking — evita N+1; timeline só no GET por id/código) |
| POST | `/api/orders/{id}/tracking-events` | Repassa pro tracking-service — usado pelo Painel do Operador pra simular avanço da entrega (não há app do entregador ainda) |
| POST | `/api/routes` | Repassa pro routing-service — cria e otimiza uma rota |
| GET | `/api/routes/{id}` | Repassa pro routing-service — busca rota por id |
| GET | `/api/orders/{id}/notifications` | Repassa pro notification-service — histórico de notificações do pedido |

**Nota de arquitetura:** a criação do tracking inicial é síncrona (BFF → tracking-service); só o
fechamento (Tracking `DELIVERED` → Order `DELIVERED`) é assíncrono via evento. Isso demonstra os
dois estilos de comunicação sem acoplamento desnecessário.

---

## Fora do MVP (fases seguintes)

React Native, API Gateway.

Já implementados: CI/CD (GitHub Actions em todos os 9 repos), microfrontends federados
(`rotahub-web` e `rotahub-customer-web` como remotes via Module Federation, carregados por
`rotahub-web-shell`), Roteirização de ponta a ponta (`routing-service` → BFF → tela "Planejar
rota" no Painel do Operador), observabilidade completa (tracing distribuído via OpenTelemetry +
Jaeger, métricas via Prometheus + Grafana — ver [`docs/observability.md`](observability.md)) e
Notificações (`notification-service`, consumidor fan-out de `tracking.status-changed`).
