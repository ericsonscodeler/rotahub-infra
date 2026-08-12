# RotaHub — Observabilidade

Tracing distribuído (OpenTelemetry + Jaeger) e métricas (Prometheus + Grafana) sobre os 4
serviços: `rotahub-bff`, `orders-service`, `tracking-service`, `routing-service`. Sem OTel
Collector como hop extra — cada serviço exporta trace direto pro Jaeger e expõe métricas que o
Prometheus raspa direto, o que já demonstra o essencial sem uma peça de infra a mais pra manter
localmente.

## O que está instrumentado

- **Tracing**: os 3 serviços Java usam Micrometer Tracing + bridge OpenTelemetry (autoconfiguração
  do Spring Boot); o BFF usa o OpenTelemetry SDK para Node com auto-instrumentação
  (`@opentelemetry/auto-instrumentations-node`, cobrindo HTTP/Express/Axios sem precisar tocar
  em cada controller/service). Sampling em 100% (`management.tracing.sampling.probability=1.0`) —
  proposital para dev local, seria reduzido em produção.
- **Propagação de contexto**: W3C Trace Context (`traceparent` header) é o default tanto no SDK
  Node quanto no Micrometer Tracing — nenhuma configuração extra foi necessária pra BFF e
  serviços Java compartilharem o mesmo trace nas chamadas REST síncronas.
- **RabbitMQ também propaga o trace**: com
  `spring.rabbitmq.template.observation-enabled=true` e
  `spring.rabbitmq.listener.simple.observation-enabled=true`, o span de publicação do
  `tracking-service` (`rotahub.events/delivery.completed send`) e o span de consumo do
  `orders-service` (`orders-service.delivery-completed receive`) aparecem no **mesmo trace** que
  disparou o evento — confirmado na prática, não só na teoria: um fluxo completo (criar pedido →
  marcar `DELIVERED`) gera um único trace com spans de `rotahub-bff`, `tracking-service` e
  `orders-service`, incluindo o hop assíncrono.
- **Métricas**: os serviços Java expõem `/actuator/prometheus` (Micrometer); o BFF expõe
  `:9464/metrics` (exportador Prometheus do OTel SDK). São dois formatos com convenções de nome
  diferentes (`http.server.requests` no Micrometer vs `http.server.duration` no OTel) — por isso
  não há um dashboard Grafana pré-fabricado tentando unificar os dois; o Prometheus/Grafana ficam
  provisionados pra explorar ad hoc (aba Explore), o que já é suficiente pra provar a instrumentação
  sem fingir uma unificação que não existe.

## Como ver um trace

1. Suba a infra (`docker compose up -d` em `rotahub-infra`) e os 4 serviços de aplicação
   normalmente.
2. Dispare um fluxo pelo BFF (criar pedido, depois marcar um evento `DELIVERED`).
3. Abra `http://localhost:16686` (Jaeger UI), selecione o serviço `rotahub-bff` e procure o trace
   mais recente — vai ter spans de `rotahub-bff`, `orders-service` e `tracking-service` no mesmo
   trace id.

## Portas

| Ferramenta | URL |
|---|---|
| Jaeger UI | http://localhost:16686 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3001 (acesso anônimo como admin — infra só local, sem exposição externa) |

Datasources do Grafana (Prometheus e Jaeger) já vêm provisionados via
`observability/grafana-datasources.yml` — não precisa configurar nada na primeira vez que abrir.

## Pegadinha: host vs Docker

Os 4 serviços de aplicação rodam no **host** (fora do Docker), só a infra (Postgres, Mongo,
RabbitMQ, Jaeger, Prometheus, Grafana) roda em container — mesmo padrão do resto do projeto. Isso
inverte a direção normal de rede em dois pontos:

- Os serviços exportam trace pra `localhost:4318` (Jaeger publica essa porta pro host) — fácil,
  sentido host → container.
- O Prometheus, rodando dentro do container, precisa raspar `/actuator/prometheus` e `/metrics`
  nos serviços que estão no host — por isso `observability/prometheus.yml` usa
  `host.docker.internal:<porta>` como alvo em vez de `localhost`, já que de dentro do container
  `localhost` seria o próprio Prometheus.

## Decisão registrada: Spring Boot 4 mudou a autoconfiguração de tracing

Igual já tinha acontecido com Mongo e Jackson nesse projeto, o Spring Boot 4.1 quebrou o caminho
"óbvio" de configurar tracing silenciosamente:

- `io.micrometer:micrometer-tracing-bridge-otel` sozinho **não é suficiente** — a autoconfiguração
  do Spring Boot que liga tudo (properties, exporter OTLP, bean `Tracer`) foi extraída pra um módulo
  próprio, `org.springframework.boot:spring-boot-micrometer-tracing-opentelemetry`. Sem ele, a app
  sobe normalmente e cai silenciosamente num `NoopTracer` — nenhum erro, nenhum log, só nenhum
  trace aparecendo em lugar nenhum.
- A propriedade `management.otlp.tracing.endpoint` (a documentada pra Boot 3.x) foi renomeada pra
  `management.opentelemetry.tracing.export.otlp.endpoint`, com depreciação em nível "error" — ou
  seja, silenciosamente ignorada, sem warning de binding.

Nenhum desses dois problemas aparece nos logs por padrão; só foram descobertos inspecionando o
relatório de condition evaluation do Spring Boot (`--debug`) e o conteúdo dos jars em
`~/.m2/repository`.

## Fora do escopo (por enquanto)

Sem OTel Collector, sem dashboards Grafana pré-fabricados, sem alerting (Alertmanager). Sampling
em 100% é aceitável só porque isso é dev local com baixíssimo volume — não é uma recomendação de
produção.
