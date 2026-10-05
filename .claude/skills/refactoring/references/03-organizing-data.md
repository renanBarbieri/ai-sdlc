# Organizing Data — Organizando Dados

> Grupo 3 de 6 · 15 técnicas · Fonte: refactoring.guru/refactoring/techniques

## Quando este grupo se aplica
- Tipos primitivos (`string`, `number`) carregando significado de domínio: `usuarioId: string`, `email: string`, `totalCentavos: number` — *primitive obsession*, e trocar dois argumentos `string` compila sem erro.
- Números e strings literais espalhados em regras de negócio (`if (total >= 20000)`, `if (tipo === 'EXPRESSA')`) sem nome nem lugar único.
- Estado mutável exportado (objeto de módulo com `let`/campos mutáveis, array interno devolvido por getter, store cujo `set` está público).
- Type codes (`string` de status, plano, método de pagamento) usados em `if`/`switch` para decidir comportamento — normalmente com `default: throw` e campos `| null` "condicionais".
- Dado de servidor copiado para `useState` e sincronizado por `useEffect` — duas fontes de verdade que divergem; DTO cru (`unknown`, `Record<string, unknown>`) navegado na camada de negócio.
- Associações entre entidades desalinhadas com o uso real: referência circular que quebra `JSON.stringify`/`structuredClone`, ou varredura O(n) para achar o "dono".

## Índice
1. [Change Value to Reference](#1-change-value-to-reference)
2. [Change Reference to Value](#2-change-reference-to-value)
3. [Duplicate Observed Data](#3-duplicate-observed-data)
4. [Self Encapsulate Field](#4-self-encapsulate-field)
5. [Replace Data Value with Object](#5-replace-data-value-with-object)
6. [Replace Array with Object](#6-replace-array-with-object)
7. [Change Unidirectional Association to Bidirectional](#7-change-unidirectional-association-to-bidirectional)
8. [Change Bidirectional Association to Unidirectional](#8-change-bidirectional-association-to-unidirectional)
9. [Encapsulate Field](#9-encapsulate-field)
10. [Encapsulate Collection](#10-encapsulate-collection)
11. [Replace Magic Number with Symbolic Constant](#11-replace-magic-number-with-symbolic-constant)
12. [Replace Type Code with Class](#12-replace-type-code-with-class)
13. [Replace Type Code with Subclasses](#13-replace-type-code-with-subclasses)
14. [Replace Type Code with State/Strategy](#14-replace-type-code-with-statestrategy)
15. [Replace Subclass with Fields](#15-replace-subclass-with-fields)

---

## 1. Change Value to Reference
**PT-BR:** Trocar Valor por Referência · **Fonte:** https://refactoring.guru/change-value-to-reference

### Problema
Existem muitas instâncias idênticas da mesma entidade (mesmo `id`) espalhadas pelo programa, cada uma criada isoladamente a partir de DTOs. Quando uma delas muda, as outras ficam desatualizadas.

### Solução
Converter as cópias em uma única referência compartilhada, obtida por uma factory/cache (identity map) que devolve sempre o mesmo objeto para o mesmo identificador.

### Sinais no código (gatilhos)
- O mapeamento DTO→domínio faz `dtos.map((d) => ({ ...d }))` e cria um objeto novo por chamada, mesmo para o mesmo `id`.
- Duas telas exibem dados divergentes da mesma entidade porque cada uma carregou por um endpoint diferente.
- Código muta o objeto recebido (`usuario.nomeExibicao = x`) e só uma das listas "vê" a mudança.
- A entidade tem identidade natural (`id`) **e** dados que mudam durante a sessão.

### Antes
```ts
type Usuario = { id: string; nomeExibicao: string };

class UsuarioRepository {
  constructor(private readonly api: ApiUsuarios) {}

  async carregarEquipe(contaId: string): Promise<Usuario[]> {
    const dtos = await this.api.buscarEquipe(contaId);
    return dtos.map((d) => ({ id: d.id, nomeExibicao: d.nome }));
  }

  async carregarAdministradores(contaId: string): Promise<Usuario[]> {
    const dtos = await this.api.buscarAdmins(contaId);
    // cria OUTRO objeto para o mesmo usuário
    return dtos.map((d) => ({ id: d.id, nomeExibicao: d.nome }));
  }
}

function renomear(usuario: Usuario, novoNome: string): void {
  // muta apenas a cópia recebida; a outra lista segue com o nome antigo
  usuario.nomeExibicao = novoNome;
}
```

### Depois
```ts
class Usuario {
  #nomeExibicao: string;
  constructor(readonly id: string, nome: string) { this.#nomeExibicao = nome; }
  get nomeExibicao(): string { return this.#nomeExibicao; }
  renomear(novo: string): void { this.#nomeExibicao = novo; }
}

/** Identity map: um id => uma instância. Criado por request/sessão, nunca global. */
function criarRegistroUsuarios() {
  const cache = new Map<string, Usuario>();
  return {
    obter(id: string, nome: string): Usuario {
      const atual = cache.get(id) ?? new Usuario(id, nome);
      cache.set(id, atual);
      return atual;
    },
    existente: (id: string): Usuario | undefined => cache.get(id),
  };
}
type RegistroUsuarios = ReturnType<typeof criarRegistroUsuarios>;

class UsuarioRepository {
  constructor(private readonly api: ApiUsuarios, private readonly registro: RegistroUsuarios) {}
  async carregarEquipe(contaId: string): Promise<Usuario[]> {
    const dtos = await this.api.buscarEquipe(contaId);
    return dtos.map((d) => this.registro.obter(d.id, d.nome));
  }
  async carregarAdministradores(contaId: string): Promise<Usuario[]> {
    const dtos = await this.api.buscarAdmins(contaId);
    return dtos.map((d) => this.registro.obter(d.id, d.nome));
  }
}
```

### Passos
1. Substitua a criação literal do objeto por uma factory: construtor de classe com campo `#` privado, ou closure que devolve o objeto.
2. Decida quem guarda as referências. Em Node, o registro é criado **por request** e injetado no construtor; nunca um `Map` no escopo do módulo (é singleton por processo e vaza dados entre usuários).
3. Escolha entre pré-carregar o cache e criar sob demanda (`cache.get(id) ?? new ...`).
4. Defina o comportamento para id desconhecido e nomeie o método de acordo: `obter` (cria se faltar) vs `existente` (devolve `undefined`).
5. Troque todas as construções diretas pela factory e feche a mutação: campo `#` + método de domínio (`renomear`) em vez de atribuição externa.

### Ganhos
- Uma única fonte de verdade por entidade: a mudança em um ponto é vista em todos.
- Elimina divergência entre telas que carregaram a mesma entidade por rotas diferentes.
- Reduz alocação quando a mesma entidade aparece em muitas listas.

### Quando NÃO aplicar
- Entidade imutável ou sem identidade própria (`Moeda`, `Endereco`, `Periodo`) — é *value*, veja a técnica 2.
- Servidor multiusuário com cache em escopo de módulo: vira vazamento de dados entre contas e memória que nunca é liberada.
- Front-end com cache de servidor já resolvido (TanStack Query, SWR): a `queryKey` **já é** o identity map; um segundo cache manual reintroduz divergência.

### Nota TypeScript
Igualdade em JS é **referencial**: `{ id: 'u-1' } !== { id: 'u-1' }`. Não existe comparação estrutural embutida, então o identity map é o que dá sentido a `===` para entidades — e é também o que faz `useMemo`/`React.memo`/`useEffect` pararem de disparar sem motivo, porque a referência deixa de mudar a cada mapeamento. Use `Map` (não objeto literal) para a chave: ele aceita qualquer chave e não herda `Object.prototype`. Se as instâncias podem ser descartadas, `WeakMap`/`WeakRef` evita retenção. Em SSR, estado no escopo do módulo é compartilhado por todas as requisições do processo — injete o registro por construtor.

---

## 2. Change Reference to Value
**PT-BR:** Trocar Referência por Valor · **Fonte:** https://refactoring.guru/change-reference-to-value

### Problema
Um objeto pequeno e raramente alterado é tratado como referência gerenciada (id, cache, ciclo de vida), obrigando o código a buscá-lo em um registro a cada uso.

### Solução
Transformá-lo em *value object* imutável, comparado pelos seus campos e construído livremente onde for necessário.

### Sinais no código (gatilhos)
- Classe de 1–3 campos com um `Map` estático só para garantir instância única.
- Código chama `repositorio.buscar(id)` apenas para ler dois campos constantes.
- O objeto atravessa fronteiras (JSON de API, `localStorage`, `postMessage`, `structuredClone`) e a identidade não importa.
- Comparações usam `a.id === b.id` quando o que interessa são os valores.
- Mutar a instância compartilhada altera, por efeito colateral, objetos já montados em outro lugar.

### Antes
```ts
class Moeda {
  private static readonly instancias = new Map<string, Moeda>();
  simbolo = '';
  casasDecimais = 2;

  private constructor(readonly codigo: string) {}

  static de(codigo: string): Moeda {
    const atual = Moeda.instancias.get(codigo) ?? new Moeda(codigo);
    Moeda.instancias.set(codigo, atual);
    return atual;
  }
}

class Fatura {
  constructor(readonly id: string, readonly centavos: number, readonly moeda: Moeda) {}
}

class FaturaRepository {
  constructor(private readonly db: Db) {}
  async carregar(faturaId: string): Promise<Fatura> {
    const row = await this.db.faturas.buscar(faturaId);
    const moeda = Moeda.de(row.moedaCodigo);
    // mutação da instância compartilhada: toda fatura já carregada muda junto
    moeda.simbolo = row.moedaSimbolo;
    moeda.casasDecimais = row.moedaCasas;
    return new Fatura(row.id, row.centavos, moeda);
  }
}
```

### Depois
```ts
export type Moeda = {
  readonly codigo: string;
  readonly simbolo: string;
  readonly casasDecimais: number;
};

export const criarMoeda = (codigo: string, simbolo: string, casasDecimais: number): Moeda =>
  Object.freeze({ codigo, simbolo, casasDecimais });

/** Não existe equals estrutural em JS: a igualdade de valor é explícita. */
export const mesmaMoeda = (a: Moeda, b: Moeda): boolean =>
  a.codigo === b.codigo && a.casasDecimais === b.casasDecimais;

export type Fatura = { readonly id: string; readonly centavos: number; readonly moeda: Moeda };

export const formatar = (f: Fatura): string =>
  `${f.moeda.simbolo} ${(f.centavos / 100).toFixed(f.moeda.casasDecimais)}`;

/** "Mutação" pontual = novo valor, via spread. */
export const trocarMoeda = (f: Fatura, moeda: Moeda): Fatura => ({ ...f, moeda });

export class FaturaRepository {
  constructor(private readonly db: Db) {}
  async carregar(faturaId: string): Promise<Fatura> {
    const row = await this.db.faturas.buscar(faturaId);
    return {
      id: row.id,
      centavos: row.centavos,
      moeda: criarMoeda(row.moedaCodigo, row.moedaSimbolo, row.moedaCasas),
    };
  }
}
```

### Passos
1. Torne o objeto imutável: todos os campos `readonly`, nenhum setter, nenhum método que altera estado.
2. Defina a igualdade explicitamente — uma função `mesmaX(a, b)` ou uma chave canônica (`` `${codigo}:${casas}` ``) para usar em `Map`/`Set`.
3. Para mudanças pontuais, use spread (`{ ...fatura, moeda }`) em vez de mutar.
4. Apague o registro estático e a factory de instância única; construa o valor onde ele é necessário, normalmente no mapeamento DTO→domínio.
5. Remova as buscas por id e passe o valor direto por parâmetro e no estado da UI.

### Ganhos
- Consistência: nenhuma instância "velha" divergente; seguro para concorrência e para cache de servidor.
- Implementação muito mais simples: sem registro, sem invalidação, sem ciclo de vida.
- Serializa e clona sem cuidado especial (`JSON.stringify`, `structuredClone`), e serve como parte de `queryKey`.

### Quando NÃO aplicar
- O objeto muda com frequência e a mudança precisa ser vista por todos os detentores (é referência — técnica 1).
- É a raiz do agregado, com identidade de negócio (`Usuario`, `Pedido`).
- Objeto grande copiado em listas enormes a cada interação — o custo do spread aparece no perfil.

### Nota TypeScript
`readonly` e `ReadonlyArray<T>` só existem em tempo de compilação: nada impede `(obj as { simbolo: string }).simbolo = 'x'`, e o valor cruzando a fronteira (`JSON.parse`, resposta de API) chega sem nenhuma proteção. Quando a garantia precisa ser real em runtime, use `Object.freeze` na construção. Sem igualdade estrutural embutida, dois valores iguais são referências diferentes: por isso value objects em props/deps de React devem ser criados uma vez (`useMemo`) ou comparados por campo — passar `criarMoeda(...)` inline em cada render invalida toda memoização abaixo.

---

## 3. Duplicate Observed Data
**PT-BR:** Duplicar Dados Observados · **Fonte:** https://refactoring.guru/duplicate-observed-data

### Problema
Dados e regras de domínio estão dentro da camada de UI, misturando apresentação com negócio, e o mesmo dado existe em dois lugares ao mesmo tempo — a cópia local e a origem — podendo divergir.

### Solução
Manter uma única fonte de verdade fora do componente (cache de servidor ou store externa) e fazer a UI **observar** essa fonte, derivando o que precisa em vez de sincronizar cópias.

### Sinais no código (gatilhos)
- `useState` inicializado vazio + `useEffect` que copia `props`/`data` para o estado (o clássico "sincronizar em vez de derivar").
- Vários `useState` que sempre mudam juntos (`itens`, `total`, `freteGratis`) — sinal de dado derivado guardado como estado.
- Regra de negócio (desconto, elegibilidade, faixa) escrita dentro do corpo do componente.
- Componente chama o cliente HTTP diretamente e guarda o resultado em estado local.
- Bug típico: "atualizei em uma tela e a outra continuou com o valor antigo".

### Antes
```tsx
function ResumoCarrinho({ carrinhoId }: { carrinhoId: string }) {
  const { data } = useQuery({
    queryKey: ['carrinho', carrinhoId],
    queryFn: () => api.buscarCarrinho(carrinhoId),
  });

  // estado duplicado: cópia do dado do servidor, livre para divergir
  const [itens, setItens] = useState<ItemCarrinho[]>([]);
  const [total, setTotal] = useState(0);
  const [freteGratis, setFreteGratis] = useState(false);

  useEffect(() => {
    if (data !== undefined) setItens(data.itens);
  }, [data]);

  function alterarQuantidade(itemId: string, quantidade: number): void {
    const novos = itens.map((i) => (i.id === itemId ? { ...i, quantidade } : i));
    setItens(novos);
    // regra de negócio no componente, espalhada por três estados
    const soma = novos.reduce((a, i) => a + i.precoCentavos * i.quantidade, 0);
    setTotal(soma);
    setFreteGratis(soma >= 20000);
  }

  return (
    <div>
      {itens.map((i) => (
        <LinhaItem key={i.id} item={i} onAlterar={(q) => alterarQuantidade(i.id, q)} />
      ))}
      <span>{freteGratis ? 'Frete grátis' : `Total ${total}`}</span>
    </div>
  );
}
```

### Depois
```tsx
// domínio puro: testável sem React
export type Carrinho = { readonly itens: ReadonlyArray<ItemCarrinho> };
const FRETE_GRATIS_A_PARTIR_DE_CENTAVOS = 20_000;

export function resumir(carrinho: Carrinho): { total: number; freteGratis: boolean } {
  const total = carrinho.itens.reduce((a, i) => a + i.precoCentavos * i.quantidade, 0);
  return { total, freteGratis: total >= FRETE_GRATIS_A_PARTIR_DE_CENTAVOS };
}

function ResumoCarrinho({ carrinhoId }: { carrinhoId: string }) {
  const chave = ['carrinho', carrinhoId] as const;
  const { data: carrinho } = useQuery({ queryKey: chave, queryFn: () => api.buscarCarrinho(carrinhoId) });
  const { mutate: alterarQuantidade } = useMutation({
    mutationFn: (v: { itemId: string; quantidade: number }) => api.alterarItem(carrinhoId, v),
    onSuccess: (atualizado: Carrinho) => queryClient.setQueryData(chave, atualizado),
  });

  if (carrinho === undefined) return <span>Carregando…</span>;
  // derive, não sincronize: nenhum useState/useEffect copiando o servidor
  const { total, freteGratis } = resumir(carrinho);

  return (
    <div>
      {carrinho.itens.map((i) => (
        <LinhaItem
          key={i.id}
          item={i}
          onAlterar={(quantidade) => alterarQuantidade({ itemId: i.id, quantidade })}
        />
      ))}
      <span>{freteGratis ? 'Frete grátis' : `Total ${total}`}</span>
    </div>
  );
}
```

### Passos
1. Classifique o dado: **de servidor** (vai para TanStack Query/SWR), **global de app** (store externa lida com `useSyncExternalStore`), ou **de UI** (`useState` local, e só ele).
2. Extraia as regras de negócio do componente para funções puras que recebem o estado e devolvem o resultado (`resumir`).
3. Elimine todo estado derivado: se o valor pode ser calculado do dado da fonte, calcule no render (`useMemo` só se medir caro).
4. Apague o `useEffect` que copiava `data`/`props` para `useState`; a leitura passa a vir direto da fonte.
5. Troque escritas locais por mutações na fonte (`useMutation` + `setQueryData`/`invalidateQueries`, ou uma ação da store).
6. Nos handlers, chame ações (`alterarQuantidade({...})`) em vez de montar o próximo estado no componente.

### Ganhos
- Uma fonte de verdade: impossível a UI mostrar um total que não corresponde aos itens.
- Regra de negócio testável com teste unitário puro, sem renderizar componente.
- Outra tela (checkout, mini-cart, widget) reutiliza o mesmo cache e as mesmas funções de domínio.

### Quando NÃO aplicar
- Estado puramente visual e efêmero (foco, scroll, acordeão aberto, texto sendo digitado) — pertence ao componente.
- Formulário controlado: o rascunho local é legitimamente diferente do valor persistido; sincronizar de volta a cada tecla é que seria errado.
- Componente de design system stateless por contrato.

### Nota TypeScript
O Observer manual (lista de listeners + `notificar()`) só é necessário para store própria — e, mesmo aí, o ponto de integração é `useSyncExternalStore(subscribe, getSnapshot)`, com `getSnapshot` devolvendo **a mesma referência** enquanto nada mudar (senão o React entra em loop de re-render). Para dado de servidor, o cache da biblioteca já é a store observável. A regra prática é o oposto do nome clássico da técnica: **não duplique** — derive. Quando um estado local precisa mesmo "resetar" ao mudar a entidade, prefira `key={id}` no componente a um `useEffect` de sincronização.

---

## 4. Self Encapsulate Field
**PT-BR:** Auto-Encapsular Campo · **Fonte:** https://refactoring.guru/self-encapsulate-field

### Problema
A própria classe acessa diretamente seu campo em vários pontos, então não há um lugar único para validar, normalizar, derivar ou calcular preguiçosamente esse valor.

### Solução
Passar a acessar o campo sempre por um acessor — em TypeScript, `get`/`set` sobre um campo privado `#`, com cache/memo quando o cálculo é caro.

### Sinais no código (gatilhos)
- A mesma normalização (`trim`, `toLowerCase`, `replace(/\D/g, '')`) repetida antes de cada atribuição.
- Flag manual de cache (`calculada`, `carregado`) com o mesmo bloco `if (!flag) { ... }` copiado em vários métodos.
- Campo derivado recalculado em três métodos em vez de um acessor só.
- Cálculo caro feito no construtor mesmo quando o resultado pode nunca ser usado.

### Antes
```ts
class AnaliseFaturas {
  private mediaCache = 0;
  private calculada = false;

  constructor(private readonly faturas: ReadonlyArray<Fatura>) {}

  media(): number {
    if (!this.calculada) {
      this.mediaCache = this.faturas.reduce((a, f) => a + f.centavos, 0) / this.faturas.length;
      this.calculada = true;
    }
    return this.mediaCache;
  }

  acimaDoTicket(): ReadonlyArray<Fatura> {
    if (!this.calculada) {
      this.mediaCache = this.faturas.reduce((a, f) => a + f.centavos, 0) / this.faturas.length;
      this.calculada = true;
    }
    return this.faturas.filter((f) => f.centavos > this.mediaCache);
  }

  resumo(): string {
    if (!this.calculada) {
      this.mediaCache = this.faturas.reduce((a, f) => a + f.centavos, 0) / this.faturas.length;
      this.calculada = true;
    }
    return `Ticket médio ${(this.mediaCache / 100).toFixed(2)}`;
  }
}
```

### Depois
```ts
class AnaliseFaturas {
  #media: number | undefined;

  constructor(private readonly faturas: ReadonlyArray<Fatura>) {}

  /** Ponto único de acesso: calcula na primeira leitura e memoiza. */
  get media(): number {
    this.#media ??= this.faturas.reduce((a, f) => a + f.centavos, 0) / this.faturas.length;
    return this.#media;
  }

  acimaDoTicket(): ReadonlyArray<Fatura> {
    return this.faturas.filter((f) => f.centavos > this.media);
  }

  resumo(): string {
    return `Ticket médio ${(this.media / 100).toFixed(2)}`;
  }
}

class CadastroCliente {
  #cpf = '';

  /** Setter: normaliza e valida em um lugar só. */
  set cpf(valor: string) {
    const digitos = valor.replace(/\D/g, '');
    if (digitos.length !== 11) throw new Error('CPF inválido');
    this.#cpf = digitos;
  }

  get cpf(): string {
    return this.#cpf;
  }
}
```

### Passos
1. Identifique o campo lido/escrito direto em vários pontos e torne-o privado (`#campo`).
2. Crie o acessor: `get` para valor derivado, `get` com `??=` para memoizar cálculo caro, `set` para validar/normalizar a escrita.
3. Substitua todos os acessos internos pelo acessor — inclusive dentro da própria classe; é isso que dá o nome à técnica.
4. Se o valor memoizado depende de entrada mutável, invalide o cache (`this.#media = undefined`) no mesmo lugar em que a entrada muda; se a entrada é `readonly`, não há o que invalidar.
5. Em código funcional, o equivalente é uma closure com variável de memo, ou `useMemo`/`useCallback` no componente — a ideia é a mesma: um ponto único de obtenção.
6. Se o par `get`/`set` tem tipos diferentes (aceita `string`, devolve `Cpf`), considere trocar o setter por um método (`definirCpf`) — assimetria em acessor confunde.

### Ganhos
- Validação, normalização, log e cache em um único ponto — impossível esquecer um caminho.
- `??=` remove a flag manual e o bloco duplicado.
- Trocar a forma de obter o valor (memo, cálculo, delegação) não afeta nenhum consumidor.

### Quando NÃO aplicar
- Campo trivial sem regra: o acessor só adiciona ruído e uma camada de indireção.
- Loop de altíssima frequência (parse de arquivo grande, laço de animação), onde a chamada do acessor pesa.
- Obtenção assíncrona ou com I/O: getter não pode ser `async` — exponha um método `Promise`.

### Nota TypeScript
Getter é **invisível no ponto de chamada**: `analise.media` parece leitura de campo e pode custar um `reduce` inteiro. Em React isso é armadilha real — um getter caro lido no corpo do componente roda a cada render, e ninguém suspeita porque não há parênteses. Regra: getter é O(1) ou memoizado; qualquer coisa acima disso deve ser um método com nome de verbo (`calcularMedia()`), que sinaliza o custo. Cuidado também com `get` acessado em dependências de hooks: `[analise.media]` executa o cálculo em toda renderização. Campos `#` não aparecem em `JSON.stringify` nem em spread — bom para blindagem, mas o `toJSON`/DTO precisa ser explícito.

---

## 5. Replace Data Value with Object
**PT-BR:** Substituir Valor de Dado por Objeto · **Fonte:** https://refactoring.guru/replace-data-value-with-object

### Problema
Um campo primitivo (`cpf: string`, `email: string`, `usuarioId: string`) acumulou dados e comportamento próprios — validação, normalização, formatação — que ficaram espalhados por várias camadas.

### Solução
Criar um tipo dedicado para o conceito, mover para ele a validação e o comportamento, e usar esse tipo no lugar do primitivo.

### Sinais no código (gatilhos)
- Assinaturas com vários `string` seguidos (`criar(clienteId: string, cpf: string, email: string)`) — trocar a ordem dos argumentos compila.
- Módulo `utils`/`helpers` com `validarCpf`, `formatarCpf`, `limparDigitos`.
- A mesma validação de formato repetida no formulário, no service e no repositório.
- Comentário explicando o formato esperado da string (`// sem pontuação`, `// sempre minúsculo`).
- `Record<string, X>` onde a chave "é" um id de outra entidade e nada impede passar o id errado.

### Antes
```ts
export const limparDigitos = (valor: string): string => valor.replace(/\D/g, '');
export const cpfValido = (cpf: string): boolean => limparDigitos(cpf).length === 11;
export const emailValido = (email: string): boolean => /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email);

export function formatarCpf(cpf: string): string {
  const d = limparDigitos(cpf);
  return `${d.slice(0, 3)}.${d.slice(3, 6)}.${d.slice(6, 9)}-${d.slice(9)}`;
}

type Cliente = { id: string; cpf: string; email: string };

export class CriarAssinaturaService {
  constructor(private readonly repo: AssinaturaRepository) {}

  async criar(clienteId: string, cpf: string, email: string): Promise<Assinatura> {
    if (!cpfValido(cpf)) throw new Error('CPF inválido');
    if (!emailValido(email)) throw new Error('E-mail inválido');
    // três strings seguidas: trocar clienteId por cpf compila sem reclamar
    return this.repo.criar(clienteId, limparDigitos(cpf), email.trim().toLowerCase());
  }
}
```

### Depois
```ts
declare const marca: unique symbol;
type Brand<T, B extends string> = T & { readonly [marca]: B };
export type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
export type ClienteId = Brand<string, 'ClienteId'>;
export type Cpf = Brand<string, 'Cpf'>;
export type Email = Brand<string, 'Email'>;
export const clienteId = (bruto: string): ClienteId => bruto as ClienteId; // único ponto que marca

/** Validação na construção: só existe Cpf válido em memória. */
export function parseCpf(bruto: string): Result<Cpf, 'cpf-invalido'> {
  const digitos = bruto.replace(/\D/g, '');
  return digitos.length === 11
    ? { ok: true, value: digitos as Cpf }
    : { ok: false, error: 'cpf-invalido' };
}

export function parseEmail(bruto: string): Result<Email, 'email-invalido'> {
  const normalizado = bruto.trim().toLowerCase();
  return /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(normalizado)
    ? { ok: true, value: normalizado as Email }
    : { ok: false, error: 'email-invalido' };
}

export const formatarCpf = (cpf: Cpf): string =>
  `${cpf.slice(0, 3)}.${cpf.slice(3, 6)}.${cpf.slice(6, 9)}-${cpf.slice(9)}`;

export type Cliente = { readonly id: ClienteId; readonly cpf: Cpf; readonly email: Email };

export class CriarAssinaturaService {
  constructor(private readonly repo: AssinaturaRepository) {}
  // trocar a ordem dos argumentos agora é erro de compilação
  async criar(clienteId: ClienteId, cpf: Cpf, email: Email): Promise<Assinatura> {
    return this.repo.criar(clienteId, cpf, email);
  }
}
```

### Passos
1. Crie o tipo do conceito. Para um único valor primitivo, **branded type**; para dois ou mais campos, um `type` com todos `readonly` (`Endereco`, `Dinheiro`).
2. Crie a única porta de entrada: `parseX(bruto: string): Result<X, ErroX>`, que normaliza e valida. O `as X` só aparece dentro dessa função.
3. Mova para o módulo do tipo o comportamento que estava em `utils` (`formatarCpf`, `mascarar`, `dominioDe`).
4. Troque o tipo do campo (`cpf: string` → `cpf: Cpf`) e siga os erros do compilador.
5. Faça o parse **na fronteira** — schema Zod com `.transform()` no DTO de entrada, ou no handler do formulário — e propague o tipo já validado para service, repositório e estado da UI.
6. Converta de volta para `string` só ao sair (query SQL, body de request, texto renderizado); como o brand é fantasma, o valor já *é* a string.

### Ganhos
- Coesão: dado e comportamento juntos; validação impossível de esquecer.
- Segurança de tipos: `Cpf`, `Email` e `ClienteId` não se confundem, mesmo sendo todos `string`.
- Erro esperado vira valor (`Result`), tratável no formulário sem `try/catch`.

### Quando NÃO aplicar
- O primitivo não tem regra nem comportamento (um `titulo: string` livre).
- DTOs de borda: mantenha primitivos crus no tipo do JSON e converta no mapper — não vaze brand para o contrato de rede.
- Valor que precisa cruzar `postMessage`/worker com métodos: brand e funções não sobrevivem à serialização; mantenha as operações como funções do módulo, não como métodos do objeto.

### Nota TypeScript
Branded type é a resposta a *primitive obsession* sem custo em runtime: `type Cpf = string & { readonly [marca]: 'Cpf' }` é apagado na compilação — em runtime é uma `string` comum, aceita em `slice`, template literal e `JSON.stringify`. O preço é que o brand só se aplica por `as` dentro do parser: mantenha esse `as` num único arquivo e o resto do código não consegue fabricar um valor inválido. Com Zod, `z.string().transform((v) => v as Cpf)` (ou `.brand<'Cpf'>()`) faz a marcação na própria validação de fronteira. Para conceitos de vários campos, `readonly` + `Object.freeze` na factory; e prefira uma factory que devolve `Result` a uma que lança, porque erro de formato de input é fluxo esperado, não exceção.

---

## 6. Replace Array with Object
**PT-BR:** Substituir Array por Objeto · **Fonte:** https://refactoring.guru/replace-array-with-object

### Problema
Um array ou tupla é usado como registro heterogêneo, com significado atrelado à posição (`linha[1]` é o nome, `linha[4]` é o total), o que é ilegível e quebra em silêncio quando a ordem muda.

### Solução
Substituir o array por um objeto com um campo nomeado por elemento, e concentrar em um único lugar a leitura das posições.

### Sinais no código (gatilhos)
- Acesso por índice literal: `linha[2]`, `campos[7]`, `resultado[0]`.
- `string[]`, `unknown[]` ou `Record<string, unknown>` representando uma entidade.
- Comentário mapeando índices para significados (`// 0=id, 1=nome, ...`).
- `Number(...)`/`String(...)` repetidos no ponto de uso em vez de no parse.
- Tupla anônima retornada de função (`[boolean, string, number]`) que o chamador desestrutura na ordem errada.

### Antes
```ts
type LinhaCsv = ReadonlyArray<string>;
const TICKET_MINIMO_CENTAVOS = 5_000;

export function importar(csv: ReadonlyArray<string>): LinhaCsv[] {
  return csv.slice(1).map((linha) => linha.split(';'));
}

export function acimaDoMinimo(linhas: ReadonlyArray<LinhaCsv>): string[] {
  return linhas
    // 0=pedidoId, 1=clienteNome, 2=cupom, 3=itens, 4=totalCentavos
    .filter((l) => Number(l[4]) / Number(l[3]) >= TICKET_MINIMO_CENTAVOS)
    .map((l) => String(l[1]));
}

export function relatorio(linhas: ReadonlyArray<LinhaCsv>): string {
  return linhas
    .map((l) => `${String(l[1])} (${String(l[0])}) - ${String(l[3])} itens`)
    .join('\n');
}

/** Tupla posicional: mesmo problema, outra sintaxe. */
export function totalELote(linhas: ReadonlyArray<LinhaCsv>): [number, number] {
  return [linhas.reduce((a, l) => a + Number(l[4]), 0), linhas.length];
}
```

### Depois
```ts
export type PedidoImportado = {
  readonly pedidoId: string;
  readonly clienteNome: string;
  readonly cupom: string | null;
  readonly itens: number;
  readonly totalCentavos: number;
};
const TICKET_MINIMO_CENTAVOS = 5_000;

/** Índices e conversões isolados em um único lugar. */
export function deLinhaCsv(linha: string): PedidoImportado {
  const [pedidoId = '', clienteNome = '', cupom = '', itens = '0', total = '0'] = linha.split(';');
  return {
    pedidoId,
    clienteNome,
    cupom: cupom === '' ? null : cupom,
    itens: Number(itens),
    totalCentavos: Number(total),
  };
}

export const ticketMedio = (p: PedidoImportado): number =>
  p.itens === 0 ? 0 : p.totalCentavos / p.itens;

export const importar = (csv: ReadonlyArray<string>): PedidoImportado[] => csv.slice(1).map(deLinhaCsv);

export const acimaDoMinimo = (itens: ReadonlyArray<PedidoImportado>): string[] =>
  itens.filter((p) => ticketMedio(p) >= TICKET_MINIMO_CENTAVOS).map((p) => p.clienteNome);

export const relatorio = (itens: ReadonlyArray<PedidoImportado>): string =>
  itens.map((p) => `${p.clienteNome} (${p.pedidoId}) - ${p.itens} itens`).join('\n');

/** Retorno nomeado em vez de tupla posicional. */
export const totalizar = (itens: ReadonlyArray<PedidoImportado>): { total: number; lote: number } =>
  ({ total: itens.reduce((a, p) => a + p.totalCentavos, 0), lote: itens.length });
```

### Passos
1. Crie o `type` com um campo `readonly` por posição do array, nomeado pelo significado.
2. Escreva o parse em um único lugar (`deLinhaCsv`), com desestruturação e valores default — isso também resolve o `T | undefined` que `noUncheckedIndexedAccess` gera em acesso por índice.
3. Troque os tipos de parâmetro/retorno de `LinhaCsv[]` para `PedidoImportado[]`.
4. Substitua cada `l[i]` pelo campo e mova para funções do módulo os cálculos que dependiam dos índices (`ticketMedio`).
5. Para retornos com mais de dois valores, troque a tupla por objeto nomeado; se mantiver tupla, use **tupla nomeada** (`[total: number, lote: number]`) para que a IDE mostre o significado na desestruturação.
6. Se a origem é um mapa dinâmico, converta na borda com `Object.entries` (`Object.entries(bruto).map(([chave, valor]) => ...)`) e devolva objetos tipados; não navegue o mapa na camada de negócio.

### Ganhos
- Campos autodocumentados; mudar a ordem das colunas deixa de ser bug silencioso.
- Comportamento derivado (`ticketMedio`) vive junto do formato dos dados.
- Objeto nomeado é resiliente: acrescentar um campo não desloca nada, ao contrário de uma tupla.

### Quando NÃO aplicar
- Coleção genuinamente homogênea (`ItemPedido[]`, `number[]` de valores) — array é o tipo certo.
- Tupla de duas posições consagrada pela convenção: entradas de `Map`/`Object.entries`, retorno de hook (`const [valor, setValor] = useState()`).
- Buffers e `TypedArray` em código sensível a performance.

### Nota TypeScript
Com `noUncheckedIndexedAccess`, `linha[4]` tem tipo `string | undefined` — o compilador já denuncia o acesso posicional, e espalhar `?? ''` pelo código é o sintoma, não a cura. Desestruturação com default no parse resolve de uma vez. Se a estrutura posicional é imposta de fora, descreva-a como tupla nomeada e `readonly` (`readonly [id: string, nome: string]`), que dá aridade fixa e rótulos na IDE. Para dados de rede, valide com Zod (`z.tuple([...]).transform(...)` ou `z.object({...})`) e devolva um objeto de domínio: o schema passa a ser o único lugar que conhece o formato bruto.

---

## 7. Change Unidirectional Association to Bidirectional
**PT-BR:** Mudar Associação Unidirecional para Bidirecional · **Fonte:** https://refactoring.guru/change-unidirectional-association-to-bidirectional

### Problema
Duas classes precisam dos recursos uma da outra, mas a associação existe só em um sentido, forçando o cliente a varrer coleções para redescobrir o lado inverso.

### Solução
Adicionar a associação que falta na classe que precisa dela, elegendo um lado dominante que mantém as duas pontas consistentes.

### Sinais no código (gatilhos)
- Busca repetida do "dono": `pedidos.find((p) => p.itens.some((i) => i.id === item.id))`.
- Função recebe os dois objetos só para descobrir a relação entre eles.
- Varredura O(n·m) rodando a cada render ou a cada item de uma lista grande.
- O cálculo que depende do pai (peso, percentual do total) vive espalhado em helpers de UI.

### Antes
```ts
class Pedido {
  readonly itens: ItemPedido[] = [];
  constructor(readonly id: string, readonly numero: string) {}
  adicionar(item: ItemPedido): void { this.itens.push(item); }
}

class ItemPedido {
  constructor(readonly id: string, readonly sku: string, readonly precoCentavos: number) {}
}

class RelatorioItemService {
  constructor(private readonly pedidos: ReadonlyArray<Pedido>) {}

  /** Varre todos os pedidos para descobrir o dono do item. */
  numeroDoPedido(item: ItemPedido): string | undefined {
    return this.pedidos.find((p) => p.itens.some((i) => i.id === item.id))?.numero;
  }

  pesoNoPedido(item: ItemPedido): number {
    const pedido = this.pedidos.find((p) => p.itens.some((i) => i.id === item.id));
    if (pedido === undefined) return 0;
    const total = pedido.itens.reduce((a, i) => a + i.precoCentavos, 0);
    return total === 0 ? 0 : item.precoCentavos / total;
  }
}
```

### Depois
```ts
class Pedido {
  #itens: ItemPedido[] = [];
  constructor(readonly id: string, readonly numero: string) {}
  get itens(): ReadonlyArray<ItemPedido> { return this.#itens; }
  get totalCentavos(): number { return this.#itens.reduce((a, i) => a + i.precoCentavos, 0); }

  /** Lado dominante: cria a associação nas duas pontas, com guarda de duplicata. */
  adicionar(item: ItemPedido): void {
    if (this.#itens.some((i) => i.id === item.id)) return;
    this.#itens.push(item);
    item.vincularPedido(this);
  }

  remover(itemId: string): void {
    const item = this.#itens.find((i) => i.id === itemId);
    if (item === undefined) return;
    this.#itens = this.#itens.filter((i) => i.id !== itemId);
    item.desvincularPedido();
  }
}

class ItemPedido {
  // campo privado: não aparece em JSON.stringify nem em spread, logo não gera ciclo
  #pedido: Pedido | undefined;
  constructor(readonly id: string, readonly sku: string, readonly precoCentavos: number) {}
  get pedido(): Pedido | undefined { return this.#pedido; }
  get peso(): number {
    const total = this.#pedido?.totalCentavos ?? 0;
    return total === 0 ? 0 : this.precoCentavos / total;
  }
  vincularPedido(dono: Pedido): void { this.#pedido = dono; }
  desvincularPedido(): void { this.#pedido = undefined; }
}
```

### Passos
1. Adicione o campo da associação inversa como privado (`#pedido`) com getter somente leitura.
2. Escolha o lado dominante — normalmente a raiz do agregado (`Pedido`); só ele cria e desfaz a associação.
3. Crie no lado não dominante métodos de nome inequívoco (`vincularPedido`/`desvincularPedido`) e **não os exporte** do módulo público (barrel/`index.ts`); a fronteira de módulo é o `internal` do TypeScript.
4. Chame esses métodos dentro de `adicionar`/`remover` do dominante, sempre com guarda contra duplicata e recursão.
5. Mova para o lado não dominante os cálculos que dependiam do pai (`peso`), agora que ele tem a referência.
6. Antes de serializar, garanta que o ciclo não vaza: campo `#`, `toJSON()` explícito devolvendo só ids, ou um mapper domínio→DTO.

### Ganhos
- Navegação direta nas duas direções; elimina a varredura O(n·m) por render.
- Cálculos que dependem do pai ficam na própria entidade filha, com dados e comportamento juntos.
- A consistência das duas pontas passa a ser garantida por um único método.

### Quando NÃO aplicar
- O lado inverso é fácil de calcular ou raramente usado.
- Os objetos são serializados, clonados ou guardados em cache: o ciclo quebra `JSON.stringify` (`TypeError: Converting circular structure to JSON`) e, em muitos formatos, também a validação de schema.
- Estado de front-end: em store/props de React, o correto é normalizar (técnica 8), não criar ciclo — o React compara referências e um grafo cíclico é imprevisível em memo e devtools.

### Nota TypeScript
Referência circular é o risco central aqui, e ele é silencioso até o momento em que o objeto precisa sair da memória: `JSON.stringify` lança, o body da API não serializa, `structuredClone` (usado por `postMessage`, IndexedDB e cache) preserva o ciclo mas explode o payload, e `console.log`/devtools ficam ilegíveis. Duas defesas: guardar o inverso em campo `#` (invisível para `JSON.stringify`, spread e `Object.keys`) ou escrever `toJSON()` devolvendo apenas `pedidoId`. Se os modelos são objetos planos imutáveis — o caso normal em front-end —, não crie o ciclo: guarde `pedidoId` e resolva pelo mapa normalizado.

---

## 8. Change Bidirectional Association to Unidirectional
**PT-BR:** Mudar Associação Bidirecional para Unidirecional · **Fonte:** https://refactoring.guru/change-bidirectional-association-to-unidirectional

### Problema
Existe associação bidirecional entre duas entidades, mas um dos lados quase não usa o outro — sobra código de sincronização, acoplamento, ciclos na serialização e objetos que nunca são coletados.

### Solução
Remover a associação não utilizada e obter o objeto por parâmetro, por id ou por consulta quando necessário.

### Sinais no código (gatilhos)
- Campo de volta (`usuario`, `pedido`, `parent`) lido em um único ponto — ou em nenhum.
- Métodos `vincular`/`desvincular` que existem só para manter o campo coerente.
- `JSON.stringify` lançando `Converting circular structure to JSON`, ou `structuredClone` devolvendo um grafo enorme.
- `console.log` do modelo imprimindo árvore infinita; snapshot de teste impossível de ler.
- Closure/objeto de vida longa (cache, store de módulo) segurando referência a algo que deveria morrer.

### Antes
```ts
class Usuario {
  private readonly _assinaturas: Assinatura[] = [];
  constructor(readonly id: string, readonly nome: string) {}
  get assinaturas(): ReadonlyArray<Assinatura> { return this._assinaturas; }

  assinar(assinatura: Assinatura): void {
    this._assinaturas.push(assinatura);
    assinatura.vincular(this);
  }

  cancelar(assinaturaId: string): void {
    const a = this._assinaturas.find((x) => x.id === assinaturaId);
    if (a !== undefined) a.desvincular();
  }
}

class Assinatura {
  usuario: Usuario | undefined;
  constructor(readonly id: string, readonly plano: string) {}
  vincular(u: Usuario): void { this.usuario = u; }
  desvincular(): void { this.usuario = undefined; }
  // único uso do campo de volta
  descricao(): string { return `${this.usuario?.nome ?? '?'} — ${this.plano}`; }
}

const usuario = new Usuario('u-1', 'Ana');
usuario.assinar(new Assinatura('a-1', 'pro'));
// lança TypeError: Converting circular structure to JSON
const corpo = JSON.stringify(usuario);
```

### Depois
```ts
export type Usuario = { readonly id: string; readonly nome: string };
export type Assinatura = {
  readonly id: string;
  readonly usuarioId: string;
  readonly plano: string;
};

/** Estado normalizado: só ids atravessam a relação — nenhum ciclo. */
export type EstadoAssinaturas = {
  readonly usuarios: Readonly<Record<string, Usuario>>;
  readonly assinaturas: Readonly<Record<string, Assinatura>>;
};

/** O nome chega por parâmetro; a assinatura não guarda o dono. */
export const descrever = (a: Assinatura, nomeUsuario: string): string =>
  `${nomeUsuario} — ${a.plano}`;

export function descricoesDoUsuario(estado: EstadoAssinaturas, usuarioId: string): string[] {
  const usuario = estado.usuarios[usuarioId];
  if (usuario === undefined) return [];
  return Object.values(estado.assinaturas)
    .filter((a) => a.usuarioId === usuarioId)
    .map((a) => descrever(a, usuario.nome));
}

export function assinar(estado: EstadoAssinaturas, nova: Assinatura): EstadoAssinaturas {
  return { ...estado, assinaturas: { ...estado.assinaturas, [nova.id]: nova } };
}

const estado = assinar(estadoInicial, { id: 'a-1', usuarioId: 'u-1', plano: 'pro' });
// serializa, clona com structuredClone e cabe em cache/localStorage
const corpo = JSON.stringify(estado);
```

### Passos
1. Confirme uma das condições: a associação inversa é inutilizada, o objeto pode chegar por parâmetro, ou pode ser resolvido por id numa consulta/mapa.
2. Substitua cada uso do campo de volta por parâmetro explícito (`descrever(a, usuario.nome)`) ou por lookup no mapa normalizado.
3. Remova `vincular`/`desvincular` e toda a lógica que mantinha as duas pontas coerentes.
4. Apague o campo e troque a referência de objeto por `usuarioId: string` no lado que ainda precisa saber do dono.
5. Normalize o estado: `Record<id, Entidade>` por tipo de entidade, com as relações expressas por id — e derive as listas com `Object.values(...).filter(...)` ou por um índice memoizado.
6. Com a ciclicidade eliminada, promova as classes a objetos planos `readonly`, atualizados por spread.

### Ganhos
- Menos código de sincronização e menos acoplamento — cada entidade é usável isolada e fácil de montar em teste.
- Serialização, clonagem estrutural e persistência voltam a funcionar; logs e snapshots ficam legíveis.
- Referências de vida longa deixam de segurar grafos inteiros na memória.

### Quando NÃO aplicar
- O lado inverso é usado com frequência e redescobri-lo é caro (aí a técnica 7 é a correta).
- Não existe caminho alternativo: sem id, sem mapa e sem repositório para consultar.
- Grafo genuinamente cíclico do domínio (árvore com `parent` navegável em editor/DOM virtual), em que a navegação para cima é requisito.

### Nota TypeScript
Em front-end esta é quase sempre a direção certa, e o alvo é o estado **normalizado**: `Record<id, Entidade>` por tipo, relações por id, seletores derivando o resto. Isso é o que torna o estado serializável (`JSON.stringify` para `localStorage`, `structuredClone` para worker/IndexedDB), diffável em devtools e comparável por referência — atualizar uma assinatura por spread mantém as outras com a mesma referência, então `React.memo` continua funcionando. Se precisar mesmo de referência de volta em memória, use `WeakMap<Assinatura, Usuario>` do lado de fora das entidades: a relação existe, não polui o objeto e não impede coleta.

---

## 9. Encapsulate Field
**PT-BR:** Encapsular Campo · **Fonte:** https://refactoring.guru/encapsulate-field

### Problema
Um campo é público e mutável, então qualquer código pode alterá-lo sem passar pelas regras da classe, quebrando invariantes em pontos impossíveis de rastrear.

### Solução
Restringir a escrita e expor acesso controlado: campo privado `#`, getter, tipo `readonly` na saída e métodos de domínio para mudar o estado.

### Sinais no código (gatilhos)
- Objeto de módulo exportado com campos mutáveis (`export const carrinho = { total: 0 }`).
- Store cujo `setState`/objeto interno é exportado junto com o `state`.
- Consumidores atribuindo direto: `carrinho.total = 0`, `usuario.papel = 'admin'`.
- Invariante do objeto (limite, status válido, total coerente com os itens) quebrável de fora.
- `Object.assign(estado, patch)` aplicado por código de UI.

### Antes
```tsx
// carrinho-store.ts — estado mutável exportado: qualquer módulo escreve
export const carrinho = {
  itens: [] as ItemCarrinho[],
  cupom: null as string | null,
  totalCentavos: 0,
};

export function CupomInput() {
  const [texto, setTexto] = useState('');
  return (
    <div>
      <input value={texto} onChange={(e) => setTexto(e.target.value)} />
      <button
        onClick={() => {
          // regra de negócio na UI e invariante quebrável de fora
          carrinho.cupom = texto;
          carrinho.totalCentavos = Math.round(carrinho.totalCentavos * 0.9);
          carrinho.itens.push({ id: 'brinde', precoCentavos: 0, quantidade: 1 });
        }}
      >
        Aplicar
      </button>
    </div>
  );
}
```

### Depois
```ts
export type CarrinhoSnapshot = {
  readonly itens: ReadonlyArray<ItemCarrinho>;
  readonly cupom: string | null;
  readonly totalCentavos: number;
};
const DESCONTO_CUPOM = 0.1;

class Carrinho {
  #itens: ItemCarrinho[] = [];
  #cupom: string | null = null;
  /** Total é derivado: não existe como campo atribuível. */
  get totalCentavos(): number {
    const bruto = this.#itens.reduce((a, i) => a + i.precoCentavos * i.quantidade, 0);
    return this.#cupom === null ? bruto : Math.round(bruto * (1 - DESCONTO_CUPOM));
  }
  aplicarCupom(codigo: string): boolean {
    if (this.#cupom !== null || codigo.trim() === '') return false;
    this.#cupom = codigo.trim().toUpperCase();
    return true;
  }
  /** readonly é só compile-time; freeze é a garantia em runtime. */
  snapshot(): CarrinhoSnapshot {
    return Object.freeze({
      itens: Object.freeze([...this.#itens]),
      cupom: this.#cupom,
      totalCentavos: this.totalCentavos,
    });
  }
}

// fronteira de módulo: a instância mutável NÃO é exportada
const carrinho = new Carrinho();
export const aplicarCupom = (codigo: string): boolean => carrinho.aplicarCupom(codigo);
export const lerCarrinho = (): CarrinhoSnapshot => carrinho.snapshot();
```

### Passos
1. Torne o campo inacessível de fora: `#campo` na classe, variável de closure na factory, ou simplesmente **não exportar** o binding mutável do módulo.
2. Exponha leitura por getter ou por um snapshot de tipo `readonly`/`ReadonlyArray<T>`; nunca devolva o objeto interno mutável.
3. Localize toda escrita externa e substitua por métodos de domínio (`aplicarCupom`) que carregam a invariante.
4. Elimine campos que são derivados de outros (`totalCentavos`) — vire getter, e o estado inválido deixa de ser representável.
5. Congele o que sai (`Object.freeze`) quando o consumidor é código que você não controla, ou quando o objeto vai para cache/props compartilhadas.
6. Compile: cada erro de "cannot assign to read-only property" ou de propriedade inexistente aponta um acoplamento a remover.

### Ganhos
- Invariantes garantidas em um ponto único; a UI não consegue deixar o objeto em estado inválido.
- Dado e comportamento juntos, o que torna a unidade testável isoladamente.
- Abre espaço para log, validação e telemetria em cada mudança de estado.

### Quando NÃO aplicar
- Objeto de dados já imutável por construção (DTO validado, props, estado de reducer) — encapsular a mais só adiciona cerimônia.
- Objeto local de escopo curto dentro de uma função.
- Caminho de altíssima frequência onde `Object.freeze` e cópias defensivas custam medida (freeze impede otimizações do motor).

### Nota TypeScript
As ferramentas aqui são quatro, em ordem de força: **tipo `readonly`** (documenta e barra atribuição no build, mas é apagado — `as`, `any` implícito de biblioteca, ou JS puro escrevem à vontade), **getter sem setter** (leitura pública, escrita interna), **campo `#`** (privacidade real em runtime, invisível a `Object.keys`/spread/`JSON.stringify` — diferente de `private`, que é só compile-time e aparece no objeto) e **fronteira de módulo** (não exportar o mutável; nada alcança o que não está exportado). Quando a garantia precisa valer em runtime — objeto entregue a plugin, guardado em cache, ou compartilhado entre componentes —, `Object.freeze` é o único que realmente impede a escrita (silenciosa em modo solto, `TypeError` em `strict mode`/módulos ESM). `Object.freeze` é raso: congele também arrays e objetos aninhados, ou use um `deepFreeze` só em desenvolvimento.

---

## 10. Encapsulate Collection
**PT-BR:** Encapsular Coleção · **Fonte:** https://refactoring.guru/encapsulate-collection

### Problema
Uma classe ou módulo expõe sua coleção diretamente, permitindo que clientes adicionem, removam ou substituam itens sem que o dono saiba — e sem passar pelas regras da coleção.

### Solução
Devolver uma visão somente-leitura e oferecer operações próprias de adicionar/remover, que aplicam as regras e produzem o novo estado.

### Sinais no código (gatilhos)
- Campo público `T[]` ou getter que devolve o array interno (`return this._itens`).
- Clientes chamando `objeto.itens.push(...)`, `.splice(...)`, `.sort(...)` ou `itens.length = 0`.
- Setter que troca a coleção inteira (`setPermissoes(novas)`), permitindo aliasing do array do chamador.
- Regras sobre a coleção (limite, sem duplicatas, ordenação) repetidas em cada chamador.
- Bug de mutação compartilhada: dois objetos "diferentes" apontando para o mesmo array.

### Antes
```ts
export class Papel {
  // coleção mutável exposta: o cliente altera sem passar pelas regras
  permissoes: string[] = [];
  constructor(readonly id: string) {}
  setPermissoes(novas: string[]): void { this.permissoes = novas; }
}

export class PapelService {
  constructor(private readonly repo: PapelRepository) {}

  async conceder(papelId: string, permissao: string): Promise<void> {
    const papel = await this.repo.buscar(papelId);
    // limite e duplicidade checados aqui, e só aqui
    if (papel.permissoes.length < 20 && !papel.permissoes.includes(permissao)) {
      papel.permissoes.push(permissao);
    }
    await this.repo.salvar(papel);
  }

  async limpar(papelId: string): Promise<void> {
    const papel = await this.repo.buscar(papelId);
    papel.permissoes.length = 0; // ninguém avisa o dono
    await this.repo.salvar(papel);
  }
}
```

### Depois
```ts
const MAX_PERMISSOES = 20;
export type Papel = { readonly id: string; readonly permissoes: ReadonlyArray<string> };

/** Construção normaliza: sem duplicatas, dentro do limite, congelada. */
export function criarPapel(id: string, permissoes: ReadonlyArray<string> = []): Papel {
  return Object.freeze({
    id,
    permissoes: Object.freeze([...new Set(permissoes)].slice(0, MAX_PERMISSOES)),
  });
}

export const cheio = (papel: Papel): boolean => papel.permissoes.length >= MAX_PERMISSOES;

/** Adicionar/remover devolvem NOVO estado; ninguém muta o array interno. */
export function conceder(papel: Papel, permissao: string): Papel {
  if (cheio(papel) || papel.permissoes.includes(permissao)) return papel;
  return criarPapel(papel.id, [...papel.permissoes, permissao]);
}

export const revogar = (papel: Papel, permissao: string): Papel =>
  criarPapel(papel.id, papel.permissoes.filter((p) => p !== permissao));

export class PapelService {
  constructor(private readonly repo: PapelRepository) {}
  async conceder(papelId: string, permissao: string): Promise<void> {
    const papel = await this.repo.buscar(papelId);
    await this.repo.salvar(conceder(papel, permissao));
  }
  async limpar(papelId: string): Promise<void> {
    const papel = await this.repo.buscar(papelId);
    await this.repo.salvar(criarPapel(papel.id, []));
  }
}
```

### Passos
1. Esconda a coleção: campo `#itens: T[]` na classe, ou objeto `readonly` cuja lista só é produzida pela factory.
2. Exponha a leitura como `ReadonlyArray<T>` (ou `ReadonlySet`/`ReadonlyMap`) e **nunca** devolva a referência interna — retorne cópia (`[...this.#itens]`) ou o resultado congelado da construção.
3. Crie as operações de domínio: `conceder`/`revogar` devolvendo o novo estado (versão imutável), ou métodos `adicionar`/`remover` que retornam `boolean` de sucesso (versão com classe).
4. Troque o setter de coleção inteira por `substituirPor`/`criarPapel(id, novas)`, que **copia** a entrada — receber e guardar o array do chamador cria aliasing.
5. Mova para dentro dessas operações as regras que estavam nos chamadores (limite, duplicidade via `Set`, ordenação).
6. Congele o que sai (`Object.freeze`, ou `as const` para literais fixos) quando a coleção vai para cache, props ou código de terceiros.
7. Substitua no cliente todo `push`/`splice`/`sort` pelas novas operações; `sort` e `reverse` mutam no lugar — use `[...itens].sort(...)`.

### Ganhos
- Conteúdo protegido: só o dono altera a coleção, com validação centralizada.
- Operações expressivas (`cheio`, `conceder`) em vez de manipulação crua de array no chamador.
- Novo array por operação dá igualdade referencial útil: `React.memo`/`useMemo` só invalidam quando a lista realmente mudou.

### Quando NÃO aplicar
- Objeto de dados já imutável e construído de uma vez (DTO validado, `props`, item de lista renderizada).
- Builder local dentro de uma função, onde a mutação não escapa do escopo.
- Listas muito grandes com alta frequência de escrita: copiar a cada operação pesa — aí use estrutura persistente (Immer, `immutable`) ou mutação encapsulada em classe.

### Nota TypeScript
`ReadonlyArray<T>` remove `push`/`splice`/`sort` do tipo, mas é apagado na compilação: um `as string[]` (ou uma biblioteca em JS) volta a mutar o mesmo array. Duas consequências práticas: (1) sempre devolva **cópia** ou resultado de `Object.freeze`, porque `get itens() { return this.#itens }` tipado como `ReadonlyArray<T>` ainda entrega a referência interna; (2) para constantes literais, `as const` dá tuple `readonly` em compile-time e `Object.freeze` dá a garantia em runtime — use os dois quando a lista sai do módulo. Prefira `Set`/`Map` quando a regra é unicidade ou lookup por chave; `[...new Set(x)]` é a forma canônica de deduplicar preservando ordem.

---

## 11. Replace Magic Number with Symbolic Constant
**PT-BR:** Substituir Número Mágico por Constante Simbólica · **Fonte:** https://refactoring.guru/replace-magic-number-with-symbolic-constant

### Problema
O código usa números ou strings literais com significado de negócio, sem nome que explique o valor nem lugar único para mudá-lo.

### Solução
Declarar constantes com nome legível — e, para conjuntos fechados de valores textuais, um tipo — e usá-las em todas as ocorrências com aquele mesmo significado.

### Sinais no código (gatilhos)
- Literais dentro de regras: `>= 50000`, `* 0.9`, `=== 3`, `'OURO'`.
- O mesmo literal repetido no service, no componente e no teste.
- Comentário explicando o que o número significa.
- Timeouts, tamanhos de página, chaves de `localStorage` e nomes de evento inline.
- `string` livre onde só três valores são válidos, sem nenhum tipo que os liste.

### Antes
```ts
export class AvaliarPedidoService {
  constructor(private readonly repo: PedidoRepository) {}

  async avaliar(pedidoId: string): Promise<ResumoPedido> {
    const itens = await this.repo.itens(pedidoId);
    const total = itens.reduce((a, i) => a + i.precoCentavos * i.quantidade, 0);

    let faixa: string;
    if (total >= 50000) faixa = 'OURO';
    else if (total >= 20000) faixa = 'PRATA';
    else faixa = 'BRONZE';

    const tentativas = await this.repo.tentativasPagamento(pedidoId);
    return {
      total,
      faixa,
      freteCentavos: total >= 20000 ? 0 : 1990,
      tentativasRestantes: Math.max(3 - tentativas, 0),
      validadeDias: 180,
    };
  }
}
```

### Depois
```ts
export const FAIXAS_PEDIDO = ['bronze', 'prata', 'ouro'] as const;
export type FaixaPedido = (typeof FAIXAS_PEDIDO)[number];

export const LIMITES = {
  faixaOuroCentavos: 50_000,
  faixaPrataCentavos: 20_000,
  freteGratisCentavos: 20_000,
  fretePadraoCentavos: 1_990,
  maxTentativasPagamento: 3,
  validadeCotacaoDias: 180,
} as const;

export function faixaDe(totalCentavos: number): FaixaPedido {
  if (totalCentavos >= LIMITES.faixaOuroCentavos) return 'ouro';
  if (totalCentavos >= LIMITES.faixaPrataCentavos) return 'prata';
  return 'bronze';
}

export class AvaliarPedidoService {
  constructor(private readonly repo: PedidoRepository) {}

  async avaliar(pedidoId: string): Promise<ResumoPedido> {
    const itens = await this.repo.itens(pedidoId);
    const total = itens.reduce((a, i) => a + i.precoCentavos * i.quantidade, 0);
    const tentativas = await this.repo.tentativasPagamento(pedidoId);
    return {
      total,
      faixa: faixaDe(total),
      freteCentavos: total >= LIMITES.freteGratisCentavos ? 0 : LIMITES.fretePadraoCentavos,
      tentativasRestantes: Math.max(LIMITES.maxTentativasPagamento - tentativas, 0),
      validadeDias: LIMITES.validadeCotacaoDias,
    };
  }
}
```

### Passos
1. Dê nome de negócio ao valor: `const` no módulo para um valor isolado, ou um objeto de constantes `as const` quando forem vários do mesmo assunto.
2. Localize todas as ocorrências do literal e confirme, uma por uma, que o significado é o mesmo — `20_000` como faixa de fidelidade e como piso de frete grátis são conceitos diferentes que hoje coincidem; unificá-los cria acoplamento falso.
3. Para literais textuais de conjunto fechado, declare a lista com `as const` e derive o tipo: `type FaixaPedido = (typeof FAIXAS_PEDIDO)[number]`. Isso dá o valor em runtime (iterável, validável) e a união em compile-time de uma só declaração.
4. Substitua as ocorrências, incluindo as dos testes — importar a constante evita que o teste reintroduza o literal e passe a testar o número, não a regra.
5. Use separador numérico (`50_000`) e sufixo de unidade no nome (`...Centavos`, `...Dias`, `...Ms`) para eliminar a ambiguidade que gera bug de escala.
6. O que é configuração (URL, feature flag, timeout por ambiente) não vira constante no código: vai para variável de ambiente validada por schema na inicialização.

### Ganhos
- A constante documenta o valor e centraliza a mudança de regra.
- Elimina duplicação e substituições acidentais de valores homônimos.
- União de literais dá autocomplete, exaustividade em `switch` e erro de compilação para valor inválido.

### Quando NÃO aplicar
- Números com significado óbvio no contexto: `0`, `1`, `length - 1`, divisores triviais.
- Valor usado uma única vez e já explicado pelo nome da variável local.
- Configuração por ambiente — constante hardcoded aí é o problema errado resolvido.

### Nota TypeScript
Para conjuntos fechados, prefira **união de literais + `as const`** a `enum`. Motivos concretos: (1) `enum` gera um objeto em runtime e **não é apagado de forma previsível** — atrapalha `isolatedModules`, transpilação por arquivo (esbuild/SWC/Vite) e tree-shaking; (2) `const enum` é apagado, mas é proibido sob `isolatedModules` e não pode ser consumido de fora do pacote com segurança; (3) `enum` numérico é *unsound*: até TS 5, `let f: Faixa = 99` compilava, e o valor numérico não tem significado nem sobrevive a reordenação em dados persistidos; (4) o valor de um `enum` de string não é atribuível a partir da string literal equivalente (`const f: Faixa = 'ouro'` falha), o que atrita com JSON de API e com Zod. `['bronze','prata','ouro'] as const` dá união para o tipo, array para iterar/renderizar e `z.enum(FAIXAS_PEDIDO)` para validar na fronteira. Use `satisfies` para checar que um objeto de constantes cobre todas as chaves sem perder o tipo literal.

---

## 12. Replace Type Code with Class
**PT-BR:** Substituir Código de Tipo por Classe · **Fonte:** https://refactoring.guru/replace-type-code-with-class

### Problema
Uma entidade tem um campo de type code primitivo (`string`/`number`) cujos valores não controlam o fluxo, mas também não têm validação nem checagem de tipo — qualquer string entra.

### Solução
Criar um tipo dedicado para o conceito e usá-lo no lugar do primitivo, com os dados atrelados a cada valor guardados junto do tipo. Em TypeScript, o primeiro degrau é **união de literais + mapa de metadados**.

### Sinais no código (gatilhos)
- Constantes soltas (`const TIPO_EXPRESSA = 'E'`) acompanhando um campo `tipo: string`.
- Comparação com literal só para exibir rótulo: `if (entrega.tipoCodigo === 'E')`.
- `switch` com `default: return 'Desconhecido'` porque o tipo permite qualquer string.
- Mapeamento código→descrição duplicado em componentes diferentes.
- Erro de código inválido só aparece em runtime, e longe da borda que o recebeu.

### Antes
```ts
export const TIPO_ENTREGA_PADRAO = 'P';
export const TIPO_ENTREGA_EXPRESSA = 'E';
export const TIPO_ENTREGA_RETIRADA = 'R';

export type Entrega = { id: string; enderecoId: string; tipoCodigo: string };

export function rotuloTipo(entrega: Entrega): string {
  switch (entrega.tipoCodigo) {
    case TIPO_ENTREGA_PADRAO: return 'Padrão';
    case TIPO_ENTREGA_EXPRESSA: return 'Expressa';
    case TIPO_ENTREGA_RETIRADA: return 'Retirada na loja';
    default: return 'Desconhecido'; // aceita qualquer string
  }
}

export function prazoDias(entrega: Entrega): number {
  if (entrega.tipoCodigo === 'E') return 1;
  if (entrega.tipoCodigo === 'R') return 0;
  return 5;
}

export const mapear = (dto: EntregaDto): Entrega => ({
  id: dto.id,
  enderecoId: dto.enderecoId,
  // nada impede 'e', 'expressa' ou '' chegarem aqui
  tipoCodigo: dto.tipo,
});
```

### Depois
```ts
export const TIPOS_ENTREGA = ['padrao', 'expressa', 'retirada'] as const;
export type TipoEntrega = (typeof TIPOS_ENTREGA)[number];

type MetaEntrega = { readonly codigo: string; readonly rotulo: string; readonly prazoDias: number };

/** Os dados atrelados ao código vivem junto do tipo, não em switches de UI. */
export const META_ENTREGA = {
  padrao: { codigo: 'P', rotulo: 'Padrão', prazoDias: 5 },
  expressa: { codigo: 'E', rotulo: 'Expressa', prazoDias: 1 },
  retirada: { codigo: 'R', rotulo: 'Retirada na loja', prazoDias: 0 },
} as const satisfies Record<TipoEntrega, MetaEntrega>;

export type Entrega = {
  readonly id: string;
  readonly enderecoId: string;
  readonly tipo: TipoEntrega;
};

export const rotuloTipo = (e: Entrega): string => META_ENTREGA[e.tipo].rotulo;
export const prazoDias = (e: Entrega): number => META_ENTREGA[e.tipo].prazoDias;

const POR_CODIGO: Readonly<Record<string, TipoEntrega>> = Object.fromEntries(
  TIPOS_ENTREGA.map((tipo) => [META_ENTREGA[tipo].codigo, tipo]),
);

/** Validação na borda: código inválido não entra no domínio. */
export function parseTipoEntrega(codigo: string): Result<TipoEntrega, string> {
  const tipo = POR_CODIGO[codigo];
  return tipo === undefined
    ? { ok: false, error: `Tipo de entrega desconhecido: ${codigo}` }
    : { ok: true, value: tipo };
}
```

### Passos
1. Declare os valores do conjunto com `as const` e derive a união (`type TipoEntrega = (typeof TIPOS_ENTREGA)[number]`).
2. Mova os dados atrelados a cada valor (código persistido, rótulo, prazo) para um `Record<Tipo, Meta>` com `as const satisfies` — `satisfies` garante que nenhum valor da união ficou sem entrada e preserva os tipos literais.
3. Troque os `switch`/`if` de leitura por acesso ao mapa (`META_ENTREGA[e.tipo].rotulo`).
4. Crie o parser da borda (`parseTipoEntrega`) devolvendo `Result`, e construa o índice inverso código→tipo a partir do próprio mapa, sem duplicar a tabela.
5. Troque o tipo do campo na entidade e aplique o parser no mapeamento DTO→domínio (com Zod, `z.enum(TIPOS_ENTREGA)`); remova as constantes soltas.
6. Se depois o tipo passar a **decidir comportamento** com dados próprios por variante, suba para a técnica 13.

### Ganhos
- Validação na borda: valor inválido nunca entra no domínio.
- Autocomplete e checagem em tempo de compilação; `default: 'Desconhecido'` desaparece.
- Rótulo, prazo e código persistido ficam em uma tabela só, em vez de espalhados por componentes.

### Quando NÃO aplicar
- O type code **controla comportamento** com dados diferentes por variante — vá para a técnica 13.
- O comportamento varia e muda em runtime, ou precisa ser injetado — técnica 14.
- Conjunto aberto, definido pelo servidor e estendido com frequência: uma união fixa obriga deploy do front a cada valor novo; use `string` validada com fallback explícito (`'desconhecido'`) e trate esse caso na UI.

### Nota TypeScript
Este é o primeiro degrau da progressão de type codes: **união de literais + `Record` de metadados** cobre "conjunto fechado, sem comportamento próprio, dados uniformes por valor". Prefira essa forma a `enum` (ver a nota da técnica 11) e a uma classe com instâncias constantes: a união é comparável por `===`, serializa direto para JSON, atravessa `structuredClone` e vira `z.enum(...)` na validação sem nenhuma conversão. O `as const satisfies Record<TipoEntrega, Meta>` é o detalhe que torna o mapa exaustivo — acrescentar `'agendada'` à união quebra a compilação exatamente no mapa que precisa ser atualizado.

---

## 13. Replace Type Code with Subclasses
**PT-BR:** Substituir Código de Tipo por Subclasses · **Fonte:** https://refactoring.guru/replace-type-code-with-subclasses

### Problema
O type code afeta diretamente o comportamento: seus valores disparam ramos diferentes em condicionais espalhadas, e cada variante precisa de campos que não fazem sentido para as outras.

### Solução
Criar uma variante por valor do type code, cada uma com exatamente os seus dados, e trocar as condicionais por despacho sobre a variante. Em TypeScript, isso é uma **discriminated union**.

### Sinais no código (gatilhos)
- `switch (tipo)` repetido em mais de um lugar, com ramos diferentes em cada um.
- Campos `| null`/opcionais que só valem para alguns valores do tipo (`parcelas` só para cartão).
- `if (x === null) throw` para "provar" ao compilador o que o tipo já implicava.
- `default: throw new Error('Tipo inválido')` — estado inválido representável no tipo.
- Adicionar um tipo novo exige caçar todos os `switch` manualmente.

### Antes
```ts
type MetodoPagamento = {
  readonly id: string;
  readonly tipo: string; // 'CARTAO' | 'PIX' | 'BOLETO'
  readonly bandeira: string | null;
  readonly parcelas: number | null;
  readonly chavePix: string | null;
  readonly linhaDigitavel: string | null;
};

function taxaCentavos(m: MetodoPagamento, valorCentavos: number): number {
  switch (m.tipo) {
    case 'CARTAO':
      return Math.round(valorCentavos * 0.029) + (m.parcelas ?? 1) * 50;
    case 'PIX':
      return 0;
    case 'BOLETO':
      return 349;
    default:
      throw new Error(`Tipo inválido: ${m.tipo}`);
  }
}

function prazoLiquidacaoDias(m: MetodoPagamento): number {
  switch (m.tipo) {
    case 'CARTAO': return 30;
    case 'PIX': return 0;
    case 'BOLETO': return 3;
    default: throw new Error(`Tipo inválido: ${m.tipo}`);
  }
}
```

### Depois
```ts
export type MetodoPagamento =
  | { readonly tipo: 'cartao'; readonly id: string; readonly bandeira: string; readonly parcelas: number }
  | { readonly tipo: 'pix'; readonly id: string; readonly chavePix: string }
  | { readonly tipo: 'boleto'; readonly id: string; readonly linhaDigitavel: string };

export function assertNever(x: never): never {
  throw new Error(`Caso não tratado: ${JSON.stringify(x)}`);
}

const TAXA_CARTAO = 0.029;
const TAXA_POR_PARCELA_CENTAVOS = 50;
const TAXA_BOLETO_CENTAVOS = 349;

export function taxaCentavos(m: MetodoPagamento, valorCentavos: number): number {
  switch (m.tipo) {
    case 'cartao':
      // parcelas existe e é number: garantido pelo estreitamento
      return Math.round(valorCentavos * TAXA_CARTAO) + m.parcelas * TAXA_POR_PARCELA_CENTAVOS;
    case 'pix':
      return 0;
    case 'boleto':
      return TAXA_BOLETO_CENTAVOS;
    default:
      return assertNever(m); // método novo => erro de compilação aqui
  }
}

export function prazoLiquidacaoDias(m: MetodoPagamento): number {
  switch (m.tipo) {
    case 'cartao': return 30;
    case 'pix': return 0;
    case 'boleto': return 3;
    default: return assertNever(m);
  }
}
```

### Passos
1. Escolha o campo discriminante e mantenha o mesmo nome em todas as variantes (`tipo`, `kind`, `status`), com valor literal único.
2. Escreva uma variante por valor do type code, contendo **só** os campos que valem para ela — os `| null` condicionais desaparecem.
3. Substitua os `switch (m.tipo)` de string livre pelo `switch` sobre o discriminante e adicione `default: return assertNever(m)`.
4. Onde o comportamento é uniforme, extraia funções que recebem a união; onde é específico, deixe o estreitamento dar acesso aos campos da variante.
5. Valide na fronteira com `z.discriminatedUnion('tipo', [...])` para que o JSON da API só entre no domínio como uma variante válida.
6. Se o valor do discriminante muda durante a vida do objeto, ou a política precisa ser injetada/trocada em runtime, use a técnica 14.

### Ganhos
- Elimina campos opcionais que só valiam para alguns tipos: estado inválido deixa de ser representável.
- Variante nova = o compilador aponta **todos** os pontos que precisam tratá-la (via `assertNever`).
- Tipagem forte: impossível montar um pagamento de cartão sem `parcelas`.

### Quando NÃO aplicar
- O type code só carrega rótulo e dados uniformes, sem decidir comportamento — união de literais + mapa (12) basta.
- O valor muda durante a vida do objeto, ou a política é escolhida por configuração/feature flag — técnica 14.
- Duas dimensões de variação independentes: a união cresce em produto cartesiano; componha (`tipo` × `politica`) em vez de multiplicar variantes.

### Nota TypeScript
Discriminated union é a forma canônica — não hierarquia de classes. Requisitos: discriminante de tipo literal (não `string`), o mesmo nome em todas as variantes e `strictNullChecks` ligado. O `switch` exaustivo com `default: return assertNever(x)` é a rede de segurança: se uma variante nova não for tratada, `x` deixa de ser `never` e o build falha no ponto exato. Um `if/else` sem `else` final não dá essa garantia — prefira `switch`, ou um `Record<Tipo, Handler>` (técnica 14), que também é checado por exaustividade. As variantes são objetos planos: serializam, clonam, entram em cache e funcionam como estado de reducer e como `UiState` (`{ status: 'carregando' } | { status: 'erro'; mensagem: string }`).

---

## 14. Replace Type Code with State/Strategy
**PT-BR:** Substituir Código de Tipo por State/Strategy · **Fonte:** https://refactoring.guru/replace-type-code-with-state-strategy

### Problema
O type code afeta o comportamento, mas o valor muda durante a vida do objeto, ou a política precisa ser escolhida de fora (configuração, feature flag, teste) — então não dá para amarrar o comportamento à forma do dado.

### Solução
Extrair o comportamento variável para objetos de estado/estratégia e escolher a estratégia pelo type code, trocando-a quando o tipo mudar. Em TypeScript, um **`Record<Tipo, Handler>` injetado**.

### Sinais no código (gatilhos)
- Campo de tipo mutável (`public situacao: string`) usado em `switch` que decide comportamento.
- `switch` sobre o mesmo type code em módulos diferentes (cálculo, validação, notificação).
- Transição de estado escrita como string literal (`return 'pausada'`) sem nenhuma máquina explícita.
- Necessidade de variar a política por ambiente, feature flag ou fake de teste, sem tocar na entidade.

### Antes
```ts
export class Assinatura {
  constructor(
    readonly id: string,
    readonly usuarioId: string,
    public situacao: string, // 'ativa' | 'pausada' | 'cancelada'
  ) {}

  podeCancelar(): boolean {
    switch (this.situacao) {
      case 'ativa':
      case 'pausada':
        return true;
      case 'cancelada':
        return false;
      default:
        throw new Error('Situação inválida');
    }
  }

  multaCentavos(mensalidadeCentavos: number): number {
    switch (this.situacao) {
      case 'ativa': return Math.round(mensalidadeCentavos * 0.3);
      case 'pausada': return 0;
      case 'cancelada': return 0;
      default: throw new Error('Situação inválida');
    }
  }

  pausar(): void {
    this.situacao = this.situacao === 'ativa' ? 'pausada' : this.situacao;
  }
}
```

### Depois
```ts
export type SituacaoAssinatura = 'ativa' | 'pausada' | 'cancelada';
export type PoliticaSituacao = {
  readonly podeCancelar: boolean;
  readonly multaCentavos: (mensalidadeCentavos: number) => number;
  readonly aoPausar: () => SituacaoAssinatura;
};
export type Politicas = Readonly<Record<SituacaoAssinatura, PoliticaSituacao>>;
const MULTA_ATIVA = 0.3;

/** Um handler por valor do type code: o mapa é a hierarquia, e é injetável. */
export const POLITICAS_PADRAO: Politicas = {
  ativa: {
    podeCancelar: true,
    multaCentavos: (m) => Math.round(m * MULTA_ATIVA),
    aoPausar: () => 'pausada',
  },
  pausada: { podeCancelar: true, multaCentavos: () => 0, aoPausar: () => 'pausada' },
  cancelada: { podeCancelar: false, multaCentavos: () => 0, aoPausar: () => 'cancelada' },
};

export class Assinatura {
  #situacao: SituacaoAssinatura;
  constructor(
    readonly id: string,
    situacaoInicial: SituacaoAssinatura,
    private readonly politicas: Politicas = POLITICAS_PADRAO,
  ) {
    this.#situacao = situacaoInicial;
  }
  get situacao(): SituacaoAssinatura { return this.#situacao; }
  get politica(): PoliticaSituacao { return this.politicas[this.#situacao]; }
  pausar(): void { this.#situacao = this.politica.aoPausar(); }
  multa(mensalidadeCentavos: number): number { return this.politica.multaCentavos(mensalidadeCentavos); }
}
```

### Passos
1. Encapsule o campo de type code antes de mexer no comportamento: `#situacao` + getter, escrita só por método.
2. Declare o contrato da estratégia (`PoliticaSituacao`) com o comportamento que varia — propriedades para valores constantes, funções para cálculo e transição.
3. Monte o mapa `Record<Tipo, Handler>` com uma entrada por valor; o `Record` sobre a união é checado por exaustividade, então valor novo sem handler não compila.
4. Injete o mapa por construtor (ou por factory de closure) com um default — é isso que permite trocar a política em teste, por feature flag ou por tenant, sem tocar na entidade.
5. Troque cada `switch` por acesso ao handler (`this.politica.multaCentavos(m)`).
6. Faça a transição de estado passar pelo próprio handler (`aoPausar()`), mantendo a escrita privada; a máquina de estados fica legível em um lugar só.
7. Se algum handler precisar de estado próprio, troque a entrada do mapa por uma factory (`(deps) => Handler`) em vez de acrescentar campos na entidade.

### Ganhos
- Suporta type code que **muda em runtime**: troca-se o handler, não a forma do objeto.
- Valor novo = entrada nova no mapa, sem editar as existentes; o compilador cobra a entrada faltante.
- Política injetável é testável com fake e substituível por configuração.
- As transições ficam explícitas e centralizadas no mapa.

### Quando NÃO aplicar
- Type code simples, sem comportamento: um mapa de metadados (12) resolve com muito menos cerimônia.
- Tipo fixo na construção e com dados próprios por variante: discriminated union (13) é mais direto e dá estreitamento de campos.
- Apenas duas variantes triviais — um `boolean` ou um `if` local resolve.
- A política é pura e sem estado e ninguém precisa injetá-la: uma função com `switch` exaustivo já basta.

### Nota TypeScript
**Critério de escolha entre os três degraus.** (1) **União de literais + `Record` de metadados** (12): conjunto fechado, dados uniformes, sem comportamento próprio — é o default, e o mais barato. (2) **Discriminated union + `switch` exaustivo** (13): o tipo decide comportamento **e** cada variante tem campos diferentes, fixos na construção; o `assertNever` garante que nada foi esquecido. (3) **`Record<Tipo, Handler>` injetado** (14): a política varia em runtime, precisa ser trocada por configuração/flag/tenant, ou substituída por fake em teste; e é também a saída quando existem duas dimensões de variação, porque mapas se compõem e uniões não. Nas três formas o dado persistido continua sendo a `string` do código — reconstrua o estado no mapper de borda (`z.enum`), nunca guarde índice numérico. Em React, o mapa de handlers costuma virar `Record<Tipo, ComponentType<Props>>`, e injetá-lo por Context é o equivalente da estratégia trocável.

---

## 15. Replace Subclass with Fields
**PT-BR:** Substituir Subclasse por Campos · **Fonte:** https://refactoring.guru/replace-subclass-with-fields

### Problema
Subclasses diferem apenas pelos valores constantes que seus métodos devolvem, sem comportamento próprio — uma hierarquia inteira só para carregar dados fixos.

### Solução
Substituir esses métodos por campos em um único tipo, com uma tabela de metadados por variante, e apagar as subclasses.

### Sinais no código (gatilhos)
- Subclasse cujo corpo é só `metodo() { return 'constante'; }`.
- Hierarquia sem nenhum método com lógica real, e nenhum `instanceof` que dependa de estrutura diferente.
- Factory com `if`/`switch` decidindo qual subclasse instanciar a partir de um código.
- Instâncias criadas com `new` só para ler valores (`new PlanoPro().maxUsuarios()`).
- As "variantes" nunca são serializadas — porque perderiam a classe no `JSON.parse`.

### Antes
```ts
abstract class Plano {
  abstract codigo(): string;
  abstract rotulo(): string;
  abstract precoMensalCentavos(): number;
  abstract maxUsuarios(): number;
}

class PlanoGratis extends Plano {
  codigo(): string { return 'gratis'; }
  rotulo(): string { return 'Grátis'; }
  precoMensalCentavos(): number { return 0; }
  maxUsuarios(): number { return 1; }
}

class PlanoPro extends Plano {
  codigo(): string { return 'pro'; }
  rotulo(): string { return 'Pro'; }
  precoMensalCentavos(): number { return 4900; }
  maxUsuarios(): number { return 10; }
}

class PlanoEmpresarial extends Plano {
  codigo(): string { return 'empresarial'; }
  rotulo(): string { return 'Empresarial'; }
  precoMensalCentavos(): number { return 29900; }
  maxUsuarios(): number { return 200; }
}

function planoDe(codigo: string): Plano {
  if (codigo === 'pro') return new PlanoPro();
  if (codigo === 'empresarial') return new PlanoEmpresarial();
  return new PlanoGratis();
}
```

### Depois
```tsx
export const CODIGOS_PLANO = ['gratis', 'pro', 'empresarial'] as const;
export type CodigoPlano = (typeof CODIGOS_PLANO)[number];

export type Plano = {
  readonly codigo: CodigoPlano;
  readonly rotulo: string;
  readonly precoMensalCentavos: number;
  readonly maxUsuarios: number;
};

/** Uma hierarquia inteira virou uma tabela de configuração. */
export const PLANOS = {
  gratis: { codigo: 'gratis', rotulo: 'Grátis', precoMensalCentavos: 0, maxUsuarios: 1 },
  pro: { codigo: 'pro', rotulo: 'Pro', precoMensalCentavos: 4_900, maxUsuarios: 10 },
  empresarial: { codigo: 'empresarial', rotulo: 'Empresarial', precoMensalCentavos: 29_900, maxUsuarios: 200 },
} as const satisfies Record<CodigoPlano, Plano>;

const ehCodigoPlano = (v: string): v is CodigoPlano => (CODIGOS_PLANO as ReadonlyArray<string>).includes(v);

export const planoDe = (codigo: string): Plano =>
  ehCodigoPlano(codigo) ? PLANOS[codigo] : PLANOS.gratis;

export function TabelaPlanos({ codigoAtual }: { codigoAtual: string }) {
  const atual = planoDe(codigoAtual);
  return (
    <ul>
      {CODIGOS_PLANO.map((c) => (
        <li key={c} aria-current={c === atual.codigo}>
          {PLANOS[c].rotulo} — até {PLANOS[c].maxUsuarios} usuários
        </li>
      ))}
    </ul>
  );
}
```

### Passos
1. Substitua as construções diretas de subclasse por uma única factory (`planoDe`), e redirecione todas as chamadas para ela.
2. Declare o tipo único com um campo `readonly` para cada método constante que as subclasses sobrescreviam.
3. Monte a tabela `Record<Codigo, Tipo>` com `as const satisfies`: cada variante passa a ser uma entrada de dados, e o compilador exige que nenhuma falte.
4. Troque cada chamada de método por leitura de campo (`plano.maxUsuarios()` → `plano.maxUsuarios`).
5. Crie o type guard do código (`ehCodigoPlano`) para validar o valor que vem do servidor, com fallback explícito.
6. Se algum construtor de subclasse fazia trabalho extra, mova esse trabalho para a factory.
7. Apague as subclasses e a classe base.

### Ganhos
- Remove uma hierarquia inteira sem perder informação; muito menos código para manter.
- A tabela é iterável (`CODIGOS_PLANO.map`) — renderizar comparativo de planos deixa de exigir uma lista paralela.
- Os valores são objetos planos: serializam, entram em cache, viram props e sobrevivem a `JSON.parse`.

### Quando NÃO aplicar
- Alguma variante tem comportamento real, não apenas constantes — mantenha o despacho (13 ou 14).
- Variantes com **conjuntos de campos diferentes**: o alvo é discriminated union (13), não campos opcionais em um tipo único.
- Conjunto de valores definido em runtime pelo backend (planos criados no painel): a tabela fixa no bundle obriga deploy; nesse caso o dado vem da API e o tipo descreve o formato, não os valores.

### Nota TypeScript
O destino natural aqui é **objeto de configuração / `Record` de metadados**, não classe nem `enum`: `as const satisfies Record<CodigoPlano, Plano>` dá tipos literais, exaustividade verificada e um objeto real para iterar e renderizar. Ganhos práticos que a hierarquia não tinha: os valores passam por `JSON.stringify`/`structuredClone` sem perder identidade (uma instância de classe volta de `JSON.parse` como objeto solto, sem métodos), a igualdade referencial das entradas do mapa é estável — `PLANOS.pro` é sempre a mesma referência, boa para `useMemo`/`React.memo` —, e nada é instanciado por leitura. Persista o `codigo` (string), nunca índice ou nome de classe, e valide na entrada com `z.enum(CODIGOS_PLANO)` mais fallback explícito.
