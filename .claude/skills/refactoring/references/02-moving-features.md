# Moving Features between Objects — Movendo Recursos entre Objetos

> Grupo 2 de 6 · 8 técnicas · Fonte: refactoring.guru/refactoring/techniques

## Quando este grupo se aplica

- Uma função usa mais dados de outro objeto do que do módulo onde vive (**Feature Envy**): o cálculo de total do pedido mora no componente ou no controller, não no módulo de domínio.
- Dois módulos se conhecem por dentro — leem os mesmos campos, replicam a mesma invariante, mudam sempre no mesmo commit (**Inappropriate Intimacy**).
- Cadeias de navegação no código cliente: `pedido.cliente.endereco.cidade.nome` dentro de componentes, hooks ou use cases (**Message Chain**).
- Uma camada só repassa chamadas, sem regra própria — repositório cujo corpo é `buscar(id) { return this.http.buscar(id); }` linha por linha (**Middle Man**).
- Componente, service ou entidade com responsabilidades misturadas: fetch + regra de preço + formatação + estado de UI no mesmo arquivo (**Large Class / God Component**).
- Você precisa de um comportamento em um tipo que não controla (`Date`, tipo gerado por OpenAPI, classe de um SDK) e a lógica vaza duplicada para os clientes.

## Índice

1. [Move Method](#1-move-method)
2. [Move Field](#2-move-field)
3. [Extract Class](#3-extract-class)
4. [Inline Class](#4-inline-class)
5. [Hide Delegate](#5-hide-delegate)
6. [Remove Middle Man](#6-remove-middle-man)
7. [Introduce Foreign Method](#7-introduce-foreign-method)
8. [Introduce Local Extension](#8-introduce-local-extension)

---

## 1. Move Method
**PT-BR:** Mover Método · **Fonte:** https://refactoring.guru/move-method

### Problema
Um método é mais usado por outro objeto do que por aquele onde está declarado. Ele lê dados alheios e quase nada do próprio dono.

### Solução
Declare o comportamento junto dos dados que ele usa — no módulo de domínio ou na classe que detém o estado —, mova o corpo e transforme o original em delegação até poder apagá-lo, atualizando os chamadores.

### Sinais no código (gatilhos)
- O corpo referencia `pedido.itens`, `pedido.desconto`, `pedido.frete` e nenhum `this`.
- Método `private` de uma classe cujo único parâmetro é uma entidade e que nunca toca nas dependências do construtor.
- Componente React ou handler de rota calculando regra de negócio direto do payload.
- Mudar um campo do `type Pedido` quebra arquivos em `ui/` e `controllers/`, não em `domain/`.
- `helpers.ts` com várias funções cujo primeiro parâmetro é sempre o mesmo tipo *seu* (se o tipo fosse de terceiros, seria a seção 7).

### Antes
```ts
export type ItemPedido = {
  readonly sku: string;
  readonly quantidade: number;
  readonly precoUnitarioCents: number;
};

export type Pedido = {
  readonly id: string;
  readonly itens: ReadonlyArray<ItemPedido>;
  readonly descontoCents: number;
  readonly freteCents: number;
};

// checkout/CheckoutController.ts
export class CheckoutController {
  constructor(private readonly pagamentos: GatewayPagamentos) {}

  // Feature Envy: só interroga Pedido, nunca usa `this.pagamentos`
  private subtotalCents(pedido: Pedido): number {
    return pedido.itens.reduce((acc, i) => acc + i.quantidade * i.precoUnitarioCents, 0);
  }

  private totalCents(pedido: Pedido): number {
    const bruto = this.subtotalCents(pedido) - pedido.descontoCents;
    return Math.max(0, bruto) + pedido.freteCents;
  }

  async cobrar(pedido: Pedido): Promise<string> {
    return this.pagamentos.cobrar(pedido.id, this.totalCents(pedido));
  }
}
```

### Depois
```ts
// domain/pedido.ts — o comportamento passa a morar junto dos dados
export type ItemPedido = {
  readonly sku: string;
  readonly quantidade: number;
  readonly precoUnitarioCents: number;
};

export type Pedido = {
  readonly id: string;
  readonly itens: ReadonlyArray<ItemPedido>;
  readonly descontoCents: number;
  readonly freteCents: number;
};

export function subtotalCents(pedido: Pedido): number {
  return pedido.itens.reduce((acc, i) => acc + i.quantidade * i.precoUnitarioCents, 0);
}

export function totalCents(pedido: Pedido): number {
  const bruto = subtotalCents(pedido) - pedido.descontoCents;
  return Math.max(0, bruto) + pedido.freteCents;
}

export function temFreteGratis(pedido: Pedido, limiteCents = 20_000): boolean {
  return subtotalCents(pedido) >= limiteCents;
}

// checkout/CheckoutController.ts — volta a ser só orquestração
export class CheckoutController {
  constructor(private readonly pagamentos: GatewayPagamentos) {}

  async cobrar(pedido: Pedido): Promise<string> {
    return this.pagamentos.cobrar(pedido.id, totalCents(pedido));
  }
}
```

### Passos
1. Liste o que a função usa do dono atual (campos, outras funções, dependências injetadas). Se algo é usado só por ela, planeje mover junto.
2. Se for método de classe, confirme que não é parte de um contrato implementado (`implements`, override em subclasse) — se for, mova a declaração inteira ou pare.
3. Crie a função exportada no módulo de domínio do tipo (`domain/pedido.ts`), com a entidade como primeiro parâmetro; ou o método na classe que detém o estado, e aí o parâmetro vira `this`.
4. Verifique o caminho de acesso: o destino precisa alcançar tudo o que a função lê, sem importar de volta a origem (ciclo de import).
5. Transforme a função antiga em uma linha delegando para a nova e rode os testes — ainda ninguém precisa mudar.
6. Apague a antiga e deixe `tsc --noEmit` apontar cada uso; corrija import por import. Se o módulo de origem ficou só com repasse, aplique **Inline Class**.

### Ganhos
- Aumenta coesão: comportamento fica junto dos dados que ele usa.
- Reduz acoplamento e a superfície pública que a origem precisava expor.
- A regra fica testável sem instanciar o controller nem renderizar a tela.

### Quando NÃO aplicar
- A função coordena vários objetos (orquestração) — o lugar dela é no service/use case, não na entidade.
- O comportamento depende de infraestrutura (`fetch`, repositório, logger, `localStorage`): mover para o domínio contamina a camada pura.
- O destino é um DTO validado na fronteira (schema Zod, tipo gerado do OpenAPI): pôr regra lá acopla o domínio ao contrato externo.
- A função é usada igualmente por 3+ módulos — provavelmente é uma política própria; extraia um módulo/classe (seção 3).

### Nota TypeScript
Em TS existem três destinos possíveis: método de classe, getter, ou **função exportada em módulo de domínio**. Com `type` + funções puras (o desenho mais comum), o módulo ganha: é tree-shakeable, testável sem instanciar nada e sobrevive ao ciclo de vida dos dados. Esse último ponto é decisivo — objeto que veio de `JSON.parse`, de `structuredClone`, de um cache ou do estado serializado do cliente é *plain object*: não tem método nenhum, mesmo que o tipo diga que tem. Preferir funções evita a reidratação manual (`Object.assign(Object.create(Pedido.prototype), json)`), que é frágil e silenciosamente errada quando o payload muda.

---

## 2. Move Field
**PT-BR:** Mover Campo · **Fonte:** https://refactoring.guru/move-field

### Problema
Um campo é lido e escrito mais por outro tipo do que pelo seu. Ele simplesmente está no lugar errado.

### Solução
Declare o campo no tipo que mais o usa, redirecione todos os acessos e remova o original.

### Sinais no código (gatilhos)
- O campo aparece quase sempre em expressões junto de campos de outro objeto (`assinatura.limiteDeUsuarios` sempre ao lado de `assinatura.plano.nome`).
- Ele descreve uma regra do outro conceito (parâmetro do plano guardado na assinatura).
- Nenhuma função do módulo dono usa o campo, ou só uma usa — e essa função também vai embora.
- Ele é copiado do outro objeto no mapper DTO→domínio e nunca mais muda.
- Surgiu como resíduo de um **Extract Class** anterior.

### Antes
```ts
export type Plano = {
  readonly id: string;
  readonly nome: string;
  readonly precoMensalCents: number;
};

export type Assinatura = {
  readonly id: string;
  readonly plano: Plano;
  // Inappropriate Intimacy: regra do plano guardada na assinatura
  readonly limiteDeUsuarios: number;
};

export function custoMensalCents(assinatura: Assinatura, usuariosAtivos: number): number {
  return assinatura.plano.precoMensalCents * usuariosAtivos;
}

export class ConvitesService {
  podeConvidar(assinatura: Assinatura, usuariosAtivos: number): boolean {
    return usuariosAtivos < assinatura.limiteDeUsuarios;
  }

  textoLimite(assinatura: Assinatura): string {
    return `Plano ${assinatura.plano.nome} permite ${assinatura.limiteDeUsuarios} usuários`;
  }
}
```

### Depois
```ts
export type Plano = {
  readonly id: string;
  readonly nome: string;
  readonly precoMensalCents: number;
  readonly limiteDeUsuarios: number;
};

export type Assinatura = {
  readonly id: string;
  readonly plano: Plano;
};

export function cabeMaisUm(plano: Plano, usuariosAtivos: number): boolean {
  return usuariosAtivos < plano.limiteDeUsuarios;
}

export function custoMensalCents(plano: Plano, usuariosAtivos: number): number {
  return plano.precoMensalCents * usuariosAtivos;
}

export class ConvitesService {
  podeConvidar(assinatura: Assinatura, usuariosAtivos: number): boolean {
    return cabeMaisUm(assinatura.plano, usuariosAtivos);
  }

  textoLimite(plano: Plano): string {
    return `Plano ${plano.nome} permite ${plano.limiteDeUsuarios} usuários`;
  }
}
```

### Passos
1. Se o campo for mutável e público, encapsule primeiro (`readonly` + função que devolve uma cópia com spread) para ter um único ponto de mudança.
2. Adicione o campo no tipo destino **sem remover o da origem**; marque o antigo com `/** @deprecated */`. Nesse estado intermediário tudo compila.
3. Garanta o caminho de acesso da origem até o destino (já existe `assinatura.plano`; se não houver, passe como parâmetro em vez de criar dependência nova).
4. Troque cada leitura/escrita para o destino, um arquivo por vez, rodando `tsc --noEmit` a cada passo.
5. Remova o campo do tipo original e siga a lista de erros: construtores/factories, schemas Zod da fronteira, mappers DTO→domínio, seeds e fixtures de teste.
6. Reveja se alguma função deve acompanhar o campo (**Move Method**) — normalmente sim, foi ela que motivou a mudança.

### Ganhos
- Elimina duplicação de estado e o risco de os dois lados divergirem.
- Reduz o número de parâmetros que circulam entre módulos.
- A regra passa a ter um único dono, o que simplifica os testes de domínio.

### Quando NÃO aplicar
- O campo tem significado diferente em cada contexto (`limiteDeUsuarios` contratado vs. limite promocional temporário) — são dois conceitos, não um campo mal posicionado.
- Mover cria import do destino em direção à origem (ciclo de módulos, que em runtime aparece como `undefined` em tempo de carga).
- O campo é estado de UI ou cache e o destino é entidade de domínio imutável.
- O destino é contrato externo estável (DTO versionado, tabela com migração publicada) — o custo supera o ganho.

### Nota TypeScript
O compilador acha os acessos que faltam, mas **não** protege contra dois riscos. Primeiro, *structural typing*: se depois da mudança `Plano` e outro tipo ficarem com o mesmo shape, eles passam um pelo outro sem erro — se o campo movido é um identificador, aproveite para brandeá-lo (`type PlanoId = string & { readonly __brand: 'PlanoId' }`). Segundo, fixtures com `Partial<Plano>` ou `as Plano`: elas silenciam exatamente o erro que você quer ver, então limpe-as antes de mover. Sem construtor posicional para inverter, a troca acidental de argumentos deixa de ser um risco — mas objetos de opções com dois `number` seguidos ainda são.

---

## 3. Extract Class
**PT-BR:** Extrair Classe · **Fonte:** https://refactoring.guru/extract-class

### Problema
Uma unidade faz o trabalho de duas ou três. No front, o caso clássico é o componente que busca dados, aplica regra de preço, formata e ainda gerencia estado; no backend, o service que valida, calcula, persiste e notifica.

### Solução
Crie uma unidade nova — classe com dependências no construtor, custom hook, ou módulo de funções puras —, mova para ela o subconjunto coeso de estado e comportamento e ligue as duas em uma única direção.

### Sinais no código (gatilhos)
- O nome precisa de "e" para descrever o que a unidade faz (`CarrinhoEValidacaoDeCupom`).
- Subconjuntos de estado que nunca são usados pelos mesmos trechos: `useState` de cupom nunca aparece perto do cálculo de total.
- Componente com 150+ linhas, vários `useEffect` sem relação entre si, ou service com 5+ dependências heterogêneas.
- Testar uma regra exige renderizar a árvore inteira, mockar `fetch` e usar timers falsos.
- Blocos separados por comentários (`// --- preço ---`) ou por linhas em branco cerimoniais.

### Antes
```tsx
// ui/Carrinho.tsx — faz tudo: I/O, regra de preço, formatação e estado de UI
export function Carrinho({ itens }: { itens: ReadonlyArray<ItemCarrinho> }) {
  const [codigo, setCodigo] = useState('');
  const [cupom, setCupom] = useState<Cupom | null>(null);
  const [validando, setValidando] = useState(false);

  useEffect(() => {
    if (codigo.length < 4) {
      setCupom(null);
      return;
    }
    setValidando(true);
    let ativo = true;
    buscarCupom(codigo)
      .then((c) => { if (ativo) setCupom(c); })
      .catch(() => { if (ativo) setCupom(null); })
      .finally(() => { if (ativo) setValidando(false); });
    return () => { ativo = false; };
  }, [codigo]);

  const subtotal = itens.reduce((acc, i) => acc + i.quantidade * i.precoUnitarioCents, 0);
  const desconto = cupom === null ? 0 : Math.round(subtotal * cupom.percentual);
  const totalReais = (subtotal - desconto) / 100;

  return (
    <form>
      <input value={codigo} onChange={(e) => setCodigo(e.target.value)} />
      {validando ? <span>validando…</span> : null}
      <strong>{totalReais.toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' })}</strong>
    </form>
  );
}
```

### Depois
```tsx
// domain/precificacao.ts — regra pura: sem React, sem I/O
export function subtotalCents(itens: ReadonlyArray<ItemCarrinho>): number {
  return itens.reduce((acc, i) => acc + i.quantidade * i.precoUnitarioCents, 0);
}
export function descontoCents(subtotal: number, cupom: Cupom | null): number {
  return cupom === null ? 0 : Math.round(subtotal * cupom.percentual);
}
export function formatarCents(cents: number): string {
  return (cents / 100).toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' });
}

// hooks/useCupom.ts — estado e I/O do cupom, testáveis sem renderizar a tela
export function useCupom(codigo: string): { cupom: Cupom | null; validando: boolean } {
  const { data, isFetching } = useQuery({
    queryKey: ['cupom', codigo],
    queryFn: () => buscarCupom(codigo),
    enabled: codigo.length >= 4,
  });
  return { cupom: data ?? null, validando: isFetching };
}

// ui/Carrinho.tsx — só orquestra e renderiza
export function Carrinho({ itens }: { itens: ReadonlyArray<ItemCarrinho> }) {
  const [codigo, setCodigo] = useState('');
  const { cupom, validando } = useCupom(codigo);
  const subtotal = useMemo(() => subtotalCents(itens), [itens]);
  const total = subtotal - descontoCents(subtotal, cupom);
  return (
    <form>
      <input value={codigo} onChange={(e) => setCodigo(e.target.value)} />
      {validando ? <span>validando…</span> : null}
      <strong>{formatarCents(total)}</strong>
    </form>
  );
}
```

### Passos
1. Nomeie a responsabilidade antes de mexer no código (`precificacao`, `useCupom`). Se você não consegue nomear, ainda não entendeu o corte.
2. Escolha a saída idiomática: **módulo de funções puras** para regra sem estado; **custom hook** quando há estado/efeito de UI; **classe com dependências no construtor** no backend (`class Precificador { constructor(private readonly cupons: RepositorioCupons) {} }`), instanciada na composição raiz.
3. Estabeleça a relação em **uma direção**: o antigo passa a consumir o novo; o novo não importa o antigo nem conhece o componente.
4. Mova estado e comportamento com **Move Field** e **Move Method**, começando pelo que não é exportado, compilando a cada movimento.
5. Só declare um `type`/`interface` de contrato se houver segunda implementação ou necessidade real de fake; caso contrário injete a classe/função concreta.
6. Ajuste a fronteira do módulo: exporte o mínimo (o resto fica sem `export`, que é o `private` real em TS) e apague qualquer import circular que tenha aparecido.
7. Escreva testes diretos na unidade nova — funções puras sem setup, hook com `renderHook`. Se ficaram simples, o corte está no lugar certo.

### Ganhos
- Cumpre o Princípio da Responsabilidade Única: mexer na validação do cupom não arrisca o cálculo de total.
- Testes menores: a regra de preço roda sem DOM, sem timers falsos e sem mock de rede.
- Reuso: `subtotalCents` serve à tela, ao resumo do pedido e ao job de faturamento.

### Quando NÃO aplicar
- A "nova" unidade teria um único trecho sem estado usado por um só chamador — extraia uma função local no mesmo arquivo.
- Você não consegue cortar sem que as duas partes se importem mutuamente.
- A extração produz peças anêmicas que apenas repassam chamadas (você criou um **Middle Man**, seção 6) — hook que só devolve `useState` cru é o exemplo típico.
- Especulação: não extraia por "vai crescer"; extraia quando o smell existe (**Speculative Generality**).

### Nota TypeScript
Antes de criar uma classe, verifique se o corte não é apenas um **módulo de funções puras** ou uma união discriminada de estados alimentando um `useReducer`. No front, a extração natural é o custom hook — mas mantenha a regra fora dele: hook é cola entre estado e regra, e regra em hook só se testa renderizando. No backend, prefira DI por construtor ou factory de closure e faça a instanciação na composição raiz; evite estado no escopo do módulo, que é singleton por processo e vaza entre requisições em SSR e entre casos de teste. Se as duas partes precisam do mesmo dado derivado, calcule uma vez no dono e passe por parâmetro em vez de recalcular em cada peça.

---

## 4. Inline Class
**PT-BR:** Classe em Linha (dissolver classe) · **Fonte:** https://refactoring.guru/inline-class

### Problema
Uma classe ou módulo quase não faz nada, não é responsável por nada e não há planos de dar responsabilidade a ele. Sobrou de uma extração anterior ou nasceu como embrulho cerimonial.

### Solução
Mova o que ele tem para quem o usa e apague-o.

### Sinais no código (gatilhos)
- A classe tem um método e esse método é `return outraCoisa(...)` — é um wrapper de uma função (**Lazy Class**).
- Ela não tem estado nem dependências: o construtor é vazio, e ainda assim ela é instanciada e injetada em todo mundo.
- Tipo só com campos e nenhuma regra, acessado sempre como `pai.filho.campo`, adicionando um nível de indireção.
- Adicionar um campo obriga a editar dois arquivos sempre juntos (**Shotgun Surgery**).
- Ela existe por generalidade especulativa ("um dia a política de desconto vai crescer").

### Antes
```ts
// pricing/desconto.ts
export function aplicarCupom(subtotalCents: number, cupom: Cupom): number {
  const bruto = Math.round(subtotalCents * cupom.percentual);
  return Math.min(bruto, cupom.tetoCents);
}

// Lazy Class: existe só para embrulhar uma função pura, sem estado nem dependências
export class CalculadoraDescontoService {
  aplicar(subtotalCents: number, cupom: Cupom): number {
    return aplicarCupom(subtotalCents, cupom);
  }
}

export class CheckoutService {
  constructor(
    private readonly calculadora: CalculadoraDescontoService,
    private readonly pedidos: RepositorioPedidos,
  ) {}

  async totalFinalCents(pedidoId: string, cupom: Cupom): Promise<number> {
    const pedido = await this.pedidos.buscar(pedidoId);
    const subtotal = subtotalCents(pedido);
    return subtotal - this.calculadora.aplicar(subtotal, cupom);
  }
}
```

### Depois
```ts
// pricing/desconto.ts — a função já era a unidade; o wrapper virou import
export function aplicarCupom(subtotalCents: number, cupom: Cupom): number {
  const bruto = Math.round(subtotalCents * cupom.percentual);
  return Math.min(bruto, cupom.tetoCents);
}

export class CheckoutService {
  constructor(private readonly pedidos: RepositorioPedidos) {}

  async totalFinalCents(pedidoId: string, cupom: Cupom): Promise<number> {
    const pedido = await this.pedidos.buscar(pedidoId);
    const subtotal = subtotalCents(pedido);
    return subtotal - aplicarCupom(subtotal, cupom);
  }
}

// composição raiz: uma dependência a menos para montar em produção e em teste
export function criarCheckoutService(pedidos: RepositorioPedidos): CheckoutService {
  return new CheckoutService(pedidos);
}
```

### Passos
1. No consumidor, passe a usar diretamente o que a unidade embrulhava (a função importada, os campos do tipo interno).
2. Faça o wrapper delegar enquanto ainda existir; compile e rode os testes nesse estado intermediário.
3. Remova o parâmetro do construtor / o nível de indireção do tipo e corrija a lista de erros: composição raiz, factories de teste, mocks que injetavam o wrapper.
4. Ajuste mappers, schemas da fronteira e fixtures que citavam o tipo dissolvido.
5. Apague o arquivo vazio e remova imports órfãos (`tsc --noEmit` e o lint de imports não usados fecham a conta).

### Ganhos
- Menos arquivos e menos indireção para ler o mesmo dado.
- Fim do trabalho duplicado de manter dois construtores/mappers em sincronia.
- Um único ponto de verdade, e um teste que não precisa montar a camada intermediária.

### Quando NÃO aplicar
- É um **value object** com validação ou unidade (`Cents`, `Email`, `CupomCodigo` como branded type) — a indireção ali é tipagem, não gordura.
- A unidade é ponto de substituição real: o wrapper existe para que o teste injete um fake, e há mais de uma implementação.
- É contrato público do pacote/módulo, consumido por outras features.
- Dissolver faria o receptor estourar (12+ campos, service com 8 métodos) — você trocaria Lazy Class por Large Class.
- Há trabalho já planejado (não especulativo) que dará comportamento a ela na próxima entrega.

### Nota TypeScript
Em TS o wrapper inútil mais comum é a **classe sem estado que embrulha uma função** — nascida do hábito de precisar de uma classe para injetar. Não precisa: a própria função é injetável (`constructor(private readonly aplicarCupom: (s: number, c: Cupom) => number)`), e o tipo da função já é o contrato. Ao dissolver um tipo aninhado em um objeto plano, lembre que `readonly` é apagado em runtime: nada impede `Object.assign` no objeto achatado, então mantenha a construção concentrada em uma factory. E prefira `type X = { ... }` a wrapper de campo único — para primitivo, branded type dá segurança com zero custo em runtime.

---

## 5. Hide Delegate
**PT-BR:** Esconder Delegação · **Fonte:** https://refactoring.guru/hide-delegate

### Problema
O cliente obtém o objeto B a partir de um campo de A e depois lê algo de B. Ele passa a depender da estrutura interna de A (`pedido.cliente.endereco.cidade.nome`).

### Solução
Ofereça em A um acesso próprio que delega para B. O cliente deixa de conhecer B.

### Sinais no código (gatilhos)
- Duas ou mais setas em uma expressão de navegação (`a.b.c`), especialmente dentro de JSX ou de um hook.
- A mesma cadeia repetida em vários arquivos, cada um resolvendo o `null` do meio à sua maneira.
- Sopa de `?.` e `??` na tela: `pedido.cliente.endereco?.cidade?.nome ?? '—'`.
- A UI importa tipos de camadas profundas do domínio só para conseguir navegar.
- Mudar a associação (`endereco` virou `enderecos: Endereco[]`) obriga a editar telas.

### Antes
```tsx
type Cidade = { readonly nome: string; readonly uf: string };
type Endereco = { readonly logradouro: string; readonly cidade: Cidade };
type Cliente = { readonly id: string; readonly nome: string; readonly endereco: Endereco | null };
export type Pedido = { readonly id: string; readonly cliente: Cliente };

export function ResumoEntrega({ pedido }: { pedido: Pedido }) {
  // Message Chain: a tela conhece quatro níveis do modelo
  return (
    <section>
      <h2>{pedido.cliente.nome}</h2>
      <p>{pedido.cliente.endereco?.logradouro ?? 'sem endereço'}</p>
      <p>
        {pedido.cliente.endereco?.cidade.nome ?? '—'}/
        {pedido.cliente.endereco?.cidade.uf ?? '—'}
      </p>
    </section>
  );
}

export function etiquetaEnvio(pedido: Pedido): string {
  const cidade = pedido.cliente.endereco?.cidade;
  return `${pedido.cliente.nome} — ${cidade?.nome ?? '?'}/${cidade?.uf ?? '?'}`;
}
```

### Depois
```tsx
// domain/pedido.ts — acesso de conveniência: ninguém mais navega a cadeia
export function cidadeEntrega(pedido: Pedido): Cidade | null {
  return pedido.cliente.endereco?.cidade ?? null;
}

export function etiquetaEnvio(pedido: Pedido): string {
  const cidade = cidadeEntrega(pedido);
  return `${pedido.cliente.nome} — ${cidade?.nome ?? '?'}/${cidade?.uf ?? '?'}`;
}

// ui/resumoEntregaVM.ts — a cadeia é achatada uma única vez, antes do componente
export type ResumoEntregaVM = {
  readonly nomeCliente: string;
  readonly logradouro: string;
  readonly cidadeUf: string;
};

export function toResumoEntregaVM(pedido: Pedido): ResumoEntregaVM {
  const cidade = cidadeEntrega(pedido);
  return {
    nomeCliente: pedido.cliente.nome,
    logradouro: pedido.cliente.endereco?.logradouro ?? 'sem endereço',
    cidadeUf: cidade === null ? 'não informado' : `${cidade.nome}/${cidade.uf}`,
  };
}
// ui/ResumoEntrega.tsx — recebe campos planos, não conhece o modelo
export function ResumoEntrega({ vm }: { vm: ResumoEntregaVM }) {
  return (
    <section>
      <h2>{vm.nomeCliente}</h2>
      <p>{vm.logradouro}</p>
      <p>{vm.cidadeUf}</p>
    </section>
  );
}
```

### Passos
1. Levante todos os pontos da cadeia que os clientes atravessam e quais deles são de fato usados.
2. Escolha a resposta certa para o caso:
   - **função/getter de conveniência no domínio** quando a navegação carrega significado de negócio (`cidadeEntrega`);
   - **optional chaining (`?.` / `??`)** quando o problema é apenas o `null` do meio e a cadeia é curta e local;
   - **ViewModel/DTO de apresentação achatado** quando quem navega é a UI: mapeie uma vez e o componente recebe strings prontas.
3. Nomeie o novo acesso no vocabulário do cliente (`cidadeEntrega`, não `getClienteEnderecoCidade`).
4. Migre os clientes um arquivo por vez; o `tsc --noEmit` guia depois que você remove o campo intermediário do tipo público.
5. Feche a porta: no domínio, deixe de exportar o tipo interno; no VM, o componente passa a aceitar só `ResumoEntregaVM`, então navegar de novo nem compila.
6. Conte quantos acessos delegantes surgiram: se forem muitos e sem regra, você está criando um **Middle Man** — pare e leia a seção 6.

### Ganhos
- O cliente depende só do que já recebe; mudança de estrutura interna não sobe para a tela.
- Os `null`-checks da cadeia ficam concentrados em um ponto, com um fallback consistente.
- Facilita trocar a origem do dado (campo → consulta → cache) sem tocar em clientes.
- Componente que recebe VM plano é trivial de memoizar e de testar.

### Quando NÃO aplicar
- O cliente precisa do objeto delegado inteiro (para passar adiante, listar, comparar) — esconder só adicionaria repasse.
- O número de acessos delegantes cresce sem regra própria: aí a resposta é expor o delegado (**Remove Middle Man**).
- A cadeia tem um nível e é local a uma função — `pedido.cliente.nome` não é um smell.
- Na UI, se a tela precisa de muitos campos aninhados, o remédio é o VM achatado, não 15 getters na entidade.

### Nota TypeScript
As três saídas resolvem problemas diferentes e a escolha errada custa caro. `?.` trata **ausência**, não acoplamento: usá-lo na tela deixa a cadeia (e a dependência estrutural) exatamente onde estava, além de espalhar fallbacks divergentes (`'—'` em um arquivo, `''` em outro). Função de conveniência no domínio resolve **significado**. VM plano resolve **acoplamento de camada** e é o único que impede regressão por construção, porque o tipo do componente nem menciona `Cliente`. Uma regra prática: se a expressão aparece dentro de JSX, ela pertence ao mapper; e como `strictNullChecks` obriga a decidir o fallback, decida uma vez, no mapper — não em cada `??`.

---

## 6. Remove Middle Man
**PT-BR:** Remover Intermediário · **Fonte:** https://refactoring.guru/remove-middle-man

### Problema
Uma camada tem métodos demais que apenas delegam para outro objeto. Ela não tem função própria, mas todo recurso novo do delegado exige um repasse nela.

### Solução
Apague os métodos delegantes, exponha o delegado (ou remova a camada) e deixe o cliente chamar direto.

### Sinais no código (gatilhos)
- Repositório cujo corpo é 100% `buscar(id) { return this.http.buscar(id); }`.
- Nenhum mapeamento, cache, validação de payload nem política de erro nos repasses.
- Toda adição no delegado gera um commit mecânico no intermediário.
- O intermediário já expõe o delegado (`readonly http`) e metade do app usa esse acesso direto.
- Testes do intermediário só verificam "chamou o mock" (`expect(http.listar).toHaveBeenCalled()`).
- O hook `useX` só devolve o resultado de outro hook, sem transformar nada.

### Antes
```ts
export type PedidoDTO = { readonly id: string; readonly totalCents: number };

export type RepositorioPedidos = {
  listar(): Promise<ReadonlyArray<PedidoDTO>>;
  buscar(id: string): Promise<PedidoDTO>;
  cancelar(id: string): Promise<void>;
  // pior ainda: vaza o delegado, então metade dos clientes já o usa direto
  readonly http: PedidosHttpClient;
};

export class RepositorioPedidosImpl implements RepositorioPedidos {
  constructor(readonly http: PedidosHttpClient) {}

  // Middle Man: nenhuma regra, só repasse
  listar(): Promise<ReadonlyArray<PedidoDTO>> {
    return this.http.listar();
  }

  buscar(id: string): Promise<PedidoDTO> {
    return this.http.buscar(id);
  }

  cancelar(id: string): Promise<void> {
    return this.http.cancelar(id);
  }
}
```

### Depois
```ts
export type RepositorioPedidos = {
  listar(): Promise<ReadonlyArray<PedidoDTO>>;
  buscar(id: string): Promise<PedidoDTO>;
  cancelar(id: string): Promise<void>;
};

// A camada de repasse desaparece: o cliente HTTP satisfaz o contrato
// e passa a ser o único lugar que valida o payload na fronteira.
export class PedidosHttpClient implements RepositorioPedidos {
  constructor(private readonly fetchJson: FetchJson) {}

  async listar(): Promise<ReadonlyArray<PedidoDTO>> {
    return pedidoSchema.array().parse(await this.fetchJson('/pedidos'));
  }

  async buscar(id: string): Promise<PedidoDTO> {
    return pedidoSchema.parse(await this.fetchJson(`/pedidos/${id}`));
  }

  async cancelar(id: string): Promise<void> {
    await this.fetchJson(`/pedidos/${id}/cancelamento`, { method: 'POST' });
  }
}

// composição raiz: um único ponto decide a implementação
export function criarRepositorioPedidos(fetchJson: FetchJson): RepositorioPedidos {
  return new PedidosHttpClient(fetchJson);
}
```

### Passos
1. Meça: conte métodos com regra vs. puro repasse. Só siga se o repasse dominar.
2. Prefira fazer o delegado **implementar o contrato de domínio** e apontar a factory da composição raiz para ele, em vez de simplesmente expor o delegado.
3. Migre os clientes para o novo alvo um por vez, mantendo os métodos antigos com `/** @deprecated */` durante a transição — o editor marca os usos remanescentes.
4. Apague os métodos delegantes e o arquivo do intermediário quando não houver mais chamadores; `tsc --noEmit` confirma.
5. Reaproveite o que era regra dispersa (validação, mapeamento, tratamento de erro) na camada que sobrou — não jogue fora.
6. Apague também os testes que só verificavam a delegação; eles não medem nada além da existência do repasse.

### Ganhos
- Menos código para manter e um arquivo a menos para editar por feature nova.
- Rastro de chamada mais curto na leitura e no stack trace.
- Fim dos testes que só verificam delegação.

### Quando NÃO aplicar
- **Tensão com Hide Delegate:** esta técnica desfaz a seção 5, e vice-versa. Critério de decisão: conte os repasses e a regra. Poucos acessos que **encapsulam uma cadeia** → mantenha `Hide Delegate`. Muitos delegantes **sem nenhuma regra** e crescendo a cada feature → `Remove Middle Man`. Se existe pelo menos uma responsabilidade real (validação de payload, mapeamento DTO→domínio, cache, retry, combinação de fontes), a camada não é Middle Man.
- O intermediário é a fronteira do módulo: mesmo fino, ele impede que tipos de infraestrutura (cliente HTTP, ORM, `Request` do framework) vazem para a UI ou para o domínio.
- Há segunda implementação real e em uso (offline, mock de testes de integração, feature flag).
- Remover exigiria que a camada de apresentação importasse tipos de rede/ORM — troca de um arquivo por acoplamento estrutural.

### Nota TypeScript
Se você quer manter a fronteira sem o boilerplate, a delegação em TS pode ser feita por composição: `return { ...delegate, async buscar(id) { /* a única regra */ } }`. Duas armadilhas: spread copia apenas propriedades **próprias e enumeráveis**, então métodos de classe (que vivem no `prototype`) **não** vêm — funciona com objeto literal/factory de closure, não com `new PedidosHttpClient()`; e o método sobrescrito perde o `this` do delegado, então chame `delegate.x()` explicitamente. Se o delegado é classe, as opções são escrever os repasses, usar um `Proxy` (custo: perde inferência e atrapalha depuração), ou trocar a classe por factory de closure. Um `type` de contrato com métodos e uma factory por implementação costuma ser o desenho mais simples — e é o que torna a decisão barata: composição para fronteira legítima, `rm` do arquivo para intermediário sem propósito.

---

## 7. Introduce Foreign Method
**PT-BR:** Introduzir Método Estrangeiro · **Fonte:** https://refactoring.guru/introduce-foreign-method

### Problema
Você precisa de um comportamento em um tipo que não controla (`Date`, tipo gerado por OpenAPI, classe de um SDK) e não pode adicioná-lo lá. A lógica então se espalha duplicada pelos clientes.

### Solução
Crie a função no seu lado, recebendo o objeto do tipo externo como primeiro parâmetro — e deixe explícito que o lugar natural dela seria o outro tipo.

### Sinais no código (gatilhos)
- O mesmo trecho manipulando um tipo de terceiros aparece em 2+ arquivos.
- Aritmética de `Date`, `URLSearchParams`, `FormData` ou `Intl` misturada com regra de negócio.
- Você escreveria `base.proximoDiaUtil()` se pudesse, mas o tipo vem da plataforma ou de um pacote.
- Comentários do tipo "a lib não tem isso" seguidos de dez linhas de workaround.
- Utilitário com funções que recebem sempre o mesmo tipo externo no primeiro parâmetro — sinal de que você já está fazendo isto, só falta consolidar em um módulo.

### Antes
```ts
export class FaturasService {
  constructor(private readonly faturas: RepositorioFaturas) {}

  async emitir(assinaturaId: string, base: Date): Promise<Fatura> {
    // método estrangeiro embutido — duplicado abaixo
    const vencimento = new Date(base.getTime());
    vencimento.setDate(vencimento.getDate() + 1);
    while (vencimento.getDay() === 0 || vencimento.getDay() === 6) {
      vencimento.setDate(vencimento.getDate() + 1);
    }
    return this.faturas.criar(assinaturaId, vencimento);
  }
}

export class NotificacoesService {
  constructor(private readonly notificacoes: RepositorioNotificacoes) {}

  async agendarAvisoDeCobranca(usuarioId: string, base: Date): Promise<void> {
    const envio = new Date(base.getTime());
    envio.setDate(envio.getDate() + 1);
    while (envio.getDay() === 0 || envio.getDay() === 6) {
      envio.setDate(envio.getDate() + 1);
    }
    await this.notificacoes.agendar(usuarioId, envio);
  }
}
```

### Depois
```ts
// core/date/proximoDiaUtil.ts
// Método estrangeiro: pertenceria a Date, que não é nosso — então vive aqui,
// com o objeto como primeiro parâmetro.
const FIM_DE_SEMANA: ReadonlySet<number> = new Set([0, 6]);

export function proximoDiaUtil(base: Date): Date {
  const data = new Date(base.getTime());
  do {
    data.setDate(data.getDate() + 1);
  } while (FIM_DE_SEMANA.has(data.getDay()));
  return data;
}

export class FaturasService {
  constructor(private readonly faturas: RepositorioFaturas) {}

  async emitir(assinaturaId: string, base: Date): Promise<Fatura> {
    return this.faturas.criar(assinaturaId, proximoDiaUtil(base));
  }
}

export class NotificacoesService {
  constructor(private readonly notificacoes: RepositorioNotificacoes) {}

  async agendarAvisoDeCobranca(usuarioId: string, base: Date): Promise<void> {
    await this.notificacoes.agendar(usuarioId, proximoDiaUtil(base));
  }
}

// NUNCA faça isto — monkey patching global:
// declare global { interface Date { proximoDiaUtil(): Date } }
// Date.prototype.proximoDiaUtil = function () { return proximoDiaUtil(this); };
```

### Passos
1. Isole o trecho que opera sobre o tipo externo e nomeie-o no vocabulário do domínio (`proximoDiaUtil`).
2. Declare-o como **função standalone exportada**, em um módulo dedicado por tipo (`core/date/proximoDiaUtil.ts`), com o objeto externo como **primeiro parâmetro**.
3. Mova o corpo; mantenha a função pura (sem `fetch`, sem `Date.now()` implícito, sem leitura de env) para que o teste seja uma tabela de entrada/saída. Se o tipo externo é mutável, copie antes de alterar (`new Date(base.getTime())`).
4. Substitua cada ocorrência duplicada pela chamada, compilando a cada troca.
5. Deixe um comentário curto dizendo que é método estrangeiro e por que não está no tipo original.
6. Se acumularem várias funções sobre o mesmo tipo externo, ou se você precisar de estado, promova para **Introduce Local Extension** (seção 8).

### Ganhos
- Elimina a duplicação sem depender do mantenedor da biblioteca.
- A chamada passa a ficar no nível de abstração do domínio.
- A regra ganha teste unitário próprio, isolado do service.
- Função em módulo próprio é tree-shakeable e trivial de substituir quando a lib finalmente oferecer o recurso.

### Quando NÃO aplicar
- A regra é do seu domínio, não do tipo externo: `assinaturaEstaAtiva(assinatura)` pertence ao módulo de assinatura, não a um utilitário de `Date`.
- Você pode editar o tipo — então **Move Method** é a resposta correta.
- O comportamento precisa de estado ou de ciclo de vida (cache, contador, throttling): vá para a seção 8.
- Vira depósito: `stringUtils.ts` com 40 funções sem relação entre si é smell, não solução — um módulo por tipo e por assunto.

### Nota TypeScript
Em TS a forma correta é a função standalone; a armadilha é tentar simular método. **Não estenda o `prototype` de tipos nativos** (`Date.prototype.proximoDiaUtil = ...`): o patch é global e por processo, colide com outra lib que escolha o mesmo nome, aparece em `for...in`, quebra quando a plataforma passa a ter o método nativo com semântica diferente, e não é tree-shakeable — quem importa o módulo por engano ganha o efeito colateral. Pelo mesmo motivo, **não use `declare global` / declaration merging** para pendurar métodos em tipos de terceiros: a declaração passa a valer para *todo* o projeto e para quem consome seus `.d.ts`, o compilador para de avisar quando o método não existe em runtime, e duas libs que declarem o mesmo membro entram em conflito irreconciliável. Se a ergonomia de encadeamento importa, use `pipe(base, proximoDiaUtil)` ou um wrapper explícito (seção 8) — nunca o `prototype` alheio.

---

## 8. Introduce Local Extension
**PT-BR:** Introduzir Extensão Local · **Fonte:** https://refactoring.guru/introduce-local-extension

### Problema
O tipo de terceiros não tem vários comportamentos de que você precisa, e você não pode adicioná-los. Já existem "métodos estrangeiros" espalhados, ou o comportamento novo exige estado.

### Solução
Crie uma unidade local que envolva (wrapper) ou estenda (subclasse) o tipo externo, concentrando os comportamentos extras — e use-a no lugar do original.

### Sinais no código (gatilhos)
- Três ou mais funções estrangeiras (seção 7) sobre o mesmo tipo externo.
- O comportamento extra precisa de **estado**: cache, retry, contador, deduplicação de requisições em voo, rate limit.
- Você precisa **substituir** o objeto externo em produção e em teste pelo mesmo contrato — função solta não permite.
- Clientes (services, repositórios) estão cheios de código que é infraestrutura do SDK, não regra de negócio.
- Toda chamada ao SDK é precedida do mesmo pré-processamento e seguida do mesmo pós-processamento.

### Antes
```ts
// SDK de terceiros: GatewayPagamentosSdk, sem cache e sem retry, que não podemos editar
export class CobrancasService {
  private readonly cache = new Map<string, StatusCobranca>();

  constructor(private readonly sdk: GatewayPagamentosSdk) {}

  // Inappropriate Intimacy: infraestrutura do SDK dentro da regra de cobrança
  async status(cobrancaId: string): Promise<StatusCobranca> {
    const emCache = this.cache.get(cobrancaId);
    if (emCache !== undefined) return emCache;
    let ultimoErro: unknown = null;
    for (let tentativa = 0; tentativa < 3; tentativa++) {
      try {
        const status = await this.sdk.status(cobrancaId);
        this.cache.set(cobrancaId, status);
        return status;
      } catch (erro) {
        ultimoErro = erro;
        await esperar(200 * (tentativa + 1));
      }
    }
    throw ultimoErro ?? new Error(`falha ao consultar cobrança ${cobrancaId}`);
  }

  async estornar(cobrancaId: string): Promise<void> {
    await this.sdk.estornar(cobrancaId);
  }
}
```

### Depois
```ts
// core/pagamentos/comCacheERetry.ts — extensão local: estado (cache) + política (retry)
export type GatewayPagamentos = {
  status(cobrancaId: string): Promise<StatusCobranca>;
  estornar(cobrancaId: string): Promise<void>;
};

export function comCacheERetry(delegate: GatewayPagamentos, tentativas = 3): GatewayPagamentos {
  const cache = new Map<string, StatusCobranca>();
  return {
    estornar: (cobrancaId) => delegate.estornar(cobrancaId), // repasse explícito
    async status(cobrancaId: string): Promise<StatusCobranca> {
      const emCache = cache.get(cobrancaId);
      if (emCache !== undefined) return emCache;
      let ultimoErro: unknown = null;
      for (let tentativa = 0; tentativa < tentativas; tentativa++) {
        try {
          const status = await delegate.status(cobrancaId);
          cache.set(cobrancaId, status);
          return status;
        } catch (erro) {
          ultimoErro = erro;
          await esperar(200 * (tentativa + 1));
        }
      }
      throw ultimoErro ?? new Error(`falha ao consultar cobrança ${cobrancaId}`);
    },
  };
}
// o service volta a ser regra de negócio e recebe o contrato, não o SDK
export class CobrancasService {
  constructor(private readonly gateway: GatewayPagamentos) {}
  status(cobrancaId: string): Promise<StatusCobranca> {
    return this.gateway.status(cobrancaId);
  }
}
```

### Passos
1. Declare o **contrato mínimo** que você usa do tipo externo (`type GatewayPagamentos` com dois métodos, não a API inteira do SDK). Ele é a costura que permite trocar implementação e escrever fake.
2. Escolha a forma: **wrapper** (padrão) por factory de closure devolvendo um objeto que satisfaz o contrato; **subclasse** somente se a lib expõe uma classe pensada para herança e você precisa passar por `instanceof`; **módulo de funções** se o comportamento extra é sem estado.
3. No wrapper, delegue o resto do contrato — repasse método a método (explícito, com boa inferência) ou por spread de um delegado que seja objeto literal. Sobrescreva só o que muda e chame `delegate.x()` dentro.
4. Se o SDK entrega uma classe e você quer aceitar as duas coisas, escreva um adaptador fino (`function adaptar(sdk: GatewayPagamentosSdk): GatewayPagamentos`) e envolva o adaptador.
5. Traga para o wrapper as funções estrangeiras dispersas (seção 7) e apague as cópias nos clientes.
6. Monte a cadeia na composição raiz (`comCacheERetry(adaptar(new GatewayPagamentosSdk(config)))`) e troque o tipo injetado nos consumidores: eles passam a depender do contrato.
7. Teste o wrapper com um fake do delegado: cache acerta na segunda chamada, retry respeita o limite, erro final propaga.

### Ganhos
- Tira do cliente o código que não é do domínio dele; o service volta a ser regra.
- Estado e política (cache, retry, métricas) ficam em um único ponto substituível e testável.
- Troca de SDK, de versão ou de fornecedor fica confinada ao wrapper e ao adaptador.
- Empilhamento: cada preocupação é um wrapper, e a ordem fica explícita na composição raiz.

### Quando NÃO aplicar
- Uma ou duas funções sem estado bastam — use a seção 7; wrapper aqui é peso morto.
- O wrapper viraria repasse puro sem regra própria: isso é **Middle Man** (seção 6).
- O tipo externo é exigido nominalmente por outra API (checagem `instanceof`, serializador, decorator de framework) e o objeto envolvido não é aceito.
- Herdar de classe de terceiros só para acessar comportamento não documentado — frágil a cada atualização, e método `#private` da classe base nem é alcançável.

### Nota TypeScript
As três formas não são equivalentes. **Módulo de funções** costuma ganhar: sem `this`, sem hierarquia, tree-shakeable, e é o único que não obriga todo o código a trocar de tipo — perde só quando é preciso guardar estado por instância ou substituir a dependência. **Wrapper por composição** é o padrão para estado e política; faça-o com factory de closure, porque o estado fica realmente privado e o spread de delegação só funciona com objetos próprios — métodos de classe ficam no `prototype` e não são copiados. **Subclasse** é a última opção: só serve quando a lib expõe a classe, quebra a cada mudança interna do fornecedor e não alcança membros `#private`. Como TS é estruturalmente tipado, o wrapper que satisfaz o contrato é aceito em qualquer lugar que peça o contrato, sem `implements` nem cast — o que torna a substituição barata, desde que você tenha declarado um contrato mínimo em vez de depender do tipo concreto do SDK.
