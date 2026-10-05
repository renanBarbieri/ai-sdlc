---
name: inicializar-documentacao
version: 1.0.0
description: "Configura o projeto-alvo em .claude/project.json e gera ou atualiza toda a documentação de contexto da casca (docs/project/ e docs/.ia/sdlc/sdlc.md) a partir da exploração real do alvo."
tags: [bootstrap, docs, context, project-setup, inventory]
when:
  - "O usuário pedir para inicializar, gerar ou atualizar a documentação de IA do projeto-alvo."
  - "O usuário pedir para configurar ou trocar o projeto-alvo desta casca."
  - "docs/project/overview.md não existir e uma tarefa depender de contexto do projeto-alvo."
---

Você é um especialista em documentação de software para IA. Esta sessão roda na raiz de uma casca externa de SDLC. Analise um projeto-alvo, sem copiar esta casca para ele, e gere na casca a documentação e a configuração necessárias para trabalhar nesse alvo ao longo do tempo.
Só finalize após a conclusão de todas as etapas.

## Passo 0 — Selecionar e persistir o projeto-alvo

Antes de explorar ou criar documentação, verifique `.claude/project.json` na raiz desta casca.

1. Se o arquivo não existir, estiver inválido ou não apontar para um diretório existente, pergunte: "Qual é o caminho relativo, a partir da raiz desta casca, do projeto que devo analisar?" Não prossiga sem resposta.
2. Resolva o caminho a partir da raiz da casca e confirme com o usuário o caminho absoluto resolvido e o nome do projeto.
3. Salve ou atualize `.claude/project.json` usando apenas caminhos relativos e este formato:

```json
{
  "projectPath": "../meu-projeto",
  "projectName": "meu-projeto"
}
```

4. Em execuções futuras, reutilize o arquivo. Pergunte novamente apenas se o alvo não existir, o JSON estiver inválido ou o usuário solicitar a troca.
5. O alvo é a única fonte de código a ser explorada. A raiz da casca é a única localização permitida para os artefatos gerados.
6. Não crie, altere ou sobrescreva `CLAUDE.md`, `.claude/`, `docs/` ou qualquer outro arquivo dentro do projeto-alvo durante esta inicialização.

Use estas variáveis no restante desta skill:

- `<CASCA>`: raiz desta sessão.
- `<ALVO>`: caminho absoluto resolvido a partir de `projectPath`.
- `<DOCS>`: `<CASCA>/docs/`.
- `<CLAUDE>`: `<CASCA>/.claude/`.

## Passo 1 — Explorar o projeto

Antes de criar qualquer arquivo, leia o suficiente do projeto para entender:

1. **Stack técnica**: linguagens, frameworks, bibliotecas principais, infra
2. **Arquitetura**: padrões usados (Clean Architecture, MVC, monolito, microsserviços, etc.)
3. **Módulos/domínios**: quais são as principais áreas de negócio do sistema
4. **Autenticação e segurança**: como funciona auth, multi-tenancy, isolamento de dados
5. **Banco de dados**: tipo, ORM, estratégias de acesso
6. **Convenções de código**: nomenclatura, idioma do código, idioma da UI
7. **Deploy**: como o sistema é publicado em produção
8. **Testes**: ferramentas e estratégia de testes existentes

Arquivos úteis para começar a explorar em `<ALVO>`: `README.md`, `package.json` (ou equivalente), arquivos de configuração, módulos/controllers/services, schema de banco se existir, rotas/telas, jobs, integrações e testes. A lista de `2-3 módulos/controllers/services` serve apenas como amostra inicial; não limita o inventário final.

### Inventário obrigatório antes da documentação

Antes de criar os documentos de funcionalidades, faça uma varredura completa do projeto e crie na casca um inventário de trabalho em `<DOCS>/project/features/_inventory.md`. Use múltiplas fontes para não depender apenas do README:

- árvore de diretórios e arquivos de código;
- módulos, pacotes, bounded contexts, services, controllers, handlers, casos de uso e repositórios;
- rotas, telas, fluxos de navegação, comandos, jobs, consumers e eventos;
- entidades, schemas, migrations e tabelas relevantes;
- integrações externas e pontos de entrada da aplicação;
- testes existentes e nomes dos cenários cobertos.

Faça o inventário em dois níveis:

1. **Domínio ou módulo:** área ampla como `admin`, `billing`, `notificações`, `autenticação` ou um domínio do cliente.
2. **Funcionalidade ou subfeature:** comportamento concreto oferecido por essa área, como listar, criar, editar, cancelar, aprovar, reenviar, configurar, pagar, receber webhook, consultar histórico ou alterar preferências.

Uma área ampla nunca é evidência suficiente de que suas funcionalidades internas foram mapeadas. Por exemplo, `admin` deve ser decomposto em cada fluxo administrativo identificado; `billing` deve separar cobrança, planos, pagamentos, faturas, cancelamentos e webhooks quando existirem; `notificações` deve separar canais, preferências, templates, envio e histórico quando existirem; e cada domínio do cliente deve ser decomposto em suas telas, ações e fluxos de navegação.

Classifique cada item encontrado como `domínio`, `módulo`, `funcionalidade`, `subfeature`, `fluxo`, `integração`, `job`, `infraestrutura` ou `desconhecido`. Um item pode ter um `parent-id`, mas nunca deve ser omitido por estar agrupado em outro item. Quando não houver evidência suficiente para descrever o comportamento, mantenha-o no inventário com status `evidência insuficiente`; não o descarte.

### Segunda passagem por artefatos concretos

Depois do inventário inicial, faça uma segunda passagem exclusivamente para reconciliar artefatos concretos com os itens inventariados. Extraia e confira, conforme existirem no projeto:

- cada controller, handler, endpoint e rota de API;
- cada service, use case, command, query e job;
- cada página, tela, componente de fluxo e rota de navegação do cliente;
- cada model, entity, schema, migration, tabela e política de acesso;
- cada consumer, webhook, evento, integração externa e template de notificação;
- cada arquivo ou suíte de teste relevante.

Para cada artefato, responda: `qual comportamento ele implementa?`, `a qual item do inventário ele pertence?` e `qual cenário de teste cobre esse comportamento?`. Se uma resposta não for possível, crie um item separado com status `evidência insuficiente` ou `subfeature não classificada`. Nunca considere um domínio amplo como cobertura automática de seus controllers, páginas, services ou models internos.

Use nomes concretos encontrados no código para descobrir subfeatures. Procure especialmente prefixos, nomes de métodos públicos, decorators, rotas, ações de UI, eventos e migrations que indiquem operações distintas. Diferencie arquivos compartilhados de funcionalidades: um utilitário comum pode ser documentado como infraestrutura, mas qualquer comportamento de negócio exposto por ele precisa estar associado a uma feature.

O inventário deve conter pelo menos:

```markdown
| ID | Parent ID | Tipo | Nome concreto | Evidência no alvo | Feature slug | Cenários | Status |
|---|---|---|---|---|---|---|---|
| D-001 | - | domínio | admin | [caminhos reais] | admin | [IDs] | pendente |
| M-001 | D-001 | módulo | gestão de usuários | [caminhos reais] | admin-usuarios | [IDs] | pendente |
| F-001 | M-001 | funcionalidade | bloquear usuário | [caminhos reais] | admin-usuarios | [IDs] | pendente |
| A-001 | F-001 | artefato | UserController.block | [caminho:linha] | admin-usuarios | [IDs] | reconciliado |
```

Não avance para a geração final enquanto a varredura não tiver sido feita. O inventário é a fonte de conferência para garantir que nenhum módulo ou funcionalidade foi omitido.

---

## Passo 2 — Criar a estrutura de arquivos

Crie todos os arquivos abaixo dentro de `<DOCS>`, com conteúdo específico para `<ALVO>`. Siga rigorosamente a hierarquia:

```
docs/project/
├── overview.md                        # Visão geral completa do sistema, com routers para especificações técnicas, em `docs/project/specs`, e funcionalidades, em `docs/project/features/<slug>/`
|── specs/                             # Especificações técnicas do projeto
|   |── arquitetura.md                 # Especificação do pardrão arquitetural do projeto, assim como system design e convenções de código. Se necessário, criar sub documentações para não inflar demais este arquivo. Sempre adicionar exemplos
|   |── seguranca.md                   # Especificação técnica de segurança do projeto (exemplos)
|   |── testes.md                      # Especificação técnica de testes do projeto (ferramentas + exemplos)
|   |── stack-<slug-name>.md           # Especificação técnica da stacks identificada `<slug>`. Se o projeto possuir mais de uma stack, gerar mais de um arquivo com o respectivo <slug-name>
|── features                           # Pasta com as funcionalidades da aplicação
|   |── <slug>                         # Espeficicação de uma funcionalidade
|   |   |── <slug>-overview.md         # Visão geral de uma funcionalidade
|   |   |── <slug>-plan.md             # Plano de implementação da funcionalidade. Gerado quando há uma nova funcionalidade ou há uma modificação na funcionalidade existente. Este arquivo não existe no mapeamento inicial
|   |   |── <slug>-tests.md            # Cenários de comportamento em Gherkin e registro das validações. Criado no planejamento da funcionalidade
|── features/_inventory.md              # Inventário completo usado para conferir cobertura da documentação
```
---

## Passo 3 — Regras de conteúdo

### `<DOCS>/.ia/sdlc/sdlc.md`
Tabela curta mapeando as 7 etapas do processo (plan, design, build, review, test, deploy, maintain) para: artefato produzido, mecanismo (rule ou skill) e onde ele mora neste projeto. O deploy deve documentar `feature/<slug> -> develop -> main`, aprovação e versionamento `AA.MM.xx`. Deve terminar com uma seção "Onde entra especialidade de stack" apontando para `.claude/rules/<stack>.md` (regra sempre-on escopada por `paths:`) e `docs/project/specs/stack-<slug-stack-name>.md` (conhecimento arquitetural detalhado) — sem escrever conteúdo de nenhuma stack específica, apenas indicando o local.

### `<DOCS>/project/overview.md`
Documento de referência central. Deve incluir:
- O que é o sistema (propósito em 2-3 linhas)
- Stack completa (tabela: camada → tecnologias)
- Módulos/domínios principais (tabela: módulo → descrição)
- Fluxo de autenticação (passo a passo)
- Multi-tenancy ou isolamento de dados (se existir)
- Conexões de banco (roles, strings de conexão, estratégias)
- Segurança implementada (resumo)
- URLs de desenvolvimento
- Comandos úteis (start, test, migrate, build)
- Arquitetura (estrutura de pastas, regras de dependência, fluxo de dados)
- Convenções de código (idioma, nomenclatura, imports)
- Estado atual e notas relevantes

### `<DOCS>/project/specs/arquitetura.md`
Documentação técnica detalhada. Inclui:
- Estrutura de pastas com exemplos reais do projeto
- Padrão de controller/service/repository (código real)
- Módulos/classes compartilhadas
- Decisões de design importantes e seus motivos
- System design e padrões de código

### `<DOCS>/project/specs/seguranca.md`
Detalhes de implementação de segurança. Inclui:
- Mecanismos de autenticação e autorização
- Criptografia de dados sensíveis (se existir)
- Isolamento de dados entre tenants (se existir)
- Validação de inputs
- Configurações de segurança HTTP
- Checklist de deploy seguro

### `<DOCS>/project/specs/testes.md`
Guia prático de testes. Inclui:
- Setup e comandos
- Exemplo completo de um teste representativo do projeto
- Como mockar dependências principais
- Prioridades (quais services/módulos testar primeiro)

### `<DOCS>/project/specs/stack-<slug-stack-name>.md`
Detalhes de uma stack do projeto. Inclui:
- Determinação do tipo de stack (backend, client ou o que conseguir identificar)
- Setup e comandos
- Integração entre as stacks
- Referências de documentação oficial

### `<DOCS>/project/features/<slug>`
Pastas com todas as funcionalidades mapeadas em `<DOCS>/project/overview.md`

Cada item do inventário deve estar associado a uma pasta `<slug>`. Módulos sem uma funcionalidade claramente identificável devem receber uma feature de documentação própria ou ser associados explicitamente à feature que os representa, com a decisão registrada no inventário.

### `<DOCS>/project/features/<slug>/<slug>-overview.md`
Documento de referência da funcionalidade. Deve incluir:
- O que é a funcionalidade (propósito em 2-3 linhas)
- Stack completa (tabela: camada → tecnologias)
- Módulos/domínios principais (tabela: módulo → descrição)
- Multi-tenancy ou isolamento de dados (se existir)
- Conexões de banco (roles, strings de conexão, estratégias)
- Segurança implementada (resumo)
- Comandos úteis (start, test, migrate, build)
- Estado atual e notas relevantes

### `<DOCS>/project/features/<slug>/<slug>-plan.md`
Guia técnico de implementação de uma funcionalidade. Inclui:
- Critérios de aceite observáveis, identificados como `CA-001`, `CA-002` etc., cada um vinculado a um cenário Gherkin
- Ordem exata dos passos (Schema → DTO → Service → ... ou Model → UseCase → ...)
- Template de código para cada camada (com placeholders do projeto real)
- Comandos de migration, geração de código, etc.
- O que registrar/exportar em cada passo

### `<DOCS>/project/features/<slug>/<slug>-tests.md`
Contrato de comportamento e validação da funcionalidade. Deve incluir:
- Cenários de aceitação escritos em Gherkin (`Funcionalidade`, `Cenário`, `Dado`, `Quando`, `Então`)
- IDs estáveis (`CT-001`, `CT-002` etc.) e vínculo explícito com critérios `CA-*`
- Cenários de sucesso, validação, autorização, limites, falhas e regressões relevantes
- Mapeamento dos cenários para testes automatizados, quando houver
- Comandos executados, resultados, cenários pendentes e limitações do ambiente

O arquivo deve ser criado durante o planejamento, confrontado com as rules e os critérios na especificação, atualizado durante a construção quando o comportamento aprovado mudar e executado/documentado pela skill de testes. Não aguarde a implementação para definir os critérios de aceitação.
Na documentação inicial, o arquivo deve ser criado, mapeando os cenários de testes pré-existentes no projeto ou cenários onde devemos ter testes que ainda não foram implementados.

Para cada item `funcionalidade`, `subfeature`, `fluxo`, `integração` ou `job` do inventário, garanta cobertura individual no `<slug>-tests.md` da feature associada. Domínios e módulos também devem ter cenários próprios quando representarem comportamento observável; caso contrário, registre a relação com suas subfeatures. Itens podem compartilhar uma feature e um arquivo quando isso refletir a organização real do projeto, mas cada item deve ser identificável pelo seu ID no inventário e possuir cenários próprios ou uma justificativa explícita de `Evidência insuficiente`. O arquivo deve existir mesmo quando não houver testes automatizados: nesse caso, registre os cenários Gherkin esperados, marque-os como `não automatizado` e explique quais evidências ainda faltam. Nunca omita o arquivo por falta de testes existentes.

---

### `<DOCS>/.ia/sdlc/skills/07_publicar.md`
Procedimento de publicação da funcionalidade. Deve incluir:
- branch dedicada `feature/<slug-da-feature>` criada a partir de `develop`;
- commits pequenos, coerentes e seguindo a convenção do projeto;
- validações antes do push e abertura de PR para `develop`;
- proteção de `develop` e `main`, checks obrigatórios, reviewers, CODEOWNERS e aprovação auditável;
- automação para abrir PR, solicitar aprovação, acompanhar checks e habilitar auto-merge somente após os gates;
- proibição de autoaprovação pelo autor ou bot sem política formal e identidade separada;
- segundo PR de `develop` para `main`;
- versionamento `AA.MM.xx`, tag `vAA.MM.xx`, changelog e validação pós-release;
- rollback e relatório de publicação.

Na documentação inicial, não invente provedor de Git, workflow, comando ou ferramenta. Registre o provedor e as políticas encontradas no projeto. Se não houver automação existente, documente a configuração recomendada como pendência, sem criar credenciais ou aprovar PRs automaticamente.

## Passo 4 — Criar `<CASCA>/CLAUDE.md`

Crie ou sobrescreva o `CLAUDE.md` na raiz da casca com o formato mínimo. Ele deve descrever `<ALVO>` e deixar claro que a documentação vive na casca; não crie esse arquivo no projeto-alvo:

```markdown
# CLAUDE.md

## Projeto
[1-2 linhas descrevendo o sistema e suas prioridades]

## Stack
[Bullet list: camada → tecnologias principais]

## Regras fixas
[Resumo de 2-3 linhas — as regras completas moram em `.claude/rules/`, não aqui]
[Penúltima regra: "Se houver dúvida sobre padrão do projeto, consultar docs/project/overview.md"]
[Última regra: "Se houver dúvida sobre em que etapa do processo uma tarefa se encaixa, consultar docs/.ia/sdlc/sdlc.md"]

## Comandos
[Bloco bash com os comandos essenciais: start, test, migrate, build]

## Referências
[Lista de todos os docs/.ia/ relevantes com path e descrição de uma linha]
[Lista de todos os docs/project/ relevantes com path e descrição de uma linha]
[Skills devem aparecer antes dos docs de arquitetura]
```

---

## Princípios gerais

1. **Conteúdo real**: use nomes, caminhos e padrões reais do projeto, não placeholders genéricos.
2. **Concisão**: cada arquivo deve ser lido em <2 minutos. Se crescer muito, divida.
3. **Arquivos de arquitetura** (`docs/project/specs/arquitetura.md`) são a exceção — podem ser mais longos pois são referência técnica.
4. **Validação do que foi gerado**: Ao finalizar a criação da documentação de funcionalidades, revisite o arquivo `<DOCS>/project/overview.md` e certifique que todas as funcionalidades e módulos foram mapeadas em `<DOCS>/project/features/<slug>`. Caso contrário, retorne o processo para criar a documentação faltante até que todas as funcionalidades e módulos estejam mapeados.
5. **Criar os cenários de teste é obrigatório**: Para cada funcionalidade já existente, gere `<slug>-tests.md`. Se houver evidência suficiente no projeto para descrever seu comportamento, escreva os cenários; Se não houver evidência, crie o arquivo mesmo assim, deixando diretrizes para forçar uma criação em uma implementação futura. Para funcionalidades novas, o arquivo será criado junto com o plano.

## Gate obrigatório de completude

Antes de finalizar, execute esta reconciliação:

1. Leia `<DOCS>/project/features/_inventory.md` e conte todos os itens classificados como domínio, módulo, funcionalidade, subfeature, fluxo, integração ou job.
2. Para cada item, confirme uma associação explícita a `<DOCS>/project/features/<slug>/`.
3. Confirme que cada feature associada contém `<slug>-overview.md` e `<slug>-tests.md`.
4. Confirme que cada artefato concreto está reconciliado com um item do inventário ou marcado como compartilhado/infraestrutura.
5. Confirme que cada ID de funcionalidade, subfeature, fluxo, integração e job aparece no `<slug>-tests.md` associado e possui cenários Gherkin próprios, ou uma seção explícita `Evidência insuficiente` com os cenários pendentes e a justificativa.
6. Confirme que cada domínio e módulo possui overview próprio ou uma associação explícita às features que o decompõem.
7. Atualize `<DOCS>/project/overview.md` para listar todos os domínios, módulos e features do inventário, sem usar apenas nomes de áreas agregadas.
8. Atualize o inventário com `overview criado`, `tests criado`, `artefatos reconciliados` e os caminhos dos arquivos gerados.
9. Se qualquer item falhar, não finalize: volte à exploração/documentação e corrija a lacuna.

Ao reportar a conclusão, mostre a contagem de itens do inventário, a contagem de features documentadas, a contagem de arquivos `-tests.md` e uma lista de eventuais itens com evidência insuficiente. A conclusão só é válida quando as contagens e associações forem reconciliadas.

Crie todos os arquivos agora, nesta ordem, sempre dentro de `<CASCA>`: overview.md → arquitetura → funcionalidades → seguranca → testes → stacks.

Ao final, mostre o caminho do alvo configurado, a lista de arquivos criados na casca e confirme explicitamente que nenhum arquivo foi criado ou alterado em `<ALVO>`.
