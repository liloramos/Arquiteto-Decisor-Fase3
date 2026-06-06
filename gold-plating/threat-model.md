# Threat Model - FinTech Wallet

## 1. Contexto

A FinTech Wallet processa dados financeiros, autenticação, autorização, pagamentos, transferências e notificações. O modelo de ameaças considera os principais containers da arquitetura: Kong, Keycloak, Wallet Service, Payment Service, Transaction Service, Notification Service, Report Service, RabbitMQ e bancos PostgreSQL por serviço.

O objetivo é identificar ameaças prováveis, decisões de mitigação e riscos residuais. A análise usa STRIDE como estrutura de raciocínio.

## 2. Ativos Críticos

| Ativo | Motivo de Proteção |
|---|---|
| Tokens JWT | Permitem acesso a recursos protegidos |
| Credenciais OAuth2 | Autenticam usuários e serviços |
| Dados de carteira | Representam saldo e informações financeiras |
| Registros de transação | Suportam auditoria e contestação |
| Eventos RabbitMQ | Propagam pagamentos, notificações e relatórios |
| Chaves RS256 | Garantem integridade e autenticidade de tokens |
| Logs de auditoria | Evidenciam operações críticas |

## 3. Fronteiras de Confiança

- Internet para API Gateway Kong.
- Kong para microsserviços internos.
- Microsserviços para Keycloak.
- Payment Service para gateway bancário externo.
- Notification Service para provedor externo de e-mail/SMS.
- Serviços para RabbitMQ.
- Serviços para seus respectivos bancos PostgreSQL.

Cada fronteira deve aplicar autenticação, autorização, criptografia em trânsito e observabilidade.

## 4. Ameaças STRIDE

| Categoria | Ameaça | Impacto | Mitigação |
|---|---|---:|---|
| Spoofing | Uso de token roubado | Alto | JWT curto, TLS, MFA, rotação de chaves e validação de audience |
| Tampering | Alteração de payload de pagamento | Alto | Assinatura JWT, validação de schema, idempotency key e auditoria |
| Repudiation | Usuário nega transação | Alto | Transaction Service com trilha imutável e correlation ID |
| Information Disclosure | Vazamento de saldo ou transações | Alto | RBAC, escopos OAuth2, criptografia em trânsito e mascaramento de logs |
| Denial of Service | Sobrecarga no Payment Service | Alto | Rate limiting no Kong, Bulkhead, Circuit Breaker e filas |
| Elevation of Privilege | Usuário comum acessa função administrativa | Alto | RBAC no Keycloak e autorização por domínio nos serviços |

## 5. Decisões de Segurança

### Autenticação e Autorização

Decisão:

- Keycloak será o provedor central de identidade.
- Usuários usarão Authorization Code + PKCE.
- Serviços usarão Client Credentials.
- Tokens serão JWT RS256.

Alternativas rejeitadas:

- Autenticação própria por serviço foi rejeitada por aumentar duplicação e risco de políticas inconsistentes.
- JWT HS256 foi rejeitado porque exigiria compartilhamento de segredo simétrico entre múltiplos serviços.

Consequências:

- A plataforma ganha padronização de identidade.
- Keycloak torna-se componente crítico e precisa de backup, monitoração e hardening.

### Autorização de Domínio

Decisão:

- Kong validará token e políticas de borda.
- Cada microsserviço validará permissões de domínio antes de executar casos de uso.

Alternativa rejeitada:

- Centralizar toda autorização apenas no gateway foi rejeitado porque regras financeiras dependem de contexto de negócio interno.

Consequências:

- Defesa em profundidade.
- Maior esforço de implementação em cada serviço.

### Proteção de Eventos

Decisão:

- Eventos RabbitMQ terão schema versionado, correlation ID, producer, timestamp e identificador único.
- Consumidores serão idempotentes.
- Filas terão dead-letter queues.

Alternativa rejeitada:

- Publicar mensagens sem contrato versionado foi rejeitado por dificultar evolução independente.

Consequências:

- Maior confiabilidade na comunicação assíncrona.
- Necessidade de governança de eventos.

## 6. Controles Recomendados

- Rate limiting por cliente e por rota no Kong.
- TLS obrigatório em toda comunicação externa.
- Secrets fora do repositório.
- Rotação periódica de chaves e credenciais.
- MFA para administradores.
- Menor privilégio em roles e clients OAuth2.
- Logs sem CPF, cartão, senha, refresh token ou access token completo.
- Auditoria de pagamentos, transferências e alterações sensíveis.
- Backups criptografados dos bancos PostgreSQL.
- DLQ monitorada no RabbitMQ.

## 7. Riscos Residuais

| Risco Residual | Justificativa | Tratamento |
|---|---|---|
| Comprometimento do dispositivo do usuário | Fora do controle direto da plataforma | MFA e detecção de comportamento anômalo |
| Erro operacional em EC2 | IaaS exige operação ativa | Runbooks, backups e automação |
| Evento duplicado | Mensageria pode entregar mais de uma vez | Idempotência e chaves únicas |
| Indisponibilidade do Keycloak | Serviço central de identidade | Health checks, backup e instância redundante em produção |

## 8. Trade-offs

A arquitetura escolhe segurança forte e defesa em profundidade, o que aumenta complexidade operacional. A decisão é adequada porque o domínio financeiro tem alto impacto em caso de acesso indevido, fraude ou perda de rastreabilidade.

O uso de Keycloak reduz esforço de implementação de identidade, mas cria dependência operacional de um componente central. A mitigação é tratar Keycloak como serviço crítico, com backup, monitoração e política clara de recuperação.

## 9. Referências

- OWASP. *Application Security Verification Standard*.
- OWASP. *API Security Top 10*.
- Nygard, Michael T. *Release It!*. Pragmatic Bookshelf.
- Newman, Sam. *Building Microservices*. O'Reilly Media.
