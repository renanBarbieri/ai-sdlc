---
name: refactoring
version: 1.0.0
description: "Refatora código existente sem alterar comportamento, usando a técnica adequada ao code smell e as referências específicas do grupo."
tags: [refactoring, code-smell, typescript, javascript, tdd, clean-code]
when:
  - "Ao revisar código e encontrar um code smell."
  - "Na etapa de refactor de um ciclo TDD."
  - "Quando pedirem para melhorar, limpar, simplificar, desacoplar, extrair, tipar, reorganizar ou refatorar código."
---

# Refactoring

## Regra zero

Refactoring não muda comportamento. Se a mudança altera requisito, contrato ou resultado observável, trate-a como feature ou bugfix separado.

## Processo

1. Identifique o sintoma, não uma técnica favorita.
2. Confirme o escopo e leia a arquitetura e as convenções da stack afetada.
3. Verifique se há teste cobrindo o comportamento. Se não houver, crie um teste de caracterização antes de refatorar.
4. Consulte somente o grupo de referências correspondente ao smell. Leia também a seção **Quando não aplicar** da técnica escolhida.
5. Rode as validações existentes antes da mudança: testes, typecheck/compilador e lint.
6. Aplique uma técnica por vez, em passos pequenos. Valide depois de cada passo.
7. Confirme que não houve mudança de comportamento, revise o diff e registre a técnica aplicada.

Não leia os seis grupos de referência de uma vez. Se houver mais de um smell, aplique a ordem abaixo e reavalie após cada etapa.

## Mapa rápido: sintoma -> grupo

| Sintoma observado | Técnicas candidatas | Referência |
|---|---|---|
| Função, hook ou componente longo; expressão difícil | Extract Method/Variable, Decompose Conditional, Substitute Algorithm | `references/01-composing-methods.md`, `04-conditionals.md` |
| Responsabilidade no módulo errado; service/componente faz tudo | Move Method, Extract Class, Hide Delegate | `references/02-moving-features.md` |
| Campo, coleção ou primitivo exposto; estado duplicado | Encapsulate Field/Collection, Replace Data Value, Change Reference/Value | `references/03-organizing-data.md` |
| `if`/`switch` repetido, aninhado ou difícil de ler | Consolidate, Decompose, Guard Clauses, Polymorphism | `references/04-conditionals.md` |
| Assinatura ruim, boolean posicional, retorno ambíguo | Parameter Object, Rename, Separate Query/Modifier, Factory, Result | `references/05-method-calls.md` |
| Herança ou abstração duplicada/inadequada | Pull Up/Push Down, Extract Interface, Replace Inheritance with Delegation, Collapse Hierarchy | `references/06-generalization.md` |

## Heurísticas de TypeScript/JavaScript

- Valide dados externos como `unknown` na fronteira; não use `as` para esconder validação ausente.
- Prefira discriminated unions com `assertNever` a hierarquias artificiais.
- Use `Result` tipado para erros esperados; reserve exceções para falhas inesperadas.
- Prefira `readonly`, `ReadonlyArray`, cópias imutáveis e funções puras; não mute props, state, parâmetros ou coleções compartilhadas.
- Use branded types para ids, dinheiro e outros valores com semântica própria.
- Prefira `as const` e unions literais a `enum` quando não houver necessidade real de runtime.
- Derive estado a partir da fonte única; não sincronize dados derivados com `useEffect`.
- Em React, hooks devem permanecer antes de early returns; extrair JSX é Extract Component e extrair lógica é Extract Hook.
- Prefira composição e fronteiras de módulo a herança, getters ou classes criados apenas para encapsular.

## Ordem quando há vários smells

1. Tipar e validar fronteiras; remover `any`, `as` e `@ts-ignore` novos.
2. Simplificar condicionais e aplicar guard clauses.
3. Extrair métodos, variáveis, hooks ou componentes.
4. Modelar dados e ajustar assinaturas.
5. Mover responsabilidades entre módulos.
6. Alterar hierarquia ou composição, somente se ainda houver necessidade.

## Guardrails

- O teste deve estar verde antes e depois da mudança.
- Rode apenas os comandos reais documentados pelo projeto; não presuma `tsc --noEmit`, lint ou ferramenta específica.
- Não misture feature, bugfix, formatação ampla ou mudança de contrato no mesmo refactor.
- Não crie abstração especulativa: ela deve resolver um problema observado e ter uso real.
- Listeners, subscriptions e timers precisam de cleanup.
- O diff deve ser menor ou igual ao necessário para o smell identificado.

## Entrega

Informe: smell encontrado, técnica aplicada, arquivos alterados, validações executadas, resultado dos testes e qualquer risco ou comportamento que não pôde ser comprovado.

Para detalhes, exemplos Antes/Depois e restrições de cada técnica, leia apenas o arquivo do grupo indicado no mapa. O catálogo completo contém 66 técnicas em seis grupos.

## Regra zero

**Refactoring não muda comportamento.** Se o comportamento muda, não é refactoring — é feature ou bugfix, e vai em commit separado.

Antes de aplicar qualquer técnica:

1. Existe teste cobrindo o trecho? Se não, escreva o teste primeiro (caracterização) — ele é a rede de segurança.
2. Rode `tsc --noEmit`, o lint e a suíte de testes; confirme verde **antes** de mexer.
3. Aplique **uma** técnica por vez, em passos pequenos, rodando os testes entre passos.
4. Confirme verde depois. Só então commite, nomeando a técnica (`refactor: extract hook de useCheckout`).

Em TypeScript o compilador é parte da rede de segurança, mas só cobre o que está tipado: **`any`, `as` e `@ts-ignore` são buracos na rede**. Refatorar código cheio de `any` sem teste é reescrever no escuro.

## Como usar esta skill

1. **Identifique o smell** — parta do sintoma observado, não da técnica preferida.
2. **Consulte o mapa** abaixo para chegar às técnicas candidatas.
3. **Abra o arquivo de referência do grupo** (`references/0N-*.md`) e leia gatilhos, Antes/Depois, passos, e principalmente **Quando NÃO aplicar**.
4. **Cheque a Nota TypeScript** — várias técnicas clássicas mudam de forma em TS/JS: umas viram um recurso da linguagem, outras (como mutação de parâmetro) ficam **mais** importantes do que em linguagens com imutabilidade por padrão.
5. **Aplique em passos pequenos**, deixando o compilador apontar os usos.

Carregue apenas o arquivo do grupo relevante. Não leia os seis de uma vez.

## Arquivos de referência

| # | Grupo | Arquivo | Técnicas |
|---|---|---|---|
| 1 | Composing Methods — decompor funções, hooks e componentes | `references/01-composing-methods.md` | 9 |
| 2 | Moving Features between Objects — mover responsabilidade entre módulos | `references/02-moving-features.md` | 8 |
| 3 | Organizing Data — modelar e encapsular dados | `references/03-organizing-data.md` | 15 |
| 4 | Simplifying Conditional Expressions — condicionais e narrowing | `references/04-conditionals.md` | 8 |
| 5 | Simplifying Method Calls — assinaturas e contratos | `references/05-method-calls.md` | 14 |
| 6 | Dealing with Generalization — herança vs composição | `references/06-generalization.md` | 12 |

Total: **66 técnicas**.

## Mapa sintoma → técnica

### Função, hook ou componente problemático

| Sintoma observável | Técnicas candidatas | Grupo |
|---|---|---|
| Função/componente longo (> ~30 linhas) ou com comentário explicando blocos | Extract Method (função, custom hook ou componente), Decompose Conditional | 1, 4 |
| `useEffect` fazendo três coisas diferentes | Extract Method (um hook por responsabilidade) | 1 |
| Corpo da função é mais curto que o nome; indireção sem valor | Inline Method | 1 |
| Expressão longa dentro de JSX ou de uma condição | Extract Variable | 1 |
| `const` intermediária usada uma vez, vinda de expressão trivial | Inline Temp | 1 |
| Valor calculado repetido em vários pontos | Replace Temp with Query (função pura / `useMemo`) | 1 |
| `let` reaproveitado para dois propósitos | Split Temporary Variable | 1 |
| Parâmetro reatribuído, ou array/objeto recebido sendo mutado | Remove Assignments to Parameters | 1 |
| Função enorme com muitas locais interdependentes | Replace Method with Method Object (classe ou closure) | 1 |
| Algoritmo confuso quando há um mais direto (`Map`/`Set`, métodos de array) | Substitute Algorithm | 1 |

### Distribuição de responsabilidade

| Sintoma observável | Técnicas candidatas | Grupo |
|---|---|---|
| Função usa mais dados de outro módulo que do próprio (Feature Envy) | Move Method | 2 |
| Cálculo de domínio morando no componente ou no controller | Move Method, Extract Class | 2 |
| Módulo/classe com responsabilidades demais (service ou componente que faz tudo) | Extract Class (novo service, módulo ou custom hook) | 2, 6 |
| Módulo/classe que quase não faz nada | Inline Class, Collapse Hierarchy | 2, 6 |
| Cadeia `pedido.cliente.endereco.cidade` no componente | Hide Delegate (ou achatar no DTO de apresentação) | 2 |
| Camada que só repassa chamadas 1:1 | Remove Middle Man | 2 |
| Falta um método em tipo de terceiro (lib, SDK) | Introduce Foreign Method (função standalone — **nunca** monkey patch) | 2 |
| Precisa de várias operações sobre um tipo de terceiro | Introduce Local Extension (wrapper ou módulo de funções) | 2 |

### Dados e modelagem

| Sintoma observável | Técnicas candidatas | Grupo |
|---|---|---|
| Campo mutável exposto para fora do módulo | Encapsulate Field (`readonly`, `#priv`, fronteira de módulo) | 3 |
| Array interno retornado por referência | Encapsulate Collection (`ReadonlyArray`, cópia) | 3 |
| Getter caro chamado em render | Self Encapsulate Field (memo) | 3 |
| `string`/`number` carregando semântica (id, e-mail, CEP, dinheiro) | Replace Data Value with Object (branded type) | 3 |
| Tupla/array com posições de significado fixo | Replace Array with Object | 3 |
| Literal solto com significado de negócio | Replace Magic Number with Symbolic Constant (`as const`) | 3 |
| `if`/`switch` repetido sobre um campo de tipo | Replace Type Code with Class / Subclasses / State-Strategy | 3 |
| Subclasses ou variantes que só diferem por constantes | Replace Subclass with Fields (`Record` de metadados) | 3 |
| Dado do servidor copiado para `useState` e podendo divergir | Duplicate Observed Data (fonte única; derive, não sincronize) | 3 |
| Igualdade/identidade de entidades confusa; cache duplicando objetos | Change Value to Reference, Change Reference to Value | 3 |
| Referência circular quebrando `JSON.stringify` ou `structuredClone` | Change Bidirectional Association to Unidirectional (normalizar) | 3 |

### Condicionais

| Sintoma observável | Técnicas candidatas | Grupo |
|---|---|---|
| Vários `if` diferentes com o mesmo resultado | Consolidate Conditional Expression (predicado nomeado) | 4 |
| Código idêntico em todos os ramos | Consolidate Duplicate Conditional Fragments | 4 |
| Condição complexa e ilegível | Decompose Conditional | 4 |
| `switch`/`if` sobre tipo, repetido em vários lugares | Replace Conditional with Polymorphism (union + `assertNever`, ou `Record` de handlers) | 4 |
| Flag booleana controlando saída de loop | Remove Control Flag (`some`/`every`/`find`) | 4 |
| `if` aninhado com mais de 2 níveis | Replace Nested Conditional with Guard Clauses (early return) | 4 |
| Checagem de `null`/`undefined` repetida em muitos pontos | Introduce Null Object (ou `?.`/`??` e estado `'vazio'` na união) | 4 |
| Suposição implícita sobre o estado, não declarada | Introduce Assertion (assertion function que **estreita o tipo**) | 4 |

### Assinaturas e contratos

| Sintoma observável | Técnicas candidatas | Grupo |
|---|---|---|
| Função precisa de mais/menos informação | Add Parameter (objeto de opções com default), Remove Parameter | 5 |
| Nome não descreve o que a função faz | Rename Method (rename do TS Server; alias `@deprecated` em API pública) | 5 |
| Lista longa de parâmetros posicionais que andam juntos | Introduce Parameter Object, Preserve Whole Object | 5 |
| Parâmetro booleano posicional escolhendo o caminho | Replace Parameter with Explicit Methods (ou union de comando) | 5 |
| Função que consulta **e** modifica | Separate Query from Modifier | 5 |
| Funções quase idênticas diferindo por um valor | Parameterize Method | 5 |
| Setter público / objeto mutado após criado | Remove Setting Method (`readonly` + spread) | 5 |
| Parâmetro que o próprio receptor poderia obter | Replace Parameter with Method Call | 5 |
| Export público que ninguém de fora usa | Hide Method (não exportar; `#priv`; closure) | 5 |
| Construção exige lógica ou escolhe o tipo concreto | Replace Constructor with Factory Method (função factory) | 5 |
| Função devolve `-1`, `null` ou `false` para sinalizar erro | Replace Error Code with Exception (na prática: `Result` tipado) | 5 |
| `try/catch` usado para fluxo previsível | Replace Exception with Test | 5 |

### Herança e hierarquia

| Sintoma observável | Técnicas candidatas | Grupo |
|---|---|---|
| Campo/método duplicado em classes irmãs (ex.: classes de erro) | Pull Up Field, Pull Up Method | 6 |
| Construtores de subclasses com corpo duplicado | Pull Up Constructor Body | 6 |
| Membro na base usado por só uma subclasse | Push Down Field, Push Down Method | 6 |
| Duas classes/módulos com código comum | Extract Superclass (ou extrair função/módulo comum) | 6 |
| Cliente precisa só de parte da API; falta fake em teste | Extract Interface (tipo estrutural — quase gratuito) | 6 |
| Parte da classe é usada só em alguns casos | Extract Subclass (em TS: discriminated union) | 6 |
| Hierarquia de uma-só-subclasse; classe base anêmica | Collapse Hierarchy | 6 |
| Variantes com a mesma estrutura de algoritmo | Form Template Method (em TS: passos injetados) | 6 |
| Herança usada só para reaproveitar código | Replace Inheritance with Delegation | 6 |
| Delegação trivial repassando tudo | Replace Delegation with Inheritance | 6 |

## Heurísticas de TypeScript

Aplicar a técnica clássica ao pé da letra pode piorar o código. Antes de refatorar, cheque se um recurso da linguagem já resolve — e onde TS **agrava** o problema em vez de resolver:

- **`sealed class` → discriminated union.** Modele variantes com `type X = A | B | C` sobre um campo discriminante e feche o `switch` com `default: assertNever(x)`. Assim o compilador acusa o caso novo esquecido; sem o `assertNever`, não acusa.
- **Erro esperado é valor.** `type Result<T, E> = { ok: true; value: T } | { ok: false; error: E }`. Em JS o `catch` recebe `unknown` e não existe exceção checada — retornar valor tipado é o único jeito de o compilador cobrar tratamento.
- **Asserção que estreita o tipo.** `function assertPedidoAberto(p: Pedido): asserts p is PedidoAberto` valida em runtime **e** refina o tipo. Já `as` não valida nada: é uma promessa ao compilador, e é o oposto de uma asserção.
- **Validação na fronteira.** Todo dado externo (HTTP, storage, query string) entra como `unknown` e é validado com Zod ou equivalente. `as MinhaResposta` num `fetch` é o buraco mais comum na tipagem.
- **`readonly` é só compile-time.** Não protege em runtime; para garantia real, `Object.freeze`. E `readonly` é superficial: objeto aninhado continua mutável.
- **Mutação de parâmetro é pior aqui.** Diferente de linguagens com imutabilidade por padrão, arrays e objetos chegam por referência e `const` não protege o conteúdo. `itens.push(x)` num array de props muta o estado do pai. Tipe entradas como `Readonly`/`ReadonlyArray` e derive com spread ou `toSorted`/`with`.
- **Branded types** matam primitive obsession: `type PedidoId = string & { readonly __brand: 'PedidoId' }`, sempre com função de construção que valida.
- **`as const` + union de literais** em vez de `enum`. O `enum` do TS tem semântica de runtime irregular e `enum` numérico aceita qualquer número.
- **Igualdade é referencial.** Não há igualdade estrutural nativa: compare por id, normalize caches em `Record<id, T>`/`Map`, e lembre que em React uma referência nova a cada render quebra memoização.
- **Fronteira de módulo é o principal mecanismo de encapsulamento.** Antes de `#priv` ou getters, pergunte se basta não exportar.
- **React: derive, não sincronize.** Estado que pode ser calculado a partir de props/estado existente não deve morar em `useState` alimentado por `useEffect`. Esse é o `Duplicate Observed Data` do front-end.
- **React: hooks vêm antes de qualquer early return.** Guard clauses em componente entram depois de todos os hooks.
- **React: extrair componente e extrair hook são Extract Method.** Um para JSX, outro para lógica com estado/efeito.
- **Assinatura sem argumentos nomeados.** O equivalente é o objeto de opções desestruturado com defaults — e ele também elimina o parâmetro booleano posicional.
- **Herança é exceção em TS.** Tipos estruturais tornam Extract Interface quase gratuito e composição quase sempre melhor; não existe delegação por palavra-chave, então delegue explicitamente os métodos que importam em vez de recorrer a `Proxy`.

## Ordem de aplicação quando há vários smells

Refatore de dentro para fora, para não mover código sujo de lugar:

1. **Tipar a fronteira** — validar entrada externa e eliminar `any`/`as` do trecho. Sem isso o compilador não ajuda nos passos seguintes.
2. **Guard clauses e decomposição de condicionais** — deixa o fluxo legível.
3. **Extract Method / Variable / hook / componente** — cria unidades nomeadas.
4. **Modelagem de dados** — parameter object, branded type, discriminated union, encapsulamento.
5. **Ajuste de assinaturas** — rename, objeto de opções, factory, sobre tipos já corretos.
6. **Movimentação entre módulos** — mover código já limpo para onde ele pertence.
7. **Hierarquia / composição** — por último, quando as responsabilidades estão nítidas.

Começar pela hierarquia é o erro mais comum: cria abstração sobre código que ainda vai mudar de forma.

## Checklist de code review

- [ ] Há teste verde cobrindo o comportamento antes e depois?
- [ ] `tsc --noEmit` e o lint passam, sem `any`, `as` ou `@ts-ignore` novos?
- [ ] O commit é só refactoring, sem mudança de comportamento nem feature embutida?
- [ ] A técnica aplicada tem nome? Está na mensagem do commit?
- [ ] A seção "Quando NÃO aplicar" da técnica foi lida e não se aplica aqui?
- [ ] O resultado é TypeScript idiomático, ou é Java/Kotlin escrito com sintaxe TS (classes e herança onde composição bastava)?
- [ ] Nada de mutação de props, state, parâmetro ou array compartilhado?
- [ ] Estado derivado é derivado, não sincronizado por `useEffect`?
- [ ] Subscrições, listeners e timers têm cleanup?
- [ ] Erro de negócio é valor tipado, e exceção ficou para o inesperado?
- [ ] A abstração criada tem pelo menos dois usos reais, ou é especulativa?
- [ ] Nomes revelam intenção de negócio, não mecânica de implementação?

## Fonte

Catálogo derivado de https://refactoring.guru/refactoring/techniques (66 técnicas, 6 grupos), com problema/solução/passos da fonte e exemplos escritos para TypeScript (React, Node, React Native).

Skill irmã: **design-patterns-typescript** (22 padrões GoF). Use esta quando o problema é código existente com smell; a outra quando o problema é **desenho** de uma solução.
