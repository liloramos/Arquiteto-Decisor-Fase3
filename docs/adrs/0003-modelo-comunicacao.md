# ADR 0003 - Modelo de Comunicação

## Status

Aceito.

## Contexto

A FinTech Wallet possui operações com diferentes necessidades de consistência, latência e acoplamento. Autenticação, consultas de saldo e início de pagamento exigem resposta imediata ao usuário. Já notificações, atualização de relatórios e propagação de eventos financeiros podem ocorrer de forma assíncrona, desde que sejam rastreáveis e confiáveis.

Em microsserviços, a comunicação define o grau de acoplamento entre equipes, serviços e dados. Sam Newman recomenda avaliar cuidadosamente quando usar comunicação síncrona e quando adotar eventos, pois cada modelo traz custos operacionais, efeitos sobre consistência e impactos na experiência do usuário.

## Opções Avaliadas

### REST Síncrono

REST oferece chamadas diretas, contrato simples e resposta imediata. É adequado para consultas, autenticação, autorização e comandos que precisam informar resultado ao usuário no mesmo fluxo.

Vantagens:

- Simplicidade de entendimento e depuração.
- Boa aderência a APIs públicas e chamadas via API Gateway.
- Resposta imediata para interfaces de usuário.
- Facilidade de aplicar autenticação JWT e políticas de gateway.

Desvantagens:

- Acoplamento temporal entre cliente e serviço.
- Maior risco de falhas em cascata sem Circuit Breaker.
- Latência acumulada quando há chamadas encadeadas.

### RabbitMQ Assíncrono

RabbitMQ permite publicar eventos e processá-los posteriormente por consumidores independentes. É adequado para notificações, eventos de pagamento, atualização de relatórios e integrações que aceitam consistência eventual.

Vantagens:

- Desacoplamento temporal entre produtores e consumidores.
- Melhor absorção de picos.
- Facilita fan-out de eventos para notificação, relatório e auditoria.
- Reduz dependência direta entre serviços.

Desvantagens:

- Introduz consistência eventual.
- Exige idempotência nos consumidores.
- Requer observabilidade de filas, dead-letter queues e rastreamento de mensagens.
- Debug distribuído é mais complexo.

## Decisão

A FinTech Wallet adotará modelo híbrido de comunicação.

REST será usado para:

- autenticação e autorização com Keycloak;
- consultas de saldo, transações e relatórios;
- comandos iniciados pelo usuário que precisam de resposta imediata;
- comunicação entre API Gateway e microsserviços.

RabbitMQ será usado para:

- eventos de pagamento, como PaymentRequested, PaymentAuthorized e PaymentFailed;
- notificações transacionais;
- atualização assíncrona de relatórios;
- propagação de eventos de transação para serviços interessados.

## Alternativas Rejeitadas

Usar apenas REST foi rejeitado porque aumentaria o acoplamento entre serviços e tornaria fluxos de notificação e relatório dependentes da disponibilidade imediata de todos os consumidores.

Usar apenas RabbitMQ foi rejeitado porque autenticação, consultas e interações diretas do usuário exigem resposta síncrona e previsível. Transformar tudo em eventos aumentaria a complexidade da interface e prejudicaria a experiência do usuário.

## Consequências

Consequências positivas:

- Melhor equilíbrio entre experiência imediata e desacoplamento.
- Maior resiliência em fluxos secundários.
- Serviços de relatório e notificação podem evoluir sem bloquear pagamentos.
- Eventos permitem trilha de auditoria e reconstrução de projeções.

Consequências negativas:

- A arquitetura passa a lidar com consistência eventual.
- É necessário padronizar contratos de eventos.
- Observabilidade distribuída torna-se obrigatória.
- Consumidores precisam ser idempotentes para evitar efeitos duplicados.

## Trade-offs

O modelo híbrido evita extremos. REST simplifica fluxos que precisam de resposta imediata, enquanto RabbitMQ reduz acoplamento em fluxos naturalmente assíncronos. O principal custo é operacional: a plataforma precisa monitorar APIs, filas, dead-letter queues, latência de consumo e correlação entre eventos.

A consistência eventual é aceitável para notificações e relatórios, mas não para confirmação imediata de saldo e autorização de pagamento. Por isso, os limites de consistência são documentados por caso de uso.

## Observabilidade Necessária

- Correlation ID propagado em HTTP e mensagens AMQP.
- Métricas de latência REST, taxa de erro e abertura de circuit breaker.
- Métricas de fila: profundidade, idade da mensagem, taxa de consumo e mensagens em DLQ.
- Logs estruturados com identificadores de usuário, carteira, transação e pagamento.
- Rastreamento distribuído ponta a ponta.

## Referências

- Newman, Sam. *Building Microservices*. O'Reilly Media.
- Newman, Sam. *Monolith to Microservices*. O'Reilly Media.
- Nygard, Michael T. *Release It!*. Pragmatic Bookshelf.
