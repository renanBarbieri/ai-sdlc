# Skill: Especificar Funcionalidade

## Trigger

Sempre que o usuário disser **"vamos especificar"**, **"revisa o plano contra as regras"**, ou logo após aprovar um documento de plano (`planejar-funcionalidade`) e antes de pedir a implementação, execute este fluxo. Não implemente código ainda — apenas confronte o plano contra as regras do projeto e registre os riscos.


## Processo

### 1. Localizar o plano

 Leia o documento de plano em `docs/project/features/<slug-da-feature>/<slug-da-feature>-plan.md` (produzido pela skill `planejar-funcionalidade`). Se não existir, peça ao usuário para planejar primeiro.

### 2. Carregar as regras aplicáveis

Leia também `docs/project/features/<slug-da-feature>/<slug-da-feature>-tests.md`. Se não existir, peça ao usuário para voltar ao planejamento.

- Sempre: `.claude/rules/arquitetura.md` e `.claude/rules/seguranca.md`.
- Para backend: `.claude/rules/backend.md`.
- Para client, web, mobile ou outra camada de apresentação: `.claude/rules/client.md`.

Se uma stack não puder ser classificada, ou se faltar documentação obrigatória, registre um bloqueante de contexto e não prossiga nessa área. Não substitua documentação ausente por suposições.


### 3. Confrontar o plano com as regras

Faça o confronto sem implementar código e sem inventar detalhes ausentes. Use os nomes reais de módulos, arquivos, regras, tecnologias e contratos encontrados no projeto. Se uma informação necessária não estiver documentada, registre-a como desconhecida e classifique a falta de contexto.

#### 3.1 Identificar o escopo real

Antes de validar o conteúdo, extraia do plano:

- Camadas e stacks afetadas: dados, backend, client, web, mobile, emails, jobs, infraestrutura, observabilidade ou outras.
- Módulos, serviços, pacotes, aplicações e arquivos criados ou alterados.
- Contratos e integrações: API, eventos, banco, autenticação, storage, filas e notificações.
- Regras aplicáveis: sempre-on, específicas da stack, do módulo ou do caminho do arquivo.

Se o plano não permitir determinar se uma área é backend, client ou outra categoria, não escolha por aproximação: registre a ambiguidade e peça esclarecimento antes de liberar a implementação.

#### 3.2 Confrontar cada decisão do plano

Para cada decisão, requisito e passo de implementação, registre mentalmente ou em uma tabela de trabalho:

| Item do plano | Regra/documento de referência | Situação | Evidência ou lacuna |
|---|---|---|---|
| [decisão concreta] | [arquivo e seção] | Conforme / Atenção / Bloqueante | [justificativa objetiva] |

Verifique, no mínimo:

- **Arquitetura:** responsabilidades, dependências permitidas, limites entre camadas, reutilização de abstrações e aderência à arquitetura documentada.
- **Dados:** entidades, relações, migrações, índices, concorrência, compatibilidade e isolamento por tenant.
- **Backend e serviços:** interfaces, DTOs/modelos, validação, autorização, erros, idempotência, transações, jobs, eventos e observabilidade.
- **Client:** navegação ou telas, estado, ciclo de vida, cache, offline, permissões, deep links, acessibilidade e requisitos da plataforma.
- **Integrações:** contratos, autenticação, versionamento, timeouts, retries, falhas e compatibilidade entre consumidores.
- **Emails e notificações:** gatilhos, destinatários, preferências, opt-in, dados expostos e falhas de entrega.
- **Segurança:** segredos, dados pessoais, autenticação, autorização, logs, armazenamento local, transporte e isolamento.
- **Cenários Gherkin:** cada critério de aceite deve ter cenário identificável, com casos positivos, negativos, autorização, limites e regressões relevantes.
- **Testes e operação:** estratégia de testes, migração, rollback, feature flags, métricas, alertas e impacto no deploy.

#### 3.3 Classificar os resultados

Classifique cada divergência ou ausência seguindo estes critérios:

- **Bloqueante:** viola regra obrigatória, deixa segurança ou isolamento indefinido, depende de documentação ausente, cria contrato incompatível ou impede determinar a implementação.
- **Atenção:** exige decisão explícita, evidencia risco técnico ou deixa comportamento importante sem definição.
- **Conformidade verificada:** há evidência de que a decisão é compatível com as regras.

Não marque como conformidade apenas porque o plano não menciona um tema. Ausências relevantes devem ser registradas como atenção ou bloqueante, conforme o impacto.



### 4. Registrar os achados

Acrescente ao **mesmo arquivo** de plano (`docs/project/features/<slug-da-feature>/<slug-da-feature>-plan.md`) uma seção nova ao final:

```markdown
## Conformidade e Riscos

### 🔴 Bloqueantes
[Itens que violam uma regra fixa de segurança/arquitetura — precisam ser resolvidos antes de implementar]

### 🟡 Atenção
[Itens que não violam regra fixa, mas merecem decisão explícita do usuário]

### ✅ Conformidade verificada
[O que já está alinhado com as regras]
```

Não crie um arquivo separado — o objetivo é manter um único documento por feature.

### 5. Resolver com o usuário

Se houver itens 🔴, pare e pergunte ao usuário como resolver antes de sugerir a implementação (`construir-funcionalidade`). Itens 🟡 podem ser decididos e registrados na mesma resposta.
