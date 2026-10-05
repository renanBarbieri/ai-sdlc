# Skill: Auditar Funcionalidade

## Papel

Você é responsável por uma rodada de manutenção do projeto. Audite segurança, arquitetura, qualidade, comportamento, testes e operação sem ampliar o escopo silenciosamente. A auditoria identifica problemas e define o encaminhamento; a implementação de correções segue o fluxo apropriado.

## Quando executar

Execute manualmente ou por agendamento, preferencialmente:

- após mudanças relevantes ou um conjunto de entregas;
- nos módulos tocados desde a última auditoria;
- quando houver suspeita de regressão, risco de segurança ou dívida técnica;
- em uma periodicidade definida pelo projeto.

Não trate esta skill como parte obrigatória de uma feature em andamento. Uma feature recém-implementada deve passar primeiro pela skill de review.

## Processo

### 1. Definir o escopo

Registre:

- período ou marco desde a última auditoria;
- módulos, arquivos, serviços e integrações incluídos;
- alterações recentes e achados anteriores ainda abertos;
- regras, critérios e riscos relevantes para o escopo.

Se o escopo não for informado, use como padrão os módulos alterados desde a última rodada documentada. Não faça uma auditoria global sem declarar esse alcance.

### 2. Carregar o contexto

Leia somente a documentação aplicável:

- `docs/.ia/sdlc/sdlc.md`;
- `docs/project/specs/arquitetura.md`;
- `docs/project/specs/seguranca.md`;
- especificações das stacks e rules correspondentes;
- documentação da camada, testes e decisões relacionadas ao escopo.

Se a documentação obrigatória estiver ausente, desatualizada ou contraditória com o código, registre a lacuna e reduza o alcance da conclusão. Não invente a arquitetura ou os requisitos.

### 3. Auditar sem implementar

Analise o código, configurações, testes, migrações, contratos e histórico relevante. Verifique, conforme aplicável:

- autenticação, autorização, exposição de dados, segredos e isolamento entre usuários ou tenants;
- validação de entradas, fronteiras de confiança, logs e dados sensíveis;
- responsabilidades, dependências, duplicação, acoplamento e aderência à arquitetura;
- regressões, comportamento incorreto, casos extremos e contratos incompatíveis;
- cobertura e confiabilidade dos testes, migrações, rollback e compatibilidade;
- performance, observabilidade, jobs, filas, integrações e operação em produção;
- documentação desatualizada e dívida técnica com impacto verificável.

Para cada achado, registre evidência, impacto, severidade e recomendação. Não altere código, testes ou configuração durante a auditoria. Correções diretas são apenas uma classificação de encaminhamento e devem ser executadas em uma etapa posterior, com validação própria.

### 4. Classificar os achados

Classifique cada item:

- **Crítico:** risco de segurança, perda/corrupção de dados, indisponibilidade grave ou violação de regra obrigatória.
- **Alto:** regressão provável, contrato quebrado, falha operacional relevante ou risco significativo sem mitigação.
- **Médio:** problema de qualidade, cobertura, manutenção, performance ou arquitetura com impacto limitado ou controlável.
- **Baixo:** melhoria de clareza, documentação ou dívida sem impacto imediato.

Depois, defina o encaminhamento:

- **Correção direta:** pequena, isolada, bem compreendida e sem impacto arquitetural relevante.
- **Novo plano:** envolve decisão de arquitetura, migração, contrato, múltiplos módulos ou risco que exige planejamento.
- **Aceitar/acompanhar:** risco conhecido, documentado e conscientemente aceito pelo responsável.

Todo item Crítico ou Alto deve ter responsável e próximo passo explícitos. Itens classificados como Novo plano devem voltar à etapa `plan` pela skill de planejamento.

### 5. Registrar e reportar

Registre a rodada em `<CASCA>/docs/project/audits/<YYYY-MM-DD>-<slug-do-escopo>.md`, salvo se o projeto tiver um local de auditoria mais específico documentado, contendo:

```markdown
# Auditoria: [escopo] - [data]

## Escopo e contexto
[Período, módulos, documentação e limitações]

## Achados críticos e altos
[Achado, evidência, impacto, responsável e encaminhamento]

## Achados médios e baixos
[Achado, evidência, impacto e encaminhamento]

## Correções diretas
[Itens pequenos e isolados recomendados para execução posterior]

## Novos planos
[Itens que reentram pela etapa plan]

## Aceitos ou descartados
[Decisão, justificativa e prazo de revisão, quando aplicável]

## Próxima auditoria
[Escopo ou data sugerida]
```

Não registre segredos, tokens, senhas ou strings de conexão no relatório.

### 6. Encaminhar a manutenção

Não implemente durante a auditoria. Para itens pequenos e isolados, encaminhe uma correção direta para execução posterior usando as rules e validações do projeto. Para os demais, crie ou solicite um plano e retorne ao ciclo `plan -> design -> build -> review -> test -> deploy`.

Informe ao final o que foi auditado, os achados por severidade, quais correções diretas foram recomendadas, quais itens viraram plano, o que foi aceito e quais limitações permaneceram.
