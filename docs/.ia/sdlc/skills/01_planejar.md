# Skill: Planejar Funcionalidade

## Trigger

Sempre que o usuário disser **"quero planejar"**, **"vamos planejar"**, ou **"crie um plano para"** uma funcionalidade, execute este fluxo. Não implemente código ainda — apenas produza o documento de planejamento.

---

## Processo

### 1. Entender o escopo

Leia a solicitação completa e identifique:
- Quais camadas são afetadas: backend, frontend, banco de dados, emails, infraestrutura
- Quais módulos existentes serão modificados
- O que é novo (novas entidades, novos endpoints, novas páginas)
- Dependências de outras funcionalidades já existentes

### 2. Ler o contexto do projeto

Antes de planejar, leia: `docs/project/overview.md` — visão geral, stack e módulos

### 3. Explorar o código existente

Para cada área afetada, leia pelo menos uma implementação representativa da stack real:
- entrypoints, handlers, controllers, commands, jobs ou use cases;
- páginas, telas, componentes de fluxo ou rotas de navegação, quando existirem;
- services, repositórios, modelos, entidades, schemas, migrations e contratos relacionados;
- testes existentes e documentação específica da camada.

### 4. Criar o documento de plano

Crie o plano em **`docs/project/features/<slug-da-feature>/<slug-da-feature>-plan.md`**. Crie também, na mesma pasta, **`<slug-da-feature>-tests.md`** com os cenários de comportamento em Gherkin. Nunca crie esses arquivos em outro local.

---

## Estrutura do Documento de Plano

```markdown
# [Nome da Funcionalidade]

## Contexto e Motivação

[Por que essa funcionalidade existe. Qual problema ela resolve.]

## Escopo

[O que está dentro e fora do escopo desta implementação.]

## Critérios de Aceite

Cada critério deve ser observável, verificável e vinculado a pelo menos um cenário Gherkin em `<slug-da-feature>-tests.md`.

- [ ] CA-001: [resultado esperado]
- [ ] CA-002: [resultado esperado]

## Impacto Arquitetural

[Módulos novos, módulos modificados, novos modelos de dados, impacto em RLS/segurança.]

## Modelo de Dados

[Novos models, campos, enums, relações. Inclua o schema da biblioteca de gerenciamento de banco de dados completo para cada model novo.]

## Backend

### Novos endpoints
[Tabela com método, rota, guarda, descrição]

### Alterações em endpoints existentes
[O que muda e por quê]

### Novos DTOs
[Lista de DTOs a criar com campos]

### Lógica de negócio relevante
[Regras não óbvias, fluxos de validação, integrações com email e banco de dados]

## Cliente / Frontend

### Navegação
[Rotas web ou telas/rotas de navegação mobile, incluindo proteção e deep links.]

### Telas, páginas e componentes
[Elementos novos ou alterados conforme a plataforma.]

### Alterações em componentes existentes
[O que muda e por quê]

### Estado e ciclo de vida
[Hooks, stores, providers, lifecycle hooks e estados de loading/error que precisam de atualização]

### Services e integração
[Clients HTTP, mappers, modelos, cache e contratos consumidos.]

### Requisitos específicos da plataforma
[Permissões, notificações, armazenamento local, câmera, localização, offline ou acessibilidade.]
```

## Ordem de Implementação

Checklist numerado e ordenado por dependência:

- [ ] 1. [Primeiro passo — ex: schema Prisma + migrate]
- [ ] 2. [...]
- [ ] N. [Último passo — ex: registrar rota no App.tsx]

## Decisões e Trade-offs

[Decisões não óbvias tomadas durante o planejamento e por quê.]

## Fora do Escopo (por ora)

[O que foi conscientemente deixado de fora e pode ser planejado depois.]

## Cenários de Teste

Escreva em `docs/project/features/<slug-da-feature>/<slug-da-feature>-tests.md` os cenários de aceitação observáveis da funcionalidade:

```gherkin
# language: pt
Funcionalidade: [comportamento de negócio]
	Como [ator]
	Quero [intenção]
	Para [valor]

	Cenário: [resultado esperado]
		Dado que [contexto]
		Quando [ação]
		Então [resultado observável]
```

Inclua sucesso, validações, autorização, limites e falhas relevantes. Use IDs estáveis (`CT-001`, `CT-002`) e vincule cada cenário aos critérios `CA-*`. O arquivo é um artefato inicial de planejamento: a especificação pode apontar riscos ou lacunas, a construção pode atualizá-lo apenas com mudanças aprovadas e a skill `testar-funcionalidade` o usa para implementar e executar os testes.


---

## Regras

- **Não implemente** nada durante o planejamento. O objetivo é apenas o documento.
- Use linguagem técnica precisa: nomes de arquivos, métodos, tabelas reais do projeto.
- Se houver ambiguidade no requisito, liste as opções e recomende uma, explicando o trade-off.
- Ao final, pergunte ao usuário se o plano está aprovado antes de sugerir a especificação.
