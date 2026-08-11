# RotaHub — Contratos (MVP)

> Contrato entre os serviços antes de escrever código. Domínio: encomendas/entregas genéricas
> (estilo transportadora — pacotes de origem A para destino B, sem itens de catálogo).
>
> Convenção de idioma: nomes de domínio no código (entidades, rotas, campos, enums) em **inglês**;
> esta documentação em português.
>
> Repositórios: `orders-service`, `tracking-service`, `rotahub-bff`, `rotahub-web`, `rotahub-web-shell`,
> `rotahub-customer-web`, `rotahub-infra`.

---

## Entidades

### Order (dono: orders-service / Postgres)
- `id` (UUID)
- `trackingCode` (string)
- `sender` { name, address }
- `recipient` { name, address }
- `status`: `CREATED → IN_TRANSIT → DELIVERED` (ou `CANCELLED`)
- `createdAt`, `updatedAt`

### Tracking (dono: tracking-service / MongoDB)
- `id`
- `orderId` (referência lógica ao Order — nunca FK direta)
- `status`: `AWAITING_PICKUP → PICKED_UP → IN_TRANSIT → OUT_FOR_DELIVERY → DELIVERED` (ou `FAILED_ATTEMPT`)
- `position` opcional { lat, lng }
- `history`: lista de eventos (status, timestamp, note, position)
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

## Evento assíncrono — `delivery.completed`

RabbitMQ, exchange `rotahub.events` (topic), routing key `delivery.completed`.

Publicado por `tracking-service`, consumido por `orders-service` (seta `Order.status = DELIVERED`).

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

## BFF — rotahub-bff (porta 3000)

Único ponto de entrada para o frontend. Nunca fala com Postgres/Mongo/RabbitMQ diretamente — só REST síncrono com `orders-service` e `tracking-service`.

| Método | Rota | Descrição |
|---|---|---|
| POST | `/api/auth/login` | JWT simples |
| POST | `/api/orders` | Orquestra: cria no orders-service, depois inicializa tracking no tracking-service; retorna visão combinada |
| GET | `/api/orders/{id}` | Combina Orders + Tracking num único DTO (status do pedido + timeline) |
| GET | `/api/orders/by-tracking-code/{trackingCode}` | Mesma combinação acima, busca pelo código de rastreio — usado pelo `rotahub-customer-web` |
| GET | `/api/orders` | Lista paginada (sem tracking — evita N+1; timeline só no GET por id/código) |
| POST | `/api/orders/{id}/tracking-events` | Repassa pro tracking-service — usado pelo Painel do Operador pra simular avanço da entrega (não há app do entregador ainda) |

**Nota de arquitetura:** a criação do tracking inicial é síncrona (BFF → tracking-service); só o
fechamento (Tracking `DELIVERED` → Order `DELIVERED`) é assíncrono via evento. Isso demonstra os
dois estilos de comunicação sem acoplamento desnecessário.

---

## Fora do MVP (fases seguintes)

Roteirização, Notificações, React Native, API Gateway, observabilidade completa
(OpenTelemetry/Prometheus/Grafana/Jaeger).

Já implementados: CI/CD (GitHub Actions em todos os 7 repos) e microfrontends federados
(`rotahub-web` e `rotahub-customer-web` como remotes via Module Federation, carregados por
`rotahub-web-shell`).
