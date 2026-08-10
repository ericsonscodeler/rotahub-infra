# rotahub-infra

Orquestração local e documentação de contrato do RotaHub. Não contém código de aplicação.

- `docker-compose.yml` — sobe Postgres, MongoDB, RabbitMQ e os serviços/BFF/frontend para rodar tudo localmente com um comando (referencia os outros repositórios/imagens)
- `docs/contracts.md` — contratos REST entre os serviços e payload dos eventos assíncronos

## Repositórios do RotaHub

- `orders-service` — Java + Spring Boot + Postgres
- `tracking-service` — Java + Spring Boot + MongoDB
- `rotahub-bff` — Node + NestJS
- `rotahub-web` — React + Vite (Painel do Operador)
- `rotahub-infra` — este repositório

## Rodando localmente

> TODO: preencher `docker-compose.yml` conforme os serviços forem criados.
