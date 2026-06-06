# src

Esta pasta foi mantida para atender à estrutura obrigatória do repositório acadêmico e para servir como ponto de entrada futuro da implementação.

Na Fase 3, a entrega principal é arquitetural: README, SAD, ADRs, diagramas e artefatos complementares. A arquitetura proposta define que, quando a implementação evoluir, cada microsserviço deverá preservar a Arquitetura Hexagonal adotada na Fase 2:

- domínio isolado de infraestrutura;
- portas de entrada para REST e consumidores de eventos;
- portas de saída para repositórios, autenticação, notificações e gateway de pagamento;
- adaptadores para PostgreSQL, RabbitMQ, Keycloak e integrações externas.

Estrutura futura sugerida:

```text
src/
├── wallet-service/
├── payment-service/
├── transaction-service/
├── notification-service/
├── report-service/
└── auth-service/
```
