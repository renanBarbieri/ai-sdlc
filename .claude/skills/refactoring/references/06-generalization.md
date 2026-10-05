# Dealing with Generalization — Lidando com Generalização

> Grupo 6 de 6 · 12 técnicas · Fonte: refactoring.guru/refactoring/techniques

## Quando este grupo se aplica
- **Leia este grupo com um filtro de TypeScript.** Herança é muito menos usada aqui do que em Java/Kotlin: em React não existe componente de classe idiomático, e no back-end composição de funções e objetos domina. Boa parte deste grupo se traduz em **composição, funções de alta ordem e uniões discriminadas**.
- Hierarquias profundas são exceção, não regra. Onde elas legitimamente aparecem: **classes de erro** (`Error` → `AppError` → `ValidationError`), **entidades/models de ORM**, **SDKs e frameworks de terceiros** que você estende, e algumas classes de serviço com estado compartilhado.
- Ainda assim, a técnica continua valendo: quando você **tem** uma hierarquia — sua ou de biblioteca — vale saber subir, descer, extrair e desmontar corretamente. Cada técnica abaixo mostra a mecânica na hierarquia e, quando existe, a rota idiomática sem herança.
- Sintomas que trazem você para cá: subclasses irmãs com campos/inicialização quase idênticos; base "sacola" com membro que só um ramo usa; campo de tipo (`ehTrial`, `tipo: string`) governando `if`/`switch` espalhados; serviço que depende de classe concreta de infra e só aceita mock em teste; `extends` usado para reuso de código, não para "é-um"; hierarquia com uma única subclasse ou base anêmica.

## Índice
1. [Pull Up Field](#1-pull-up-field)
2. [Pull Up Method](#2-pull-up-method)
3. [Pull Up Constructor Body](#3-pull-up-constructor-body)
4. [Push Down Field](#4-push-down-field)
5. [Push Down Method](#5-push-down-method)
6. [Extract Subclass](#6-extract-subclass)
7. [Extract Superclass](#7-extract-superclass)
8. [Extract Interface](#8-extract-interface)
9. [Collapse Hierarchy](#9-collapse-hierarchy)
10. [Form Template Method](#10-form-template-method)
11. [Replace Inheritance with Delegation](#11-replace-inheritance-with-delegation)
12. [Replace Delegation with Inheritance](#12-replace-delegation-with-inheritance)

---

## 1. Pull Up Field
**PT-BR:** Subir Campo · **Fonte:** https://refactoring.guru/pull-up-field

### Problema
Duas ou mais subclasses declaram o mesmo campo, com o mesmo propósito.

### Solução
Remover o campo das subclasses e declará-lo na superclasse, recebendo-o pelo construtor (parameter property `protected readonly`).

### Sinais no código (gatilhos)
- Mesmo dado com nomes diferentes nas subclasses (`traceId`, `requestId`, `correlationId`).
- Método idêntico nas subclasses só porque cada uma tem sua própria cópia do campo.
- O campo é lido pelo mesmo tipo de lógica em todas as subclasses.
- Cada subclasse repete a mesma atribuição `this.x = x` logo depois do `super(...)`.

### Antes
```ts
abstract class AppError extends Error {
  abstract readonly status: number;
}

class ValidationError extends AppError {
  readonly status = 400;
  private readonly traceId: string;

  constructor(campo: string, traceId: string) {
    super(`campo inválido: ${campo}`);
    this.traceId = traceId;
  }

  logLine(): string {
    return `[${this.traceId}] ${this.message}`;
  }
}

class NotFoundError extends AppError {
  readonly status = 404;
  private readonly requestId: string;

  constructor(recurso: string, requestId: string) {
    super(`${recurso} não encontrado`);
    this.requestId = requestId;
  }

  logLine(): string {
    return `[${this.requestId}] ${this.message}`;
  }
}
```

### Depois
```ts
abstract class AppError extends Error {
  abstract readonly status: number;

  constructor(
    message: string,
    protected readonly traceId: string,
  ) {
    super(message);
  }

  logLine(): string {
    return `[${this.traceId}] ${this.message}`;
  }
}

class ValidationError extends AppError {
  readonly status = 400;

  constructor(campo: string, traceId: string) {
    super(`campo inválido: ${campo}`, traceId);
  }
}

class NotFoundError extends AppError {
  readonly status = 404;

  constructor(recurso: string, traceId: string) {
    super(`${recurso} não encontrado`, traceId);
  }
}
```

### Passos
1. Confirme que o campo tem o mesmo propósito em todas as subclasses (não apenas o mesmo tipo).
2. Se os nomes divergem, unifique com Rename e atualize todas as referências.
3. Declare o campo na base como parameter property: `protected readonly` se as subclasses leem, `private readonly` se só a base usa.
4. Nas subclasses, remova a declaração do campo e passe o valor em `super(...)`.
5. Recompile. Com `readonly`, o compilador aponta qualquer subclasse que dependia de reatribuição.
6. Se o campo era só dado compartilhado (sem comportamento na base), avalie a rota sem herança: um `type` comum e uma função que o recebe como primeiro parâmetro.

### Ganhos
- Elimina duplicação de estado; um só ponto de verdade.
- Habilita Pull Up Method para os métodos que dependiam do campo.
- O campo na base documenta o contrato compartilhado da hierarquia.

### Quando NÃO aplicar
- Os campos têm o mesmo nome mas semântica diferente (`id` do DTO vs `id` do banco).
- Subir o campo forçaria a base a conhecer detalhe de uma única subclasse.
- As subclasses precisam de tipos diferentes para o campo — use um genérico na base (`class Base<T>`) em vez de subir um `unknown`.

### Nota TypeScript
Parameter properties (`constructor(protected readonly traceId: string)`) tornam o passo quase mecânico: um lugar declara e atribui. Atenção a `useDefineForClassFields`/target moderno: **inicializadores de campo da subclasse rodam depois do `super()`** e sobrescrevem o que a base atribuiu — se a base preenche `this.x`, a subclasse não pode declarar `x` com inicializador. Prefira `readonly` a campos mutáveis herdados; para estado observável compartilhado, exponha um getter e mantenha o valor em `#priv`, nunca um campo público mutável.

---

## 2. Pull Up Method
**PT-BR:** Subir Método · **Fonte:** https://refactoring.guru/pull-up-method

### Problema
Subclasses contêm métodos que fazem o mesmo trabalho, com corpo idêntico ou quase idêntico.

### Solução
Tornar os métodos idênticos (mesmo nome, mesma assinatura) e movê-los para a superclasse — ou, se não dependem de estado, para um módulo compartilhado.

### Sinais no código (gatilhos)
- Corpos idênticos linha a linha em duas classes irmãs (dois gateways, dois repositórios).
- Mesma lógica com nomes diferentes (`mensagemDeErro`, `traduzErro`, `textoDoErro`).
- Subclasse que sobrescreve método da base para fazer essencialmente o mesmo.
- Copiar/colar entre arquivos irmãos aparece no diff de todo PR do módulo.

### Antes
```ts
abstract class GatewayBase {
  constructor(protected readonly http: HttpClient) {}

  abstract cobrar(valorEmCentavos: number): Promise<Result<string, string>>;
}

class GatewayStripe extends GatewayBase {
  async cobrar(valorEmCentavos: number): Promise<Result<string, string>> {
    try {
      const r = await this.http.post('/stripe/charges', { valorEmCentavos });
      return { ok: true, value: r.id };
    } catch (e: unknown) {
      return { ok: false, error: this.mensagemDeErro(e) };
    }
  }

  private mensagemDeErro(e: unknown): string {
    return e instanceof TypeError ? 'Sem conexão' : 'Falha no pagamento';
  }
}

class GatewayPix extends GatewayBase {
  async cobrar(valorEmCentavos: number): Promise<Result<string, string>> {
    try {
      const r = await this.http.post('/pix/charges', { valorEmCentavos });
      return { ok: true, value: r.id };
    } catch (e: unknown) {
      return { ok: false, error: this.traduzErro(e) };
    }
  }

  private traduzErro(e: unknown): string {
    return e instanceof TypeError ? 'Sem conexão' : 'Falha no pagamento';
  }
}
```

### Depois
```ts
abstract class GatewayBase {
  constructor(protected readonly http: HttpClient) {}

  abstract cobrar(valorEmCentavos: number): Promise<Result<string, string>>;

  protected mensagemDeErro(e: unknown): string {
    return e instanceof TypeError ? 'Sem conexão' : 'Falha no pagamento';
  }
}

class GatewayStripe extends GatewayBase {
  async cobrar(valorEmCentavos: number): Promise<Result<string, string>> {
    try {
      const r = await this.http.post('/stripe/charges', { valorEmCentavos });
      return { ok: true, value: r.id };
    } catch (e: unknown) {
      return { ok: false, error: this.mensagemDeErro(e) };
    }
  }
}

class GatewayPix extends GatewayBase {
  async cobrar(valorEmCentavos: number): Promise<Result<string, string>> {
    try {
      const r = await this.http.post('/pix/charges', { valorEmCentavos });
      return { ok: true, value: r.id };
    } catch (e: unknown) {
      return { ok: false, error: this.mensagemDeErro(e) };
    }
  }
}

// Rota sem hierarquia, quando o método não usa estado:
// `export function mensagemDeErro(e: unknown): string` em `gateways/erros.ts`,
// importada pelos dois gateways — nenhuma classe base necessária.
```

### Passos
1. Compare os corpos e normalize formatação, nomes de variáveis locais e ordem das instruções até ficarem idênticos.
2. Unifique nome e assinatura do método nas subclasses (Rename).
3. **Decida o destino:** se o método não lê `this`, extraia uma função exportada em um módulo compartilhado e pare aqui — é a rota idiomática.
4. Se ele depende de estado da base, mova o corpo para a superclasse como `protected` e apague as versões das subclasses.
5. Se ele usa campo de subclasse, aplique Pull Up Field antes, ou declare `protected abstract` getter na base.
6. Nos pontos de chamada, avalie declarar o tipo da base (ou do contrato) em vez do tipo concreto.

### Ganhos
- Uma única alteração propaga para toda a hierarquia.
- Reduz superfície de teste: a regra é testada uma vez.
- Torna explícito qual é de fato a responsabilidade da base.

### Quando NÃO aplicar
- A "duplicação" é coincidência temporal (duas mensagens que vão divergir por regra de produto).
- Subir o método obriga a base a importar dependência de camada superior (domínio importando cliente HTTP).
- O método não depende de `this` — nesse caso extraia uma função, não suba na hierarquia. Este caso é a maioria em TypeScript.

### Nota TypeScript
Métodos em TS não são `final`: qualquer subclasse pode sobrescrever o método subido, e só o flag `noImplicitOverride` obriga a marcar `override` (ligue-o — é o que torna a hierarquia auditável). Antes de criar/alimentar uma base, pergunte se uma **função exportada** resolve: funções são mais fáceis de importar, tree-shakeáveis, testáveis sem instanciar nada e não consomem o único slot de `extends`. Reserve a base para o que realmente lê estado compartilhado.

---

## 3. Pull Up Constructor Body
**PT-BR:** Subir Corpo do Construtor · **Fonte:** https://refactoring.guru/pull-up-constructor-body

### Problema
Construtores das subclasses têm código de inicialização quase idêntico (validações, normalizações, timestamps, geração de id).

### Solução
Mover o trecho comum para o construtor da superclasse e chamá-lo das subclasses via `super(...)`.

### Sinais no código (gatilhos)
- O mesmo bloco de validação repetido logo após `super()` em cada subclasse.
- Campos da base declarados com `!` (definite assignment) e preenchidos por cada subclasse.
- `throw new Error('... obrigatório')` sobre o mesmo parâmetro em todas as subclasses.
- Objeto que pode existir meio-inicializado entre `new` e a primeira chamada.

### Antes
```ts
abstract class Notificacao {
  protected id!: string;
  protected criadaEm!: Date;
}

class NotificacaoEmail extends Notificacao {
  constructor(
    readonly destinatario: string,
    readonly assunto: string,
  ) {
    super();
    if (!destinatario.includes('@')) throw new Error('destinatário inválido');
    this.id = `NT-${destinatario.slice(-4)}`;
    this.criadaEm = new Date();
  }
}

class NotificacaoPush extends Notificacao {
  constructor(
    readonly destinatario: string,
    readonly deviceToken: string,
  ) {
    super();
    if (!destinatario.includes('@')) throw new Error('destinatário inválido');
    this.id = `NT-${destinatario.slice(-4)}`;
    this.criadaEm = new Date();
  }
}
```

### Depois
```ts
abstract class Notificacao {
  readonly id: string;
  readonly criadaEm: Date;

  constructor(readonly destinatario: string) {
    if (!destinatario.includes('@')) throw new Error('destinatário inválido');
    this.id = `NT-${destinatario.slice(-4)}`;
    this.criadaEm = new Date();
  }
}

class NotificacaoEmail extends Notificacao {
  readonly assunto: string;

  constructor(destinatario: string, assunto: string) {
    super(destinatario);
    if (assunto.trim() === '') throw new Error('assunto obrigatório');
    this.assunto = assunto;
  }
}

class NotificacaoPush extends Notificacao {
  constructor(
    destinatario: string,
    readonly deviceToken: string,
  ) {
    super(destinatario);
  }
}
```

### Passos
1. Identifique o trecho comum no começo dos construtores das subclasses.
2. Adicione à base apenas os parâmetros necessários para esse trecho.
3. Mova o trecho para o construtor da base; declare os campos resultantes como `readonly` e atribua-os ali.
4. Nas subclasses, chame `super(...)` como **primeira instrução** e remova o trecho duplicado; nada de `this` antes do `super()` (é erro de compilação e de runtime).
5. Onde a subclasse ainda precisa guardar algo próprio, atribua **depois** do `super()`; se o valor é fixo, declare como parameter property.
6. Elimine `!` e campos mutáveis da base: com a inicialização centralizada, eles deixam de ser necessários.

### Ganhos
- Validação e normalização acontecem em um único lugar, sempre.
- Fim do estado meio-inicializado e das declarações com `!`.
- Campos da base tornam-se `readonly`.

### Quando NÃO aplicar
- O código comum depende de valor que só a subclasse conhece depois de inicializar-se — a ordem `super()` → campos da subclasse não permite.
- As listas de parâmetros são incompatíveis e subir exigiria parâmetros artificiais/opcionais.
- A classe é instanciada por um ORM/serializador que reidrata objetos sem passar pelo construtor — nesse caso mova a validação para uma factory (`static criar`) em vez do construtor.

### Nota TypeScript
Ordem de inicialização: argumentos passados a `super()` → corpo do construtor da base → **inicializadores de campo da subclasse** → corpo do construtor da subclasse. Duas consequências práticas. (1) **Nunca** chame um método sobrescrevível dentro do construtor da base: a subclasse ainda não inicializou seus campos e você lê `undefined`. (2) Com `useDefineForClassFields: true` (padrão em target ES2022+), um campo declarado com inicializador na subclasse *sobrescreve* o que a base atribuiu ao mesmo nome — se a base preenche o campo, a subclasse não deve declará-lo de novo. Se a base precisa de algo da subclasse, receba por parâmetro do construtor.

---

## 4. Push Down Field
**PT-BR:** Descer Campo · **Fonte:** https://refactoring.guru/push-down-field

### Problema
Campo declarado na superclasse é usado apenas por uma ou poucas subclasses.

### Solução
Mover o campo para as subclasses que realmente o usam e removê-lo da superclasse.

### Sinais no código (gatilhos)
- Campo da base tipado como `T | null` e passado como `null` por metade das subclasses.
- Toda leitura do campo vem depois de um `instanceof` ou de um narrowing por discriminante.
- `!` ou `?? valorDefault` sempre que o campo é usado.
- Campo criado para uma feature que só se materializou em um ramo.

### Antes
```ts
abstract class MetodoPagamento {
  constructor(
    readonly id: string,
    readonly bandeira: string | null,
    readonly chavePix: string | null,
  ) {}

  rotulo(): string {
    return this.bandeira ?? 'PIX';
  }
}

class Cartao extends MetodoPagamento {
  constructor(id: string, bandeira: string) {
    super(id, bandeira, null);
  }
}

class Pix extends MetodoPagamento {
  constructor(id: string, chavePix: string) {
    super(id, null, chavePix);
  }
}

function descrever(m: MetodoPagamento): string {
  if (m instanceof Cartao) return `Cartão ${m.rotulo()}`;
  if (m instanceof Pix) return `PIX ${m.chavePix ?? ''}`;
  return m.id;
}
```

### Depois
```ts
abstract class MetodoPagamento {
  constructor(readonly id: string) {}
}

class Cartao extends MetodoPagamento {
  constructor(
    id: string,
    readonly bandeira: string,
  ) {
    super(id);
  }

  rotulo(): string {
    return this.bandeira;
  }
}

class Pix extends MetodoPagamento {
  constructor(
    id: string,
    readonly chavePix: string,
  ) {
    super(id);
  }
}

function descrever(m: MetodoPagamento): string {
  if (m instanceof Cartao) return `Cartão ${m.rotulo()}`;
  if (m instanceof Pix) return `PIX ${m.chavePix}`;
  return m.id;
}
```

### Passos
1. Liste todos os usos do campo (busca por nome + "Find all references") e confirme que se concentram em um subconjunto das subclasses.
2. Declare o campo nas subclasses que o usam, como parameter property `readonly`.
3. Remova o parâmetro da base e o argumento correspondente de cada `super(...)`.
4. Se algum método da base usava o campo, desça-o também (Push Down Method) ou torne-o `abstract`.
5. Recompile: os erros apontam exatamente os clientes que liam o campo pela referência da base.
6. Avalie o passo seguinte: com os campos onde pertencem, a hierarquia costuma virar uma **discriminated union** com `switch` exaustivo — mais barata que `instanceof`.

### Ganhos
- Acaba com `null` e defaults artificiais na base; o tipo passa a dizer a verdade.
- Cada variante carrega só o que faz sentido para ela.
- A base fica pequena e realmente comum.

### Quando NÃO aplicar
- Quase todas as subclasses usam o campo — descer só multiplica declarações.
- Existe código genérico que lê o campo pela base; descer exigiria `instanceof` espalhado, piorando o design.
- O campo faz parte de um contrato serializado (DTO, coluna única para toda a hierarquia em ORM com single-table inheritance).

### Nota TypeScript
Campo `T | null` na base é o sintoma número um, e em TS ele fica visível no tipo — `strictNullChecks` cobra `??`/`!` em cada leitura. Depois de descer, considere trocar a hierarquia por união discriminada: `type MetodoPagamento = { kind: 'cartao'; bandeira: string } | { kind: 'pix'; chavePix: string }`. O `switch` exaustivo com `default: assertNever(m)` dá a mesma garantia de cobertura que o compilador daria com classes seladas, sem `instanceof` (que quebra em objetos vindos de JSON, `structuredClone` ou de outro realm).

---

## 5. Push Down Method
**PT-BR:** Descer Método · **Fonte:** https://refactoring.guru/push-down-method

### Problema
Comportamento implementado na superclasse é usado por apenas uma (ou poucas) subclasses.

### Solução
Mover o método — e as dependências exclusivas dele — para as subclasses que o usam, removendo-o da superclasse.

### Sinais no código (gatilhos)
- Método da base referenciado em um único arquivo de subclasse.
- Base recebendo no construtor uma dependência que só um ramo usa (cache, gerador de PDF, cliente de e-mail).
- Método da base que lança `new Error('não suportado')` em algumas subclasses.
- Teste da base precisa de fake para colaborador que a base não usa.

### Antes
```ts
abstract class RepositorioBase<T> {
  constructor(
    protected readonly db: Db,
    private readonly cache: Cache,
  ) {}

  abstract porId(id: string): Promise<T | null>;

  async invalidarCache(id: string): Promise<void> {
    await this.cache.del(`pedido:${id}`);
  }
}

class PedidoRepository extends RepositorioBase<Pedido> {
  async porId(id: string): Promise<Pedido | null> {
    return this.db.pedidos.buscar(id);
  }
}

class UsuarioRepository extends RepositorioBase<Usuario> {
  async porId(id: string): Promise<Usuario | null> {
    return this.db.usuarios.buscar(id);
  }
}

// `cache` é exigido no construtor mesmo por quem nunca invalida nada
const usuarios = new UsuarioRepository(db, cache);
```

### Depois
```ts
abstract class RepositorioBase<T> {
  constructor(protected readonly db: Db) {}

  abstract porId(id: string): Promise<T | null>;
}

class PedidoRepository extends RepositorioBase<Pedido> {
  constructor(
    db: Db,
    private readonly cache: Cache,
  ) {
    super(db);
  }

  async porId(id: string): Promise<Pedido | null> {
    return this.db.pedidos.buscar(id);
  }

  async invalidarCache(id: string): Promise<void> {
    await this.cache.del(`pedido:${id}`);
  }
}

class UsuarioRepository extends RepositorioBase<Usuario> {
  async porId(id: string): Promise<Usuario | null> {
    return this.db.usuarios.buscar(id);
  }
}

const usuarios = new UsuarioRepository(db);
```

### Passos
1. Declare o método na subclasse que o usa e copie o corpo da base.
2. Mova junto as dependências exclusivas: o parâmetro de construtor sai da base e entra na subclasse (que passa a chamar `super(...)` só com o que sobrou).
3. Remova o método da base.
4. Ajuste os pontos de chamada e as construções (`new`, factories, composição raiz) para o tipo que agora tem o método.
5. Se mais de uma subclasse precisa do método e o corpo divergiu, não duplique: extraia um colaborador injetado.
6. Compile com `noUnusedParameters`/lint para encontrar restos do parâmetro removido.

### Ganhos
- A base deixa de exigir dependências que não usa; construtores e wiring encolhem.
- O método fica onde o leitor espera encontrá-lo.
- Testes da base ficam menores e sem fakes desnecessários.

### Quando NÃO aplicar
- Duas ou mais subclasses usam o método com corpo idêntico — descer cria duplicação.
- Clientes chamam o método pela referência da base (é contrato público da hierarquia).
- O método é ponto de extensão intencional com implementação default, mesmo que hoje só um ramo o use.

### Nota TypeScript
Em TS a alternativa a descer é quase sempre melhor: transforme o comportamento em **colaborador injetado por construtor** (`new PedidoRepository(db, cache)`) ou em função que recebe o que precisa. Herdar utilitário significa que a base importa o módulo do utilitário — e isso vaza para o bundle de todo mundo que estende a base. Uma função importada só por quem usa é tree-shakeável; um método herdado, não.

---

## 6. Extract Subclass
**PT-BR:** Extrair Subclasse · **Fonte:** https://refactoring.guru/extract-subclass

### Problema
Uma classe tem campos e comportamento usados só em certos casos, controlados por uma flag de tipo.

### Solução
Separar cada caso em seu próprio tipo e substituir os condicionais por despacho sobre o tipo. Em TypeScript, o alvo correto quase sempre é uma **discriminated union** (ou composição), não uma subclasse.

### Sinais no código (gatilhos)
- Campo booleano/enum de "tipo" (`ehTrial`, `modalidade`) consultado em vários métodos.
- Campos `T | null` que só fazem sentido quando a flag tem certo valor, lidos com `!` ou `?? default`.
- `throw` no construtor garantindo combinações inválidas de campos — invariante que o tipo poderia expressar.
- Todo novo caso obriga a tocar todos os métodos da classe.

### Antes
```ts
class Assinatura {
  constructor(
    readonly usuarioId: string,
    readonly ehTrial: boolean,
    readonly diasDeTrial: number | null,
    readonly valorMensal: number | null,
    readonly inicio: Date,
  ) {
    if (ehTrial && diasDeTrial === null) throw new Error('trial exige duração');
    if (!ehTrial && valorMensal === null) throw new Error('paga exige valor');
  }

  expiraEm(): Date {
    const d = new Date(this.inicio);
    if (this.ehTrial) d.setDate(d.getDate() + (this.diasDeTrial ?? 0));
    else d.setMonth(d.getMonth() + 1);
    return d;
  }

  cobrancaDoMes(): number {
    return this.ehTrial ? 0 : (this.valorMensal ?? 0);
  }

  podeBaixarNotaFiscal(): boolean {
    return !this.ehTrial;
  }
}
```

### Depois
```ts
type AssinaturaBase = { readonly usuarioId: string; readonly inicio: Date };

type Assinatura =
  | (AssinaturaBase & { readonly kind: 'trial'; readonly diasDeTrial: number })
  | (AssinaturaBase & { readonly kind: 'paga'; readonly valorMensal: number });

function assertNever(x: never): never {
  throw new Error(`variante não tratada: ${JSON.stringify(x)}`);
}

export function expiraEm(a: Assinatura): Date {
  const d = new Date(a.inicio);
  switch (a.kind) {
    case 'trial':
      d.setDate(d.getDate() + a.diasDeTrial);
      return d;
    case 'paga':
      d.setMonth(d.getMonth() + 1);
      return d;
    default:
      return assertNever(a);
  }
}

export function cobrancaDoMes(a: Assinatura): number {
  return a.kind === 'trial' ? 0 : a.valorMensal;
}

export function podeBaixarNotaFiscal(a: Assinatura): boolean {
  return a.kind === 'paga';
}
```

### Passos
1. Nomeie as variantes que a flag esconde e escolha o discriminante (`kind`, `type`, `status`).
2. Declare a união com um membro por variante, contendo **só** os campos daquela variante — os `| null` desaparecem.
3. Extraia o que é comum a todas as variantes para um `type` base e intersecte (`&`).
4. Troque cada ponto de construção (factory, mapper DTO→domínio) pelo literal da variante correta; use `satisfies Assinatura` para validar sem perder inferência.
5. Converta cada `if (flag)` em `switch (a.kind)` com `default: assertNever(a)`; o compilador passa a exigir tratamento de toda variante nova.
6. Só use subclasse de verdade se as variantes precisam de identidade de objeto, estado mutável interno ou têm de encaixar em uma hierarquia existente (ex.: classes de erro com `instanceof`).

### Ganhos
- Fim dos campos nuláveis e das invariantes verificadas só em runtime — o tipo torna o estado inválido inexpressável.
- Cada regra fica isolada e testável por variante.
- Adicionar `'institucional'` provoca erro de compilação em todo lugar que precisa decidir — nada é esquecido em silêncio.

### Quando NÃO aplicar
- Há dois eixos de variação independentes (plano × forma de pagamento): a união vira produto cartesiano; use composição (campo de estratégia).
- A variação é de dados, não de comportamento — um campo extra resolve.
- O caso especial é temporário (experimento, feature flag de vida curta).

### Nota TypeScript
Uniões discriminadas são o substituto direto de `sealed class` e dominam este caso em TS: são serializáveis (sobrevivem a `JSON.parse`, ao cache e à fronteira do servidor, ao contrário de `instanceof`), casam com Zod (`z.discriminatedUnion`) e com `useReducer` no front. Se o comportamento precisa ficar junto do dado, use um `Record<Assinatura['kind'], Handler>` — o `Record` mapeado obriga a preencher toda variante. Reserve `extends` para hierarquias que já existem e que você não controla.

---

## 7. Extract Superclass
**PT-BR:** Extrair Superclasse · **Fonte:** https://refactoring.guru/extract-superclass

### Problema
Duas classes têm campos e métodos comuns, sem relação de herança entre elas.

### Solução
Extrair o que é comum. Em TS, primeiro tente **função/módulo compartilhado** ou um objeto base composto; crie a superclasse abstrata quando há estado e inicialização realmente compartilhados.

### Sinais no código (gatilhos)
- Mesmos campos de identidade/auditoria (`id`, `criadoEm`) em duas entidades independentes.
- A mesma validação copiada em dois construtores.
- Dois arquivos com blocos idênticos e o mesmo conceito de domínio por trás.
- Um bug corrigido em um arquivo e esquecido no irmão.

### Antes
```ts
class Cupom {
  constructor(
    readonly id: string,
    readonly criadoEm: Date,
    readonly codigo: string,
    readonly percentual: number,
  ) {
    if (id.trim() === '') throw new Error('id obrigatório');
  }

  idadeEmDias(agora: Date): number {
    return Math.floor((agora.getTime() - this.criadoEm.getTime()) / 86_400_000);
  }
}

class Assinatura {
  constructor(
    readonly id: string,
    readonly criadoEm: Date,
    readonly plano: string,
  ) {
    if (id.trim() === '') throw new Error('id obrigatório');
  }

  idadeEmDias(agora: Date): number {
    return Math.floor((agora.getTime() - this.criadoEm.getTime()) / 86_400_000);
  }
}
```

### Depois
```ts
abstract class EntidadeAuditada {
  constructor(
    readonly id: string,
    readonly criadoEm: Date,
  ) {
    if (id.trim() === '') throw new Error('id obrigatório');
  }

  idadeEmDias(agora: Date): number {
    return Math.floor((agora.getTime() - this.criadoEm.getTime()) / 86_400_000);
  }
}

class Cupom extends EntidadeAuditada {
  constructor(
    id: string,
    criadoEm: Date,
    readonly codigo: string,
    readonly percentual: number,
  ) {
    super(id, criadoEm);
  }
}

class Assinatura extends EntidadeAuditada {
  constructor(id: string, criadoEm: Date, readonly plano: string) {
    super(id, criadoEm);
  }
}

// Alternativa idiomática, se o comum não guarda estado próprio: `type Auditada
// = { readonly id: string; readonly criadoEm: Date }` + uma função exportada
// `idadeEmDias(e: Auditada, agora: Date): number` no módulo compartilhado.
```

### Passos
1. Liste o que é idêntico: campos, validação de construtor, métodos.
2. **Escolha a forma.** Sem estado compartilhado → módulo com `type` comum + funções que recebem o objeto. Com estado e inicialização comuns → classe abstrata. Objetos sem classe → uma factory que compõe (`{ ...base(id, criadoEm), ...especifico }`).
3. Na rota da superclasse: crie a base `abstract`, suba os campos (Pull Up Field) e depois o corpo do construtor (Pull Up Constructor Body).
4. Suba os métodos idênticos (Pull Up Method).
5. Nos clientes, troque o tipo declarado pelo da base onde só a API comum é usada.
6. Confira se o nome da base descreve um conceito real do domínio. Se você só achou nomes como `BaseModel`, o certo era o módulo compartilhado.

### Ganhos
- Deduplicação de campos, validação e comportamento em um lugar só.
- Clientes genéricos passam a operar sobre um tipo único (`readonly EntidadeAuditada[]`).
- Uma correção vale para todas as entidades derivadas.

### Quando NÃO aplicar
- Uma das classes já tem superclasse: só existe um slot de `extends`. Extraia um contrato (`type`) e componha.
- As classes só coincidem estruturalmente, sem "é-um" real (`Cupom` e `Pagamento` ambos com `id` e `criadoEm` não são a mesma coisa).
- O comum é comportamento sem estado — função resolve sem acoplar hierarquia.
- São objetos planos que atravessam a fronteira (props, DTO, resposta de API): herança não sobrevive à serialização.

### Nota TypeScript
A superclasse ganha em poucos casos: estado mutável compartilhado, inicialização validada em um ponto, ou encaixe em uma hierarquia que já existe (classes de erro, base model do ORM). Nos outros, prefira `type` comum + funções, ou mixins (`function Auditada<T extends new (...a: unknown[]) => object>(Base: T)`) quando precisar compor mais de um eixo — TS suporta mixins justamente porque `extends` é único. Lembre: para tipos de dados puros a resposta é `type` e spread, não hierarquia.

---

## 8. Extract Interface
**PT-BR:** Extrair Interface · **Fonte:** https://refactoring.guru/extract-interface

### Problema
Vários clientes usam somente uma parte da API de uma classe concreta, ou duas classes compartilham a mesma parte de API.

### Solução
Declarar essa parte comum como um tipo próprio e fazer os clientes dependerem dele; a implementação passa a ser um detalhe injetado.

### Sinais no código (gatilhos)
- Serviço/caso de uso com parâmetro de construtor tipado como `...RepositoryPrisma`, `...ClientHttp`.
- Teste que só consegue isolar a dependência com `jest.mock` do módulo.
- Import de `infra/` dentro de `domain/`.
- A dependência traz métodos que o cliente nunca chama (`limparCache`, `desconectar`).

### Antes
```ts
class PedidoRepositoryPrisma {
  constructor(
    private readonly db: PrismaClient,
    private readonly cache: Cache,
  ) {}

  async porId(id: string): Promise<Pedido | null> {
    return this.db.pedido.findUnique({ where: { id } });
  }

  async salvar(pedido: Pedido): Promise<void> {
    await this.db.pedido.update({ where: { id: pedido.id }, data: pedido });
    await this.cache.del(`pedido:${pedido.id}`);
  }
}

class FinalizarCompra {
  constructor(private readonly repo: PedidoRepositoryPrisma) {}

  async executar(id: string): Promise<Result<Pedido, 'nao_encontrado'>> {
    const pedido = await this.repo.porId(id);
    if (pedido === null) return { ok: false, error: 'nao_encontrado' };
    const pago = { ...pedido, status: 'pago' as const };
    await this.repo.salvar(pago);
    return { ok: true, value: pago };
  }
}
```

### Depois
```ts
// domínio: só o que o caso de uso realmente chama
type PedidoRepository = {
  porId(id: string): Promise<Pedido | null>;
  salvar(pedido: Pedido): Promise<void>;
};

class FinalizarCompra {
  constructor(private readonly repo: PedidoRepository) {}

  async executar(id: string): Promise<Result<Pedido, 'nao_encontrado'>> {
    const pedido = await this.repo.porId(id);
    if (pedido === null) return { ok: false, error: 'nao_encontrado' };
    const pago = { ...pedido, status: 'pago' as const };
    await this.repo.salvar(pago);
    return { ok: true, value: pago };
  }
}

// infra: encaixa por tipagem estrutural; `implements` é checagem, não requisito
class PedidoRepositoryPrisma implements PedidoRepository {
  constructor(private readonly db: PrismaClient, private readonly cache: Cache) {}

  async porId(id: string): Promise<Pedido | null> {
    return this.db.pedido.findUnique({ where: { id } });
  }

  async salvar(pedido: Pedido): Promise<void> {
    await this.db.pedido.update({ where: { id: pedido.id }, data: pedido });
    await this.cache.del(`pedido:${pedido.id}`);
  }
}

// teste: um objeto literal já satisfaz o contrato — sem classe, sem mock
const fakeRepo = { porId: async () => pedidoFixo, salvar: async () => undefined } satisfies PedidoRepository;
```

### Passos
1. Liste o subconjunto de membros que os clientes chamam de fato.
2. Declare o tipo no módulo do **cliente** (domínio), não no da implementação — é isso que inverte a dependência.
3. Use apenas tipos de domínio na assinatura: nada de `Prisma.PedidoGetPayload`, `AxiosResponse`, `Row`.
4. Troque o tipo declarado no construtor do cliente pelo novo contrato. Nada mais precisa mudar: a classe concreta já encaixa por estrutura.
5. Opcionalmente adicione `implements PedidoRepository` na classe concreta — serve para o erro aparecer no arquivo da implementação, não no ponto de uso.
6. Monte o wiring na composição raiz (`new FinalizarCompra(new PedidoRepositoryPrisma(db, cache))`) ou em uma factory de closure. Sem container mágico.
7. No teste, passe um objeto literal com `satisfies` no lugar do fake de classe.

### Ganhos
- Testes sem framework de mock: fake determinístico, rápido e legível.
- Inversão de dependência real: `domain/` deixa de importar `infra/`.
- Permite empilhar implementações (cache, retry, log) sem tocar o cliente.
- Documenta em duas linhas exatamente o que o caso de uso precisa do mundo externo.

### Quando NÃO aplicar
- Existe uma implementação só, o teste não precisa substituí-la e não há outra prevista — o contrato é indireção pura.
- O problema é código duplicado entre as classes: contrato isola API, não corpo. Use Extract Superclass/Extract Class.
- Para tipos de dados puros (entidades, DTOs) — não "interfaceie" modelos; `type` já é o modelo.

### Nota TypeScript
Tipagem é **estrutural**: qualquer objeto com os membros certos satisfaz o contrato, sem declarar `implements`. Isso muda a economia da técnica em três pontos. (1) Extrair o contrato é quase gratuito e retroativo — funciona para classes de biblioteca que você não pode alterar. (2) O fake de teste é um objeto literal; use `satisfies Contrato` para checar sem alargar o tipo e sem perder inferência (`as Contrato` esconderia campos faltando). (3) A injeção é por construtor ou factory de closure. Use `type` para o contrato (permite uniões e utility types); `interface` só quando precisar de declaration merging ou de `implements` em várias classes. E prefira contratos pequenos, por caso de uso: uma dependência com `porId`/`salvar` é infinitamente mais fácil de fingir que um repositório com quinze métodos.

---

## 9. Collapse Hierarchy
**PT-BR:** Colapsar Hierarquia · **Fonte:** https://refactoring.guru/collapse-hierarchy

### Problema
Uma subclasse é praticamente igual à superclasse; a hierarquia não carrega informação.

### Solução
Fundir subclasse e superclasse em uma única classe.

### Sinais no código (gatilhos)
- Base `abstract` com uma única subclasse e nenhuma outra prevista.
- Base anêmica: só guarda um campo de estado e um setter `protected`.
- Subclasse que só chama `super` em tudo, sem adicionar comportamento.
- Entender um fluxo exige abrir dois arquivos que somados dão 40 linhas.
- `protected` usado onde `#privado` bastaria, apenas porque a base existe.

### Antes
```ts
abstract class BaseCarrinhoService {
  protected itens: readonly ItemCarrinho[] = [];

  protected emitir(novos: readonly ItemCarrinho[]): void {
    this.itens = novos;
  }

  protected total(): number {
    return this.itens.reduce((s, i) => s + i.precoEmCentavos * i.quantidade, 0);
  }
}

export class CarrinhoService extends BaseCarrinhoService {
  constructor(private readonly repo: CarrinhoRepository) {
    super();
  }

  async adicionar(item: ItemCarrinho): Promise<number> {
    this.emitir([...this.itens, item]);
    await this.repo.salvar(this.itens);
    return this.total();
  }

  async limpar(): Promise<void> {
    this.emitir([]);
    await this.repo.salvar(this.itens);
  }
}
```

### Depois
```ts
export class CarrinhoService {
  #itens: readonly ItemCarrinho[] = [];

  constructor(private readonly repo: CarrinhoRepository) {}

  async adicionar(item: ItemCarrinho): Promise<number> {
    this.#itens = [...this.#itens, item];
    await this.repo.salvar(this.#itens);
    return this.#total();
  }

  async limpar(): Promise<void> {
    this.#itens = [];
    await this.repo.salvar(this.#itens);
  }

  #total(): number {
    return this.#itens.reduce((s, i) => s + i.precoEmCentavos * i.quantidade, 0);
  }
}
```

### Passos
1. Decida qual classe desaparece: normalmente a base anêmica.
2. Se remove a subclasse, aplique Pull Up Field/Method; se remove a base, aplique Push Down Field/Method.
3. Feche a visibilidade do que era `protected`: use `#campo`/`#metodo` (privado em runtime) ou `private`.
4. Substitua todas as referências ao tipo removido — construções, tipos declarados, testes, wiring.
5. Apague o arquivo vazio e remova imports órfãos; rode o lint de imports não usados.

### Ganhos
- Menos arquivos e menos saltos para entender um fluxo.
- Estado volta a ser privado de verdade, em vez de `protected` visível a qualquer herdeiro.
- Elimina a ilusão de extensibilidade que nunca foi usada.

### Quando NÃO aplicar
- Existem outras subclasses reais — fundir ramos distintos tende a violar Liskov.
- A base é ponto de extensão publicado (você exporta a base em um pacote consumido por terceiros).
- A base carrega comportamento compartilhado real e a segunda subclasse já está especificada, não apenas imaginada.

### Nota TypeScript
O caso mais comum em TS é a "base de serviço" criada para hospedar um campo de estado e dois helpers. Colapse: cada serviço declara seu próprio `#estado`. `#` é privacidade de runtime (nem cast nem `as unknown` alcançam), então o colapso costuma vir junto de um ganho real de encapsulamento — algo que `protected` nunca deu. Se a intenção original era compartilhar comportamento, o substituto é um módulo de funções ou um colaborador injetado, não uma base. No front-end o equivalente é a hierarquia de componentes: não existe base de componente idiomática — compartilhe via custom hook ou composição de props.

---

## 10. Form Template Method
**PT-BR:** Formar Método Template · **Fonte:** https://refactoring.guru/form-template-method

### Problema
Duas ou mais implementações executam os mesmos passos, na mesma ordem, variando só o corpo de alguns passos.

### Solução
Isolar a estrutura do algoritmo em um lugar e receber os passos variáveis de fora: como funções injetadas (rota idiomática em TS) ou como membros abstratos de uma superclasse.

### Sinais no código (gatilhos)
- Dois pipelines (importar, sincronizar, exportar) com a mesma sequência buscar → filtrar → mapear → salvar.
- Mudar a ordem dos passos obriga a editar N arquivos.
- Comentários numerando as mesmas etapas em arquivos diferentes.
- O diff entre os dois arquivos toca só duas linhas do meio.

### Antes
```ts
class ImportadorProdutos {
  constructor(
    private readonly api: ProdutoApi,
    private readonly repo: ProdutoRepository,
  ) {}

  async importar(lojaId: string): Promise<number> {
    const dtos = await this.api.listar(lojaId);
    const validos = dtos.filter((d) => d.id.length > 0);
    const itens = validos.map((d) => ({ id: d.id, nome: d.nome }));
    await this.repo.inserir(itens);
    return itens.length;
  }
}

class ImportadorCupons {
  constructor(
    private readonly api: CupomApi,
    private readonly repo: CupomRepository,
  ) {}

  async importar(lojaId: string): Promise<number> {
    const dtos = await this.api.listar(lojaId);
    const validos = dtos.filter((d) => d.id.length > 0);
    const itens = validos.map((d) => ({ id: d.id, codigo: d.codigo }));
    await this.repo.inserir(itens);
    return itens.length;
  }
}
```

### Depois
```ts
type Passos<Dto, Item> = {
  buscar(lojaId: string): Promise<readonly Dto[]>;
  valido?(dto: Dto): boolean;
  mapear(dto: Dto): Item;
  salvar(itens: readonly Item[]): Promise<void>;
};

// (A) rota idiomática: passos injetados, sem herança
export function criarImportador<Dto, Item>(p: Passos<Dto, Item>) {
  return async (lojaId: string): Promise<number> => {
    const dtos = await p.buscar(lojaId);
    const itens = dtos.filter((d) => p.valido?.(d) ?? true).map((d) => p.mapear(d));
    await p.salvar(itens);
    return itens.length;
  };
}
const importarProdutos = criarImportador<ProdutoDto, Produto>({
  buscar: (lojaId) => produtoApi.listar(lojaId),
  valido: (d) => d.id.length > 0,
  mapear: (d) => ({ id: d.id, nome: d.nome }),
  salvar: (itens) => produtoRepo.inserir(itens),
});
// (B) a mesma estrutura com herança, quando as variantes compartilham estado
export abstract class Importador<Dto, Item> {
  async importar(lojaId: string): Promise<number> {
    const dtos = await this.buscar(lojaId);
    const itens = dtos.filter((d) => this.valido(d)).map((d) => this.mapear(d));
    await this.salvar(itens);
    return itens.length;
  }
  protected valido(_dto: Dto): boolean { return true; }
  protected abstract buscar(lojaId: string): Promise<readonly Dto[]>;
  protected abstract mapear(dto: Dto): Item;
  protected abstract salvar(itens: readonly Item[]): Promise<void>;
}
```

### Passos
1. Decomponha cada versão duplicada em etapas nomeadas (Extract Method/Function).
2. Renomeie as etapas para nomes iguais nas duas versões e alinhe as assinaturas; genéricos cobrem a diferença de tipo do DTO e do item.
3. Escreva o esqueleto uma única vez, com as etapas como buracos: um objeto de hooks (`Passos<Dto, Item>`) recebido por uma função de alta ordem, ou métodos `protected abstract` na base.
4. Marque como opcional (`valido?`) o hook com default, resolvido com `?? true` — o equivalente do método com implementação default.
5. Troque cada implementação antiga por uma chamada da factory (ou por uma subclasse), e mantenha o esqueleto fora do alcance de override.
6. Escolha entre (A) e (B): funções injetadas se as "variantes" não têm estado próprio ou se as etapas variam em eixos independentes; superclasse se as variantes compartilham dependências e ciclo de vida.

### Ganhos
- A ordem do algoritmo existe em um único lugar; mudança de regra é feita uma vez.
- Aberto para extensão, fechado para modificação: nova variante = novo objeto de passos.
- Os pontos de variação ficam explícitos na assinatura — o tipo `Passos` é a documentação.

### Quando NÃO aplicar
- Os passos variam em **ordem**, não só em corpo: template method engessa e reaparece como flags.
- Existem duas variantes e a duplicação cabe em cinco linhas: o esqueleto custa mais que o ganho.
- Cada variação precisa de um eixo diferente de customização — parametrize com funções isoladas.

### Nota TypeScript
A versão (A) é a que envelhece melhor em TS: testar não exige subclasse (passe um objeto de passos falsos), os genéricos inferem sozinhos a partir do literal, e não se gasta o slot de `extends`. Se preferir (B), ligue `noImplicitOverride` para que ninguém sobrescreva o esqueleto por acidente — TS não tem `final`, e um `override importar()` silencioso destrói a técnica (se precisar de garantia, exponha o esqueleto como função livre que chama os hooks). Hooks opcionais com `?.()` e `??` substituem bem o método com implementação default; e um objeto de hooks tem a vantagem de poder ser montado em runtime, por configuração.

---

## 11. Replace Inheritance with Delegation
**PT-BR:** Substituir Herança por Delegação · **Fonte:** https://refactoring.guru/replace-inheritance-with-delegation

### Problema
Uma subclasse herda de uma implementação para reusar parte dela, mas usa só uma fração da API da base e passa a expor comportamento que não deveria.

### Solução
Guardar a implementação em um campo (tipado pelo **contrato**, não pela classe concreta), delegar a ela o que precisa e remover o `extends`.

### Sinais no código (gatilhos)
- `extends` de uma classe de infra apenas para reaproveitar um método.
- Subclasse que chama `super.metodo()` em um método e ignora todos os outros.
- Subclasse que não satisfaz Liskov: quebra se usada onde a base é esperada.
- Uso de membro `protected` da base como se fosse utilitário global (`this.agora()`).
- Você estende uma classe de biblioteca só para envolver duas chamadas.

### Antes
```ts
class PedidoRepositoryHttp implements PedidoRepository {
  constructor(protected readonly http: HttpClient) {}

  async porId(id: string): Promise<Pedido | null> {
    return this.http.get<Pedido>(`/pedidos/${id}`);
  }

  async listar(usuarioId: string): Promise<readonly Pedido[]> {
    return this.http.get<Pedido[]>(`/usuarios/${usuarioId}/pedidos`);
  }

  protected agora(): number {
    return Date.now();
  }
}

// herda a implementação inteira só para reaproveitar `porId`
class PedidoRepositoryComCache extends PedidoRepositoryHttp {
  constructor(
    http: HttpClient,
    private readonly cache: Map<string, { pedido: Pedido; em: number }>,
  ) {
    super(http);
  }

  override async porId(id: string): Promise<Pedido | null> {
    const hit = this.cache.get(id);
    if (hit !== undefined) return hit.pedido;
    const pedido = await super.porId(id);
    if (pedido !== null) this.cache.set(id, { pedido, em: this.agora() });
    return pedido;
  }
}
```

### Depois
```ts
class PedidoRepositoryHttp implements PedidoRepository {
  constructor(private readonly http: HttpClient) {}

  async porId(id: string): Promise<Pedido | null> {
    return this.http.get<Pedido>(`/pedidos/${id}`);
  }

  async listar(usuarioId: string): Promise<readonly Pedido[]> {
    return this.http.get<Pedido[]>(`/usuarios/${usuarioId}/pedidos`);
  }
}

// delegação explícita: repassa só o que o contrato declara
class PedidoRepositoryComCache implements PedidoRepository {
  constructor(
    private readonly delegate: PedidoRepository,
    private readonly cache: Map<string, { pedido: Pedido; em: number }>,
    private readonly agora: () => number,
  ) {}

  async porId(id: string): Promise<Pedido | null> {
    const hit = this.cache.get(id);
    if (hit !== undefined) return hit.pedido;
    const pedido = await this.delegate.porId(id);
    if (pedido !== null) this.cache.set(id, { pedido, em: this.agora() });
    return pedido;
  }

  listar(usuarioId: string): Promise<readonly Pedido[]> {
    return this.delegate.listar(usuarioId);
  }
}
```

### Passos
1. Crie na subclasse um campo do tipo da **abstração** (`PedidoRepository`), não da classe concreta.
2. Troque cada `super.metodo()` por `this.delegate.metodo()`, e cada uso de membro `protected` da base por um colaborador injetado (`this.agora()` vira `() => number` no construtor).
3. Remova o `extends` e declare `implements Contrato`.
4. Escreva à mão o repasse **apenas dos métodos do contrato** — normalmente dois ou três. Se a lista for longa, o contrato está grande demais (veja Extract Interface).
5. Monte a cadeia no wiring: `new PedidoRepositoryComCache(new PedidoRepositoryHttp(http), new Map(), () => Date.now())`.
6. Feche a base: campos voltam a `private`, `protected` desaparece, e o teste do decorator passa a precisar só de um fake do delegate.

### Ganhos
- A classe não expõe nada além do contrato; API mínima e Liskov respeitada.
- O delegate é trocável em wiring/runtime: é Strategy, e habilita decorators empilháveis (cache, log, retry, métricas).
- Cada camada é testável com um objeto literal no lugar do delegate.
- Some a dependência de detalhes `protected` da implementação — o que sobra é o contrato.

### Quando NÃO aplicar
- A relação é genuinamente "é-um" e o subtipo usa e honra toda a API da base.
- Você depende de estado interno da base que o contrato não expõe.
- Você depende de o código da base chamar o seu método sobrescrito — delegação não oferece isso (ver nota).

### Nota TypeScript
**TS não tem delegação por `by`.** As três opções: (1) **delegação manual** — métodos que repassam, explicitamente; (2) **spread de objeto** — `const repo = { ...base, porId }`, que funciona para objetos criados por factory, mas *não* para instâncias de classe (métodos vivem no prototype e não entram no spread) e perde qualquer `this` ligado; (3) **`Proxy`** com `get` que encaminha ao delegate — repassa tudo com zero linhas, mas é difícil de tipar (você acaba em `as` ou em tipos falsos), invisível ao "go to definition" e caro de depurar. Recomendação: **delegue explicitamente os poucos métodos realmente usados**. A verbosidade é o preço de um contrato honesto, e se ela incomoda, o contrato está inflado. Um limite real: o delegate **não vê seus overrides** — se `delegate.listar()` chamar internamente seu próprio `porId`, será o `porId` do delegate, não o seu com cache; quando o algoritmo interno precisa enxergar a especialização, delegação não substitui herança. Por fim: "composição > herança" vale ainda mais em TS porque o tipo é estrutural — o composto encaixa em qualquer contrato que satisfaça, sem precisar estar na mesma hierarquia.

---

## 12. Replace Delegation with Inheritance
**PT-BR:** Substituir Delegação por Herança · **Fonte:** https://refactoring.guru/replace-delegation-with-inheritance

### Problema
Uma classe é só uma casca: contém muitos métodos simples que repassam **todos** os membros públicos de outra classe, sem adicionar nada.

### Solução
Fazer a classe herdar do delegate, o que torna os métodos de repasse desnecessários.

### Sinais no código (gatilhos)
- Todo método do wrapper tem corpo `return this.delegate.mesmoMetodo(args)`, sem nenhuma linha extra.
- Cada método novo no delegate exige adicionar um repasse no wrapper — e o repasse é esquecido.
- Wrapper e delegate representam a mesma abstração, com o mesmo conjunto de operações.
- O wrapper não tem estado nem dependência próprios.

### Antes
```ts
class Carrinho {
  protected readonly itens: ItemCarrinho[] = [];

  constructor(readonly usuarioId: string) {}

  adicionar(item: ItemCarrinho): void {
    this.itens.push(item);
  }

  remover(produtoId: string): void {
    const i = this.itens.findIndex((it) => it.produtoId === produtoId);
    if (i >= 0) this.itens.splice(i, 1);
  }

  listar(): readonly ItemCarrinho[] {
    return [...this.itens];
  }

  total(): number {
    return this.itens.reduce((s, i) => s + i.precoEmCentavos * i.quantidade, 0);
  }
}

class CarrinhoDoUsuario {
  constructor(private readonly delegate: Carrinho) {}

  get usuarioId(): string { return this.delegate.usuarioId; }
  adicionar(item: ItemCarrinho): void { this.delegate.adicionar(item); }
  remover(produtoId: string): void { this.delegate.remover(produtoId); }
  listar(): readonly ItemCarrinho[] { return this.delegate.listar(); }
  total(): number { return this.delegate.total(); }
}
```

### Depois
```ts
class Carrinho {
  protected readonly itens: ItemCarrinho[] = [];

  constructor(readonly usuarioId: string) {}

  adicionar(item: ItemCarrinho): void {
    this.itens.push(item);
  }

  remover(produtoId: string): void {
    const i = this.itens.findIndex((it) => it.produtoId === produtoId);
    if (i >= 0) this.itens.splice(i, 1);
  }

  listar(): readonly ItemCarrinho[] {
    return [...this.itens];
  }

  total(): number {
    return this.itens.reduce((s, i) => s + i.precoEmCentavos * i.quantidade, 0);
  }
}

// herda em vez de repassar: nenhum método de encaminhamento sobra,
// e a classe continua sem estado nem dependência próprios
class CarrinhoDoUsuario extends Carrinho {}
```

### Passos
1. Confirme que o wrapper repassa **todos** os membros públicos do delegate, sem filtrar, adaptar nem acrescentar.
2. Confirme que o wrapper ainda não tem superclasse — só existe um slot de `extends`.
3. Faça a classe estender o delegate e repasse os argumentos necessários em `super(...)`.
4. Remova os métodos de repasse um a um, renomeando antes se algum nome divergia.
5. Troque as referências a `this.delegate` por `this`.
6. Apague o campo `delegate` e a criação do objeto interno; ajuste os `new` nos clientes e testes.

### Ganhos
- Remove todo o código de repasse: menos linhas, menos manutenção.
- Métodos novos no delegate ficam disponíveis automaticamente.
- Deixa explícita a relação "é-um" quando ela realmente existe.

### Quando NÃO aplicar
- O wrapper repassa só **parte** dos membros públicos: herdar expõe o resto e viola Liskov.
- A classe já tem superclasse.
- O wrapper adiciona comportamento em algum método (cache, log, auditoria) — é decorator; mantenha a delegação.
- O delegate vem de biblioteca cuja evolução você não controla: cada versão nova amplia sua API pública sem você pedir.

### Nota TypeScript
É a técnica mais rara deste grupo em TS, e só vale no caso estreito: repasse trivial de 100% da API, mesma abstração, wrapper sem estado próprio e sem superclasse. Três razões para desconfiar. (1) Tipagem estrutural: se o objetivo era "ser aceito onde o delegate é aceito", você não precisa herdar — basta satisfazer o tipo, e um wrapper que já repassa tudo **já** satisfaz. (2) O slot único de `extends` é caro; gastá-lo em um alias é desperdício. (3) Se o incômodo era só o boilerplate, o alvo talvez seja apagar o wrapper e usar o delegate direto, ou expor o contrato via `type`. Na dúvida, aplique o inverso (técnica 11): composição é o default em TypeScript.
