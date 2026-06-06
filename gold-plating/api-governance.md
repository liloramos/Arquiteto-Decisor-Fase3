# API Governance - FinTech Wallet

## 1. Contexto

A FinTech Wallet expõe APIs para usuários, administradores e integrações internas entre microsserviços. Em uma arquitetura distribuída, APIs e eventos são contratos de evolução. Sem governança, alterações pequenas podem quebrar consumidores, gerar falhas em cascata ou abrir brechas de segurança.

Este documento define regras para versionamento, autenticação, autorização, contratos, erros, paginação, idempotência e eventos.

## 2. Decisão

A plataforma adotará governança de API baseada em:

- contratos REST versionados;
- eventos RabbitMQ versionados;
- autenticação OAuth2/OIDC;
- autorização por escopos e RBAC;
- padronização de erros;
- idempotência em comandos financeiros;
- documentação OpenAPI para APIs REST;
- schema versionado para eventos.

## 3. Alternativas Rejeitadas

### APIs sem versionamento explícito

Rejeitado porque clientes mobile, web e serviços internos podem evoluir em ritmos diferentes. Mudanças incompatíveis sem versão quebram consumidores existentes.

### Autorização apenas por endpoint

Rejeitado porque operações financeiras exigem autorização por recurso e contexto de domínio. Um usuário pode ter acesso ao endpoint de carteira, mas não a uma carteira que não pertence a ele.

### Eventos sem contrato

Rejeitado porque consumidores assíncronos dependem de campos estáveis. Eventos informais dificultam compatibilidade e reprocessamento.

## 4. Convenções REST

### Versionamento

APIs públicas devem usar versão no caminho:

```text
/api/v1/wallets
/api/v1/payments
/api/v1/transactions
/api/v1/reports
```

Mudanças compatíveis:

- adicionar campo opcional;
- adicionar endpoint;
- ampliar enum de forma documentada.

Mudanças incompatíveis:

- remover campo;
- alterar significado de campo;
- mudar tipo;
- tornar campo opcional em obrigatório;
- alterar semântica de status HTTP.

Mudanças incompatíveis exigem nova versão.

### Status HTTP

| Código | Uso |
|---:|---|
| 200 | Consulta ou comando concluído |
| 201 | Recurso criado |
| 202 | Comando aceito para processamento assíncrono |
| 400 | Erro de validação |
| 401 | Token ausente ou inválido |
| 403 | Token válido sem permissão |
| 404 | Recurso inexistente ou inacessível |
| 409 | Conflito, duplicidade ou idempotência |
| 422 | Regra de negócio inválida |
| 429 | Rate limit |
| 500 | Erro inesperado |
| 503 | Dependência indisponível |

### Erros

Formato padrão:

```json
{
  "error": "PAYMENT_LIMIT_EXCEEDED",
  "message": "Payment amount exceeds the configured limit.",
  "correlationId": "corr-7f3c",
  "details": [
    {
      "field": "amount",
      "reason": "must be less than or equal to daily limit"
    }
  ]
}
```

### Paginação

Endpoints de coleção devem suportar paginação:

```text
GET /api/v1/transactions?page=1&pageSize=50
```

Limites:

- `pageSize` padrão: 20;
- `pageSize` máximo: 100;
- ordenação padrão por data decrescente.

## 5. Segurança de API

### Escopos OAuth2

Escopos recomendados:

| Escopo | Permite |
|---|---|
| wallet:read | Consultar carteira |
| wallet:write | Alterar carteira |
| payment:create | Criar pagamento |
| transaction:read | Consultar transações |
| report:read | Consultar relatórios |
| notification:read | Consultar notificações |
| admin:manage | Administração |

### Regras

- Kong valida token, issuer, audience e expiração.
- Serviços validam escopos e propriedade do recurso.
- Tokens expirados não devem ser aceitos por tolerância local.
- Operações administrativas exigem MFA.

## 6. Idempotência

Comandos financeiros devem aceitar `Idempotency-Key`.

Aplicável a:

- criação de pagamento;
- transferência;
- baixa de saldo;
- reprocessamento de evento crítico.

Regra:

- mesma chave e mesmo payload retornam o mesmo resultado;
- mesma chave com payload diferente retorna 409;
- chaves expiram conforme política operacional.

## 7. Governança de Eventos

Formato mínimo:

```json
{
  "eventId": "evt-123",
  "eventType": "PaymentAuthorized",
  "schemaVersion": "1.0",
  "occurredAt": "2026-06-06T10:15:30Z",
  "correlationId": "corr-7f3c",
  "causationId": "cmd-456",
  "producer": "payment-service",
  "payload": {
    "paymentId": "pay-789",
    "walletId": "wal-456",
    "amount": 150.75,
    "currency": "BRL"
  }
}
```

Regras:

- eventos são fatos ocorridos, não comandos disfarçados;
- consumidores devem ser idempotentes;
- eventos incompatíveis exigem nova versão;
- DLQ deve preservar payload original e motivo de falha;
- eventos financeiros devem ser auditáveis.

## 8. Consequências

Consequências positivas:

- Evolução mais segura dos serviços.
- Redução de quebras entre consumidores.
- Melhor auditabilidade.
- Contratos claros para APIs e eventos.

Consequências negativas:

- Maior disciplina de documentação.
- Necessidade de revisão de contratos antes de mudanças.
- Mais esforço para manter compatibilidade.

## 9. Trade-offs

Governança reduz velocidade aparente em mudanças simples, mas evita regressões caras em integrações financeiras. Para uma FinTech, previsibilidade de contrato é mais importante que alterações rápidas sem controle.

## 10. Referências

- Newman, Sam. *Building Microservices*. O'Reilly Media.
- OWASP. *API Security Top 10*.
- Fielding, Roy T. *Architectural Styles and the Design of Network-based Software Architectures*.
