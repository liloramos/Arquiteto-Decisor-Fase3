# Observability - FinTech Wallet

## 1. Contexto

A FinTech Wallet possui microsserviços, API Gateway, RabbitMQ, Keycloak e bancos PostgreSQL independentes. Esse desenho melhora isolamento e escalabilidade, mas torna incidentes mais difíceis de diagnosticar sem observabilidade consistente.

Observabilidade é requisito arquitetural para operar segurança, disponibilidade, desempenho e confiabilidade. Sem logs estruturados, métricas e tracing, padrões como Circuit Breaker e Bulkhead perdem efetividade operacional.

## 2. Decisão

A plataforma adotará observabilidade baseada em três pilares:

- logs estruturados;
- métricas técnicas e de negócio;
- tracing distribuído.

Todos os serviços devem propagar `correlationId` em chamadas HTTP e mensagens RabbitMQ. Eventos também devem carregar `causationId` para reconstruir fluxos assíncronos.

## 3. Alternativas Rejeitadas

### Logs textuais sem padronização

Rejeitado porque dificultam busca, correlação e alertas. Em incidentes financeiros, mensagens livres não bastam para reconstruir a jornada de uma transação.

### Métricas apenas de infraestrutura

Rejeitado porque CPU, memória e disco não explicam sozinhos falhas de pagamento, filas acumuladas ou rejeições de autorização.

### Tracing apenas na borda

Rejeitado porque chamadas internas e eventos assíncronos são justamente onde ocorrem muitos atrasos e perdas de contexto.

## 4. Logs Estruturados

Formato recomendado: JSON.

Campos mínimos:

```json
{
  "timestamp": "2026-06-06T10:15:30Z",
  "level": "INFO",
  "service": "payment-service",
  "operation": "authorizePayment",
  "correlationId": "corr-7f3c",
  "causationId": "req-93a1",
  "userId": "usr-123",
  "walletId": "wal-456",
  "paymentId": "pay-789",
  "message": "Payment authorization requested"
}
```

Dados proibidos em logs:

- senha;
- refresh token;
- access token completo;
- número completo de cartão;
- dados sensíveis sem mascaramento;
- payload bancário bruto.

## 5. Métricas Técnicas

### API Gateway Kong

- taxa de requisições por rota;
- latência p50, p95 e p99;
- respostas 4xx e 5xx;
- rejeições por rate limit;
- tokens inválidos ou expirados.

### Keycloak

- tentativas de login;
- falhas de autenticação;
- uso de MFA;
- emissão de tokens;
- latência dos endpoints OIDC.

### Microsserviços

- latência por endpoint;
- throughput por operação;
- taxa de erro por dependência;
- circuit breakers abertos;
- tempo de acesso ao banco;
- número de retries.

### RabbitMQ

- profundidade das filas;
- idade da mensagem mais antiga;
- taxa de publicação;
- taxa de consumo;
- mensagens em DLQ;
- consumidores ativos.

### PostgreSQL

- conexões ativas;
- queries lentas;
- locks;
- uso de disco;
- taxa de erro;
- tempo médio de consulta.

## 6. Métricas de Negócio

- pagamentos iniciados;
- pagamentos autorizados;
- pagamentos recusados;
- transferências concluídas;
- notificações enviadas;
- notificações falhadas;
- tempo médio entre PaymentAuthorized e TransactionRecorded;
- atraso de atualização do Report Service.

Essas métricas conectam saúde técnica a impacto real no usuário.

## 7. Tracing Distribuído

Cada requisição ou evento deve gerar uma trilha ponta a ponta.

Exemplo de fluxo:

```text
Kong -> Payment Service -> Gateway Bancário Externo
Payment Service -> RabbitMQ -> Transaction Service
Transaction Service -> RabbitMQ -> Notification Service
Transaction Service -> RabbitMQ -> Report Service
```

Regras:

- `traceId` acompanha a jornada completa.
- `spanId` identifica cada etapa.
- Eventos RabbitMQ propagam contexto no cabeçalho e no corpo.
- Erros externos são marcados com dependência, timeout e status.

## 8. Alertas

| Alerta | Condição | Severidade |
|---|---|---:|
| Pagamentos falhando | erro > 3% por 5 minutos | Alta |
| Circuit breaker aberto | Payment Service -> Gateway Bancário | Alta |
| DLQ crescendo | mais de 100 mensagens em 10 minutos | Alta |
| Latência crítica | p95 > 2s em pagamento | Alta |
| Login falhando | aumento anormal de falhas | Média |
| Report atrasado | lag > 15 minutos | Média |
| Disco PostgreSQL | uso > 80% | Alta |

## 9. Consequências

Consequências positivas:

- Redução do tempo médio de diagnóstico.
- Melhor rastreabilidade de pagamentos e transferências.
- Capacidade de validar se Circuit Breaker, Bulkhead e Retry estão funcionando.
- Base para auditoria técnica.

Consequências negativas:

- Aumento de volume de dados operacionais.
- Necessidade de retenção e mascaramento.
- Custo adicional de armazenamento e ferramentas.

## 10. Trade-offs

Observabilidade aumenta custo e trabalho operacional, mas reduz incerteza em incidentes. Em uma plataforma financeira, diagnosticar rapidamente uma falha de pagamento tem valor maior que economizar armazenamento de logs e métricas.

## 11. Referências

- Nygard, Michael T. *Release It!*. Pragmatic Bookshelf.
- Newman, Sam. *Building Microservices*. O'Reilly Media.
- OpenTelemetry. *Observability Framework and Specification*.
