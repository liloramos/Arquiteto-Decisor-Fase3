# ADR 0001 - Estratégia de Nuvem

## Status

Aceito.

## Contexto

A FinTech Wallet precisa hospedar microsserviços críticos para pagamentos, transferências, carteira digital, notificações e relatórios financeiros. Os requisitos não funcionais prioritários são segurança, disponibilidade, escalabilidade, desempenho e confiabilidade.

A decisão de nuvem precisa considerar maturidade operacional, custo, controle sobre a infraestrutura, facilidade de implantação acadêmica e aderência ao desenho de microsserviços definido para a Fase 3. Como Pressman e Maxim destacam ao tratar decisões de projeto, atributos de qualidade devem ser tratados como forças arquiteturais, não como detalhes tardios de implementação. Sam Newman também reforça que a escolha de plataforma influencia diretamente autonomia, deploy, observabilidade e isolamento de serviços em arquiteturas distribuídas.

## Opções Avaliadas

### IaaS - AWS EC2

Na abordagem IaaS, a equipe controla máquinas virtuais, sistema operacional, rede, runtime, containers, atualizações e políticas de implantação. A AWS EC2 permite executar Docker Compose, Kong, Keycloak, RabbitMQ, PostgreSQL e os microsserviços com maior previsibilidade de ambiente.

Vantagens:

- Alto controle sobre rede, portas, certificados, volumes, logs e dependências.
- Compatível com Docker Compose e com o objetivo acadêmico de demonstrar a arquitetura completa.
- Permite isolamento por instância, security groups, sub-redes e políticas de acesso.
- Facilita evolução posterior para ECS, EKS ou serviços gerenciados sem redesenhar o domínio.

Desvantagens:

- Maior responsabilidade operacional sobre patching, hardening, backup e monitoramento.
- Escalabilidade exige configuração explícita de balanceamento e automação.
- Risco de sobrecarga de operação se não houver disciplina de infraestrutura.

### PaaS

Em PaaS, a plataforma abstrai parte do runtime, deploy e escala. Seria possível hospedar serviços em ambientes gerenciados, reduzindo trabalho operacional.

Vantagens:

- Menor esforço com sistema operacional e runtime.
- Escala e deploy podem ser mais simples.
- Boa produtividade para times pequenos.

Desvantagens:

- Menor controle sobre topologia, rede e serviços auxiliares.
- Pode dificultar a reprodução acadêmica completa com Kong, Keycloak, RabbitMQ e múltiplos bancos.
- Lock-in maior em recursos específicos da plataforma.

### Serverless

Serverless delega provisionamento e escala para o provedor. Funções sob demanda podem ser eficientes para cargas event-driven e tarefas pontuais.

Vantagens:

- Escalabilidade automática.
- Cobrança por uso.
- Boa aderência a eventos simples e jobs curtos.

Desvantagens:

- Menor previsibilidade para fluxos transacionais longos.
- Cold start e limites de execução podem afetar desempenho percebido.
- Complexidade para manter arquitetura hexagonal e fronteiras consistentes em muitos handlers.
- Observabilidade e testes end-to-end ficam mais distribuídos.

## Decisão

A FinTech Wallet adotará IaaS com AWS EC2 como estratégia de nuvem da Fase 3.

A decisão foi tomada porque EC2 oferece o nível de controle necessário para demonstrar a arquitetura completa de microsserviços com Kong, Keycloak, RabbitMQ, PostgreSQL por serviço e Docker Compose. Essa escolha permite evidenciar decisões arquiteturais de segurança, comunicação, resiliência e persistência sem esconder componentes críticos atrás de abstrações de plataforma.

## Alternativas Rejeitadas

PaaS foi rejeitado nesta fase porque reduziria a carga operacional, mas também reduziria a visibilidade da arquitetura e poderia limitar a composição controlada dos componentes exigidos. Para uma entrega acadêmica focada em arquitetura, a clareza da topologia é mais importante que a conveniência de deploy.

Serverless foi rejeitado porque a plataforma possui fluxos financeiros que exigem rastreabilidade, controle de sessão operacional, integração com mensageria e consistência transacional por serviço. Serverless pode ser útil futuramente para jobs auxiliares, mas não será o modelo principal da arquitetura.

## Consequências

Consequências positivas:

- Maior controle sobre segurança, rede e execução.
- Ambiente compatível com Docker Compose, facilitando demonstração local e implantação em EC2.
- Boa base para evoluir para orquestração mais robusta em fases futuras.
- Facilita demonstrar padrões arquiteturais explicitamente.

Consequências negativas:

- A equipe assume responsabilidade por atualização de imagens, sistema operacional e dependências.
- Alta disponibilidade exige configuração adicional, como múltiplas instâncias, balanceador e backup.
- Custos podem crescer se instâncias ficarem superdimensionadas.

## Trade-offs

A escolha prioriza controle, transparência arquitetural e reprodutibilidade sobre automação nativa de plataforma. Em troca, aumenta a responsabilidade operacional. Para a Fase 3, esse trade-off é aceitável porque a disciplina avalia a capacidade de justificar decisões de arquitetura e não apenas consumir serviços prontos.

Em termos de custo versus desempenho, EC2 permite escolher instâncias conforme a carga prevista e ajustar verticalmente no curto prazo. No médio prazo, a arquitetura deve evoluir para autoscaling, banco gerenciado e observabilidade centralizada para reduzir risco operacional.

## Referências

- Pressman, Roger S.; Maxim, Bruce R. *Software Engineering: A Practitioner's Approach*. McGraw-Hill.
- Newman, Sam. *Building Microservices*. O'Reilly Media.
- Newman, Sam. *Monolith to Microservices*. O'Reilly Media.
