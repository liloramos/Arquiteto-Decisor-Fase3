# ADR 0001 - Estratégia de Nuvem e Escalabilidade

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

### SaaS

Em SaaS, a equipe consumiria uma solução pronta de gestão financeira, autenticação, pagamentos ou relatórios, delegando a maior parte da arquitetura ao fornecedor.

Vantagens:

- Menor esforço inicial de infraestrutura e operação.
- Tempo de adoção reduzido para funcionalidades padronizadas.
- Responsabilidade operacional transferida em grande parte ao fornecedor.

Desvantagens:

- Baixo controle sobre arquitetura interna, modelos de dados e mecanismos de resiliência.
- Dificuldade para demonstrar microsserviços, Arquitetura Hexagonal, RabbitMQ e Database per Service.
- Forte dependência de fornecedor e limitação de customização.
- Não atende ao objetivo acadêmico de justificar decisões arquiteturais próprias.

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

### Escalabilidade Vertical

Escalabilidade vertical consiste em aumentar CPU, memória, disco ou capacidade de rede de uma instância EC2.

Vantagens:

- Simples de executar no curto prazo.
- Não exige mudança imediata na topologia.
- Útil para absorver crescimento inicial controlado.

Desvantagens:

- Possui limite físico e econômico.
- Mantém risco de ponto único de falha quando usada isoladamente.
- Pode gerar custo elevado por superdimensionamento.

### Escalabilidade Horizontal

Escalabilidade horizontal consiste em adicionar mais instâncias ou réplicas dos serviços, distribuindo carga por balanceamento.

Vantagens:

- Melhor alinhamento com microsserviços.
- Permite escalar apenas serviços pressionados, como Payment Service ou Wallet Service.
- Melhora disponibilidade quando combinada com múltiplas instâncias e health checks.

Desvantagens:

- Exige balanceamento, observabilidade e disciplina de configuração.
- Requer serviços stateless ou estado externo bem definido.
- Aumenta complexidade operacional em EC2.

## Decisão

A FinTech Wallet adotará IaaS com AWS EC2 como estratégia de nuvem da Fase 3. Para escalabilidade, a decisão arquitetural é priorizar escala horizontal dos microsserviços e usar escala vertical apenas como ajuste tático de curto prazo.

A decisão foi tomada porque EC2 oferece o nível de controle necessário para demonstrar a arquitetura completa de microsserviços com Kong, Keycloak, RabbitMQ, PostgreSQL por serviço e Docker Compose. Essa escolha permite evidenciar decisões arquiteturais de segurança, comunicação, resiliência e persistência sem esconder componentes críticos atrás de abstrações de plataforma.

A escala horizontal é preferida porque respeita a decomposição em microsserviços: Payment Service, Wallet Service, Transaction Service, Notification Service e Report Service podem receber réplicas conforme pressão de carga. A escala vertical continua aceitável para o ambiente acadêmico e para ajustes iniciais, mas não será tratada como estratégia principal de crescimento.

## Alternativas Rejeitadas

PaaS foi rejeitado nesta fase porque reduziria a carga operacional, mas também reduziria a visibilidade da arquitetura e poderia limitar a composição controlada dos componentes exigidos. Para uma entrega acadêmica focada em arquitetura, a clareza da topologia é mais importante que a conveniência de deploy.

SaaS foi rejeitado porque eliminaria a maior parte das decisões arquiteturais avaliadas na disciplina. Embora possa ser útil para componentes específicos, como envio de e-mail ou antifraude, não atende ao objetivo de projetar a FinTech Wallet como sistema próprio baseado em microsserviços.

Serverless foi rejeitado porque a plataforma possui fluxos financeiros que exigem rastreabilidade, controle de sessão operacional, integração com mensageria e consistência transacional por serviço. Serverless pode ser útil futuramente para jobs auxiliares, mas não será o modelo principal da arquitetura.

Escalabilidade exclusivamente vertical foi rejeitada porque aumenta capacidade sem melhorar proporcionalmente disponibilidade ou isolamento de falhas. Em uma FinTech, crescer apenas aumentando a instância mantém risco operacional concentrado.

## Consequências

Consequências positivas:

- Maior controle sobre segurança, rede e execução.
- Ambiente compatível com Docker Compose, facilitando demonstração local e implantação em EC2.
- Boa base para evoluir para orquestração mais robusta em fases futuras.
- Facilita demonstrar padrões arquiteturais explicitamente.
- Permite evoluir para escala horizontal seletiva por microsserviço.

Consequências negativas:

- A equipe assume responsabilidade por atualização de imagens, sistema operacional e dependências.
- Alta disponibilidade exige configuração adicional, como múltiplas instâncias, balanceador e backup.
- Custos podem crescer se instâncias ficarem superdimensionadas.
- Escala horizontal demanda serviços stateless, health checks, logs centralizados e automação.

## Trade-offs

A escolha prioriza controle, transparência arquitetural e reprodutibilidade sobre automação nativa de plataforma. Em troca, aumenta a responsabilidade operacional. Para a Fase 3, esse trade-off é aceitável porque a disciplina avalia a capacidade de justificar decisões de arquitetura e não apenas consumir serviços prontos.

Em termos de custo versus desempenho, EC2 permite escolher instâncias conforme a carga prevista e ajustar verticalmente no curto prazo. No médio prazo, a arquitetura deve evoluir para autoscaling, múltiplas instâncias, banco gerenciado e observabilidade centralizada. O trade-off principal é aceitar maior complexidade operacional para obter melhor disponibilidade e escalabilidade horizontal por serviço.

## Referências

- Pressman, Roger S.; Maxim, Bruce R. *Software Engineering: A Practitioner's Approach*. McGraw-Hill.
- Newman, Sam. *Building Microservices*. O'Reilly Media.
- Newman, Sam. *Monolith to Microservices*. O'Reilly Media.
