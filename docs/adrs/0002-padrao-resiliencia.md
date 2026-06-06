# ADR 0002 - Padrão de Resiliência

## Status

Aceito.

## Contexto

A FinTech Wallet depende de chamadas entre microsserviços e integrações externas, especialmente no fluxo de pagamentos. Em uma arquitetura distribuída, latência, indisponibilidade parcial, timeouts e erros transitórios deixam de ser exceções raras e passam a fazer parte do comportamento esperado do sistema.

Michael Nygard descreve que sistemas integrados podem falhar em cascata quando uma dependência lenta consome threads, conexões ou filas de serviços chamadores. Sam Newman também enfatiza que microsserviços exigem mecanismos explícitos de isolamento e tolerância a falhas, pois a rede introduz modos de falha que não aparecem em uma aplicação monolítica.

## Opções Avaliadas

### API Gateway

O API Gateway atua como ponto de entrada controlado para a plataforma. Na FinTech Wallet, Kong concentra roteamento, TLS, rate limiting, validação inicial de JWT, políticas de borda e coleta de métricas.

Vantagens:

- Reduz exposição direta dos microsserviços.
- Aplica rate limiting e proteção contra abuso antes da carga chegar aos serviços.
- Centraliza políticas transversais de borda.
- Melhora observabilidade de tráfego externo.

Desvantagens:

- Torna-se componente crítico de disponibilidade.
- Não substitui autorização de domínio dentro dos serviços.
- Pode virar gargalo se não for monitorado e escalado.

### Circuit Breaker

O Circuit Breaker monitora falhas em chamadas a dependências e interrompe novas tentativas quando a taxa de erro ou timeout excede um limite. O serviço chamador passa a falhar rapidamente ou executar fallback, evitando saturação.

Vantagens:

- Reduz falhas em cascata.
- Implementa fail fast.
- Protege threads, conexões e pools de requisição.
- Permite recuperação gradual por estados fechado, aberto e meio aberto.

Desvantagens:

- Exige configuração cuidadosa de thresholds, janelas e timeouts.
- Pode negar chamadas durante recuperação se mal calibrado.
- Requer métricas para operação segura.

### Bulkhead

Bulkhead isola recursos por dependência, caso de uso ou classe de tráfego. Assim, uma falha no gateway bancário não consome todos os recursos do Payment Service ou do API Gateway.

Vantagens:

- Protege recursos críticos.
- Evita que uma dependência degrade toda a plataforma.
- Permite separar pools para pagamento, relatório, notificação e autenticação.
- Melhora previsibilidade operacional.

Desvantagens:

- Aumenta complexidade de configuração.
- Pode subutilizar recursos se os limites forem rígidos demais.
- Exige entendimento da carga por fluxo de negócio.

### Retry com Backoff

Retry com backoff tenta novamente uma operação após falha transitória, aumentando o intervalo entre tentativas. É útil para instabilidades temporárias de rede ou indisponibilidades curtas.

Vantagens:

- Recupera falhas transitórias sem intervenção do usuário.
- Simples de aplicar em chamadas idempotentes.
- Melhora confiabilidade percebida.

Desvantagens:

- Pode amplificar carga durante incidentes.
- É perigoso em operações não idempotentes, como pagamento.
- Sem jitter e limites, contribui para tempestades de retry.

## Decisão

A FinTech Wallet adotará API Gateway, Circuit Breaker e Bulkhead como padrões principais de resiliência. Retry com Backoff será permitido apenas como padrão complementar, aplicado a operações idempotentes, com limite de tentativas, jitter e observabilidade.

O API Gateway Kong será usado como primeira camada de proteção, limitando tráfego abusivo, validando tokens na borda e reduzindo exposição direta dos microsserviços.

Circuit Breaker será usado em chamadas REST entre serviços e integrações externas, especialmente Payment Service para gateway bancário, Notification Service para provedor de e-mail/SMS e chamadas do API Gateway para serviços internos.

Bulkhead será usado para separar recursos por integração e criticidade. Fluxos de pagamento e autenticação terão limites próprios, evitando que relatórios pesados, notificações atrasadas ou falhas externas comprometam operações financeiras essenciais.

## Alternativas Rejeitadas

Usar apenas Retry foi rejeitado porque retries indiscriminados podem piorar incidentes, multiplicando requisições contra dependências degradadas. Em sistemas financeiros, repetir pagamento sem idempotência forte também pode gerar duplicidade operacional.

Usar apenas API Gateway foi rejeitado porque proteção de borda não resolve falhas internas entre microsserviços nem indisponibilidade de provedores externos. O gateway reduz exposição, mas Circuit Breaker e Bulkhead continuam necessários dentro da malha de serviços.

Não aplicar padrões explícitos de resiliência foi rejeitado porque microsserviços aumentam chamadas remotas e tornam falhas parciais inevitáveis. A ausência desses padrões comprometeria disponibilidade e confiabilidade, dois RNFs prioritários.

## Consequências

Consequências positivas:

- Menor risco de falhas em cascata.
- Melhor proteção de recursos críticos.
- Menor exposição direta dos serviços internos.
- Resposta mais previsível quando dependências estão indisponíveis.
- Maior aderência aos RNFs de disponibilidade e confiabilidade.

Consequências negativas:

- Maior complexidade operacional.
- Necessidade de métricas, alertas e ajuste contínuo.
- Kong passa a ser componente crítico e precisa de health checks, logs e escala.
- Alguns usuários podem receber falha rápida em vez de aguardar recuperação lenta.

## Trade-offs

A decisão privilegia estabilidade sistêmica em vez de insistir indefinidamente em uma operação degradada. O fail fast pode parecer menos amigável em uma requisição isolada, mas protege a plataforma inteira.

O uso de Bulkhead aumenta a configuração inicial, porém reduz o risco de uma fila ou dependência consumir todos os recursos compartilhados. Para uma plataforma financeira, essa proteção é mais importante que a máxima utilização de recursos em cenários normais.

O API Gateway simplifica políticas transversais, mas introduz um ponto de concentração de tráfego. O trade-off é aceitável porque a plataforma ganha controle de borda, desde que Kong seja tratado como componente crítico e escalável.

## Referências

- Nygard, Michael T. *Release It!*. Pragmatic Bookshelf.
- Newman, Sam. *Building Microservices*. O'Reilly Media.
- Newman, Sam. *Monolith to Microservices*. O'Reilly Media.
