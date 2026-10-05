# Simplifying Method Calls — Simplificando Chamadas de Métodos

> Grupo 5 de 6 · 14 técnicas · Fonte: refactoring.guru/refactoring/techniques

## Quando este grupo se aplica
- Assinaturas difíceis de ler: listas longas de parâmetros posicionais, `boolean` soltos no call site, parâmetros nunca lidos ou passados "por precaução".
- Nomes que não descrevem o efeito (`getData`, `process`, `handle`) ou que mentem sobre I/O e mutação.
- Grupos de parâmetros que sempre viajam juntos, ou valores desestruturados de um objeto imediatamente antes da chamada.
- Funções que retornam um valor **e** mutam estado (violação de CQS), especialmente handlers de React e métodos de service que gravam e devolvem leitura.
- Erro sinalizado por código numérico, `-1`, `null` ou `throw` usado para fluxo esperado.
- Construtores que fazem mais que atribuir campos, e campos mutáveis públicos em modelos que deveriam ser `readonly`.
- Módulo exportando helpers que só interessam a ele mesmo (barrel file vazando detalhe interno).

## Índice
1. [Add Parameter](#1-add-parameter)
2. [Remove Parameter](#2-remove-parameter)
3. [Rename Method](#3-rename-method)
4. [Separate Query from Modifier](#4-separate-query-from-modifier)
5. [Parameterize Method](#5-parameterize-method)
6. [Introduce Parameter Object](#6-introduce-parameter-object)
7. [Preserve Whole Object](#7-preserve-whole-object)
8. [Remove Setting Method](#8-remove-setting-method)
9. [Replace Parameter with Explicit Methods](#9-replace-parameter-with-explicit-methods)
10. [Replace Parameter with Method Call](#10-replace-parameter-with-method-call)
11. [Hide Method](#11-hide-method)
12. [Replace Constructor with Factory Method](#12-replace-constructor-with-factory-method)
13. [Replace Error Code with Exception](#13-replace-error-code-with-exception)
14. [Replace Exception with Test](#14-replace-exception-with-test)

---

## 1. Add Parameter
**PT-BR:** Adicionar Parâmetro · **Fonte:** https://refactoring.guru/add-parameter

### Problema
A função não tem dados suficientes para executar a nova regra exigida.

### Solução
Adicionar um parâmetro que carrega o dado necessário, em vez de criar estado mutável só para transportá-lo.

### Sinais no código (gatilhos)
- A função lê um `let` de módulo (ou um campo mutável de classe) que existe apenas para "passar" um valor entre chamadas.
- A regra nova depende de contexto do chamador (frete, cupom, política de campanha) que a função não recebe.
- Você está tentado a criar uma variável de módulo, um `Context` global ou um campo de configuração para um dado efêmero.
- O chamador já tem o dado e o descarta antes de chamar (ou o "instala" com um setter antes).

### Antes
```ts
type ItemPedido = {
  readonly produtoId: string;
  readonly preco: number;
  readonly quantidade: number;
};

// o frete mora num let de módulo só para "chegar" até o cálculo
let freteCorrente = 0;

export function definirFrete(valor: number): void {
  freteCorrente = valor;
}

export function calcularTotal(itens: ReadonlyArray<ItemPedido>): number {
  const subtotal = itens.reduce((acc, item) => acc + item.preco * item.quantidade, 0);
  return subtotal + freteCorrente;
}

declare const itensDoCarrinho: ReadonlyArray<ItemPedido>;
// o chamador tem o frete, mas precisa instalá-lo antes de calcular
definirFrete(19.9);
const total = calcularTotal(itensDoCarrinho);
```

### Depois
```ts
type ItemPedido = {
  readonly produtoId: string;
  readonly preco: number;
  readonly quantidade: number;
};

type OpcoesTotal = {
  readonly frete?: number;
  readonly cupomPercentual?: number;
};

export function calcularTotal(
  itens: ReadonlyArray<ItemPedido>,
  { frete = 0, cupomPercentual = 0 }: OpcoesTotal = {},
): number {
  const subtotal = itens.reduce((acc, item) => acc + item.preco * item.quantidade, 0);
  const desconto = subtotal * cupomPercentual;
  return subtotal - desconto + frete;
}

declare const itensDoCarrinho: ReadonlyArray<ItemPedido>;
// nomes no call site: legível e independente de ordem
const total = calcularTotal(itensDoCarrinho, { frete: 19.9, cupomPercentual: 0.1 });
const totalSemFrete = calcularTotal(itensDoCarrinho);
```

### Passos
1. Verifique se a função implementa uma `interface` ou é passada como callback tipado; se sim, o parâmetro precisa entrar no tipo do contrato também.
2. Se o novo dado é opcional, coloque-o no **objeto de opções** com default representando o comportamento atual (`frete = 0`). Todos os call sites continuam compilando.
3. Se o dado é obrigatório, adicione-o como parâmetro posicional e deixe o compilador listar cada chamada a corrigir (`tsc --noEmit`).
4. Nunca invente default "mentiroso" para calar o compilador: se não há comportamento neutro, o parâmetro é obrigatório.
5. Se a função é API pública de um pacote, mantenha a assinatura antiga como overload marcado com `@deprecated` no JSDoc por um ciclo de release.

### Ganhos
- Dado efêmero fica no parâmetro em vez de virar estado mutável compartilhado.
- Com objeto de opções + defaults, a evolução é retrocompatível e o call site fica autoexplicativo.
- A dependência da regra fica explícita na assinatura, facilitando teste unitário.

### Quando NÃO aplicar
- A lista já tem 4+ parâmetros: prefira `Introduce Parameter Object` (§6) ou `Preserve Whole Object` (§7).
- O dado pode ser obtido dentro da função por uma dependência que ela já tem: use `Replace Parameter with Method Call` (§10).
- O parâmetro é um `boolean` que liga/desliga trechos distintos: use `Replace Parameter with Explicit Methods` (§9).
- O dado é dependência de longo prazo (repositório, `clock`, `fetch`): injete por construtor ou por factory de closure.

### Nota TypeScript
TS não tem argumento nomeado. O idioma equivalente é o **objeto de opções desestruturado com defaults** (`function calcularTotal(itens, { frete = 0, cupom }: Opcoes = {})`): dá o efeito de named args, permite adicionar campos sem quebrar chamadas e elimina a dependência de ordem. Parâmetro posicional `boolean` é smell — `criar(pedido, true, false)` não se lê; promova-o a campo do objeto de opções ou a união de literais (`'imediato' | 'agendado'`). `Add Parameter` recorrente é sinal de **parameter object faltando**: três adições no mesmo mês significam que existe um `type` de critérios esperando para nascer (§6). Use `= {}` no objeto de opções para manter a chamada de um argumento válida, e prefira `readonly` nos campos para que o objeto não seja mutado dentro da função.

---

## 2. Remove Parameter
**PT-BR:** Remover Parâmetro · **Fonte:** https://refactoring.guru/remove-parameter

### Problema
Um parâmetro não é usado no corpo da função.

### Solução
Remover o parâmetro, junto com o código do chamador que o produzia.

### Sinais no código (gatilhos)
- `noUnusedParameters` / regra `@typescript-eslint/no-unused-vars` acusando o parâmetro (ou um `_` prefixado para silenciá-lo).
- Parâmetro adicionado "para o futuro" e nunca lido.
- Parâmetro só repassado adiante para outra função que também o ignora.
- Todos os call sites passam o mesmo literal (`false`, `null`, `''`).

### Antes
```ts
type PedidoDto = { readonly id: string; readonly status: string; readonly total: number };
type Pedido = { readonly id: string; readonly status: 'rascunho' | 'pago'; readonly total: number };

declare function toPedido(dto: PedidoDto): Pedido;
declare const api: { pedidos(usuarioId: string): Promise<ReadonlyArray<PedidoDto>> };

export interface PedidoRepository {
  listar(
    usuarioId: string,
    incluirRascunhos: boolean,
    formatoLegado: boolean,
  ): Promise<ReadonlyArray<Pedido>>;
}

export class PedidoRepositoryHttp implements PedidoRepository {
  async listar(
    usuarioId: string,
    incluirRascunhos: boolean,
    formatoLegado: boolean,
  ): Promise<ReadonlyArray<Pedido>> {
    const dtos = await api.pedidos(usuarioId);
    return dtos.filter((d) => incluirRascunhos || d.status !== 'rascunho').map(toPedido);
  }
}

declare const repo: PedidoRepository;
// nenhum chamador passa algo diferente de false no último argumento
const formatoLegado = false;
const pagos = repo.listar('usuario-1', false, formatoLegado);
```

### Depois
```ts
type PedidoDto = { readonly id: string; readonly status: string; readonly total: number };
type Pedido = { readonly id: string; readonly status: 'rascunho' | 'pago'; readonly total: number };

declare function toPedido(dto: PedidoDto): Pedido;
declare const api: { pedidos(usuarioId: string): Promise<ReadonlyArray<PedidoDto>> };

type OpcoesListagem = { readonly incluirRascunhos?: boolean };

export interface PedidoRepository {
  listar(usuarioId: string, opcoes?: OpcoesListagem): Promise<ReadonlyArray<Pedido>>;
}

export class PedidoRepositoryHttp implements PedidoRepository {
  async listar(
    usuarioId: string,
    { incluirRascunhos = false }: OpcoesListagem = {},
  ): Promise<ReadonlyArray<Pedido>> {
    const dtos = await api.pedidos(usuarioId);
    return dtos.filter((d) => incluirRascunhos || d.status !== 'rascunho').map(toPedido);
  }
}

declare const repo: PedidoRepository;
const pagos = repo.listar('usuario-1');
```

### Passos
1. Confirme que nenhuma implementação da `interface` usa o parâmetro — busque por todas as classes/objetos que a implementam.
2. Remova o parâmetro da declaração da `interface` e das implementações; rode `tsc --noEmit`: **o compilador aponta todos os usos**, nada fica silenciosamente errado.
3. Apague nos chamadores o código que existia apenas para calcular o argumento (variáveis, buscas, flags de config).
4. Se a função é API pública de um pacote, mantenha por um release um overload com a assinatura antiga marcado `@deprecated`, delegando para a nova.
5. Atualize fakes/mocks de teste — costumam ser os últimos a ainda declarar o parâmetro. Cuidado com callbacks passados a APIs de terceiros: remover parâmetro de uma função *tipada estruturalmente* é seguro, mas quem chama via `apply`/`arguments` não é verificado.

### Ganhos
- Menos carga cognitiva: quem lê a assinatura não investiga um dado inerte.
- Elimina código morto no chamador que produzia o argumento.
- Fakes de teste ficam menores e menos frágeis.

### Quando NÃO aplicar
- Alguma implementação da `interface` usa o parâmetro.
- A assinatura é imposta por contrato externo (handler de framework, callback de SDK, middleware `(req, res, next)`).
- É um parâmetro posicional intermediário exigido por um callback de terceiro (`(item, indice, lista)`): prefixe com `_` em vez de remover, para não deslocar os demais.

### Nota TypeScript
Remover parâmetro em TS é uma das refatorações mais seguras que existem: com `strict` ligado, `tsc --noEmit` lista cada chamada incompatível antes do runtime. A exceção é a **compatibilidade estrutural de funções**: TS aceita passar uma função com *menos* parâmetros onde se espera uma com mais, então tirar um parâmetro de um callback não gera erro nos pontos que já o ignoravam — bom para migração, ruim se você contava com o compilador para encontrá-los. Se o parâmetro removido era `boolean` posicional, aproveite para mover o que sobrou para um objeto de opções (§1); e ative `noUnusedParameters` no `tsconfig` para que o próximo parâmetro morto apareça no mesmo dia em que nasce.

---

## 3. Rename Method
**PT-BR:** Renomear Método · **Fonte:** https://refactoring.guru/rename-method

### Problema
O nome da função não explica o que ela faz.

### Solução
Renomear para um nome que descreva a intenção e o efeito.

### Sinais no código (gatilhos)
- Nomes genéricos: `getData`, `process`, `handle`, `doWork`, `run` sem contexto.
- Comentário acima da função explicando o que o nome deveria dizer.
- O nome diz `get`, mas a função faz `fetch`, escreve cache ou muta estado.
- O escopo cresceu: `salvarPedido` agora também publica evento e dispara notificação.

### Antes
```ts
type Pedido = { readonly id: string; readonly total: number; readonly atualizadoEm: number };

const TTL_MS = 15 * 60 * 1000;

declare const api: { pedidos(usuarioId: string): Promise<ReadonlyArray<Pedido>> };
declare const cache: {
  listar(usuarioId: string): Promise<ReadonlyArray<Pedido>>;
  substituir(usuarioId: string, pedidos: ReadonlyArray<Pedido>): Promise<void>;
};

export class PedidoService {
  // Lê do cache local; se estiver vencido, busca na API e regrava o cache
  async getData(usuarioId: string): Promise<ReadonlyArray<Pedido>> {
    const local = await cache.listar(usuarioId);
    const vencido =
      local.length === 0 || local.some((p) => p.atualizadoEm < Date.now() - TTL_MS);
    if (!vencido) return local;
    const remoto = await api.pedidos(usuarioId);
    await cache.substituir(usuarioId, remoto);
    return remoto;
  }
}
```

### Depois
```ts
type Pedido = { readonly id: string; readonly total: number; readonly atualizadoEm: number };

const TTL_MS = 15 * 60 * 1000;

declare const api: { pedidos(usuarioId: string): Promise<ReadonlyArray<Pedido>> };
declare const cache: {
  listar(usuarioId: string): Promise<ReadonlyArray<Pedido>>;
  substituir(usuarioId: string, pedidos: ReadonlyArray<Pedido>): Promise<void>;
};

export class PedidoService {
  async sincronizarPedidosDoUsuario(usuarioId: string): Promise<ReadonlyArray<Pedido>> {
    const local = await cache.listar(usuarioId);
    const vencido =
      local.length === 0 || local.some((p) => p.atualizadoEm < Date.now() - TTL_MS);
    if (!vencido) return local;
    const remoto = await api.pedidos(usuarioId);
    await cache.substituir(usuarioId, remoto);
    return remoto;
  }

  /**
   * @deprecated O nome não dizia que há I/O e escrita de cache.
   * Use `sincronizarPedidosDoUsuario`. Será removido na v3.
   */
  getData(usuarioId: string): Promise<ReadonlyArray<Pedido>> {
    return this.sincronizarPedidosDoUsuario(usuarioId);
  }
}
```

### Passos
1. Verifique se o nome faz parte de uma `interface` ou de um tipo de contrato; renomeie na declaração para o erro propagar a todas as implementações.
2. Use o **rename do TS Server** (F2 no editor) em vez de find/replace: ele cobre implementações, referências como valor (`obj.metodo` passado adiante), JSDoc e testes, e não toca em strings homônimas.
3. Para código interno da aplicação, renomeie e apague o nome antigo no mesmo commit — o compilador garante que nada ficou para trás.
4. Para **API pública / pacote publicado**, mantenha o nome antigo como alias delegando ao novo, marcado com `@deprecated` no JSDoc (o editor risca os usos); só remova no próximo major.
5. Cuidado com nomes usados por dados, não por código: chaves de payload JSON, colunas de banco, campos de schema Zod, nomes de eventos de analytics. Renomeie o símbolo TS e preserve o nome do wire no mapeamento.
6. Remova comentários que existiam apenas para compensar o nome ruim.

### Ganhos
- Elimina o smell de comentário explicativo e de nomes divergentes entre módulos equivalentes.
- Code review fica mais rápido: o nome comunica o efeito (`sincronizar` sinaliza I/O + escrita).
- O alias `@deprecated` dá migração gradual sem quebrar consumidores.

### Quando NÃO aplicar
- O nome é ditado por contrato externo (`toString`, `then`, `handler`, `default` de rota, hooks de ciclo de vida de framework).
- O rename só troca sinônimos sem ganho de clareza (`obter` → `buscar`) — ruído de diff.
- A função deveria ser dividida: se o nome honesto precisa de "E" (`salvarEEnviar`), o problema é a função, não o nome (veja §4).

### Nota TypeScript
O rename do TS Server é seguro porque é baseado no grafo de tipos, não em texto: renomeia declaração, implementações e referências indiretas de uma vez. O que ele **não** vê é o mundo dinâmico — `obj['getData']`, chaves construídas em runtime, nomes em arquivos de config, snapshots de teste. Depois do rename, faça um `grep` pela string antiga para pegar esses casos. Em pacote publicado, renomear é *breaking change*: exporte o alias antigo com `@deprecated` no JSDoc (e, se quiser barulho no build, um `@deprecated` + regra de lint que falhe em CI) antes de remover no major. Reexportações em **barrel file** (`index.ts`) precisam ser atualizadas junto, senão o nome antigo continua público mesmo tendo desaparecido do módulo de origem.

---

## 4. Separate Query from Modifier
**PT-BR:** Separar Consulta de Modificador · **Fonte:** https://refactoring.guru/separate-query-from-modifier

### Problema
Uma função retorna um valor e, ao mesmo tempo, altera o estado observável do objeto ou do sistema.

### Solução
Dividir em duas funções: uma consulta pura que retorna o valor e um modificador que aplica o efeito.

### Sinais no código (gatilhos)
- Nome com "E"/"and", ou verbo de comando retornando dado: `consumirEObter`, `salvarERetornarId`.
- Chamar a função duas vezes dá resultado diferente sem que isso seja óbvio.
- A função é usada dentro de `if`/`while`, escondendo mutação em condicional.
- Custom hook cujo retorno é consultado durante o render, mas que dispara `setState`/POST no caminho.

### Antes
```tsx
declare const api: {
  usoDeCupom(carrinhoId: string): Promise<{ readonly limite: number; readonly usados: number }>;
  registrarUso(carrinhoId: string, usados: number): Promise<void>;
  aplicarCupom(carrinhoId: string): Promise<void>;
};

function useCupom(carrinhoId: string) {
  const [restantes, setRestantes] = useState(0);

  // consulta E muta: devolve o saldo já debitado
  const consumirEObterRestantes = useCallback(async (): Promise<number> => {
    const uso = await api.usoDeCupom(carrinhoId);
    await api.registrarUso(carrinhoId, uso.usados + 1);
    const sobrando = Math.max(uso.limite - (uso.usados + 1), 0);
    setRestantes(sobrando);
    return sobrando;
  }, [carrinhoId]);

  return { restantes, consumirEObterRestantes };
}

export function BotaoAplicarCupom({ carrinhoId }: { carrinhoId: string }) {
  const { restantes, consumirEObterRestantes } = useCupom(carrinhoId);
  const onClick = async () => {
    // o teste "tem saldo?" já gastou o cupom
    if ((await consumirEObterRestantes()) >= 0) await api.aplicarCupom(carrinhoId);
  };
  return <button onClick={onClick}>Aplicar cupom ({restantes})</button>;
}
```

### Depois
```tsx
declare const api: {
  usoDeCupom(carrinhoId: string): Promise<{ readonly limite: number; readonly usados: number }>;
  registrarUso(carrinhoId: string, usados: number): Promise<void>;
  aplicarCupom(carrinhoId: string): Promise<void>;
};

function useCupom(carrinhoId: string) {
  const [restantes, setRestantes] = useState(0);

  const consultarRestantes = useCallback(async (): Promise<number> => {
    const uso = await api.usoDeCupom(carrinhoId);
    const saldo = Math.max(uso.limite - uso.usados, 0);
    setRestantes(saldo);
    return saldo;
  }, [carrinhoId]);

  const consumirCupom = useCallback(async (): Promise<void> => {
    const uso = await api.usoDeCupom(carrinhoId);
    await api.registrarUso(carrinhoId, uso.usados + 1);
    setRestantes(Math.max(uso.limite - (uso.usados + 1), 0));
  }, [carrinhoId]);

  return { restantes, consultarRestantes, consumirCupom };
}

export function BotaoAplicarCupom({ carrinhoId }: { carrinhoId: string }) {
  const { restantes, consultarRestantes, consumirCupom } = useCupom(carrinhoId);
  const onClick = async () => {
    if ((await consultarRestantes()) > 0) {
      await consumirCupom();
      await api.aplicarCupom(carrinhoId);
    }
  };
  return <button onClick={onClick}>Aplicar cupom ({restantes})</button>;
}
```

### Passos
1. Crie a função de consulta que devolve exatamente o mesmo valor da original, sem nenhum efeito (sem `setState`, sem escrita, sem POST).
2. Faça a função original delegar o `return` à nova consulta, mantendo temporariamente os efeitos.
3. Em cada call site, troque a chamada única por: consultar, decidir, e então executar o comando — nunca deixe mutação dentro da condição de um `if`/`while`.
4. Remova o `return` da função original, deixando-a `Promise<void>`, e renomeie-a com verbo de comando.
5. Se a operação precisa ser atômica (transação, seção crítica), não separe: exponha um comando único que devolve um `Result` com o resultado do domínio.

### Ganhos
- Consultas ficam idempotentes: podem ser chamadas em log, em teste e durante render sem efeito colateral.
- Testes verificam leitura e escrita separadamente.
- Elimina o bug clássico de React: consulta chamada no corpo do componente disparando mutação a cada render.

### Quando NÃO aplicar
- A operação precisa ser atômica e o valor só existe dentro da transação (linhas afetadas por um `DELETE`, id gerado pelo `INSERT`).
- Concorrência: separar cria janela de corrida (verifica-e-consome). Mantenha um comando único devolvendo união de resultado.
- A "mutação" é só memoização em campo privado, sem estado observável de fora.

### Nota TypeScript
Isto é CQS (Command-Query Separation) com dois casos concretos. **React:** consulta é o que você pode chamar no render ou dentro de `useMemo` — se um hook/handler consulta e muta ao mesmo tempo, o Strict Mode (que invoca render e efeitos duas vezes em dev) duplica a mutação e o bug aparece só em produção; separe em `consultarX` (pura, ou query do TanStack Query) e `xMutation`/handler que só é chamado por evento. **Node:** o service que grava e devolve leitura derivada esconde a mesma armadilha — tipar o comando como `Promise<void>` e a consulta como `Promise<T>` faz o compilador reclamar quando alguém tenta usar o retorno de um comando. Quando atomicidade é obrigatória, prefira um comando que devolve `Result<T, E>` (§13) em vez de forçar a separação.

---

## 5. Parameterize Method
**PT-BR:** Parametrizar Método · **Fonte:** https://refactoring.guru/parameterize-method

### Problema
Várias funções fazem a mesma coisa, diferindo apenas por valores internos.

### Solução
Unificá-las em uma única função que recebe o valor variável como parâmetro.

### Sinais no código (gatilhos)
- Funções com nomes paralelos: `precoMensal`, `precoAnual`, `precoBienal`.
- Corpos idênticos exceto por uma constante numérica ou uma string.
- Cada nova variante do negócio exige copiar/colar uma função.
- A diferença é *dado*, não *comportamento*.

### Antes
```ts
const DESCONTO_CUPOM = 0.9;

export function precoMensal(precoBase: number, cupomAtivo: boolean): number {
  const comFidelidade = precoBase * (1 - 0);
  const comCupom = cupomAtivo ? comFidelidade * DESCONTO_CUPOM : comFidelidade;
  return comCupom * 1;
}

export function precoAnual(precoBase: number, cupomAtivo: boolean): number {
  const comFidelidade = precoBase * (1 - 0.15);
  const comCupom = cupomAtivo ? comFidelidade * DESCONTO_CUPOM : comFidelidade;
  return comCupom * 12;
}

export function precoBienal(precoBase: number, cupomAtivo: boolean): number {
  const comFidelidade = precoBase * (1 - 0.25);
  const comCupom = cupomAtivo ? comFidelidade * DESCONTO_CUPOM : comFidelidade;
  return comCupom * 24;
}
```

### Depois
```ts
const DESCONTO_CUPOM = 0.9;

export const PERIODICIDADES = {
  mensal: { meses: 1, descontoFidelidade: 0 },
  anual: { meses: 12, descontoFidelidade: 0.15 },
  bienal: { meses: 24, descontoFidelidade: 0.25 },
} as const;

export type Periodicidade = keyof typeof PERIODICIDADES;

type OpcoesPreco = { readonly cupomAtivo?: boolean };

export function precoAssinatura(
  precoBase: number,
  periodicidade: Periodicidade,
  { cupomAtivo = false }: OpcoesPreco = {},
): number {
  const { meses, descontoFidelidade } = PERIODICIDADES[periodicidade];
  const comFidelidade = precoBase * (1 - descontoFidelidade);
  const comCupom = cupomAtivo ? comFidelidade * DESCONTO_CUPOM : comFidelidade;
  return comCupom * meses;
}

const anual = precoAssinatura(49.9, 'anual');
const bienalComCupom = precoAssinatura(49.9, 'bienal', { cupomAtivo: true });
```

### Passos
1. Extraia para uma nova função a parte realmente idêntica dos corpos; se as funções só coincidem em parte, extraia apenas o trecho comum.
2. Substitua cada valor especial por um parâmetro. Se os valores formam um conjunto fechado do domínio, modele como **união de literais** + `Record`/objeto `as const` com os dados de cada variante — assim o compilador rejeita `'trienal'` antes do runtime.
3. Se a variação é de *tipo* e não de valor, generalize com **generics** e `extends` em vez de parâmetro de dado.
4. Atualize os chamadores; apague as funções antigas (ou mantenha alias `@deprecated`, se públicas).
5. Verifique se sobrou um `switch` grande dentro da nova função — se sim, o caso é `Replace Parameter with Explicit Methods` (§9), não parametrização.

### Ganhos
- Uma regra, um lugar: nova periodicidade é uma linha na tabela, não uma função nova.
- Elimina duplicação e o risco de corrigir o bug em só uma das cópias.
- Os testes cobrem todas as variantes com uma tabela de casos (`it.each`).

### Quando NÃO aplicar
- As funções diferem em *lógica*, não em valores: parametrizar cria um `switch` monstruoso.
- O parâmetro seria um `boolean` ligando/desligando blocos distintos (§9).
- Superparametrização: uma função com 5 parâmetros de configuração é mais difícil de ler que 3 funções nomeadas.

### Nota TypeScript
A dupla que substitui o parâmetro primitivo é **união de literais + objeto `as const`**: `keyof typeof PERIODICIDADES` deriva o tipo da tabela, então acrescentar uma variante atualiza o tipo automaticamente e um `switch` sobre ela fica exaustivo (com `default: assertNever(x)`). Prefira isso a `enum` — `enum` gera código em runtime, não é apagável por `isolatedModules`/`erasableSyntaxOnly` e não interopera bem com JSON. Quando a diferença entre as funções é o **tipo** manipulado (`buscarPedido`/`buscarCupom`), o parâmetro certo é um type parameter com constraint (`function buscar<T extends Recurso>(rota: Rota<T>): Promise<T>`), não um dado. Com `noUncheckedIndexedAccess`, indexar um `Record<string, X>` devolve `X | undefined`; indexar um objeto `as const` por uma união de literais não — mais um motivo para a tabela literal.

---

## 6. Introduce Parameter Object
**PT-BR:** Introduzir Objeto de Parâmetro · **Fonte:** https://refactoring.guru/introduce-parameter-object

### Problema
Um mesmo grupo de parâmetros se repete em várias funções.

### Solução
Substituir o grupo por um objeto imutável que o represente.

### Sinais no código (gatilhos)
- 4 ou mais parâmetros, especialmente do mesmo tipo (`string, string, string`) — trocar a ordem compila.
- O mesmo trio/quarteto aparece no hook, no service e no client HTTP.
- O chamador monta os argumentos sempre a partir da mesma origem.
- Validações do grupo repetidas em cada função.

### Antes
```tsx
type Pedido = { readonly id: string; readonly total: number };

declare function buscarPedidos(texto: string, status: string, de: string, ate: string,
  pagina: number): Promise<ReadonlyArray<Pedido>>;

function useBuscaPedidos(
  texto: string,
  status: string,
  de: string,
  ate: string,
  pagina: number,
): { pedidos: ReadonlyArray<Pedido>; erro: string | null } {
  const [pedidos, setPedidos] = useState<ReadonlyArray<Pedido>>([]);
  const [erro, setErro] = useState<string | null>(null);
  useEffect(() => {
    let ativo = true;
    buscarPedidos(texto, status, de, ate, pagina)
      .then((r) => { if (ativo) setPedidos(r); })
      .catch((e: unknown) => { if (ativo) setErro(String(e)); });
    return () => { ativo = false; };
  }, [texto, status, de, ate, pagina]);
  return { pedidos, erro };
}

export function ListaPedidos(): JSX.Element {
  const [texto, setTexto] = useState('');
  // fácil inverter 'de' e 'ate': mesmos tipos, nenhum erro de compilação
  const { pedidos, erro } = useBuscaPedidos(texto, 'pago', '2026-12-31', '2026-01-01', 1);
  if (erro !== null) return <p>Falha ao buscar pedidos: {erro}</p>;
  return <ul>{pedidos.map((p) => <li key={p.id}>{p.total}</li>)}</ul>;
}
```

### Depois
```tsx
type Pedido = { readonly id: string; readonly total: number };
export type FiltroPedidos = {
  readonly texto: string;
  readonly status: 'todos' | 'rascunho' | 'pago';
  readonly periodo: { readonly de: string; readonly ate: string };
  readonly pagina: number;
};

export const periodoValido = (f: FiltroPedidos): boolean => Date.parse(f.periodo.de) <= Date.parse(f.periodo.ate);
declare function buscarPedidos(filtro: FiltroPedidos): Promise<ReadonlyArray<Pedido>>;

function useBuscaPedidos(filtro: FiltroPedidos): { pedidos: ReadonlyArray<Pedido>; erro: string | null } {
  const [pedidos, setPedidos] = useState<ReadonlyArray<Pedido>>([]);
  const [erro, setErro] = useState<string | null>(null);
  useEffect(() => {
    if (!periodoValido(filtro)) return;
    let ativo = true;
    buscarPedidos(filtro)
      .then((r) => { if (ativo) setPedidos(r); })
      .catch((e: unknown) => { if (ativo) setErro(String(e)); });
    return () => { ativo = false; };
  }, [filtro]);
  return { pedidos, erro };
}

export function ListaPedidos(): JSX.Element {
  const [texto, setTexto] = useState('');
  // objeto literal recriado a cada render dispararia o efeito sempre: estabilize
  const filtro = useMemo<FiltroPedidos>(() => ({
    texto, status: 'pago', periodo: { de: '2026-01-01', ate: '2026-12-31' }, pagina: 1,
  }), [texto]);
  const { pedidos, erro } = useBuscaPedidos(filtro);
  if (erro !== null) return <p>Falha ao buscar pedidos: {erro}</p>;
  return <ul>{pedidos.map((p) => <li key={p.id}>{p.total}</li>)}</ul>;
}
```

### Passos
1. Crie um `type` com campos `readonly` para o grupo; dê um nome do domínio (`FiltroPedidos`), não `Params`/`Args`.
2. Adicione o objeto como parâmetro extra nas funções alvo, mantendo os antigos por um passo (`Add Parameter`, §1).
3. Remova os parâmetros antigos um a um, trocando os usos internos por acessos ao objeto; rode `tsc` e os testes a cada passo.
4. Migre os call sites para construir o objeto uma vez e reutilizá-lo entre as chamadas.
5. Mova para funções do módulo do objeto as regras que operam só sobre esses dados (`periodoValido`) — senão o parameter object vira um saco de dados anêmico.
6. Em React, cada objeto que entra em array de dependências ou em prop precisa de referência estável: `useMemo` na construção, ou eleve-o a estado/`useReducer`.

### Ganhos
- Impossível inverter dois `string` sem erro: os campos têm nome.
- Regras do grupo (validação, derivações) ficam num lugar só.
- Assinaturas curtas e estáveis: campo novo entra no `type`, não em N funções.

### Quando NÃO aplicar
- Os parâmetros não formam um conceito coeso — agrupar por conveniência cria acoplamento artificial.
- São 2 parâmetros já claros.
- Já existe uma entidade que contém esses valores: use `Preserve Whole Object` (§7).
- Você não vai mover nenhum comportamento para lá e a função é chamada de um único ponto.

### Nota TypeScript
Use `type` com campos `readonly` (e `ReadonlyArray`) para o parameter object; "modificar" é spread (`{ ...filtro, pagina: 2 }`). O custo específico do React: **objeto literal quebra a igualdade referencial**. `{ a: 1 } !== { a: 1 }`, então um objeto criado no corpo do componente muda de identidade a cada render e (a) dispara `useEffect`/query keys sem necessidade, (b) invalida `React.memo` do componente filho, (c) faz `useMemo`/`useCallback` que dependem dele recalcular sempre. Soluções, em ordem de preferência: derivar o objeto com `useMemo` das partes primitivas; guardá-lo em `useState`/`useReducer` (a referência só muda quando você troca); ou, para queries, passar as primitivas na key e montar o objeto dentro da função de fetch. Mesmo cuidado com defaults inline em props (`opcoes = {}` cria um objeto novo por render — extraia para uma constante de módulo).

---

## 7. Preserve Whole Object
**PT-BR:** Preservar o Objeto Inteiro · **Fonte:** https://refactoring.guru/preserve-whole-object

### Problema
Você extrai vários valores de um objeto e os passa como parâmetros separados.

### Solução
Passar o objeto inteiro.

### Sinais no código (gatilhos)
- Sequência de acessos no call site imediatamente antes da chamada (`endereco.cep, endereco.uf, endereco.cidade`).
- Uma desestruturação feita só para reenviar os campos um a um.
- Todos os argumentos vêm da mesma instância.
- Adicionar um campo à regra exige mudar a assinatura e todos os chamadores.

### Antes
```ts
type Endereco = {
  readonly cep: string;
  readonly logradouro: string;
  readonly numero: string;
  readonly cidade: string;
  readonly uf: string;
  readonly pais: string;
};

export function enderecoEntregavel(
  cep: string,
  logradouro: string,
  numero: string,
  cidade: string,
  uf: string,
): boolean {
  const cepOk = cep.replace(/\D/g, '').length === 8;
  const logradouroOk = logradouro.trim().length > 3;
  const numeroOk = numero.trim() !== '';
  const cidadeOk = cidade.trim() !== '';
  const ufOk = uf.length === 2;
  return cepOk && logradouroOk && numeroOk && cidadeOk && ufOk;
}

declare const endereco: Endereco;
const podeEntregar = enderecoEntregavel(
  endereco.cep,
  endereco.logradouro,
  endereco.numero,
  endereco.cidade,
  endereco.uf,
);
```

### Depois
```ts
type Endereco = {
  readonly cep: string;
  readonly logradouro: string;
  readonly numero: string;
  readonly cidade: string;
  readonly uf: string;
  readonly pais: string;
};

export function cepValido(endereco: Endereco): boolean {
  return endereco.cep.replace(/\D/g, '').length === 8;
}

export function enderecoEntregavel(endereco: Endereco): boolean {
  return (
    cepValido(endereco) &&
    endereco.logradouro.trim().length > 3 &&
    endereco.numero.trim() !== '' &&
    endereco.cidade.trim() !== '' &&
    endereco.uf.length === 2
  );
}

declare const endereco: Endereco;
const podeEntregar = enderecoEntregavel(endereco);
```

### Passos
1. Adicione o objeto completo como novo parâmetro da função.
2. Troque, um a um, os usos dos parâmetros primitivos por acessos ao objeto; rode os testes a cada troca.
3. Remova os parâmetros antigos da assinatura e de todas as implementações; `tsc --noEmit` aponta os call sites.
4. Apague no call site o código de extração que antecedia a chamada.
5. Se a função só usa dados do próprio objeto, mova-a para o módulo do tipo (função standalone que recebe o objeto como 1º parâmetro).
6. Se o chamador tem um objeto **maior** que o necessário, tipe o parâmetro pelo mínimo (`Pick<Endereco, 'cep' | 'uf'>`): tipagem estrutural aceita o objeto inteiro sem acoplar a função a ele.

### Ganhos
- Mudança de regra fica contida na função; chamadores não mudam.
- Assinatura curta e autoexplicativa.
- Frequentemente revela que a lógica pertence ao próprio conceito (Feature Envy).

### Quando NÃO aplicar
- A função precisa de 1 ou 2 campos e passar tudo criaria dependência desnecessária (dificulta reuso com dados de outra origem).
- Cruzaria fronteira de camada: um componente de design system não deve receber a entidade de domínio, e sim props enxutas.
- O objeto é mutável e a função pode observar estado inconsistente.
- Torna o teste caro: montar a entidade inteira só para validar um número.

### Nota TypeScript
Com objetos `readonly`, passar o inteiro é barato (é uma referência) e seguro. Aproveite a **tipagem estrutural** para não acoplar: declare o parâmetro como o subconjunto que você realmente usa (`Pick<Endereco, 'cep' | 'uf'>` ou um `type EnderecoEntregavel = { readonly cep: string; readonly uf: string }`) e qualquer objeto compatível serve — inclusive fixtures mínimas de teste. Funções standalone que recebem o objeto como primeiro parâmetro ocupam o lugar das extension functions e mantêm tree-shaking. Na fronteira de UI, aplique o inverso: derive um view model e passe-o ao componente — e lembre que passar o objeto inteiro como prop reintroduz o problema de igualdade referencial do §6, então derive-o com `useMemo` ou memoize o filho com comparador por campo.

---

## 8. Remove Setting Method
**PT-BR:** Remover Método de Atribuição · **Fonte:** https://refactoring.guru/remove-setting-method

### Problema
Um campo só deveria ser definido na criação do objeto, mas existe setter permitindo alterá-lo depois.

### Solução
Remover o setter e definir o valor na criação.

### Sinais no código (gatilhos)
- Campos públicos mutáveis (ou `set x(...)`) em modelo de domínio.
- Sequência "construir e depois configurar": `const u = new Usuario(); u.email = ...; u.papel = ...`.
- Setter chamado exatamente uma vez, logo após a construção.
- Objeto de estado exposto e mutado em vários lugares, com `Object.assign(estado, patch)`.

### Antes
```ts
type UsuarioDto = {
  readonly id: string;
  readonly email: string;
  readonly papel: string;
  readonly planoId: string | null;
};

export class Usuario {
  public id = '';
  public email = '';
  public papel = 'leitor';
  public planoId: string | null = null;

  public setEmail(email: string): void {
    this.email = email;
  }

  public setPapel(papel: string): void {
    this.papel = papel;
  }
}

export function carregarUsuario(dto: UsuarioDto): Usuario {
  const usuario = new Usuario();
  usuario.id = dto.id;
  usuario.setEmail(dto.email);
  usuario.setPapel(dto.papel);
  usuario.planoId = dto.planoId;
  return usuario;
}
```

### Depois
```ts
type UsuarioDto = {
  readonly id: string;
  readonly email: string;
  readonly papel: string;
  readonly planoId: string | null;
};

export type Papel = 'leitor' | 'editor' | 'admin';

export type Usuario = {
  readonly id: string;
  readonly email: string;
  readonly papel: Papel;
  readonly planoId: string | null;
};

const PAPEIS: ReadonlyArray<Papel> = ['leitor', 'editor', 'admin'];

function paraPapel(bruto: string): Papel {
  return PAPEIS.find((p) => p === bruto) ?? 'leitor';
}

export function usuarioDeDto(dto: UsuarioDto): Usuario {
  return {
    id: dto.id,
    email: dto.email,
    papel: paraPapel(dto.papel),
    planoId: dto.planoId,
  };
}

// "mudar" é produzir uma nova instância, não escrever no campo
export function comPapel(usuario: Usuario, papel: Papel): Usuario {
  return { ...usuario, papel };
}
```

### Passos
1. Garanta que a criação (construtor ou factory) recebe todo campo hoje ajustado por setter; adicione o que faltar, com default quando fizer sentido.
2. Marque todos os campos como `readonly` — o compilador passa a apontar cada escrita.
3. Para cada setter chamado logo após a construção, mova o argumento para a factory/construtor e apague o setter.
4. Onde o valor realmente muda, substitua a escrita por uma função que devolve novo objeto via spread (`{ ...usuario, papel }`).
5. Se a classe precisa evoluir estado internamente, use campo privado `#estado` com getter público — leitura de fora, escrita só dentro.
6. Rode `tsc --noEmit`: os erros restantes são os pontos que realmente precisam de redesenho (estado compartilhado disfarçado de modelo).

### Ganhos
- Objeto sempre válido desde a criação — sem estado meio-inicializado.
- Previsibilidade: um valor imutável pode circular entre módulos, requests e closures sem risco de mutação remota.
- Em React, novo objeto por mudança é exatamente o que `useState`/`memo` precisam para detectar alteração por identidade.

### Quando NÃO aplicar
- Bibliotecas que exigem construtor vazio + atribuição (alguns ORMs, mappers antigos, formatos de serialização legados).
- O campo é genuinamente estado mutável de longa vida (posição de scroll, progresso de upload): encapsule com `#campo` + método, em vez de eliminar a escrita.
- A criação viraria gigantesca: aplique primeiro `Introduce Parameter Object` (§6).

### Nota TypeScript
A tríade em TS é: **`readonly` nos campos**, **construtor/factory** como único ponto de criação, e **spread** para "atualizar" devolvendo novo objeto. Lembre que `readonly` é só compile-time (nada impede mutação de JS puro em runtime; `Object.freeze` custa e raramente é necessário) e que é **superficial**: aninhados precisam de `readonly` próprio ou de `ReadonlyArray`/`Readonly<T>` — para estruturas profundas, um `DeepReadonly` ou um `type` com todos os níveis marcados. Para diferenciar leitura de escrita sem duplicar o tipo, mantenha o tipo `readonly` e derive o mutável interno com `-readonly` num mapped type. Em classe, `#campo` privado é privacidade real em runtime (diferente de `private`, que só existe nos tipos); um `get` sem `set` publica leitura sem publicar escrita.

---

## 9. Replace Parameter with Explicit Methods
**PT-BR:** Substituir Parâmetro por Métodos Explícitos · **Fonte:** https://refactoring.guru/replace-parameter-with-explicit-methods

### Problema
A função é dividida em blocos executados conforme o valor de um parâmetro.

### Solução
Extrair cada bloco para sua própria função com nome explícito e chamar diretamente.

### Sinais no código (gatilhos)
- `boolean` posicional no call site: `alterarAssinatura(id, 'CANCELAR', true, '', 'caro')` — ninguém lê isso.
- Parâmetro `string`/`number` de "tipo de operação" com `switch` no corpo, cada ramo com lógica não trivial.
- Chamadores sempre passam literal constante, nunca variável.
- Parâmetros que só fazem sentido para alguns valores do discriminador (e recebem `''`/`undefined` nos outros).

### Antes
```ts
declare const repo: {
  pausar(id: string, retomaEm: string): Promise<void>;
  cancelar(id: string, motivo: string): Promise<void>;
  reativar(id: string, cobrarAgora: boolean): Promise<void>;
};

export async function alterarAssinatura(
  id: string,
  acao: string,
  cobrarAgora: boolean,
  retomaEm: string,
  motivo: string,
): Promise<void> {
  switch (acao) {
    case 'PAUSAR':
      await repo.pausar(id, retomaEm);
      break;
    case 'CANCELAR':
      await repo.cancelar(id, motivo);
      break;
    case 'REATIVAR':
      await repo.reativar(id, cobrarAgora);
      break;
    default:
      throw new Error(`Ação desconhecida: ${acao}`);
  }
}

// call site: parâmetros vazios "porque a assinatura pede"
void alterarAssinatura('as-1', 'CANCELAR', true, '', 'preço alto');
```

### Depois
```ts
declare const repo: {
  pausar(id: string, retomaEm: string): Promise<void>;
  cancelar(id: string, motivo: string): Promise<void>;
  reativar(id: string, cobrarAgora: boolean): Promise<void>;
};
declare function assertNever(x: never): never;

export function pausarAssinatura(id: string, retomaEm: string): Promise<void> {
  return repo.pausar(id, retomaEm);
}

export function cancelarAssinatura(id: string, motivo: string): Promise<void> {
  return repo.cancelar(id, motivo);
}

export function reativarAssinatura(id: string): Promise<void> {
  return repo.reativar(id, true);
}

// quando a variante vem de runtime (deep link, webhook), despache por união
export type ComandoAssinatura =
  | { readonly kind: 'pausar'; readonly retomaEm: string }
  | { readonly kind: 'cancelar'; readonly motivo: string }
  | { readonly kind: 'reativar' };

export function executarComando(id: string, cmd: ComandoAssinatura): Promise<void> {
  switch (cmd.kind) {
    case 'pausar': return pausarAssinatura(id, cmd.retomaEm);
    case 'cancelar': return cancelarAssinatura(id, cmd.motivo);
    case 'reativar': return reativarAssinatura(id);
    default: return assertNever(cmd);
  }
}

void cancelarAssinatura('as-1', 'preço alto');
```

### Passos
1. Crie uma função por variante, com nome no imperativo do domínio (`pausarAssinatura`, `cancelarAssinatura`), contendo só o código daquele ramo e só os parâmetros que ele usa.
2. Faça a função original delegar cada ramo do `switch` para a função nova correspondente.
3. Percorra os call sites: cada literal constante passa a chamar a função explícita.
4. Quando não sobrar chamador da original, apague-a (ou marque `@deprecated`, se pública).
5. Se algum chamador escolhe a variante em runtime, mantenha um único ponto de despacho e modele o comando como **união discriminada** por `kind`, com `switch` exaustivo + `assertNever`.

### Ganhos
- Call site autoexplicativo: `cancelarAssinatura(id, motivo)` em vez de `alterarAssinatura(id, 'CANCELAR', true, '', motivo)`.
- Cada função declara apenas os parâmetros que realmente precisa.
- Elimina o `default: throw` e o risco de string errada só descoberto em produção.

### Quando NÃO aplicar
- Os ramos são triviais e a variante é escolhida dinamicamente — explodir em N funções só duplica o despacho.
- Novas variantes surgem sempre e todas compartilham a maior parte do fluxo: o correto é `Parameterize Method` (§5).
- Existe um contrato único obrigatório (handler de rota, `dispatch(action)` de reducer) — nesse caso o despacho é intencional.

### Nota TypeScript
Todo `boolean` posicional é candidato imediato a esta técnica; se a flag precisa existir, promova-a a campo de objeto de opções ou a união de literais (`{ cobranca: 'imediata' | 'proximo-ciclo' }`). Quando os parâmetros **variam por caso**, a união discriminada de comando é melhor que N funções e melhor que a flag por um motivo que a flag não oferece: ela **elimina combinações inválidas de parâmetros**. Com `{ kind: 'cancelar'; motivo: string }` é impossível passar `retomaEm` num cancelamento ou esquecer o `motivo` — o tipo não existe. É o mesmo desenho de `Action` de `useReducer` e de mensagem de fila: um `switch` sobre `kind` com `default: assertNever(cmd)` faz o compilador exigir tratamento de cada variante nova. Só não transforme *tudo* em comando: para chamadas diretas do próprio código, a função explícita continua sendo a API mais legível.

---

## 10. Replace Parameter with Method Call
**PT-BR:** Substituir Parâmetro por Chamada de Método · **Fonte:** https://refactoring.guru/replace-parameter-with-method-call

### Problema
O chamador executa uma consulta e passa o resultado como parâmetro, quando a própria função poderia fazer essa consulta.

### Solução
Remover o parâmetro e obter o valor dentro do corpo da função.

### Sinais no código (gatilhos)
- Todos os call sites calculam o argumento do mesmo jeito.
- O argumento vem de uma dependência que a função já tem (repositório, sessão, `clock`).
- Encadeamento burocrático: valor buscado só para ser repassado por três camadas.
- A regra de obtenção do valor está duplicada em vários chamadores.

### Antes
```ts
type Assinatura = { readonly ativa: boolean; readonly creditos: number };
type Resgate = { readonly id: string };
type Result<T, E> = { readonly ok: true; readonly value: T } | { readonly ok: false; readonly error: E };

declare const assinaturaRepo: { atual(usuarioId: string): Promise<Assinatura> };

export class ResgatarBeneficioService {
  constructor(private readonly resgates: { criar(beneficioId: string, usuarioId: string): Promise<Resgate> }) {}

  async executar(
    beneficioId: string,
    usuarioId: string,
    planoAtivo: boolean,
    creditosDisponiveis: number,
  ): Promise<Result<Resgate, string>> {
    if (!planoAtivo) return { ok: false, error: 'plano-inativo' };
    if (creditosDisponiveis <= 0) return { ok: false, error: 'sem-credito' };
    return { ok: true, value: await this.resgates.criar(beneficioId, usuarioId) };
  }
}

// o controller precisa conhecer a regra de assinatura para poder chamar o service
export async function handler(beneficioId: string, usuarioId: string, service: ResgatarBeneficioService) {
  const assinatura = await assinaturaRepo.atual(usuarioId);
  return service.executar(beneficioId, usuarioId, assinatura.ativa, assinatura.creditos);
}
```

### Depois
```ts
type Assinatura = { readonly ativa: boolean; readonly creditos: number };
type Resgate = { readonly id: string };
type Result<T, E> = { readonly ok: true; readonly value: T } | { readonly ok: false; readonly error: E };

type AssinaturaRepo = { atual(usuarioId: string): Promise<Assinatura> };
type ResgateRepo = { criar(beneficioId: string, usuarioId: string): Promise<Resgate> };

export class ResgatarBeneficioService {
  constructor(
    private readonly resgates: ResgateRepo,
    private readonly assinaturas: AssinaturaRepo,
  ) {}

  async executar(beneficioId: string, usuarioId: string): Promise<Result<Resgate, string>> {
    const assinatura = await this.assinaturas.atual(usuarioId);
    if (!assinatura.ativa) return { ok: false, error: 'plano-inativo' };
    if (assinatura.creditos <= 0) return { ok: false, error: 'sem-credito' };
    return { ok: true, value: await this.resgates.criar(beneficioId, usuarioId) };
  }
}

export function handler(beneficioId: string, usuarioId: string, service: ResgatarBeneficioService) {
  return service.executar(beneficioId, usuarioId);
}
```

### Passos
1. Verifique que a obtenção do valor não depende de outros parâmetros do chamador — se depender, a consulta não pode migrar.
2. Garanta que a função alvo tem (ou pode receber) a dependência que produz o valor: injete por construtor ou por factory de closure.
3. Dentro da função, substitua os usos do parâmetro pela chamada de consulta.
4. Remova o parâmetro (`Remove Parameter`, §2) e apague nos chamadores o código que o produzia.
5. Atualize os testes: o argumento fake vira um fake da dependência injetada (um objeto literal já satisfaz o tipo estrutural).

### Ganhos
- Regra de negócio deixa de vazar para a borda: o controller não precisa saber que o resgate depende do estado da assinatura.
- Menos parâmetros, menos chance de o chamador passar dado desatualizado.
- Uma única fonte de verdade para o valor.

### Quando NÃO aplicar
- O valor legitimamente varia por chamador (filtro escolhido pelo usuário, contexto de tela).
- A consulta é caro e o chamador já tem o valor em cache — internalizar duplicaria I/O por chamada.
- Criaria dependência que quebra a direção das camadas (domínio importando infra) ou ciclo de imports.
- A função precisa ser pura e determinística: injete `clock`/`randomUUID` em vez de chamar `Date.now()` dentro.

### Nota TypeScript
A dependência entra por **construtor** ou por **closure factory** (`const criarService = (deps: Deps) => ({ executar: (...) => ... })`), não por parâmetro de chamada — e como a tipagem é estrutural, o fake do teste é um objeto literal, sem framework de mock. Cuidado especial em **React**: mover a consulta "para dentro" de um hook ou callback muda a lista de dependências. Se você trocar um parâmetro por uma leitura de contexto/store dentro de um `useCallback`, esse valor precisa entrar em `deps` — senão a closure congela o valor antigo (bug de stale closure que o lint `react-hooks/exhaustive-deps` acusa). Quando o valor muda ao longo do tempo, não o leia uma vez: obtenha-o via hook (`useContext`, `useSyncExternalStore`, query) no topo do componente e deixe a função pura recebê-lo — aqui a técnica se aplica no service, não no componente.

---

## 11. Hide Method
**PT-BR:** Esconder Método · **Fonte:** https://refactoring.guru/hide-method

### Problema
Uma função não é usada de fora, mas está exposta na API pública do módulo ou da classe.

### Solução
Reduzir a visibilidade: campo privado de classe, closure, ou simplesmente não exportar.

### Sinais no código (gatilhos)
- "Find all references" mostra usos apenas dentro do próprio arquivo/pasta.
- Helpers de mapeamento, formatação, parsing e cache exportados junto com a API real.
- `export *` no barrel file publicando tudo que existe no módulo.
- Testes importando helpers internos para "testar detalhe" em vez de comportamento.

### Antes
```ts
export type Notificacao = { readonly id: string; readonly titulo: string; readonly lida: boolean };

export const SEPARADOR = '|';
export const TTL_MS = 900_000;

declare const api: { titulos(usuarioId: string): Promise<string> };

export function desserializarTitulos(bruto: string): ReadonlyArray<string> {
  return bruto.trim() === '' ? [] : bruto.split(SEPARADOR).map((t) => t.trim());
}

export function cacheVencido(atualizadoEm: number): boolean {
  return atualizadoEm < Date.now() - TTL_MS;
}

export class NotificacaoStore {
  public itens: ReadonlyArray<Notificacao> = [];
  public atualizadoEm = 0;

  public async carregar(usuarioId: string): Promise<void> {
    if (!cacheVencido(this.atualizadoEm)) return;
    const bruto = await api.titulos(usuarioId);
    this.itens = desserializarTitulos(bruto).map((titulo, i) => ({
      id: String(i),
      titulo,
      lida: false,
    }));
    this.atualizadoEm = Date.now();
  }
}
```

### Depois
```ts
export type Notificacao = { readonly id: string; readonly titulo: string; readonly lida: boolean };

const SEPARADOR = '|';
const TTL_MS = 900_000;

declare const api: { titulos(usuarioId: string): Promise<string> };

// não exportados: invisíveis fora deste módulo
function desserializarTitulos(bruto: string): ReadonlyArray<string> {
  return bruto.trim() === '' ? [] : bruto.split(SEPARADOR).map((t) => t.trim());
}

function cacheVencido(atualizadoEm: number): boolean {
  return atualizadoEm < Date.now() - TTL_MS;
}

export class NotificacaoStore {
  #itens: ReadonlyArray<Notificacao> = [];
  #atualizadoEm = 0;

  get itens(): ReadonlyArray<Notificacao> {
    return this.#itens;
  }

  async carregar(usuarioId: string): Promise<void> {
    if (!cacheVencido(this.#atualizadoEm)) return;
    const bruto = await api.titulos(usuarioId);
    this.#itens = desserializarTitulos(bruto).map((titulo, i) => ({
      id: String(i),
      titulo,
      lida: false,
    }));
    this.#atualizadoEm = Date.now();
  }
}
```

### Passos
1. Varra o módulo procurando `export` sem consumidor externo (o editor mostra a contagem de referências; `ts-prune`/`knip` acham exports mortos).
2. Apague o `export` do helper. Se algo quebrar, o `tsc` mostra exatamente quem dependia dele — decida se é uso legítimo (mantenha) ou vazamento (mova o chamador).
3. Em classes, troque campos e métodos internos por `#nome` (privacidade real em runtime) e publique leitura por getter.
4. No barrel file, substitua `export *` por reexports nomeados: o que não está listado não é API.
5. Se um teste quebrar por acessar interno, reescreva-o contra o comportamento público em vez de reabrir o export.
6. Repita depois de cada refatoração maior — visibilidade tende a vazar com o tempo.

### Ganhos
- Superfície pública menor: mudar o interno não quebra ninguém.
- O leitor entende rápido qual é o contrato real do módulo.
- Habilita tree-shaking e minificação de nomes internos, e reduz uso indevido entre camadas.

### Quando NÃO aplicar
- A função é ponto de extensão deliberado, ou parte documentada da API do pacote.
- É um método que implementa `interface` pública — não pode ser escondido.
- O helper é genuinamente compartilhado por dois módulos: extraia para um módulo interno próprio em vez de duplicar ou de exportar do lugar errado.

### Nota TypeScript
Em TS a privacidade tem **três níveis, do mais forte ao mais fraco**. (1) `#campo`/`#metodo` de classe: privado de verdade em runtime, inacessível por `obj['x']` ou `Object.keys`, e o `private` da linguagem não é isso — `private` só existe em tempo de compilação e desaparece no JS emitido. (2) **Closure**: o que é criado dentro de uma factory e não retornado é inalcançável (`const criarStore = () => { let itens = []; return { listar: () => itens }; }`). (3) **Fronteira de módulo**: não exportar. Este é o nível mais usado e o mais barato — todo módulo ES é fechado por padrão, então a regra prática é `export` explícito, um por símbolo, e nunca `export *`. Cuidado com **barrel files**: um `export *` transforma qualquer helper novo em API pública sem ninguém decidir isso, e ainda atrapalha tree-shaking. Se um helper precisa ser visível para outro módulo do mesmo pacote mas não para fora dele, o equivalente do `internal` é um subpath não publicado (`src/internal/…`) mais a lista de `exports` do `package.json`.

---

## 12. Replace Constructor with Factory Method
**PT-BR:** Substituir Construtor por Método Fábrica · **Fonte:** https://refactoring.guru/replace-constructor-with-factory-method

### Problema
Um construtor faz mais que atribuir valores: valida, normaliza, escolhe subtipo ou monta colaboradores.

### Solução
Encapsular a criação em uma função fábrica, deixando o construtor privado ou eliminando a classe.

### Sinais no código (gatilhos)
- Construtor com validação, parsing ou `switch` sobre um "type code".
- Necessidade de devolver formas diferentes de objeto conforme o dado de entrada.
- Criação que pode falhar — construtor não consegue devolver `Result`, só `throw`.
- Vários "modos" de criação que sobrecargas de construtor não expressam (`deDto`, `rascunho`, `restaurado`).

### Antes
```ts
type ProdutoDto = {
  readonly id: string;
  readonly nome: string;
  readonly tipo: number;
  readonly variacoes: string;
  readonly urlDownload: string;
};

export class Produto {
  readonly variacoes: ReadonlyArray<string>;
  readonly digital: boolean;
  readonly urlDownload: string | null;

  constructor(readonly id: string, readonly nome: string, tipo: number,
    variacoes: string, urlDownload: string) {
    if (nome.trim() === '') throw new Error('Nome vazio');
    this.digital = tipo === 2;
    if (tipo === 1) {
      this.variacoes = variacoes.split(';').map((v) => v.trim());
      this.urlDownload = null;
    } else if (tipo === 2) {
      this.variacoes = [];
      this.urlDownload = urlDownload;
    } else {
      throw new Error(`Tipo inválido: ${tipo}`);
    }
  }
}

declare const dto: ProdutoDto;
const produto = new Produto(dto.id, dto.nome, dto.tipo, dto.variacoes, dto.urlDownload);
```

### Depois
```ts
type ProdutoDto = {
  readonly id: string;
  readonly nome: string;
  readonly tipo: number;
  readonly variacoes: string;
  readonly urlDownload: string;
};

type Result<T, E> = { readonly ok: true; readonly value: T } | { readonly ok: false; readonly error: E };

export type Produto =
  | { readonly kind: 'fisico'; readonly id: string; readonly nome: string; readonly variacoes: ReadonlyArray<string> }
  | { readonly kind: 'digital'; readonly id: string; readonly nome: string; readonly urlDownload: string };

export function produtoDeDto(dto: ProdutoDto): Result<Produto, string> {
  const nome = dto.nome.trim();
  if (nome === '') return { ok: false, error: 'nome-vazio' };
  switch (dto.tipo) {
    case 1:
      return { ok: true, value: { kind: 'fisico', id: dto.id, nome,
        variacoes: dto.variacoes.split(';').map((v) => v.trim()) } };
    case 2:
      return { ok: true, value: { kind: 'digital', id: dto.id, nome, urlDownload: dto.urlDownload } };
    default: return { ok: false, error: `tipo-invalido:${dto.tipo}` };
  }
}

// factory de closure: injeta dependências sem container
export function criarFabricaDeProduto(logger: { warn(msg: string): void }) {
  return (dto: ProdutoDto): Result<Produto, string> => {
    const resultado = produtoDeDto(dto);
    if (!resultado.ok) logger.warn(`produto inválido: ${resultado.error}`);
    return resultado;
  };
}
```

### Passos
1. Crie a função fábrica chamando o construtor atual, sem mudar nada de comportamento.
2. Troque todas as chamadas `new X(...)` por chamadas à fábrica.
3. Torne o construtor `private` (`private constructor(...)`) — ou elimine a classe se ela era só dados.
4. Mova para a fábrica o que não é atribuição simples: validação, normalização, escolha da variante.
5. Se a criação pode falhar, faça a fábrica devolver `Result<T, E>` (ou `T | undefined`) em vez de lançar do construtor.
6. Dê nomes de domínio às fábricas (`produtoDeDto`, `carrinhoVazio`, `pedidoRestaurado`) — essa é a principal vantagem sobre sobrecargas.

### Ganhos
- A fábrica escolhe a variante correta, o que um construtor não consegue fazer.
- Nomes descritivos por modo de criação, e liberdade para devolver instância já existente (cache/pool).
- Criação falível expressa como valor em vez de exceção durante a construção.

### Quando NÃO aplicar
- A criação apenas monta um objeto de dados: um literal tipado já basta e a fábrica é cerimônia.
- Um schema Zod (`schema.parse`/`safeParse`) na fronteira já valida e tipa — não duplique a validação numa fábrica manual.
- A classe é instanciada por um framework/ORM que exige construtor público com forma fixa.
- Uma função de mapeamento simples (`dtoParaProduto`) resolveria com menos acoplamento.

### Nota TypeScript
Em TS a **função fábrica exportada é mais idiomática que `static create`**: é tree-shakeable, não obriga a existir uma classe, e o nome carrega o modo de criação. Dois usos que só a fábrica permite: devolver uma **união** de tipos diferentes (`Produto = ProdutoFisico | ProdutoDigital`) conforme a entrada — o `new` está preso ao tipo da classe, enquanto o retorno `Result<Produto, E>` deixa o chamador estreitar por `kind` com `switch` exaustivo; e a **factory de closure**, que recebe dependências e devolve as operações já ligadas a elas (`criarPedidoService({ repo, clock })`) — é injeção de dependência sem container e sem `this`. Se quiser manter a validação num único lugar, deixe o parse da fronteira (Zod) devolver o DTO tipado e a fábrica cuidar apenas da regra de domínio. Quando a classe precisa mesmo existir, `private constructor` + `static deDto` funciona, mas reserve isso para casos com estado ou identidade real.

---

## 13. Replace Error Code with Exception
**PT-BR:** Substituir Código de Erro por Exceção · **Fonte:** https://refactoring.guru/replace-error-code-with-exception

### Problema
A função devolve um valor especial (`-1`, `0`, `null`, código numérico) para sinalizar erro, obrigando o chamador a conhecer a convenção.

### Solução
Sinalizar a falha por um mecanismo explícito — em TS moderno, um `Result<T, E>` como união discriminada; exceção apenas para o inesperado.

### Sinais no código (gatilhos)
- Retorno `number`/`string` de código de erro, com constantes `ERRO_*`.
- `if (codigo === -3)` espalhado pelos chamadores.
- Retorno `null` com significados diferentes (não achou vs. sem permissão vs. offline).
- Comentário/JSDoc explicando o significado de cada número.

### Antes
```ts
const SUCESSO = 0;
const ERRO_NAO_ENCONTRADA = -1;
const ERRO_JA_ATIVA = -2;
const ERRO_CUPOM = -3;

declare const repo: {
  assinaturaAtual(usuarioId: string): Promise<{ readonly ativa: boolean } | null>;
  cupomValido(cupom: string): Promise<boolean>;
  ativar(usuarioId: string, cupom: string): Promise<{ readonly validaAte: string }>;
};

/** Retorna 0 em caso de sucesso, -1 não encontrada, -2 já ativa, -3 cupom inválido. */
export async function ativarAssinatura(usuarioId: string, cupom: string): Promise<number> {
  const assinatura = await repo.assinaturaAtual(usuarioId);
  if (assinatura === null) return ERRO_NAO_ENCONTRADA;
  if (assinatura.ativa) return ERRO_JA_ATIVA;
  if (cupom !== '' && !(await repo.cupomValido(cupom))) return ERRO_CUPOM;
  await repo.ativar(usuarioId, cupom);
  return SUCESSO;
}

declare function mostrarErroCupom(): void;
export async function onAtivarClick(usuarioId: string, cupom: string): Promise<void> {
  const codigo = await ativarAssinatura(usuarioId, cupom);
  // convenção implícita: -3 é cupom; os outros silenciosamente ignorados
  if (codigo === -3) mostrarErroCupom();
}
```

### Depois
```ts
type Result<T, E> = { readonly ok: true; readonly value: T } | { readonly ok: false; readonly error: E };
type Ativacao = { readonly validaAte: string };

export type ErroAtivacao =
  | { readonly kind: 'assinatura-nao-encontrada' }
  | { readonly kind: 'ja-esta-ativa' }
  | { readonly kind: 'cupom-invalido'; readonly cupom: string };

declare const repo: {
  assinaturaAtual(usuarioId: string): Promise<{ readonly ativa: boolean } | null>;
  cupomValido(cupom: string): Promise<boolean>;
  ativar(usuarioId: string, cupom: string): Promise<Ativacao>;
};
declare function assertNever(x: never): never;
declare function mostrarErro(msg: string): void;

export async function ativarAssinatura(usuarioId: string, cupom: string): Promise<Result<Ativacao, ErroAtivacao>> {
  const assinatura = await repo.assinaturaAtual(usuarioId);
  if (assinatura === null) return { ok: false, error: { kind: 'assinatura-nao-encontrada' } };
  if (assinatura.ativa) return { ok: false, error: { kind: 'ja-esta-ativa' } };
  if (cupom !== '' && !(await repo.cupomValido(cupom))) return { ok: false, error: { kind: 'cupom-invalido', cupom } };
  return { ok: true, value: await repo.ativar(usuarioId, cupom) };
}

export async function onAtivarClick(usuarioId: string, cupom: string): Promise<void> {
  const r = await ativarAssinatura(usuarioId, cupom);
  if (r.ok) return;
  switch (r.error.kind) {
    case 'assinatura-nao-encontrada': return mostrarErro('Assinatura não encontrada.');
    case 'ja-esta-ativa': return mostrarErro('Sua assinatura já está ativa.');
    case 'cupom-invalido': return mostrarErro(`Cupom ${r.error.cupom} inválido.`);
    default: return assertNever(r.error);
  }
}
```

### Passos
1. Liste todos os códigos retornados e o significado de cada um.
2. Classifique cada caso: **erro esperado de negócio** (cupom inválido, assinatura ausente) vs. **falha inesperada** (JSON corrompido, invariante violada, indisponibilidade de infra).
3. Para os esperados, crie a união de erro (uma variante por caso, com o contexto que a UI precisa) e devolva `Result<T, E>`.
4. Para os inesperados, `throw` de um `Error` específico (com `cause` preservando o original) e trate na borda: middleware de erro, `error boundary`, handler global.
5. Troque cada `if (codigo === X)` por `if (!r.ok)` + `switch` exaustivo sobre `r.error.kind` com `default: assertNever(...)`.
6. Apague as constantes de código e o JSDoc que as explicava.

### Ganhos
- Casos de erro deixam de ser convenção implícita e passam a ser tipos verificados na compilação.
- `switch` exaustivo impede esquecer um caso quando um erro novo é adicionado.
- Cada erro carrega o contexto necessário para a mensagem, sem parsing de código.

### Quando NÃO aplicar
- O "código" é contrato externo (status HTTP, código do gateway de pagamento): mantenha-o no DTO e traduza para a união na fronteira.
- Existe um único modo de falha, sem dados: `boolean` ou `T | undefined` já basta.
- Trocar por exceção um fluxo esperado: exceção como controle de fluxo é anti-padrão (veja §14).

### Nota TypeScript
Critério prático: **erro esperado é valor; exceção é para o inesperado**. Esperado = a regra de negócio prevê (validação, "não encontrado", cupom expirado, sem permissão) → `Result<T, E>` com `E` sendo união discriminada. Inesperado = nada a decidir no ponto da chamada (bug, invariante rompida, infra fora) → `throw`, tratado na borda. O que reforça esse desenho em TS é que **`throw` não é tipado**: qualquer valor pode ser lançado, não existem checked exceptions, e com `useUnknownInCatchVariables` (parte do `strict`) a variável do `catch` é `unknown` — você precisa fazer narrowing (`e instanceof Error`) antes de qualquer coisa, e a assinatura da função nunca conta quais falhas ela pode ter. Um `Result` conta. Dois pontos operacionais: `Promise` rejeitada sem tratamento vira `unhandledRejection` (que derruba o processo no Node por padrão) ou um erro invisível no browser — sempre `await` ou trate; e ao converter exceção em erro de domínio, preserve a origem com `new Error('falha ao ativar', { cause: e })`, para o log manter a stack real.

---

## 14. Replace Exception with Test
**PT-BR:** Substituir Exceção por Teste · **Fonte:** https://refactoring.guru/replace-exception-with-test

### Problema
Uma exceção é lançada e capturada em uma situação previsível, que uma verificação prévia resolveria.

### Solução
Substituir o `try/catch` por uma condição que checa o caso de borda antes de executar.

### Sinais no código (gatilhos)
- `try/catch` em torno de acesso a índice, `JSON.parse`, `Number.parseFloat` ou `undefined` esperado.
- `catch` que só devolve default (`catch { return []; }`).
- Exceção usada rotineiramente no caminho normal (visitante deslogado, carrinho vazio).
- `catch (e)` genérico cobrindo motivos distintos — e, sem narrowing, engolindo bug real junto.

### Antes
```ts
declare const repo: {
  itens(carrinhoId: string): Promise<ReadonlyArray<{ readonly id: string; readonly precoTexto: string }>>;
  validoAte(carrinhoId: string): Promise<string>;
};

export async function carregarItem(
  carrinhoId: string,
  indice: number,
): Promise<{ readonly id: string; readonly preco: number } | null> {
  try {
    const itens = await repo.itens(carrinhoId);
    const item = itens[indice]!;
    const preco = Number.parseFloat(item.precoTexto);
    if (Number.isNaN(preco)) throw new Error('preço inválido');
    if (Date.now() > Date.parse(await repo.validoAte(carrinhoId))) {
      throw new Error('carrinho expirado');
    }
    return { id: item.id, preco };
  } catch (e) {
    // engole índice fora, preço inválido, expiração — e qualquer bug real
    return null;
  }
}
```

### Depois
```ts
type Result<T, E> = { readonly ok: true; readonly value: T } | { readonly ok: false; readonly error: E };

type ErroItem =
  | { readonly kind: 'indice-fora-do-carrinho' }
  | { readonly kind: 'carrinho-expirado' }
  | { readonly kind: 'preco-invalido'; readonly bruto: string };

declare const repo: {
  itens(carrinhoId: string): Promise<ReadonlyArray<{ readonly id: string; readonly precoTexto: string }>>;
  validoAte(carrinhoId: string): Promise<string>;
};

export async function carregarItem(
  carrinhoId: string,
  indice: number,
  agora: () => number = Date.now,
): Promise<Result<{ readonly id: string; readonly preco: number }, ErroItem>> {
  if (agora() > Date.parse(await repo.validoAte(carrinhoId))) {
    return { ok: false, error: { kind: 'carrinho-expirado' } };
  }
  const itens = await repo.itens(carrinhoId);
  const item = itens[indice];
  if (item === undefined) {
    return { ok: false, error: { kind: 'indice-fora-do-carrinho' } };
  }
  const preco = Number.parseFloat(item.precoTexto);
  if (Number.isNaN(preco)) {
    return { ok: false, error: { kind: 'preco-invalido', bruto: item.precoTexto } };
  }
  return { ok: true, value: { id: item.id, preco } };
}
```

### Passos
1. Crie a verificação da condição de borda **antes** do bloco `try` (índice, `undefined`, string vazia, prazo, permissão).
2. Mova para dentro dessa condição o que estava no `catch` (retornar default ou a variante de erro).
3. Deixe temporariamente no `catch` um `throw` e rode a suíte de testes.
4. Se nenhum teste dispara mais a exceção, remova o `try/catch` inteiro.
5. Se há vários motivos de falha, troque o retorno `null`/default por `Result<T, E>` com união de erro, para o chamador distinguir os casos.
6. Mantenha `try/catch` só para o genuinamente imprevisível (rede, disco, payload de terceiro) — e converta lá para o seu tipo de erro, preservando `cause`.

### Ganhos
- Fluxo linear e legível, sem desvio por exceção no caminho esperado.
- Mais rápido: montar stack trace é caro em hot path e em loop sobre listas grandes.
- Elimina o `catch` genérico que escondia bug real atrás de um `return null`.

### Quando NÃO aplicar
- A verificação prévia cria condição de corrida (checa e o recurso muda antes do uso): aí `try/catch` é o correto.
- A checagem duplicaria uma validação caro já feita pela camada de baixo.
- A falha é realmente imprevisível (rede, disco, JSON de terceiro malformado).
- A API que você chama só comunica falha por `throw` e não oferece variante segura (`safeParse`, `tryX`).

### Nota TypeScript
As verificações idiomáticas quase sempre dispensam o `try/catch`: `at()`/indexação com `noUncheckedIndexedAccess` (que tipa o acesso como `T | undefined` e **obriga** a checagem), `??`, `?.`, `Number.isNaN`, `schema.safeParse` em vez de `schema.parse`. O alvo moderno não é "exceção vs. checagem", é **erro esperado como valor**: `Result<T, E>` com `E` como união discriminada, e exceção reservada ao inesperado. Isso pesa mais em TS do que em linguagens com exceções tipadas, porque em JS **`throw` aceita qualquer valor e `catch` recebe `unknown`** (com `useUnknownInCatchVariables`): não existe checked exception nem tipo de erro na assinatura, então quem chama não tem como saber o que capturar nem como estreitar sem `instanceof`. Cuide também do assíncrono: `try/catch` só pega a rejeição se houver `await` — um `promise.then(...)` sem `catch` escapa para `unhandledRejection`. E quando você realmente capturar algo de infra, converta para o seu erro de domínio com `new Error('mensagem', { cause: e })` em vez de descartar a origem.
