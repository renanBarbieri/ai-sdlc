# Composing Methods — Compondo Métodos

> Grupo 1 de 6 · 9 técnicas · Fonte: refactoring.guru/refactoring/techniques

## Quando este grupo se aplica

- Funções longas: componente React acima de ~60 linhas, service Node que faz validação + regra + persistência no mesmo corpo, handler que monta DTO, chama repositório e formata resposta.
- `useEffect` fazendo três coisas (buscar dado, sincronizar título da página, disparar analytics) — cada responsabilidade é um hook próprio.
- Comentários que existem só para rotular blocos (`// calcula subtotal`, `// monta os rótulos`) — o comentário é o nome da função que falta.
- Expressões booleanas ou aritméticas com 3+ operandos dentro de `if`, de um ternário ou de uma prop no JSX.
- `let` reatribuído com dois papéis diferentes, ou temporária trivial que só renomeia uma expressão.
- Parâmetro objeto/array sendo mutado (`push`, `sort`, `splice`, atribuição de propriedade) — o efeito escapa para o chamador por aliasing.
- Emaranhado de locais interdependentes que impede extrair qualquer trecho isoladamente.
- Algoritmo artesanal (`while` com índices, dedupe com flag booleana) onde métodos de array, `Set` ou `Map` já resolvem.

## Índice

1. [Extract Method](#1-extract-method)
2. [Inline Method](#2-inline-method)
3. [Extract Variable](#3-extract-variable)
4. [Inline Temp](#4-inline-temp)
5. [Replace Temp with Query](#5-replace-temp-with-query)
6. [Split Temporary Variable](#6-split-temporary-variable)
7. [Remove Assignments to Parameters](#7-remove-assignments-to-parameters)
8. [Replace Method with Method Object](#8-replace-method-with-method-object)
9. [Substitute Algorithm](#9-substitute-algorithm)

---

## 1. Extract Method
**PT-BR:** Extrair Método · **Fonte:** https://refactoring.guru/extract-method

### Problema
Existe um fragmento de código que pode ser agrupado. Funções longas são difíceis de entender e concentram responsabilidades distintas.

### Solução
Mova o fragmento para uma nova função com nome autoexplicativo e substitua o código original por uma chamada a ela. Em TS/React a extração tem três destinos: **função pura no módulo** (cálculo sem estado), **custom hook** (estado/efeito/assinatura), **componente** (pedaço de JSX).

### Sinais no código (gatilhos)
- Função com mais de ~25 linhas, ou componente com mais de ~60, misturando busca de dados + cálculo + markup.
- Comentário rotulando um bloco: o texto do comentário já é o nome da função a extrair.
- `useEffect` com mais de um assunto, ou dois `useState` que só existem juntos → custom hook.
- Bloco de JSX com 8+ linhas, ou `.map()` com corpo multilinha dentro do `return` → componente.
- O mesmo trecho aparece em dois ou mais lugares (duplicação).
- Cálculo de regra de negócio escrito no corpo do componente, antes do `return`.

### Antes
```tsx
type ItemCarrinho = { readonly id: string; readonly preco: number; readonly quantidade: number };
type Props = { readonly itens: readonly ItemCarrinho[]; readonly cupom: string | null };

export function ResumoCarrinho({ itens, cupom }: Props) {
  const [frete, setFrete] = useState<number | null>(null);
  const [erro, setErro] = useState<string | null>(null);
  useEffect(() => {
    let ativo = true;
    fetch(`/api/frete?itens=${itens.length}`)
      .then((r) => r.json() as Promise<{ valor: number }>)
      .then((d) => { if (ativo) setFrete(d.valor); })
      .catch((e: unknown) => { if (ativo) setErro(String(e)); });
    return () => { ativo = false; };
  }, [itens.length]);

  // calcula subtotal, desconto e total
  let subtotal = 0;
  for (const item of itens) subtotal += item.preco * item.quantidade;
  const desconto = cupom === 'BEMVINDO10' ? subtotal * 0.1 : 0;
  const total = subtotal - desconto + (frete ?? 0);
  return (
    <section>
      <ul>
        {itens.map((item) => (
          <li key={item.id}>
            {item.quantidade} x {(item.preco / 100).toFixed(2)}
          </li>
        ))}
      </ul>
      {erro !== null && <p>Frete indisponível: {erro}</p>}
      <p>Total: {(total / 100).toFixed(2)}</p>
    </section>
  );
}
```

### Depois
```tsx
// 1. função pura no módulo: regra de negócio testável sem React
export function calcularTotal(itens: readonly ItemCarrinho[], cupom: string | null, frete: number): number {
  const subtotal = itens.reduce((soma, item) => soma + item.preco * item.quantidade, 0);
  const desconto = cupom === 'BEMVINDO10' ? subtotal * 0.1 : 0;
  return subtotal - desconto + frete;
}
// 2. custom hook: o Extract Method do estado e do efeito (rejeição sempre tratada)
function useFrete(itens: number): { valor: number | null; erro: string | null } {
  const [valor, setValor] = useState<number | null>(null);
  const [erro, setErro] = useState<string | null>(null);
  useEffect(() => {
    let ativo = true;
    fetch(`/api/frete?itens=${itens}`)
      .then((r) => r.json() as Promise<{ valor: number }>)
      .then((d) => { if (ativo) setValor(d.valor); })
      .catch((e: unknown) => { if (ativo) setErro(String(e)); });
    return () => { ativo = false; };
  }, [itens]);
  return { valor, erro };
}
// 3. componente: o Extract Method da UI
const ListaDeItens = ({ itens }: { itens: readonly ItemCarrinho[] }) => (
  <ul>{itens.map((i) => <li key={i.id}>{i.quantidade} x {(i.preco / 100).toFixed(2)}</li>)}</ul>
);
export function ResumoCarrinho({ itens, cupom }: Props) {
  const { valor: frete, erro } = useFrete(itens.length);
  const total = calcularTotal(itens, cupom, frete ?? 0);
  return (
    <section>
      <ListaDeItens itens={itens} />
      {erro !== null && <p>Frete indisponível: {erro}</p>}
      <p>Total: {(total / 100).toFixed(2)}</p>
    </section>
  );
}
```

### Passos
1. Escolha o destino antes de cortar: cálculo puro → função no módulo (ou em `domain/`); usa hook → custom hook `useX`; devolve JSX → componente.
2. Crie a função com nome que descreva a intenção, não a mecânica (`calcularTotal`, não `loopItens`).
3. Mova o fragmento, remova-o do local original e chame a nova função ali.
4. Variáveis declaradas e usadas só dentro do fragmento viram locais da nova função; as declaradas antes viram parâmetros (`readonly`, com nomes de negócio).
5. Deixe o compilador achar os usos: apague o trecho antigo e siga os erros do TS Server até zerar.
6. Se o fragmento produz mais de um valor, retorne um objeto tipado (`{ subtotal, desconto }`), não parâmetros de saída.
7. Se o fragmento faz I/O, a nova função é `async` e retorna `Promise` — não crie `void promise` interno escondido.
8. No caso do custom hook, mantenha a **ordem** dos hooks: a extração não pode ficar atrás de um `if`.
9. Para renomear depois, use o rename do TS Server (F2), nunca find-and-replace textual.

### Ganhos
- Nome descritivo substitui comentário: o código passa a se documentar.
- Elimina duplicação — a função extraída é reusada.
- Isola trechos, reduzindo o risco de alteração acidental de variáveis vizinhas.
- Habilita teste unitário do cálculo sem renderizar componente nem subir servidor.
- Pré-requisito para quase todas as outras técnicas do grupo.

### Quando NÃO aplicar
- Extração que exige 5+ parâmetros: o corte foi no lugar errado — reavalie o limite ou use Replace Method with Method Object.
- Extrair componente só para "encurtar o arquivo", passando 10 props que são o estado inteiro do pai: a indireção custa mais que ganha.
- Extrair um componente que recebe `children` renderizado inline cria nova identidade a cada render e derruba memoização do filho — extraia para fora do corpo do pai, não para dentro.
- Wrapper de uma linha que só renomeia a chamada — é o smell inverso (ver Inline Method).
- Hot path medido (parser de payload grande, virtualização de lista): meça antes de multiplicar chamadas.

### Nota TypeScript
Três destinos, três regras. Função pura no módulo é sempre a primeira opção: não depende de React, é testável e o TS infere o retorno. Custom hook é o único jeito de extrair estado/efeito — a função extraída precisa começar com `use` para o lint de hooks validar as regras. Extrair JSX é literalmente o Extract Method da UI: um `return (...)` com 40 linhas é uma função longa. O erro mais comum é declarar o componente extraído **dentro** do corpo do componente pai: cada render cria um tipo de componente novo, o React desmonta e remonta a subárvore e todo o estado local do filho é perdido. Declare no escopo do módulo. O segundo erro é extrair para dentro de um `useEffect` uma função que passa a ser recriada a cada render e entra no array de dependências — nesse caso mova a função para o módulo (se pura) ou envolva com `useCallback`.

---

## 2. Inline Method
**PT-BR:** Método em Linha (Embutir Método) · **Fonte:** https://refactoring.guru/inline-method

### Problema
O corpo da função é mais óbvio que seu nome. Ela só delega para outra função ou envolve uma única expressão trivial.

### Solução
Substitua as chamadas pelo conteúdo da função e apague a função.

### Sinais no código (gatilhos)
- Método de uma linha que só chama outro com os mesmos argumentos.
- Cadeia de delegação: `a()` chama `b()` que chama `c()`, sem lógica adicional.
- Nome do wrapper não agrega informação (`getDados`, `handleIt`, `validar` que só faz `x !== null`).
- Método `private` (ou função não exportada) com um único ponto de chamada e corpo trivial.
- Custom hook que só devolve `useState` sem nenhuma regra em volta.
- Resíduo de refatoração anterior: a função existia para um caso já removido.

### Antes
```ts
export class PedidoService {
  constructor(
    private readonly api: PedidoApi,
    private readonly cache: PedidoCache,
  ) {}

  async listarDoUsuario(usuarioId: string): Promise<readonly Pedido[]> {
    const dtos = await this.buscarNaApi(usuarioId);
    return dtos.map(paraPedido);
  }

  private async buscarNaApi(usuarioId: string): Promise<readonly PedidoDto[]> {
    return this.api.listarPedidos(usuarioId);
  }

  async cancelar(pedidoId: string): Promise<void> {
    if (await this.podeCancelar(pedidoId)) {
      await this.api.cancelar(pedidoId);
      this.cache.remover(pedidoId);
    }
  }

  private async podeCancelar(pedidoId: string): Promise<boolean> {
    return this.temStatusAberto(pedidoId);
  }

  private async temStatusAberto(pedidoId: string): Promise<boolean> {
    const pedido = await this.cache.buscar(pedidoId);
    return pedido?.status === 'aberto';
  }
}
```

### Depois
```ts
export class PedidoService {
  constructor(
    private readonly api: PedidoApi,
    private readonly cache: PedidoCache,
  ) {}

  async listarDoUsuario(usuarioId: string): Promise<readonly Pedido[]> {
    const dtos = await this.api.listarPedidos(usuarioId);
    return dtos.map(paraPedido);
  }

  async cancelar(pedidoId: string): Promise<void> {
    const pedido = await this.cache.buscar(pedidoId);
    if (pedido?.status === 'aberto') {
      await this.api.cancelar(pedidoId);
      this.cache.remover(pedidoId);
    }
  }
}
```

### Passos
1. Confirme que a função não é `abstract`, não é sobrescrita por subclasse e não implementa um método de interface — se for polimórfica, não aplique.
2. Verifique que não está exportada e consumida por outro módulo: `export` é contrato público. Remova o `export` primeiro e deixe o compilador listar os usos externos.
3. Encontre todas as chamadas com "Find All References" do TS Server e substitua cada uma pelo corpo.
4. Ajuste os nomes: troque os parâmetros formais pelos argumentos reais de cada chamada.
5. Preserve `await`: embutir uma função `async` dentro de outra exige manter o `await` da expressão, senão a `Promise` vaza.
6. Se o corpo embutido deixar a expressão ilegível, extraia uma `const` com nome de negócio (Extract Variable) em vez de manter a função.
7. Apague a função, rode `tsc --noEmit` e os testes.

### Ganhos
- Menos indireção: a leitura não precisa saltar entre arquivos para entender uma linha.
- Menos superfície interna para manter e para mockar em testes.
- Aproxima o comportamento do ponto de uso, revelando o próximo refactoring.
- Reduz o grafo de módulos e, com ele, o risco de import circular.

### Quando NÃO aplicar
- Função polimórfica ou membro de interface implementada por várias classes — embutir quebra o contrato.
- O nome é a única documentação de uma regra (`podeUsarCupom()` embutido como `usuario.pedidos.length === 0 && !usuario.bloqueado` perde o significado).
- É ponto de extensão para teste (seam), instrumentação ou feature flag.
- Muitos pontos de chamada: embutir cria duplicação — o problema real pode ser o nome, não a existência.
- Wrapper que existe para estabilizar identidade referencial (`useCallback` passado a filho memoizado).

### Nota TypeScript
Boa parte da delegação trivial desaparece com recursos da linguagem, não com inline manual: getter (`get total() { return ... }`), parâmetros default e objeto de opções desestruturado (eliminam sobrecargas-wrapper), `?.` e `??` (eliminam funções `temX`/`getXOuPadrao`), e reexport (`export { x } from './y'`) em vez de função-ponte. Antes de embutir, verifique se o wrapper não pode virar uma dessas construções. O erro mais comum em Node/React é embutir uma função `async` esquecendo o `await` — o `if` passa a testar um objeto `Promise`, que é sempre truthy, e o bug é silencioso; ligue `@typescript-eslint/no-misused-promises`. Em React, cuidado ao embutir um handler extraído dentro do JSX: uma arrow inline nova a cada render invalida `React.memo` do componente filho.

---

## 3. Extract Variable
**PT-BR:** Extrair Variável · **Fonte:** https://refactoring.guru/extract-variable

### Problema
Uma expressão é difícil de entender: condição composta, cálculo aritmético longo, ou expressão que ocupa várias linhas.

### Solução
Coloque o resultado da expressão (ou de suas partes) em `const` com nomes autoexplicativos.

### Sinais no código (gatilhos)
- Condição de `if`/ternário com 3+ operandos ou mistura de `&&` e `||`.
- Expressão composta dentro do JSX: `{a && b && c && <Aviso />}` ou `disabled={...}` com duas linhas de lógica.
- `.reduce()`/`.filter()` encadeado direto dentro de uma prop ou de um template string.
- Expressão repetida no mesmo escopo.
- Comentário logo acima de um `if` explicando o que a condição significa.
- Optional chaining longo repetido (`pedido?.pagamento?.cartao?.bandeira`).

### Antes
```tsx
type Props = {
  readonly assinatura: Assinatura;
  readonly usuario: Usuario;
  readonly itens: readonly ItemCarrinho[];
  readonly onRenovar: () => void;
};

export function PainelAssinatura({ assinatura, usuario, itens, onRenovar }: Props) {
  return (
    <section>
      <h2>{assinatura.plano}</h2>
      {assinatura.status === 'ativa' &&
        assinatura.diasParaVencer <= 7 &&
        usuario.papel !== 'convidado' && <Aviso texto="Sua assinatura vence em breve" />}
      <button
        type="button"
        onClick={onRenovar}
        disabled={
          assinatura.status === 'cancelada' ||
          (assinatura.diasParaVencer > 30 && !usuario.permissoes.includes('billing'))
        }
      >
        Renovar por {(itens.reduce((s, i) => s + i.preco * i.quantidade, 0) / 100).toFixed(2)}
      </button>
    </section>
  );
}
```

### Depois
```tsx
export function PainelAssinatura({ assinatura, usuario, itens, onRenovar }: Props) {
  const estaAtiva = assinatura.status === 'ativa';
  const venceEmBreve = assinatura.diasParaVencer <= 7;
  const podeVerAvisos = usuario.papel !== 'convidado';
  const deveAvisarVencimento = estaAtiva && venceEmBreve && podeVerAvisos;

  const podeAdiantarRenovacao =
    assinatura.diasParaVencer <= 30 || usuario.permissoes.includes('billing');
  const renovacaoBloqueada = assinatura.status === 'cancelada' || !podeAdiantarRenovacao;

  // itens pode ter centenas de linhas: o memo evita refazer o reduce a cada render
  const totalCentavos = useMemo(
    () => itens.reduce((soma, item) => soma + item.preco * item.quantidade, 0),
    [itens],
  );

  return (
    <section>
      <h2>{assinatura.plano}</h2>
      {deveAvisarVencimento && <Aviso texto="Sua assinatura vence em breve" />}
      <button type="button" onClick={onRenovar} disabled={renovacaoBloqueada}>
        Renovar por {(totalCentavos / 100).toFixed(2)}
      </button>
    </section>
  );
}
```

### Passos
1. Antes da expressão, declare uma `const` e atribua a ela uma parte da expressão complexa.
2. Substitua essa parte pela nova variável; use o refactor "Extract to constant" do TS Server para não errar o escopo.
3. Repita para cada parte restante, nomeando pelo conceito de negócio (`venceEmBreve`, não `cond1`).
4. Use sempre `const`; se precisou de `let`, o caso é Split Temporary Variable.
5. Deixe o TS inferir o tipo, exceto quando o tipo explícito documenta a intenção (`const total: Centavos = ...`).
6. Em componente React, declare as `const` derivadas depois de todos os hooks e antes do `return` — nunca dentro do JSX.
7. Se a expressão é caro de calcular ou seu resultado é objeto/array consumido por filho memoizado, envolva com `useMemo` e liste as dependências reais.
8. Se a mesma `const` reaparece em outra função, promova com Replace Temp with Query ou mova para o módulo de domínio.

### Ganhos
- Legibilidade: a condição passa a ser lida como regra de negócio.
- Reduz a necessidade de comentários explicativos.
- Cada parte nomeada fica inspecionável no debugger e nos logs.
- O JSX volta a descrever estrutura, não lógica.
- Prepara Extract Method e Replace Temp with Query.

### Quando NÃO aplicar
- Quando quebra o short-circuit e o operando à direita é caro ou tem efeito: `if (temCache() || baixarCatalogo())` executa só o primeiro; extrair ambos para `const` força as duas chamadas.
- Quando o operando não é puro — extrair muda a ordem e a quantidade de execuções.
- Expressão já trivial (`itens.length > 0`): a variável só adiciona ruído.
- Extrair uma condição para fora de um type guard: `const ok = x !== null` não estreita o tipo de `x` no `if` seguinte (só o guard inline ou uma assertion function estreitam).

### Nota TypeScript
Dentro de um componente, toda `const` derivada é recalculada em **cada render** — isso é normal e barato para comparações e aritmética simples. `useMemo` só se justifica em dois casos: (a) o cálculo é realmente caro (reduce/sort/agrupamento sobre lista grande, parse, `Intl` recriado); (b) o resultado é objeto ou array que entra em props de componente `memo`, em dependência de `useEffect` ou em valor de Context — aí o que importa é a **igualdade referencial**, não o custo. Fora desses casos `useMemo` é ruído: adiciona um array de dependências para manter errado. O erro mais comum é o oposto: extrair para `const` uma expressão que dependia de short-circuit e passar a executar sempre o lado caro. Em booleanos, prefira nome com prefixo verbal (`pode`, `esta`, `tem`, `deve`) e extraia a forma positiva em vez de encadear `!`.

---

## 4. Inline Temp
**PT-BR:** Variável Temporária em Linha · **Fonte:** https://refactoring.guru/inline-temp

### Problema
Uma variável temporária recebe o resultado de uma expressão simples e nada mais — ela não acrescenta significado.

### Solução
Substitua as referências à variável pela própria expressão e remova a declaração.

### Sinais no código (gatilhos)
- `const x = outraCoisa.propriedade` usada uma única vez logo abaixo.
- Temporária cujo nome é sinônimo da expressão (`const total = itens.length`).
- `const resultado = { ... }` seguido imediatamente de `return resultado`.
- `const dados = await resposta.json()` usado uma vez só na linha seguinte.
- Temporária existindo apenas para caber na largura da linha.

### Antes
```ts
type Fatura = { readonly id: string; readonly valorCentavos: number; readonly pagoEm: string | null };

export class ResumoFaturasService {
  constructor(private readonly repo: FaturaRepository) {}

  async resumir(usuarioId: string): Promise<ResumoFaturas> {
    const faturas: readonly Fatura[] = await this.repo.listarPorUsuario(usuarioId);
    const abertas = faturas.filter((f) => f.pagoEm === null);
    const quantidadeAberta = abertas.length;
    const totalCentavos = abertas.reduce((soma, f) => soma + f.valorCentavos, 0);
    const totalEmReais = totalCentavos / 100;
    const inadimplente = quantidadeAberta > 3;
    const proxima = abertas.at(0);
    const proximaFaturaId = proxima?.id ?? null;
    const resumo: ResumoFaturas = { totalEmReais, inadimplente, proximaFaturaId };
    return resumo;
  }
}
```

### Depois
```ts
type Fatura = { readonly id: string; readonly valorCentavos: number; readonly pagoEm: string | null };

export class ResumoFaturasService {
  constructor(private readonly repo: FaturaRepository) {}

  async resumir(usuarioId: string): Promise<ResumoFaturas> {
    const faturas: readonly Fatura[] = await this.repo.listarPorUsuario(usuarioId);
    const abertas = faturas.filter((f) => f.pagoEm === null);
    const totalCentavos = abertas.reduce((soma, f) => soma + f.valorCentavos, 0);
    return {
      totalEmReais: totalCentavos / 100,
      inadimplente: abertas.length > 3,
      proximaFaturaId: abertas.at(0)?.id ?? null,
    };
  }
}
```

### Passos
1. Localize todos os usos da variável — deve ser um, ou poucos e triviais.
2. Substitua cada uso pela expressão atribuída a ela (o TS Server tem "Inline variable" para isso).
3. Apague a declaração.
4. Se a expressão embutida deixar a linha ilegível, reverta — a temporária estava documentando algo.
5. Ao embutir dentro da construção de um objeto, use a propriedade nomeada como documentação (`totalEmReais: totalCentavos / 100`).
6. Verifique que a expressão é pura: se tem efeito colateral e passou a ser avaliada mais de uma vez, não embuta.
7. Confira o narrowing: se a `const` estreitava um tipo `T | null`, embutir pode exigir `?.` ou `!` extra — nesse caso mantenha a `const`.

### Ganhos
- Remove ruído entre a intenção e o código.
- Encurta a função, expondo a estrutura real do fluxo.
- Menos nomes vivos no escopo, menos chance de reusar o errado.
- Passo preparatório para Replace Temp with Query e Extract Method.

### Quando NÃO aplicar
- A temporária faz cache de operação cara (query, `await`, parse de JSON grande) reutilizada em vários pontos — embutir multiplica o custo, e no caso do `await` multiplica requisições.
- A expressão não é pura (`iterator.next()`, `crypto.randomUUID()`, `Date.now()`, `Math.random()`).
- Narrowing dependente da `const`: `const u = mapa.get(id); if (u !== undefined) { u.nome }` — embutir perde o estreitamento (`noUncheckedIndexedAccess` torna isso frequente).
- O nome da temporária carrega a única explicação de um número mágico ou de uma unidade (`const pesoKg = gramas / 1000`).
- Em React, embutir uma `const` que é objeto/array usado como dependência de hook: a identidade passa a mudar a cada render.

### Nota TypeScript
TS/JS favorece o inline: propriedades nomeadas em object literal, `?.`, `??`, template strings e arrow com corpo de expressão absorvem a expressão sem perder clareza. O ponto de atenção real é o **control flow narrowing**: uma `const` sobre `T | undefined` mantém o tipo estreitado no bloco seguinte, e a expressão embutida não — com `strictNullChecks` + `noUncheckedIndexedAccess`, `arr[0]` e `map.get(k)` são `T | undefined` e o inline reintroduz `undefined`. O erro mais comum é embutir um `await`: `const dados = await buscar()` usado em três lugares vira três chamadas de rede. Para cache de valor caro, mantenha a `const` (ou use um getter com memo/`useMemo`), não o inline.

---

## 5. Replace Temp with Query
**PT-BR:** Substituir Variável Temporária por Consulta · **Fonte:** https://refactoring.guru/replace-temp-with-query

### Problema
O resultado de uma expressão é guardado em variável local para uso posterior, e a mesma expressão reaparece em outros métodos da classe (ou em outros componentes).

### Solução
Mova a expressão para um getter, uma propriedade calculada ou uma função pura, e consulte isso em vez de usar a variável.

### Sinais no código (gatilhos)
- A mesma expressão aparece como `const` local em dois ou mais métodos da mesma classe.
- A expressão depende só do estado do objeto, não de parâmetros.
- Temporária calculada no topo de uma função longa e usada em vários pontos.
- Regra de negócio (limiar de frete grátis, corte de desconto) reescrita em lugares diferentes.
- Dois componentes calculam o mesmo derivado a partir das mesmas props.

### Antes
```ts
type ItemCarrinho = { readonly produtoId: string; readonly precoCentavos: number; readonly quantidade: number };

export class Carrinho {
  constructor(
    private readonly itens: readonly ItemCarrinho[],
    private readonly cupom: string | null,
  ) {}

  totalCentavos(): number {
    const subtotal = this.itens.reduce((s, i) => s + i.precoCentavos * i.quantidade, 0);
    const temFreteGratis = subtotal >= 20_000;
    const desconto = this.cupom === 'BEMVINDO10' ? Math.round(subtotal * 0.1) : 0;
    return subtotal - desconto + (temFreteGratis ? 0 : 1_990);
  }

  rotulos(): readonly string[] {
    const subtotal = this.itens.reduce((s, i) => s + i.precoCentavos * i.quantidade, 0);
    const temFreteGratis = subtotal >= 20_000;
    const rotulos: string[] = [];
    if (temFreteGratis) rotulos.push('Frete grátis');
    if (subtotal > 100_000) rotulos.push('Pedido de alto valor');
    return rotulos;
  }
}
```

### Depois
```ts
type ItemCarrinho = { readonly produtoId: string; readonly precoCentavos: number; readonly quantidade: number };

const LIMITE_FRETE_GRATIS_CENTAVOS = 20_000;
const FRETE_PADRAO_CENTAVOS = 1_990;

export class Carrinho {
  constructor(
    private readonly itens: readonly ItemCarrinho[],
    private readonly cupom: string | null,
  ) {}

  get subtotalCentavos(): number {
    return this.itens.reduce((soma, item) => soma + item.precoCentavos * item.quantidade, 0);
  }

  get temFreteGratis(): boolean {
    return this.subtotalCentavos >= LIMITE_FRETE_GRATIS_CENTAVOS;
  }

  get descontoCentavos(): number {
    return this.cupom === 'BEMVINDO10' ? Math.round(this.subtotalCentavos * 0.1) : 0;
  }

  totalCentavos(): number {
    const frete = this.temFreteGratis ? 0 : FRETE_PADRAO_CENTAVOS;
    return this.subtotalCentavos - this.descontoCentavos + frete;
  }

  rotulos(): readonly string[] {
    return [
      ...(this.temFreteGratis ? ['Frete grátis'] : []),
      ...(this.subtotalCentavos > 100_000 ? ['Pedido de alto valor'] : []),
    ];
  }
}
```

### Passos
1. Confirme que a variável recebe valor exatamente uma vez. Se não, aplique Split Temporary Variable primeiro.
2. Extraia a expressão para um getter (`get x(): T`) ou função pura no módulo, usando Extract Method.
3. Garanta que a consulta é pura: só lê estado e retorna valor — sem mutar, sem log, sem I/O, sem `async`.
4. Substitua todos os usos da variável pela consulta e apague a declaração; deixe o compilador achar os usos remanescentes.
5. Repita nos outros métodos que continham a expressão duplicada.
6. Se a consulta precisa de dados externos (HTTP, banco), ela não pertence à entidade — mova para um service que recebe o dado pronto.
7. Em React, o equivalente é uma função pura no módulo chamada no corpo do componente; só troque por `useMemo` se o custo ou a identidade referencial exigirem.

### Ganhos
- `descontoCentavos` é mais claro que `subtotal * 0.1`: o nome expõe a regra.
- Elimina duplicação da expressão; a regra passa a ter um único ponto de alteração.
- Encurta funções longas, viabilizando Extract Method.
- A regra fica testável isoladamente, sem montar o fluxo inteiro.

### Quando NÃO aplicar
- A expressão é caro de calcular e é consultada dentro de laço ou a cada render — getter recalcula em **todo** acesso (note que `totalCentavos` acima lê `subtotalCentavos` três vezes indiretamente).
- O cálculo depende de I/O: getter `async` não existe de forma honesta, e getter que dispara `fetch` esconde trabalho e falha.
- O valor precisa ser estável em um snapshot (preço congelado no fechamento do pedido) — isso é dado persistido, não consulta.
- Getter com efeito colateral (log, contador, preenchimento de cache observável) — quebra a expectativa de leitura barata e idempotente.
- O valor deriva de estado externo que muda no meio do render (React exige leitura consistente — use `useSyncExternalStore`).

### Nota TypeScript
A consulta idiomática é um `get x()` na classe ou uma função pura `calcularX(entrada)` no módulo. Getters não aparecem em `JSON.stringify` nem em spread (`{ ...carrinho }` perde `subtotalCentavos`) — se o valor precisa atravessar a fronteira serializada, calcule-o explicitamente no mapeamento para o DTO. Para valor caro em objeto de vida longa, use memo por closure ou cache em campo `#priv` preenchido na primeira leitura; em componente, `useMemo`. O erro mais comum é transformar a temporária num getter que faz I/O ou mutação: um `get pedidos()` que chama `fetch` (ou um `useMemo` cujo callback dispara requisição) roda em momentos imprevisíveis, quebra em StrictMode e é impossível de cancelar. Consulta lê; quem busca é `async` e explícito.

---

## 6. Split Temporary Variable
**PT-BR:** Dividir Variável Temporária · **Fonte:** https://refactoring.guru/split-temporary-variable

### Problema
Uma variável local armazena valores intermediários diferentes ao longo da função (exceto variáveis de laço), acumulando propósitos distintos.

### Solução
Use variáveis diferentes para valores diferentes. Cada variável responde por uma única coisa.

### Sinais no código (gatilhos)
- `let` reatribuído com significado diferente da primeira atribuição.
- Nomes genéricos: `temp`, `valor`, `aux`, `result`, `data`, `x`.
- `let` declarado no topo da função e atribuído em pontos distantes.
- Mapper DTO→domínio que reaproveita a mesma `let` para normalizar campos diferentes.
- O tipo da variável muda de papel (era `string` bruta, virou `string` normalizada) sem mudar de nome.

### Antes
```ts
type UsuarioDto = {
  readonly nome_completo: string;
  readonly documento: string;
  readonly limite_credito_centavos: number;
  readonly peso_cesta_gramas: number;
  readonly email: string | null;
};

export function paraUsuario(dto: UsuarioDto): Usuario {
  let temp = dto.nome_completo.trim();
  const nome = temp.length === 0 ? 'Usuário sem nome' : temp;

  temp = dto.documento.replace(/\D/g, '');
  const cpf = temp.length === 11 ? temp : '';

  let valor = dto.limite_credito_centavos / 100;
  const limite = valor;

  valor = dto.peso_cesta_gramas / 1_000;
  const peso = valor;

  return { nome, cpf, limiteReais: limite, pesoCestaKg: peso, email: dto.email?.trim() ?? '' };
}
```

### Depois
```ts
const CPF_DIGITOS = 11;

export function paraUsuario(dto: UsuarioDto): Usuario {
  const nomeNormalizado = dto.nome_completo.trim();
  const nome = nomeNormalizado.length === 0 ? 'Usuário sem nome' : nomeNormalizado;
  const documentoApenasDigitos = dto.documento.replace(/\D/g, '');
  const cpf = documentoApenasDigitos.length === CPF_DIGITOS ? documentoApenasDigitos : '';
  const limiteReais = dto.limite_credito_centavos / 100;
  const pesoCestaKg = dto.peso_cesta_gramas / 1_000;

  return {
    nome,
    cpf,
    limiteReais,
    pesoCestaKg,
    email: dto.email?.trim() ?? '',
  };
}
```

### Passos
1. Encontre a primeira atribuição e renomeie a variável para refletir aquele valor específico (rename do TS Server, F2).
2. Substitua os usos que se referem a esse valor pelo novo nome.
3. Repita para cada atribuição subsequente, criando uma variável por valor.
4. Troque cada `let` resultante por `const`; se alguma não aceitar `const`, é acumulador legítimo ou pede outro refactoring.
5. Deixe o compilador reclamar dos usos remanescentes — cada erro marca um ponto onde a semântica estava sobreposta.
6. Extraia números mágicos que apareceram junto com os nomes (`CPF_DIGITOS`).
7. Depois da divisão, reavalie Inline Temp (temporárias que ficaram triviais) e Extract Method (blocos agora independentes).

### Ganhos
- Cada elemento do código responde por uma única coisa — manutenção mais segura.
- Nomes descritivos (`documentoApenasDigitos`) substituem `temp`/`valor`.
- `const` elimina a classe de bug "quem alterou isso no meio da função".
- Prepara Extract Method, porque cada valor tem escopo claro.

### Quando NÃO aplicar
- Variável de laço/índice ou acumulador genuíno (`let somaCentavos`), cujo propósito é acumular.
- Máquina de estado onde a variável representa o estado corrente por design (use uma discriminated union, não `string`).
- Reatribuição em loop de retentativa (`let tentativa`), que é o mesmo conceito evoluindo.
- Variável que evolui um único valor por etapas de pipeline (`let precoAtual` sendo descontado) — é um conceito só.

### Nota TypeScript
`const` por padrão previne a maior parte dos casos: com `prefer-const` ligado, um `let` no código já é um sinal para revisar. Atenção ao que `const` **não** garante: `const carrinho = { itens: [] }` continua mutável por dentro — imutabilidade real vem de `readonly`, `ReadonlyArray<T>` e `as const`. A divisão normalmente termina em encadeamento sem variável intermediária (`dto.nome_completo.trim() || 'Usuário sem nome'`) ou em desestruturação com rename (`const { nome_completo: nomeBruto } = dto`). Acumuladores desaparecem com `reduce`, `flatMap` e `Object.groupBy`. O erro mais comum em React é o inverso da técnica: reaproveitar um `useState` para dois papéis (`const [valor, setValor]` guardando ora o rascunho, ora o resultado) — dois papéis pedem dois estados, ou um `useReducer` com union discriminada.

---

## 7. Remove Assignments to Parameters
**PT-BR:** Remover Atribuições a Parâmetros · **Fonte:** https://refactoring.guru/remove-assignments-to-parameters

### Problema
Um valor é atribuído a um parâmetro dentro do corpo da função. Fica ambíguo o que o parâmetro contém em cada ponto — e se o parâmetro é objeto ou array, mutá-lo altera o dado do chamador.

### Solução
Use uma variável local em vez do parâmetro. Para objetos e arrays recebidos, derive uma cópia (spread) em vez de mutar a original.

### Sinais no código (gatilhos)
- Atribuição direta ao nome do parâmetro (`preco = 0`, `nome = nome.trim()`).
- Chamadas mutantes sobre parâmetro array: `push`, `pop`, `splice`, `sort`, `reverse`, `fill`.
- Atribuição de propriedade em parâmetro objeto (`usuario.nome = ...`, `Object.assign(config, ...)`).
- Função retorna um valor **e** altera a entrada — dupla saída não documentada.
- Testes que precisam recriar o array de entrada antes de cada `expect`.
- Em React, mutar um array/objeto vindo de props ou de estado antes de passá-lo para `setState`.

### Antes
```ts
type Cupom = { readonly codigo: string; readonly percentual: number };

export function aplicarCupons(
  precoCentavos: number,
  cupons: Cupom[],
  historico: string[],
): number {
  if (precoCentavos < 0) precoCentavos = 0;

  cupons.sort((a, b) => b.percentual - a.percentual);

  for (const cupom of cupons) {
    if (historico.includes(cupom.codigo)) continue;
    precoCentavos = precoCentavos - Math.round(precoCentavos * cupom.percentual);
    historico.push(cupom.codigo);
  }

  if (precoCentavos < 500) precoCentavos = 500;
  return precoCentavos;
}
```

### Depois
```ts
type Cupom = { readonly codigo: string; readonly percentual: number };
type ResultadoCupons = { readonly precoCentavos: number; readonly historico: readonly string[] };

const PRECO_MINIMO_CENTAVOS = 500;

export function aplicarCupons(
  precoCentavos: number,
  cupons: readonly Cupom[],
  historico: readonly string[],
): ResultadoCupons {
  const doMaiorParaOMenor = [...cupons].sort((a, b) => b.percentual - a.percentual);
  const jaUsados = new Set(historico);
  const aplicados: string[] = [];
  let precoAtual = Math.max(precoCentavos, 0);

  for (const cupom of doMaiorParaOMenor) {
    if (jaUsados.has(cupom.codigo)) continue;
    precoAtual -= Math.round(precoAtual * cupom.percentual);
    jaUsados.add(cupom.codigo); // mesmo código repetido na lista não conta duas vezes
    aplicados.push(cupom.codigo);
  }

  return {
    precoCentavos: Math.max(precoAtual, PRECO_MINIMO_CENTAVOS),
    historico: [...historico, ...aplicados],
  };
}
```

### Passos
1. Declare uma local com nome próprio para o valor que evolui (`let precoAtual`), inicializada a partir do parâmetro, e pare de atribuir ao parâmetro.
2. Substitua os usos posteriores do parâmetro pela local; deixe o compilador achar os que sobraram.
3. Tipe os parâmetros de entrada como somente leitura: `readonly T[]` / `ReadonlyArray<T>`, `Readonly<Config>`. O compilador passa a **proibir** `push`/`sort`/atribuição de propriedade.
4. Troque cada operação destrutiva pela versão que devolve novo valor: `sort` → `[...arr].sort(...)` (ou `toSorted`), `push` → `[...arr, x]`, `splice`/`filter` in place → `filter`, `obj.campo = v` → `{ ...obj, campo: v }`.
5. Faça a função retornar explicitamente tudo que produz — nenhuma saída via parâmetro. Se são dois valores, retorne um objeto tipado.
6. Atualize os chamadores e remova as cópias defensivas que existiam por causa da mutação.
7. Se a mutação era por performance, meça antes de reintroduzi-la; se necessária, mute apenas uma coleção **criada localmente**.

### Ganhos
- Sem efeitos colaterais no chamador: o mesmo array pode ser reusado com segurança.
- Função pura e determinística — testável sem setup nem reconstrução de fixtures.
- O nome de cada variável volta a significar uma coisa só.
- Em React, evita o bug clássico de mutar estado e a UI não atualizar.
- Permite extrair as etapas em funções próprias.

### Quando NÃO aplicar
- Parâmetro default (`function f(pagina = 1)`) — isso não é atribuição a parâmetro, é valor padrão.
- Normalização de argumento na primeira linha de uma função pequena, sobre **primitivo**, quando o nome continua significando a mesma coisa: `precoCentavos = Math.max(precoCentavos, 0)` é aceitável; o problema é o parâmetro trocar de papel no meio.
- Coleção enorme em hot path onde cada cópia importa: crie o buffer localmente e mute-o ali, sem expor.
- Builders cuja API é intencionalmente mutável e documentada (acumulador de query, `URLSearchParams`, `Headers`).
- Estruturas de terceiros que exigem mutação in place por contrato.

### Nota TypeScript
Diferente de Kotlin, onde parâmetros são `val` e a atribuição nem compila, em TS/JS **todo parâmetro é reatribuível** — a técnica clássica continua valendo integralmente, e o lint (`no-param-reassign`) existe justamente por isso. O caso perigoso, porém, é o outro: parâmetros de objeto e array chegam por **referência**, então `cupons.sort(...)` reordena o array do chamador e `historico.push(...)` grava no array dele; se esse array veio de estado do React ou de um cache em memória, você corrompeu dado compartilhado sem nenhuma pista no ponto da chamada. A defesa é estrutural: tipar entradas como `readonly`/`ReadonlyArray<T>`/`Readonly<T>` (o compilador barra a mutação) e derivar com spread ou com os métodos não destrutivos `toSorted`/`toSpliced`/`with`. Note que `readonly` é apenas superficial: para objeto aninhado, copie o nível que muda (`{ ...pedido, endereco: { ...pedido.endereco, cep } }`). O erro mais comum é achar que `const` protege: `const itens = props.itens; itens.push(x)` compila e muta a props do pai.

---

## 8. Replace Method with Method Object
**PT-BR:** Substituir Método por Objeto-Método · **Fonte:** https://refactoring.guru/replace-method-with-method-object

### Problema
Uma função longa tem variáveis locais tão entrelaçadas que não é possível aplicar Extract Method sem passar meia dúzia de parâmetros.

### Solução
Transforme a função em uma classe própria, onde as locais viram propriedades — ou em uma closure/factory que captura o contexto. Depois divida o algoritmo em métodos pequenos dessa unidade.

### Sinais no código (gatilhos)
- Extrair qualquer bloco exigiria 4+ parâmetros e/ou retorno de vários valores.
- Função com 30+ linhas e 5+ locais que se referenciam mutuamente.
- Vários `Map`/`Set`/array locais alimentados em fases sequenciais.
- Algoritmo em etapas (coletar → ponderar → distribuir → completar) num único corpo.
- A função quase não usa a classe/módulo onde está — é um algoritmo à parte.
- Um `useReducer` cujo reducer virou uma função de 60 linhas com quatro fases.

### Antes
```ts
export class GeradorDeVitrine {
  constructor(private readonly repo: CatalogoRepository) {}

  async gerar(usuarioId: string, config: ConfigVitrine): Promise<Vitrine> {
    const historico = await this.repo.pedidosDoUsuario(usuarioId);
    const catalogo = await this.repo.produtosPorCategorias(config.categorias);
    const pesoPorCategoria = new Map<string, number>();
    let totalPeso = 0;
    for (const categoria of config.categorias) {
      const compras = historico.filter((p) => p.categoria === categoria).length;
      pesoPorCategoria.set(categoria, compras);
      totalPeso += compras;
    }
    const cotas = new Map<string, number>();
    let distribuidas = 0;
    for (const categoria of config.categorias) {
      const fracao = totalPeso === 0
        ? 1 / config.categorias.length
        : (pesoPorCategoria.get(categoria) ?? 0) / totalPeso;
      const cota = Math.floor(config.totalDeSlots * fracao);
      cotas.set(categoria, cota);
      distribuidas += cota;
    }
    const selecionados: Produto[] = [];
    for (const [categoria, cota] of cotas) {
      selecionados.push(...catalogo.filter((p) => p.categoria === categoria).slice(0, cota));
    }
    const escolhidos = new Set(selecionados.map((p) => p.id));
    const sobra = config.totalDeSlots - distribuidas;
    const complemento = catalogo.filter((p) => !escolhidos.has(p.id)).slice(0, sobra);
    return { usuarioId, produtos: [...selecionados, ...complemento], config };
  }
}
```

### Depois
```ts
export class GeradorDeVitrine {
  constructor(private readonly repo: CatalogoRepository) {}

  async gerar(usuarioId: string, config: ConfigVitrine): Promise<Vitrine> {
    const historico = await this.repo.pedidosDoUsuario(usuarioId);
    const catalogo = await this.repo.produtosPorCategorias(config.categorias);
    return new MontagemDeVitrine(config, historico, catalogo).montar(usuarioId);
  }
}

class MontagemDeVitrine {
  private readonly pesoPorCategoria: ReadonlyMap<string, number>;
  private readonly totalPeso: number;

  constructor(private readonly config: ConfigVitrine, historico: readonly Pedido[], private readonly catalogo: readonly Produto[]) {
    this.pesoPorCategoria = new Map(config.categorias.map((c) => [c, historico.filter((p) => p.categoria === c).length]));
    this.totalPeso = [...this.pesoPorCategoria.values()].reduce((a, b) => a + b, 0);
  }

  private cotaDe(categoria: string): number {
    const peso = this.pesoPorCategoria.get(categoria) ?? 0;
    const fracao = this.totalPeso === 0 ? 1 / this.config.categorias.length : peso / this.totalPeso;
    return Math.floor(this.config.totalDeSlots * fracao);
  }

  montar(usuarioId: string): Vitrine {
    const selecionados = this.config.categorias.flatMap((c) =>
      this.catalogo.filter((p) => p.categoria === c).slice(0, this.cotaDe(c)));
    const escolhidos = new Set(selecionados.map((p) => p.id));
    const distribuidas = this.config.categorias.reduce((t, c) => t + this.cotaDe(c), 0);
    const sobra = this.config.totalDeSlots - distribuidas;
    const complemento = this.catalogo.filter((p) => !escolhidos.has(p.id)).slice(0, sobra);
    return { usuarioId, produtos: [...selecionados, ...complemento], config: this.config };
  }
}
```

### Passos
1. Nomeie a nova unidade pelo propósito do algoritmo (`MontagemDeVitrine`), não pela função (`GerarVitrineHelper`).
2. Receba no construtor os **dados prontos** e as dependências necessárias — não uma referência à classe original inteira.
3. Transforme cada local de fase 1 em `private readonly` calculada no construtor; mantenha as entradas `readonly`.
4. Não exporte a classe se ela é detalhe interno: a fronteira do módulo já é o encapsulamento.
5. Mova o corpo original para o método principal (`montar`, `executar`), trocando locais por propriedades.
6. Quebre o corpo em métodos `private` pequenos — agora eles compartilham estado sem passar parâmetros.
7. Substitua o corpo da função original pela instanciação + chamada do método principal.
8. Rode os testes existentes contra a API pública antes de refinar a divisão interna.

### Ganhos
- Isola um algoritmo que crescia sem controle, sem poluir a classe original.
- Permite dividir em submétodos que não fariam sentido como membros da classe hospedeira.
- Torna cada etapa nomeável e testável.
- Uma instância por execução elimina estado compartilhado entre chamadas concorrentes.

### Quando NÃO aplicar
- A função é longa mas linear: Extract Method resolve com 1-2 parâmetros — não crie classe.
- Não vale para funções de até ~20 linhas: a classe extra é complexidade líquida.
- O "objeto" acabaria sem estado nenhum — nesse caso são funções puras num módulo.
- Instanciação por item em render de lista virtualizada ou por evento de alta frequência: meça antes.
- Objeto-método com estado mutável reusado entre chamadas (singleton de módulo): volta a ser a mesma armadilha, agora com escopo maior.

### Nota TypeScript
Há duas saídas idiomáticas. A classe com dependências no construtor (acima) é a melhor quando o algoritmo tem várias fases nomeadas e você quer cada uma testável; `private readonly` no parâmetro do construtor evita a repetição de `this.x = x`. A alternativa é uma **factory de closure**: `export function criarMontagemDeVitrine(config, historico, catalogo) { const pesoPorCategoria = ...; function cotaDe(c) { ... }; return { montar }; }` — as locais viram variáveis capturadas, as fases viram funções internas, e nada disso escapa porque só `montar` é retornado. Escolha closure quando não houver herança nem necessidade de `instanceof`, e classe quando quiser injeção de dependência explícita e mocks por interface. O erro mais comum é dar ao objeto-método uma vida mais longa que a execução: instância guardada em variável de módulo, ou hook que cria o objeto fora de `useMemo` e acumula estado entre renders. Uma execução, uma instância — e nada de I/O no construtor, que não pode ser `async`; busque os dados antes e injete-os prontos.

---

## 9. Substitute Algorithm
**PT-BR:** Substituir Algoritmo · **Fonte:** https://refactoring.guru/substitute-algorithm

### Problema
O algoritmo atual está confuso demais para melhorar aos poucos, já existe implementação equivalente na plataforma/biblioteca, ou os requisitos mudaram e ele não é mais salvável.

### Solução
Substitua o corpo da função pelo novo algoritmo, mantendo a mesma assinatura e o mesmo contrato observável.

### Sinais no código (gatilhos)
- `while`/`for` com índices manuais onde `filter`/`map`/`flatMap`/`Object.groupBy` resolvem.
- Deduplicação, ordenação ou busca implementadas à mão (flag `duplicado`, laço aninhado).
- `includes` sobre array dentro de laço — O(n²) que um `Set` torna O(n).
- Agrupamento com objeto literal e checagem de chave existente em vez de `Map`.
- Parsing de data ou de JSON artesanal em vez de `Intl`/`Temporal`/Zod.
- Debounce, throttle ou retry reimplementados com contadores em vez de `AbortSignal` + utilitário testado.

### Antes
```ts
type Produto = { readonly id: string; readonly nome: string; readonly categoria: string };

export function buscarProdutos(
  todos: readonly Produto[],
  termo: string,
  categorias: readonly string[],
): readonly Produto[] {
  const resultado: Produto[] = [];
  let i = 0;
  while (i < todos.length) {
    const produto = todos[i];
    if (produto !== undefined) {
      let casaCategoria = false;
      let j = 0;
      while (j < categorias.length) {
        if (categorias[j] === produto.categoria) casaCategoria = true;
        j++;
      }
      let casaTermo = false;
      if (produto.nome.toLowerCase().indexOf(termo.toLowerCase()) >= 0) casaTermo = true;
      if (casaCategoria && casaTermo) {
        let duplicado = false;
        for (const r of resultado) if (r.id === produto.id) duplicado = true;
        if (!duplicado) resultado.push(produto);
      }
    }
    i++;
  }
  return resultado;
}
```

### Depois
```ts
type Produto = { readonly id: string; readonly nome: string; readonly categoria: string };

export function buscarProdutos(
  todos: readonly Produto[],
  termo: string,
  categorias: readonly string[],
): readonly Produto[] {
  const categoriasBuscadas = new Set(categorias);
  const termoNormalizado = termo.toLowerCase();
  const porId = new Map<string, Produto>();

  for (const produto of todos) {
    const casaCategoria = categoriasBuscadas.has(produto.categoria);
    const casaTermo = produto.nome.toLowerCase().includes(termoNormalizado);
    if (casaCategoria && casaTermo && !porId.has(produto.id)) {
      porId.set(produto.id, produto);
    }
  }

  return [...porId.values()];
}
```

### Passos
1. Garanta que existe suíte cobrindo o contrato observável: entradas típicas, lista vazia, duplicatas, acentuação, ordem, limites. Sem testes, não substitua.
2. Simplifique o algoritmo antigo o quanto der com Extract Method — isso revela o contrato real, inclusive as bordas não documentadas.
3. Escreva o novo algoritmo em uma função nova, mantendo a assinatura pública intacta.
4. Troque a chamada e rode os testes; compare saídas caso a caso se divergirem (ordem, empates, `undefined`, case/acento).
5. Confirme as características não funcionais que importam: complexidade, alocações, se ainda é síncrono, se ainda é cancelável.
6. Verifique o alvo de compilação: `Object.groupBy`, `toSorted`, `Array.prototype.at` exigem `lib`/runtime recentes.
7. Com tudo verde, apague o algoritmo antigo — não deixe as duas versões atrás de uma flag permanente.

### Ganhos
- Elimina complexidade acidental de uma vez, em vez de dezenas de micro-refatorações.
- Substitui código próprio por implementação de plataforma/biblioteca já testada.
- Costuma corrigir bugs latentes de borda que o laço manual escondia.
- Melhora a complexidade real (`Set.has` é O(1) contra `Array.includes` O(n)).
- Reduz a superfície para `noUncheckedIndexedAccess` reclamar: sem índices, sem `undefined` espúrio.

### Quando NÃO aplicar
- Sem testes que fixem o comportamento atual: substituir é reescrever no escuro.
- O algoritmo antigo codifica regra de negócio ou compatibilidade legada não documentada (arredondamento de desconto, critério de desempate no ranking).
- O novo algoritmo depende de biblioteca que pesa no bundle ou exige runtime acima do suportado.
- Trecho crítico de performance onde a versão idiomática aloca mais arrays intermediários: meça com benchmark antes de trocar.
- Substituição sem paridade de contrato: mudar estabilidade da ordenação ou o tratamento de `null`/`undefined` é mudança de comportamento, não refactoring.

### Nota TypeScript
A maioria das substituições é troca de laço manual pela plataforma: `filter`, `map`, `flatMap`, `some`, `every`, `Object.groupBy`, `Map`/`Set` para índice e dedupe, `toSorted`/`toSpliced` para ordenar sem mutar, `Intl.NumberFormat`/`Intl.DateTimeFormat` para formatação, Zod na fronteira em vez de validação manual. Dois pontos específicos de TS: (a) dedupe por chave se resolve com `new Map(itens.map((i) => [i.id, i]))`, mas isso mantém a **última** ocorrência, enquanto o laço com flag mantinha a primeira — é diferença de contrato, escolha consciente (o exemplo acima preserva a primeira); (b) trocar índice por `for...of` costuma remover metade dos `undefined` que `noUncheckedIndexedAccess` obriga a tratar. O erro mais comum é confundir substituição de algoritmo com mudança de semântica: `sort` sem comparador ordena por string (`[2, 10]` vira `[10, 2]`), `filter(Boolean)` derruba `0` e `''` junto com `null`, e `JSON.parse` sem validação não é equivalente a um parser que rejeitava payload inválido. Troque o "como", nunca o "o quê".
