# Simplifying Conditional Expressions — Simplificando Condicionais

> Grupo 4 de 6 · 8 técnicas · Fonte: refactoring.guru/refactoring/techniques

## Quando este grupo se aplica
- Funções com aninhamento de `if` de 3+ níveis, formando "seta" apontando para a direita.
- Condições booleanas longas (`&&`/`||` encadeados) sem nome que explique a intenção de negócio.
- `switch`/`if-else` que ramifica por `status`, string literal ou `typeof`/`in` e repete a mesma forma em vários módulos.
- Ramos de `if`/`else` (ou de `switch`) que começam ou terminam com as mesmas linhas (`logger.info`, `analytics.track`, `setState`).
- `let` booleano (`let encontrado`, `let valido`, `let deveParar`) controlando a saída de um `for`/`while`.
- Cascatas de `if (x !== null)`, `?.` empilhado ou `||` usado como default só para produzir um valor padrão.
- Props opcionais e flags soltas (`loading`, `error`, `data`) que permitem combinações impossíveis.
- Pré-condições documentadas em comentário (`// só chamar depois de abrir o pedido`) em vez de verificadas — ou "resolvidas" com `as`.

## Índice
1. [Consolidate Conditional Expression](#1-consolidate-conditional-expression)
2. [Consolidate Duplicate Conditional Fragments](#2-consolidate-duplicate-conditional-fragments)
3. [Decompose Conditional](#3-decompose-conditional)
4. [Replace Conditional with Polymorphism](#4-replace-conditional-with-polymorphism)
5. [Remove Control Flag](#5-remove-control-flag)
6. [Replace Nested Conditional with Guard Clauses](#6-replace-nested-conditional-with-guard-clauses)
7. [Introduce Null Object](#7-introduce-null-object)
8. [Introduce Assertion](#8-introduce-assertion)

---

## 1. Consolidate Conditional Expression
**PT-BR:** Consolidar Expressão Condicional · **Fonte:** https://refactoring.guru/consolidate-conditional-expression

### Problema
Várias condicionais diferentes levam ao mesmo resultado ou à mesma ação. O código não explica por que elas estão separadas.

### Solução
Unificar todas em uma única expressão (`||` para condicionais consecutivas, `&&` para aninhadas) e extrair essa expressão para uma função com nome de domínio.

### Sinais no código (gatilhos)
- Dois ou mais `if` sequenciais com corpos idênticos (`return 0`, `throw`, mesma chamada).
- `if (a) return 0;` seguido de `if (b) return 0;` seguido de `if (c) return 0;`.
- Aninhamento em que cada nível só existe para acrescentar mais uma condição, sem `else`.
- Condições espalhadas que juntas formam uma regra com nome no negócio ("inelegível", "pode publicar").
- A condição é pura: só lê propriedades, não muta estado, não faz `await` nem I/O.

### Antes
```ts
type Assinatura = {
  readonly suspensa: boolean;
  readonly faturasEmAtraso: number;
  readonly plano: 'gratuito' | 'mensal' | 'anual';
  readonly canceladaEm: string | null;
  readonly mesesAtivos: number;
};

export function calcularDesconto(assinatura: Assinatura, valorMensal: number): number {
  if (assinatura.suspensa) return 0;
  if (assinatura.faturasEmAtraso > 0) return 0;
  if (assinatura.plano === 'gratuito') return 0;
  if (assinatura.canceladaEm !== null) return 0;

  if (assinatura.mesesAtivos >= 24) {
    return valorMensal * 0.2;
  }
  return valorMensal * 0.1;
}
```

### Depois
```ts
type Assinatura = {
  readonly suspensa: boolean;
  readonly faturasEmAtraso: number;
  readonly plano: 'gratuito' | 'mensal' | 'anual';
  readonly canceladaEm: string | null;
  readonly mesesAtivos: number;
};

const MESES_PARA_DESCONTO_MAIOR = 24;

function inelegivelParaDesconto(assinatura: Assinatura): boolean {
  return (
    assinatura.suspensa ||
    assinatura.faturasEmAtraso > 0 ||
    assinatura.plano === 'gratuito' ||
    assinatura.canceladaEm !== null
  );
}

function percentualPorTempoDeCasa(assinatura: Assinatura): number {
  return assinatura.mesesAtivos >= MESES_PARA_DESCONTO_MAIOR ? 0.2 : 0.1;
}

export function calcularDesconto(assinatura: Assinatura, valorMensal: number): number {
  return inelegivelParaDesconto(assinatura)
    ? 0
    : valorMensal * percentualPorTempoDeCasa(assinatura);
}
```

### Passos
1. Confirme que todas as condições são puras: sem mutação, sem `await`, sem `track`/`log` embutido, sem getter que dispara requisição.
2. Confirme que todos os ramos produzem exatamente o mesmo resultado (mesmo valor, mesmo `throw`, mesma chamada).
3. Junte as condições: consecutivas com `||`, aninhadas com `&&`. Preserve a ordem quando o curto-circuito importa (checar `!== null` antes de acessar propriedade).
4. Extraia a expressão para uma função nomeada com o vocabulário do domínio (`inelegivelParaDesconto`, `podeFinalizarPedido`) e tipo de retorno `boolean` explícito.
5. Se a condição também estreita tipo, declare-a como type guard (`(p: Pedido): p is PedidoAberto`) para que o chamador ganhe o narrowing junto com o nome.
6. Rode os testes; depois avalie mover o predicado para o módulo do próprio modelo (`pedido.ts`) e exportá-lo como função pura reutilizável.

### Ganhos
- Deixa explícito que se trata de UMA verificação composta com UMA consequência.
- O nome da função documenta a regra de negócio; a condição fica testável isoladamente.
- Reduz a chance de alguém acrescentar um quinto `if` divergente no meio da sequência.

### Quando NÃO aplicar
- Alguma condição tem efeito colateral (incrementa contador, registra analytics, faz `fetch`) — consolidar muda quantas vezes ela executa por curto-circuito.
- Os ramos parecem iguais mas retornam por motivos semanticamente distintos que precisam de log ou erro específico por caso.
- Uma condição é caríssima e hoje está protegida por outra barata: consolidar sem preservar a ordem degrada performance.
- A expressão resultante passa de ~4 termos e mistura assuntos — extraia dois predicados nomeados e combine-os.

### Nota TypeScript
`&&`/`||` fazem curto-circuito, então a ordem preserva o narrowing: `usuario !== null && usuario.assinatura.ativa` continua compilando com `strictNullChecks`. Predicados compostos podem virar **type guards** (`function ehPedidoAberto(p: Pedido): p is PedidoAberto`), e aí a consolidação entrega narrowing de graça — dois guards combinados com `&&` estreitam para a interseção. Cuidado ao consolidar com `??`: ele tem precedência que exige parênteses ao lado de `||`/`&&` (o compilador acusa). Para predicados de coleção, `every`/`some` costumam ler melhor que uma corrente de `&&`.

---

## 2. Consolidate Duplicate Conditional Fragments
**PT-BR:** Consolidar Fragmentos Condicionais Duplicados · **Fonte:** https://refactoring.guru/consolidate-duplicate-conditional-fragments

### Problema
O mesmo código aparece em todos os ramos de uma condicional.

### Solução
Mover o código duplicado para fora da condicional: para antes, se está no início dos ramos; para depois, se está no fim.

### Sinais no código (gatilhos)
- Primeira e/ou última linha idêntica em todos os ramos de `if`/`else` ou em todos os `case` de um `switch`.
- `analytics.track(...)`, `logger.info(...)`, `setCarregando(false)` repetidos em cada ramo com os mesmos argumentos.
- Todos os ramos terminam liberando o mesmo recurso, chamando o mesmo `onFinish()` ou fazendo o mesmo `res.json(...)`.
- Todos os ramos fazem o mesmo `await` inicial (buscar entidade, abrir transação) e só divergem no meio.
- Um `switch` em que cada `case` difere apenas no "miolo", com abertura e fechamento iguais.

### Antes
```ts
type Pedido = { readonly id: string; readonly cupomId: string | null };

type Deps = {
  readonly estoque: { reservar(pedidoId: string): Promise<void> };
  readonly pagamentos: {
    cobrarComDesconto(pedido: Pedido, cupomId: string): Promise<void>;
    cobrarValorCheio(pedido: Pedido): Promise<void>;
  };
  readonly analytics: { track(evento: string, pedidoId: string): void };
};

export function criarCheckout({ estoque, pagamentos, analytics }: Deps) {
  return async function finalizar(pedido: Pedido): Promise<void> {
    if (pedido.cupomId !== null) {
      analytics.track('checkout_iniciado', pedido.id);
      await estoque.reservar(pedido.id);
      await pagamentos.cobrarComDesconto(pedido, pedido.cupomId);
      analytics.track('checkout_finalizado', pedido.id);
    } else {
      analytics.track('checkout_iniciado', pedido.id);
      await estoque.reservar(pedido.id);
      await pagamentos.cobrarValorCheio(pedido);
      analytics.track('checkout_finalizado', pedido.id);
    }
  };
}
```

### Depois
```ts
type Pedido = { readonly id: string; readonly cupomId: string | null };

type Deps = {
  readonly estoque: { reservar(pedidoId: string): Promise<void> };
  readonly pagamentos: {
    cobrarComDesconto(pedido: Pedido, cupomId: string): Promise<void>;
    cobrarValorCheio(pedido: Pedido): Promise<void>;
  };
  readonly analytics: { track(evento: string, pedidoId: string): void };
};

export function criarCheckout({ estoque, pagamentos, analytics }: Deps) {
  return async function finalizar(pedido: Pedido): Promise<void> {
    // os fragmentos identicos saem dos dois ramos: um antes, outro depois
    analytics.track('checkout_iniciado', pedido.id);
    await estoque.reservar(pedido.id);

    if (pedido.cupomId !== null) {
      await pagamentos.cobrarComDesconto(pedido, pedido.cupomId);
    } else {
      await pagamentos.cobrarValorCheio(pedido);
    }

    analytics.track('checkout_finalizado', pedido.id);
  };
}
```

### Passos
1. Identifique as linhas idênticas — idênticas em código E em argumentos, não apenas parecidas.
2. Duplicatas no início dos ramos: mova para imediatamente antes da condicional.
3. Duplicatas no fim: mova para imediatamente depois — ou para um `finally`, se também devem rodar quando o ramo lança.
4. Duplicatas no meio: só mova se der para deslocar para o início ou fim sem alterar a ordem observável dos efeitos (inclusive a ordem dos `await`).
5. Se o fragmento tem várias linhas, extraia para uma função nomeada antes de mover — o diff fica revisável.
6. Cuidado com `return`, `throw`, `break` e `continue` nos ramos: se um ramo sai antes, o código movido para depois da condicional deixa de executar naquele caminho. Aí use `try`/`finally` ou não mova.
7. Depois de encolher os ramos, veja se a condicional inteira não virou uma expressão: `const cobrar = pedido.cupomId !== null ? comDesconto : cheio;` e uma única chamada fora do `if`.

### Ganhos
- Elimina duplicação real; corrigir o fragmento passa a ser uma edição em um lugar.
- Encolhe os ramos até sobrar só a diferença de verdade, revelando qual é a decisão.
- Frequentemente expõe que a condicional pode virar uma única linha, um ternário ou uma escolha de função.

### Quando NÃO aplicar
- Um dos ramos faz `return`/`throw` antes do fragmento comum — mover para depois muda o comportamento silenciosamente.
- O fragmento tem efeito cuja ordem relativa às outras linhas do ramo importa (reservar antes de cobrar, `commit` antes de notificar).
- O "duplicado" difere em algum argumento (mesmo evento de analytics com propriedades diferentes): não é duplicação, é coincidência de forma.
- O fragmento precisa rodar mesmo em caso de exceção e você moveria para fora sem `finally` — a ferramenta certa é `try`/`finally`.
- Em React, a "linha comum" é uma chamada de hook: hooks não podem sair de dentro de um ramo condicional para outro sem quebrar a ordem de chamada. Nesse caso mover é obrigatório, mas para **antes** de qualquer condicional.

### Nota TypeScript
`if` não é expressão em TS: quando o objetivo é "calcular no ramo, agir uma vez fora", use ternário, um `switch` dentro de função extraída, ou um mapa de opções. Em `async`, mover uma linha para fora do `if` muda o ponto de suspensão — verifique se ninguém dependia da ordem dos `await`. Para "sempre no fim", `try/finally` cobre exceção e `return`; em React o par equivalente é `useEffect` com cleanup, e em TanStack Query os callbacks `onSettled`. `switch` com fall-through intencional é uma forma legítima de consolidar `case`s idênticos, mas exige comentário — `noFallthroughCasesInSwitch` reclama do resto.

---

## 3. Decompose Conditional
**PT-BR:** Decompor Condicional · **Fonte:** https://refactoring.guru/decompose-conditional

### Problema
Você tem uma condicional complexa (`if-else` ou `switch`) em que tanto a condição quanto os corpos exigem esforço para entender.

### Solução
Extrair para funções nomeadas as três partes complicadas: a condição, o bloco `then` e o bloco `else`.

### Sinais no código (gatilhos)
- Condição com aritmética, comparação de datas/timestamps ou 3+ termos, sem nome.
- Ramos com 4+ linhas cada, cada um fazendo algo conceitualmente distinto.
- É preciso ler o corpo para descobrir o que a condição significa em termos de negócio.
- Comentário acima do `if` explicando o que a condição quer dizer.
- Em JSX, um ternário com duas árvores grandes dentro do `return`.
- Números mágicos (`199`, `0.4`, `1000 * 60 * 30`) misturados na condição.

### Antes
```ts
type ItemCarrinho = { readonly precoUnitario: number; readonly quantidade: number; readonly digital: boolean };
type Carrinho = {
  readonly itens: ReadonlyArray<ItemCarrinho>;
  readonly cepDestino: string;
  readonly assinante: boolean;
  readonly criadoEm: Date;
};

export function calcularFrete(carrinho: Carrinho, agora: Date): number {
  if (
    carrinho.assinante &&
    carrinho.itens.reduce((s, i) => s + i.precoUnitario * i.quantidade, 0) >= 199 &&
    agora.getTime() - carrinho.criadoEm.getTime() < 1000 * 60 * 30
  ) {
    const pesoTotal = carrinho.itens
      .filter((i) => !i.digital)
      .reduce((s, i) => s + i.quantidade * 0.4, 0);
    return pesoTotal === 0 ? 0 : Math.max(0, pesoTotal * 1.5 - 20);
  } else {
    const fisicos = carrinho.itens.filter((i) => !i.digital);
    const base = fisicos.length === 0 ? 0 : 14.9;
    const adicional = fisicos.reduce((s, i) => s + i.quantidade, 0) * 2.5;
    const zonaRemota = carrinho.cepDestino.startsWith('69') ? 1.4 : 1;
    return (base + adicional) * zonaRemota;
  }
}
```

### Depois
```ts
const SUBTOTAL_MINIMO_PROMOCIONAL = 199;
const JANELA_PROMOCIONAL_MS = 1000 * 60 * 30;
const PESO_POR_ITEM_KG = 0.4;
const FRETE_BASE = 14.9;

const subtotal = (carrinho: Carrinho): number =>
  carrinho.itens.reduce((s, i) => s + i.precoUnitario * i.quantidade, 0);

const itensFisicos = (carrinho: Carrinho): ReadonlyArray<ItemCarrinho> =>
  carrinho.itens.filter((i) => !i.digital);

function elegivelAFretePromocional(carrinho: Carrinho, agora: Date): boolean {
  return (
    carrinho.assinante &&
    subtotal(carrinho) >= SUBTOTAL_MINIMO_PROMOCIONAL &&
    agora.getTime() - carrinho.criadoEm.getTime() < JANELA_PROMOCIONAL_MS
  );
}

function fretePromocional(carrinho: Carrinho): number {
  const peso = itensFisicos(carrinho).reduce((s, i) => s + i.quantidade * PESO_POR_ITEM_KG, 0);
  return peso === 0 ? 0 : Math.max(0, peso * 1.5 - 20);
}

function freteTabelado(carrinho: Carrinho): number {
  const fisicos = itensFisicos(carrinho);
  const base = fisicos.length === 0 ? 0 : FRETE_BASE;
  const adicional = fisicos.reduce((s, i) => s + i.quantidade, 0) * 2.5;
  return (base + adicional) * (carrinho.cepDestino.startsWith('69') ? 1.4 : 1);
}

export function calcularFrete(carrinho: Carrinho, agora: Date): number {
  return elegivelAFretePromocional(carrinho, agora) ? fretePromocional(carrinho) : freteTabelado(carrinho);
}
```

### Passos
1. Extraia a condição para uma função (ou `const` local) com nome que descreva o significado, não a mecânica: `elegivelAFretePromocional`, não `checaAssinanteEValor`.
2. Extraia o ramo `then` para uma função nomeada pelo resultado que produz; repita para o `else` e para cada `case`.
3. Deixe a função original com uma expressão só: ternário ou `switch` curto que se lê como a regra de negócio.
4. Promova os números mágicos que sobraram para `const` no topo do módulo (ou `as const` em um objeto de configuração).
5. Se as funções extraídas dependem de um único parâmetro, mova-as para o módulo daquele tipo — passam a ser reutilizáveis e testáveis sem montar o resto do contexto.
6. Em JSX, extraia cada ramo para um componente nomeado (`<CarrinhoVazio />`, `<ResumoFrete />`) em vez de aninhar ternários; extraia a condição para um `const` nomeado ou um custom hook.
7. Rode os testes e, se surgir uma explosão de parâmetros, pare: o sinal é Extract Class / objeto de contexto, não mais extração de função.

### Ganhos
- A função original vira um resumo legível da decisão.
- Cada parte fica testável em unidade, com nome que aparece no stack trace.
- Reduz carga cognitiva: não é preciso manter condição e dois corpos na cabeça ao mesmo tempo.

### Quando NÃO aplicar
- Condição e corpos já são triviais (`if (itens.length === 0) return 0;`) — extrair só adiciona indireção.
- A extração exigiria passar 5+ parâmetros ou devolver um objeto artificial só para carregar estado intermediário; o problema real é a modelagem.
- Os ramos precisam de `return`/`break` da função externa ou mutam muitas variáveis locais compartilhadas.
- Você acabaria com nomes vazios (`parte1`, `tratarElse`) — nome ruim é pior que o `if` inline.

### Nota TypeScript
Um `const` local nomeado (`const elegivel = ...`) já decompõe sem criar função, e mantém o narrowing no escopo — extrair para função **perde** o narrowing do chamador, a menos que a função seja um type guard (`(x: Pedido) => x is PedidoAberto`). Prefira arrow functions com tipo de retorno explícito: o retorno anotado congela o contrato e evita inferência acidental de `any` implícito em callbacks. Em React, ramos extraídos para componentes reduzem re-render e ficam isoláveis em teste/Storybook; a condição extraída para um custom hook (`useElegivelAFrete`) mantém `useMemo` junto da regra. Evite extrair para dentro do próprio componente (função recriada a cada render) quando ela for passada como prop — nesse caso, módulo ou `useCallback`.

---

## 4. Replace Conditional with Polymorphism
**PT-BR:** Substituir Condicional por Polimorfismo · **Fonte:** https://refactoring.guru/replace-conditional-with-polymorphism

### Problema
Uma condicional executa ações diferentes conforme o tipo do objeto, o valor de um campo de tipo (`status`, string literal) ou o resultado de uma verificação de forma.

### Solução
Modelar cada variante explicitamente — em TypeScript, uma **discriminated union** — e despachar com um `switch` exaustivo (`default: assertNever(x)`) ou com um mapa `Record<Tipo, Handler>`; comportamento que é do domínio vai para o módulo/método de cada variante.

### Sinais no código (gatilhos)
- `switch (tipo)` / `if (typeof x === ...)` / `if ('campo' in x)` sobre o mesmo discriminador repetido em 2+ arquivos.
- União de string literais com `switch` que precisa mudar em toda a base a cada valor novo.
- `default: throw new Error('tipo desconhecido')` — sinal de que o compilador não está ajudando.
- Estado de tela representado por flags soltas (`loading`, `error`, `data`, `vazio`) que se contradizem entre si.
- Props opcionais que só fazem sentido em combinação (`erro?` preenchido junto com `itens?`).
- `as` usado para "convencer" o compilador de qual variante é aquela.

### Antes
```tsx
type Props = {
  readonly carregando: boolean;
  readonly itens: ReadonlyArray<ItemCarrinho>;
  readonly mensagemErro: string | null;
  readonly cupomExpirado: boolean;
  readonly onTentarNovamente: () => void;
};

export function ResumoCarrinho(props: Props) {
  if (props.carregando) {
    return <Spinner />;
  } else {
    if (props.mensagemErro !== null) {
      return (
        <div>
          <p>{props.mensagemErro}</p>
          <button onClick={props.onTentarNovamente}>Tentar novamente</button>
        </div>
      );
    } else {
      if (props.itens.length === 0) {
        return <p>Seu carrinho esta vazio</p>;
      }
      return <ListaItens itens={props.itens} avisoCupom={props.cupomExpirado} />;
    }
  }
}
```

### Depois
```tsx
export type CarrinhoState =
  | { readonly status: 'carregando' }
  | { readonly status: 'vazio' }
  | { readonly status: 'erro'; readonly mensagem: string }
  | { readonly status: 'pronto'; readonly itens: ReadonlyArray<ItemCarrinho> };

export function assertNever(valor: never): never {
  throw new Error(`Variante nao tratada: ${JSON.stringify(valor)}`);
}

// mapa de despacho: as chaves do Record ja sao exaustivas por construcao
const TITULO: Record<CarrinhoState['status'], string> = {
  carregando: 'Carregando seu carrinho',
  vazio: 'Carrinho vazio',
  erro: 'Nao foi possivel carregar',
  pronto: 'Resumo do pedido',
};

export function ResumoCarrinho({ state, onTentarNovamente }: { state: CarrinhoState; onTentarNovamente: () => void }) {
  switch (state.status) {
    case 'carregando':
      return <Spinner titulo={TITULO.carregando} />;
    case 'vazio':
      return <p>{TITULO.vazio}</p>;
    case 'erro':
      // aqui state e' estreitado: state.mensagem existe, sem cast
      return <ErroCarrinho mensagem={state.mensagem} onTentarNovamente={onTentarNovamente} />;
    case 'pronto':
      return <ListaItens itens={state.itens} />;
    default:
      // ao adicionar uma variante nova, este argumento deixa de ser `never` e o build quebra
      return assertNever(state);
  }
}
```

### Passos
1. Identifique o discriminador (campo de status, `typeof`, conjunto de flags) e liste todas as variantes reais — incluindo as combinações impossíveis que as flags permitem hoje.
2. Declare a união com um campo literal comum (`status`/`kind`/`type`) e apenas os dados que aquela variante realmente tem. Sem campos opcionais para "às vezes existe".
3. Extraia o `switch` para uma função própria se ele estiver misturado com outras responsabilidades.
4. Comportamento que pertence ao domínio: dê à união uma função por variante (ou um método na classe de cada variante) e apague o ramo correspondente.
5. Comportamento que pertence à borda (renderizar, mapear para DTO, navegar): mantenha **um** `switch` exaustivo com `default: assertNever(state)`. Adicionar variante passa a ser erro de compilação em todos os pontos que faltam tratar.
6. Quando cada variante vira um handler homogêneo (mesma assinatura), troque o `switch` por `Record<Status, Handler>`: fica declarativo, cada handler é testável isolado e o `Record` já exige todas as chaves. Para handlers que recebem o payload da variante, tipe com mapped type: `{ [K in Status]: (s: Extract<CarrinhoState, { status: K }>) => ReactNode }`.
7. Remova as flags antigas e ajuste os testes para construir variantes em vez de combinações de booleanos.

### Ganhos
- Estados impossíveis deixam de ser representáveis (`carregando` com `mensagemErro` preenchido não compila mais).
- Adicionar variante é adicionar um membro na união; o compilador aponta todos os pontos que precisam mudar (Open/Closed com rede de segurança).
- Elimina condicionais quase idênticas espalhadas e segue Tell-Don't-Ask onde o comportamento é do domínio.

### Quando NÃO aplicar
- A condicional aparece em um único lugar e tem 2 ramos simples — a união custa mais do que economiza.
- O discriminador vem de fonte externa aberta e instável (código de erro de API): valide na fronteira e mapeie para a sua união fechada, com fallback explícito.
- O comportamento por ramo é puramente de apresentação e não pode entrar no domínio (não coloque JSX dentro do tipo de domínio).
- Você precisaria de `default: return null` só para compilar — isso anula o benefício; revise a modelagem.
- As variantes têm exatamente o mesmo comportamento e diferem apenas em dados: aí é parâmetro, não polimorfismo.

### Nota TypeScript
Discriminated union + `switch` exaustivo + `assertNever` é o substituto direto de hierarquia com método abstrato quando a decisão vive na borda; dentro do `case` o narrowing dispensa `as`. Escolha: **`switch`** quando os ramos têm formas de retorno/assinaturas diferentes, precisam de `await`, ou usam o payload de cada variante (narrowing é automático); **`Record<Tipo, Handler>`** quando os handlers são homogêneos, você quer registrá-los/testá-los separado ou compor o mapa (`{ ...base, ...overrides }`). O `Record` garante exaustividade nas chaves, mas ao indexá-lo com uma variável você perde o narrowing do payload — daí o mapped type com `Extract`. Em React, renderizar por união discriminada é o caso mais comum: `useReducer<Estado, Acao>` com duas uniões (estado e ação) troca dezenas de `setState` booleanos por transições verificadas. Evite `enum`: prefira união de literais ou `as const`.

---

## 5. Remove Control Flag
**PT-BR:** Remover Flag de Controle · **Fonte:** https://refactoring.guru/remove-control-flag

### Problema
Uma variável booleana serve de flag de controle para decidir quando parar ou pular iterações de um laço.

### Solução
Substituir a flag por `return`, `break` ou `continue` — ou, melhor ainda, por um método de array que já expressa a intenção.

### Sinais no código (gatilhos)
- `let encontrado = false` / `let valido = true` / `let deveParar = false` seguido de laço.
- Condição de laço combinando índice e flag: `while (i < n && !achou)`.
- Atribuição da flag dentro do `if` e leitura dela depois do laço para decidir o retorno.
- Laço que percorre a coleção inteira mesmo depois de já saber a resposta.
- `forEach` com um `if` que gostaria de ser um `break` (e não pode).
- Flag negada no retorno (`return !temExpirado`) — quase sempre é `every`.

### Antes
```ts
type ItemPedido = { readonly sku: string; readonly quantidade: number; readonly estoqueReservado: boolean };
type Cupom = { readonly codigo: string; readonly expirado: boolean };
type Pedido = { readonly itens: ReadonlyArray<ItemPedido>; readonly cupons: ReadonlyArray<Cupom> };

export function primeiroItemSemEstoque(pedido: Pedido): ItemPedido | undefined {
  let encontrado: ItemPedido | undefined = undefined;
  let achou = false;
  let i = 0;
  while (i < pedido.itens.length && !achou) {
    const item = pedido.itens[i];
    if (item !== undefined && !item.estoqueReservado && item.quantidade > 0) {
      encontrado = item;
      achou = true;
    }
    i += 1;
  }
  return encontrado;
}

export function todosCuponsValidos(pedido: Pedido): boolean {
  let temExpirado = false;
  for (const cupom of pedido.cupons) {
    if (cupom.expirado) {
      temExpirado = true;
    }
  }
  return !temExpirado;
}
```

### Depois
```ts
type ItemPedido = { readonly sku: string; readonly quantidade: number; readonly estoqueReservado: boolean };
type Cupom = { readonly codigo: string; readonly expirado: boolean };
type Pedido = { readonly itens: ReadonlyArray<ItemPedido>; readonly cupons: ReadonlyArray<Cupom> };

export function primeiroItemSemEstoque(pedido: Pedido): ItemPedido | undefined {
  return pedido.itens.find((item) => !item.estoqueReservado && item.quantidade > 0);
}

export function todosCuponsValidos(pedido: Pedido): boolean {
  return pedido.cupons.every((cupom) => !cupom.expirado);
}

export function temCupomExpirado(pedido: Pedido): boolean {
  return pedido.cupons.some((cupom) => cupom.expirado);
}

export function posicaoDoPrimeiroInvalido(pedido: Pedido): number {
  return pedido.itens.findIndex((item) => item.quantidade <= 0);
}

export async function reservarAteFalhar(
  pedido: Pedido,
  reservar: (sku: string) => Promise<boolean>,
): Promise<string | null> {
  for (const item of pedido.itens) {
    const ok = await reservar(item.sku);
    if (!ok) {
      return item.sku; // sai na hora, sem flag
    }
  }
  return null;
}
```

### Passos
1. Localize a atribuição da flag que significa "já decidi" ou "pode parar".
2. Troque a atribuição pelo operador de fluxo adequado: `return` para sair da função, `break` para sair do laço, `continue` para pular a iteração.
3. Remova a flag, a leitura dela na condição do laço e o `if` residual depois do laço.
4. Prefira substituir o laço inteiro por um método de array quando a intenção for padrão: `find`, `findIndex`, `findLast`, `some`, `every`, `filter`, `includes`. Todos fazem curto-circuito onde faz sentido.
5. Se o corpo do laço é assíncrono e sequencial, mantenha `for...of` com `return`/`break` — `forEach` não aceita `break` e não espera `await`. Para paralelo com curto-circuito, `Promise.all` + `some` no resultado, ou `Promise.race`.
6. Em laços aninhados, use `break` com label (`externo: for (...) { break externo; }`) em vez de reintroduzir flag — ou extraia o laço interno para uma função e use `return`.
7. Troque `let` por `const` no que sobrou e ajuste os testes; com `noUncheckedIndexedAccess`, usar `find` também elimina o `item !== undefined` que a indexação exigia.

### Ganhos
- Menos estado mutável: nada de `let` que pode ficar dessincronizado do laço.
- Curto-circuito de graça — `find`/`some` param na primeira ocorrência.
- Intenção legível no nome da operação (`every`, `some`) em vez de decifrada de uma flag negada.

### Quando NÃO aplicar
- A "flag" não controla fluxo: é resultado de negócio que precisa ser acumulado e reportado (relatório que lista todas as pendências, não só a primeira).
- É necessário continuar o laço após encontrar o caso, por causa de efeitos em todos os itens.
- A coleção é enorme e você precisa de uma passada só produzindo vários resultados: `find` + `filter` + `some` percorrem três vezes; aí um `for...of` explícito (ou `reduce`) é mais barato.
- Sair antes pularia liberação de recurso e não há `try/finally`.
- O laço já é um pipeline assíncrono/stream: a saída antecipada é `break` no `for await` (que fecha o iterador), não flag.

### Nota TypeScript
Os métodos de array cobrem quase todos os casos e já devolvem `const`: `find`/`findIndex`/`findLast`, `some`/`every`, `includes`. `find` retorna `T | undefined` — com `strictNullChecks` o compilador força você a tratar a ausência, o que a flag escondia. Quando o predicado é um type guard, `find` estreita o resultado: `itens.find((i): i is ItemDigital => i.digital)`. `forEach` não suporta `break` nem `await` — use `for...of`. Em `for await (const x of stream)`, o `break` chama `return()` no iterador e encerra o stream, substituindo flag + leitura completa. `label: for` existe em JS/TS e é preferível a uma flag em laço duplo, apesar da má fama.

---

## 6. Replace Nested Conditional with Guard Clauses
**PT-BR:** Substituir Condicional Aninhada por Cláusulas de Guarda · **Fonte:** https://refactoring.guru/replace-nested-conditional-with-guard-clauses

### Problema
Condicionais aninhadas escondem qual é o fluxo normal de execução; a indentação forma uma seta para a direita.

### Solução
Isolar cada caso especial em uma cláusula de guarda que retorna ou lança imediatamente, no topo da função, deixando o caminho principal sem indentação.

### Sinais no código (gatilhos)
- 3+ níveis de `if` com o `return` de sucesso no ponto mais profundo.
- `if (x !== null) { if (y !== null) { if (autorizado) { ... } } }`.
- Blocos `else` distantes do `if`, tratando erro dezenas de linhas depois.
- Escadas de `?.` e `if (typeof x === 'undefined')` só para desembrulhar ausências.
- Em componente React, JSX de sucesso enterrado dentro de ternários encadeados de `loading`/`error`.

### Antes
```ts
type Result<T, E> = { readonly ok: true; readonly value: T } | { readonly ok: false; readonly error: E };
type CheckoutError =
  | 'usuario-inexistente' | 'assinatura-inativa' | 'carrinho-inexistente'
  | 'carrinho-vazio' | 'endereco-nao-atendido';

export async function iniciarCheckout(
  usuarioId: string,
  carrinhoId: string,
  deps: Deps,
): Promise<Result<Pedido, CheckoutError>> {
  const usuario = await deps.usuarios.buscar(usuarioId);
  if (usuario !== null) {
    if (usuario.assinatura.ativa) {
      const carrinho = await deps.carrinhos.buscar(carrinhoId);
      if (carrinho !== null) {
        if (carrinho.itens.length > 0) {
          if (deps.entrega.atende(carrinho.cepDestino)) {
            return { ok: true, value: await deps.pedidos.criar(usuario, carrinho) };
          } else {
            return { ok: false, error: 'endereco-nao-atendido' };
          }
        } else {
          return { ok: false, error: 'carrinho-vazio' };
        }
      } else {
        return { ok: false, error: 'carrinho-inexistente' };
      }
    } else {
      return { ok: false, error: 'assinatura-inativa' };
    }
  } else {
    return { ok: false, error: 'usuario-inexistente' };
  }
}
```

### Depois
```ts
type Result<T, E> = { readonly ok: true; readonly value: T } | { readonly ok: false; readonly error: E };
type CheckoutError =
  | 'usuario-inexistente'
  | 'assinatura-inativa'
  | 'carrinho-inexistente'
  | 'carrinho-vazio'
  | 'endereco-nao-atendido';

const falha = (error: CheckoutError): Result<never, CheckoutError> => ({ ok: false, error });

export async function iniciarCheckout(
  usuarioId: string,
  carrinhoId: string,
  deps: Deps,
): Promise<Result<Pedido, CheckoutError>> {
  const usuario = await deps.usuarios.buscar(usuarioId);
  if (usuario === null) return falha('usuario-inexistente');
  if (!usuario.assinatura.ativa) return falha('assinatura-inativa');

  const carrinho = await deps.carrinhos.buscar(carrinhoId);
  if (carrinho === null) return falha('carrinho-inexistente');
  if (carrinho.itens.length === 0) return falha('carrinho-vazio');
  if (!deps.entrega.atende(carrinho.cepDestino)) return falha('endereco-nao-atendido');

  // daqui para baixo: usuario e carrinho estreitados para nao-nulo
  return { ok: true, value: await deps.pedidos.criar(usuario, carrinho) };
}
```

### Passos
1. Remova efeitos colaterais das condições antes de reordenar (separe consulta de modificação).
2. Escolha o caso especial mais externo, inverta a condição e transforme em guarda com `return`/`throw` imediato no topo.
3. Repita de fora para dentro até o caminho principal ficar sem indentação, sem `else`.
4. Para ausências, use `if (x === null) return ...` (ou `assertX(x)`): o control flow analysis estreita o tipo para não-nulo no resto da função — ganho que `?.` encadeado não dá.
5. Rode os testes após cada guarda; a ordem define qual erro o chamador recebe, então preserve a precedência original.
6. Se várias guardas produzem o mesmo resultado, aplique Consolidate Conditional Expression nelas.
7. Em componente React, mova todas as guardas para **depois** de todas as chamadas de hook e faça early return de JSX: `if (isLoading) return <Spinner />;`, `if (error !== null) return <ErroCarrinho mensagem={error.message} />;`, e o caminho feliz fica no fim, sem ternários aninhados.

### Ganhos
- O fluxo feliz fica linear e legível de cima para baixo.
- Cada caso especial tem seu erro específico, ao lado da condição que o gera.
- Elimina `else` órfãos e reduz o risco de mexer no ramo errado durante manutenção.

### Quando NÃO aplicar
- Os dois ramos são caminhos de negócio igualmente válidos e simétricos — `if/else` comunica melhor que guarda + fluxo principal.
- A função precisa liberar recurso antes de sair e não há `try/finally`: `return` antecipado vaza.
- A função é uma expressão única (um `switch`/ternário que devolve valor) e múltiplos `return` só somariam ruído.
- A guarda mascararia um erro que deveria propagar (devolver `Result` genérico onde o chamador precisava da causa original).
- Em React, o early return viria antes de um hook — isso é proibido; reorganize.

### Nota TypeScript
Guardas casam com o control flow analysis: depois de `if (usuario === null) return ...`, `usuario` é não-nulo até o fim do escopo, sem `!` e sem `as`. `if (!x)` como guarda é armadilha para `number`/`string` (`0` e `''` são falsy) — compare explicitamente com `null`/`undefined`. Uma guarda que lança e é declarada como `asserts` ou retorna `never` também estreita (`function naoEncontrado(): never { throw ... }`), o que permite `const u = usuario ?? naoEncontrado();`. **Regra dos hooks:** `useState`/`useEffect`/`useQuery` precisam rodar na mesma ordem em todo render, então nenhum hook pode ficar depois de um `return` condicional — chame todos os hooks primeiro e só então faça os early returns de JSX; o ESLint `react-hooks/rules-of-hooks` pega o erro. Se um hook só faz sentido no caminho feliz, extraia o subcomponente que o usa e renderize-o após a guarda.

---

## 7. Introduce Null Object
**PT-BR:** Introduzir Objeto Nulo · **Fonte:** https://refactoring.guru/introduce-null-object

### Problema
Funções retornam `null` em vez de objetos reais e o código enche-se de verificações de ausência repetidas, cada uma reinventando o mesmo valor padrão.

### Solução
Fornecer uma implementação padrão que se comporta como "ausência" — um objeto nulo (`PERFIL_ANONIMO`), uma variante `'vazio'` na união de estado, ou (o mais comum em TS) manter o tipo opcional e resolver o default em um único lugar com `??`.

### Sinais no código (gatilhos)
- O mesmo `if (usuario === null) ... else ...` repetido em 3+ lugares com o mesmo padrão.
- Cadeias `?.nome ?? 'Visitante'`, `?.plano ?? 'gratuito'`, `?.permissoes ?? []` espalhadas por várias telas.
- Cada chamador decide sozinho o default para ausência, e as decisões divergem.
- `!` (non-null assertion) ou `as` aparecendo porque "aqui nunca é nulo".
- Props de componente com `| undefined` e um `if` de fallback dentro de cada componente que as consome.

### Antes
```ts
type Usuario = {
  readonly id: string;
  readonly nome: string;
  readonly permissoes: ReadonlyArray<string>;
  readonly plano: 'gratuito' | 'pro';
};

type Usuarios = { buscar(id: string): Usuario | null };

export function criarPerfilPresenter(usuarios: Usuarios) {
  function cabecalho(usuarioId: string | null): string {
    const usuario = usuarioId === null ? null : usuarios.buscar(usuarioId);
    return usuario === null ? 'Visitante' : usuario.nome;
  }

  function podeEditar(usuarioId: string | null, recurso: string): boolean {
    const usuario = usuarioId === null ? null : usuarios.buscar(usuarioId);
    if (usuario === null) return false;
    return usuario.permissoes.includes(`editar:${recurso}`);
  }

  function percentualDesconto(usuarioId: string | null): number {
    const usuario = usuarioId === null ? null : usuarios.buscar(usuarioId);
    if (usuario === null) return 0;
    return usuario.plano === 'pro' ? 0.2 : 0;
  }

  return { cabecalho, podeEditar, percentualDesconto };
}
```

### Depois
```ts
export type Perfil = {
  readonly nomeExibicao: string;
  podeEditar(recurso: string): boolean;
  percentualDesconto(): number;
};

export const PERFIL_ANONIMO: Perfil = {
  nomeExibicao: 'Visitante',
  podeEditar: () => false,
  percentualDesconto: () => 0,
};

function perfilDe(usuario: Usuario): Perfil {
  return {
    nomeExibicao: usuario.nome,
    podeEditar: (recurso) => usuario.permissoes.includes(`editar:${recurso}`),
    percentualDesconto: () => (usuario.plano === 'pro' ? 0.2 : 0),
  };
}

export function criarPerfilPresenter(usuarios: Usuarios) {
  // unico ponto do sistema que converte ausencia em objeto nulo
  function perfil(usuarioId: string | null): Perfil {
    const usuario = usuarioId === null ? null : usuarios.buscar(usuarioId);
    return usuario === null ? PERFIL_ANONIMO : perfilDe(usuario);
  }

  // ?? cai no default so' para null/undefined; || cairia tambem para 0 e ''
  const limiteDeItens = (config: { maximo?: number }): number => config.maximo ?? 10;

  return { perfil, limiteDeItens };
}
```

### Passos
1. Confirme que a ausência tem comportamento padrão bem definido e igual em todos os chamadores; se cada chamador reage diferente, pare — mantenha o tipo opcional.
2. Antes de criar objeto nulo, tente o caminho barato do TS: `strictNullChecks` + `?.` + `??` resolvendo o default em UM lugar (a factory, o mapper do DTO, o hook).
3. Se o comportamento padrão é rico e repetido, extraia o `type` com só o que os chamadores usam (não a entidade inteira) e crie duas implementações: a real e a nula.
4. Declare o objeto nulo como `const` do módulo, congelado e sem estado (`PERFIL_ANONIMO`). Evite classe: um literal `satisfies Perfil` já basta.
5. Concentre a conversão `null -> objeto nulo` em um único ponto e substitua as verificações dos chamadores por chamadas polimórficas.
6. Nunca adicione `ehNulo()` ao contrato — isso reintroduz o `if` que você quis remover.
7. Se a ausência precisa aparecer na UI, use uma variante `{ status: 'vazio' }` na união de estado: o `switch` exaustivo força tratamento explícito, em vez de renderizar dados falsos.
8. Remova os defaults duplicados e ajuste os testes para injetar `PERFIL_ANONIMO` em vez de `null`.

### Ganhos
- Um único lugar define o significado de "sem usuário"; chamadores ficam livres de checagens.
- Elimina divergência de defaults (uma tela mostrava "Visitante", outra string vazia).
- Objeto nulo é imutável e sem estado: barato, seguro para concorrência, trivial em teste e Storybook.

### Quando NÃO aplicar
- Ausência é um erro que precisa ser tratado: o objeto nulo o transforma em sucesso silencioso (cobra R$ 0,00, salva pedido de ninguém, aplica cupom fantasma). É o risco central desta técnica.
- Os chamadores precisam distinguir "não existe" de "existe e está vazio" — o objeto nulo apaga essa diferença.
- Só há um ou dois checks: `??` resolve com menos código e mantendo a ausência visível no tipo.
- A UI deve renderizar algo diferente para ausência: esconder isso atrás de defaults produz tela "normal" com dados falsos.
- A fronteira é de erro (`Result`, resposta HTTP): ali a ausência é informação; escondê-la produz bug difícil de rastrear.

### Nota TypeScript
Com `strictNullChecks`, ausência já é parte do tipo (`Usuario | null`), então o caminho padrão é `?.` + `??` — o compilador não deixa esquecer o caso. Use objeto nulo (`PERFIL_ANONIMO`) quando o comportamento padrão é rico e repetido em muitos chamadores; use uma variante `'vazio'` na união discriminada quando a UI deve reagir à ausência (o `switch` exaustivo impede esquecer). Armadilhas: `||` cai no default para todo valor falsy — `0`, `''`, `NaN`, `false` —, por isso `quantidade || 1` transforma zero em um; use `??`, que só reage a `null`/`undefined`. `?.` retorna `undefined` e propaga silenciosamente: `usuario?.assinatura?.plano` esconde em qual nível a cadeia quebrou. `!` e `as` não checam nada em runtime. `Object.freeze` (ou só `readonly` + literal `satisfies`) mantém o objeto nulo realmente imutável — em um singleton de módulo, uma mutação acidental contamina todo o processo (inclusive entre requisições em SSR).

---

## 8. Introduce Assertion
**PT-BR:** Introduzir Asserção · **Fonte:** https://refactoring.guru/introduce-assertion

### Problema
Para o código funcionar, certas condições precisam ser verdadeiras, mas essa suposição vive apenas em comentário, em nome de variável ou na cabeça de quem escreveu.

### Solução
Tornar a suposição explícita e sempre verificada — em TypeScript, com **assertion functions** (`asserts x is T`) e type guards, que além de validar em runtime estreitam o tipo em tempo de compilação.

### Sinais no código (gatilhos)
- Comentário do tipo `// só chamar depois de abrir o pedido` ou `// percentual sempre entre 0 e 1`.
- `!` (non-null assertion) ou `as` justificado por suposição não verificada.
- Função que aceita `number` mas só funciona para valores positivos, ou `string` que precisa ser um CEP válido.
- Ordem de chamada obrigatória entre funções sem verificação.
- Divisão pelo tamanho de uma coleção que pode ser zero.
- `JSON.parse(...) as Dto` na fronteira, sem validação.

### Antes
```ts
type StatusPedido = 'rascunho' | 'aberto' | 'pago' | 'cancelado';
type ItemPedido = { readonly preco: number; readonly quantidade: number };
type Pedido = {
  readonly id: string;
  readonly status: StatusPedido;
  readonly itens: ReadonlyArray<ItemPedido>;
  readonly cupomPercentual: number;
};
type PedidoAberto = Pedido & { readonly status: 'aberto' };

// atencao: chamar apenas com pedido ja aberto
// cupomPercentual precisa estar entre 0 e 1
export function totalComDesconto(pedido: Pedido): number {
  const bruto = pedido.itens.reduce((s, i) => s + i.preco * i.quantidade, 0);
  return bruto * (1 - pedido.cupomPercentual);
}

// atencao: pedido precisa ter itens
export function ticketMedio(pedido: Pedido): number {
  return totalComDesconto(pedido) / pedido.itens.length;
}

export function cobrar(pedido: Pedido, gateway: Gateway): Promise<void> {
  const aberto = pedido as PedidoAberto; // "aqui sempre esta aberto"
  return gateway.cobrar(aberto.id, totalComDesconto(aberto));
}
```

### Depois
```ts
export function assertPedidoAberto(pedido: Pedido): asserts pedido is PedidoAberto {
  if (pedido.status !== 'aberto') {
    throw new Error(`Invariante violada: pedido ${pedido.id} esta ${pedido.status}`);
  }
}

function assertTemItens(pedido: Pedido): void {
  if (pedido.itens.length === 0) {
    throw new Error(`Invariante violada: pedido ${pedido.id} sem itens`);
  }
}

export function totalComDesconto(pedido: PedidoAberto): number {
  const bruto = pedido.itens.reduce((s, i) => s + i.preco * i.quantidade, 0);
  return bruto * (1 - pedido.cupomPercentual);
}

export function ticketMedio(pedido: Pedido): number {
  assertPedidoAberto(pedido);
  assertTemItens(pedido);
  // pedido agora e' PedidoAberto: totalComDesconto aceita sem cast
  return totalComDesconto(pedido) / pedido.itens.length;
}

// entrada externa NAO e' assercao: valide na fronteira e devolva erro ao cliente
const cupomSchema = z.object({ percentual: z.number().min(0).max(1) });

export function lerCupom(json: unknown): { ok: true; percentual: number } | { ok: false; erro: string } {
  const parsed = cupomSchema.safeParse(json);
  if (!parsed.success) return { ok: false, erro: 'cupom-invalido' };
  return { ok: true, percentual: parsed.data.percentual };
}
```

### Passos
1. Procure comentários de pré-condição, `!` e `as`; cada um é uma suposição candidata.
2. Classifique a suposição: **entrada externa** (body HTTP, query string, `localStorage`, resposta de API, input do usuário) → validação na fronteira com Zod, devolvendo erro; **invariante interna** (bug do programador, contrato entre módulos seus) → assertion function que lança.
3. Escreva a asserção como função declarada com predicado: `function assertPedidoAberto(p: Pedido): asserts p is PedidoAberto`. Só funciona em `function` declarada (ou variável com tipo explícito), nunca inferida de arrow anônima.
4. Chame a asserção na primeira linha, antes de qualquer efeito colateral, e inclua o valor real na mensagem.
5. Apague o comentário — a asserção passou a ser documentação executável.
6. Onde possível, prefira **tornar o estado impossível de representar**: se a função só aceita `PedidoAberto`, o parâmetro tipado já elimina a asserção. Asserção é a rede quando o tipo não alcança (dados vindos de fora do módulo, narrowing perdido).
7. Nunca coloque efeito colateral dentro da asserção; ela só observa.
8. Cubra cada asserção com um teste (`expect(() => ticketMedio(pago)).toThrow()`), e remova asserções que o sistema de tipos já garante.

### Ganhos
- Falha rápido e perto da causa, antes de corromper carrinho, cobrança ou histórico do pedido.
- Documentação viva que não desatualiza, complementando os tipos.
- A mensagem aponta o valor real, encurtando o diagnóstico no log/Sentry.
- Em TS a asserção **estreita o tipo**: depois de `assertPedidoAberto(p)`, `p` é `PedidoAberto` no resto do escopo — validação e tipagem no mesmo passo, algo que `require`/`check` de outras linguagens não oferecem.

### Quando NÃO aplicar
- A condição depende de entrada do usuário, resposta de API ou conectividade — isso é erro esperado: `Result`, estado de erro, resposta 4xx; não crash.
- O tipo já garante a invariante (parâmetro não-nulo, união fechada, branded type validado na criação).
- Você usaria a asserção para controlar fluxo (capturar o `Error` como parte da lógica) — exceção não é `if`.
- Em caminho quente (render, scroll, loop de milhões de itens) onde a checagem custa e a invariante já é garantida a montante.
- A mensagem é caríssima de montar e você a construiria antes de saber se falhou — monte dentro do `if`.

### Nota TypeScript
Assertion function (`asserts x is T`) e type guard (`x is T`) são as duas ferramentas: o guard devolve `boolean` e serve para ramificar; a asserção lança e serve para exigir. Ambas exigem tipo de retorno anotado explicitamente — TS não infere predicado, e uma arrow atribuída sem tipo explícito perde o `asserts`. O oposto disso é `as`: **type assertion não valida nada em runtime**, é só uma promessa ao compilador; junto com `!` e `any`, é a origem clássica do `undefined is not a function` em produção. `console.assert` **não** interrompe a execução — nunca use como asserção; `node:assert` funciona, mas prefira lançar seu próprio erro com mensagem de domínio, pois `assert` acoplado a Node não roda no browser. Na fronteira, quem valida é o schema (`schema.safeParse`), que ao mesmo tempo tipa o DTO e devolve o erro para o cliente. Para asserção de exaustividade, `assertNever(x: never)` no `default` do `switch`. E ative `strictNullChecks` + `noUncheckedIndexedAccess`: boa parte das asserções que você escreveria vira erro de compilação de graça.
