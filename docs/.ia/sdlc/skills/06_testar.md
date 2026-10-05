# Skill: Testar Funcionalidade

## Papel

Você é responsável por transformar os critérios de aceite em cenários verificáveis, executar os testes da funcionalidade e reportar evidências. Use Gherkin para descrever o comportamento esperado, independentemente da ferramenta de automação adotada pelo projeto.

## Artefato principal

O cenário de testes da funcionalidade deve ficar em:

`docs/project/features/<slug>/<slug>-tests.md`

Esse arquivo é criado na etapa de planejamento, validado na especificação, atualizado durante a construção quando necessário e executado nesta etapa. Se ainda não existir, crie-o antes de implementar os testes automatizados.

## Pré-condições

1. Leia `docs/.ia/sdlc/sdlc.md`.
2. Leia o plano e a especificação da funcionalidade:
	 - `docs/project/features/<slug>/<slug>-plan.md`;
	 - seção `Conformidade e Riscos` no mesmo plano;
	 - `docs/project/features/<slug>/<slug>-overview.md`, quando existir.
3. Leia `docs/project/specs/testes.md`, a especificação de cada stack afetada e as rules aplicáveis.
4. Identifique os comandos reais de teste, cobertura, build e validação do projeto.

Se houver bloqueante não resolvido ou se a stack e os comandos não puderem ser determinados, pare e registre a pendência. Não invente uma ferramenta de testes.

## Processo

### 1. Definir a estratégia

Mapeie cada critério de aceite para pelo menos um cenário. Determine, conforme o projeto:

- testes unitários, de integração, contrato, componente, interface, end-to-end ou equivalentes;
- dados, fixtures, mocks, stubs, ambiente e pré-condições necessários;
- cenários positivos, negativos, limites, autorização, isolamento, falhas de integração e regressões;
- validações específicas de web, mobile, backend, jobs, emails, eventos ou infraestrutura, quando aplicável.

Priorize o comportamento de negócio e as fronteiras entre camadas. Não transforme detalhes de implementação em cenários Gherkin quando o usuário não observa essa diferença.

### 2. Criar ou atualizar os cenários Gherkin

Edite `docs/project/features/<slug>/<slug>-tests.md` usando linguagem do domínio e a estrutura:

```gherkin
# language: pt
Funcionalidade: [comportamento de negócio]
	Como [tipo de usuário ou sistema]
	Quero [intenção]
	Para [valor ou resultado]

	Contexto:
		Dado que [pré-condição comum]

	Cenário: [resultado principal]
		Dado que [estado inicial]
		E [outra pré-condição]
		Quando [ação observável]
		Então [resultado observável]
		E [efeito adicional verificável]

	Esquema do Cenário: [variações relevantes]
		Quando [ação com <entrada>]
		Então [resultado <esperado>]

		Exemplos:
			| entrada | esperado |
			| valor   | resultado |
```

Regras para os cenários:

- um cenário deve verificar um resultado principal;
- use `Dado`, `Quando` e `Então` para fatos observáveis, não chamadas internas;
- cubra sucesso, validação, autorização, falhas e limites relevantes;
- não duplique cenários que só mudam detalhes irrelevantes;
- marque cenários que ainda não têm automação como `Pendente` no documento;
- nunca inclua segredos, dados reais sensíveis ou strings de conexão.

### 3. Implementar ou ajustar os testes automatizados

Para cada cenário `CT-*`, localize uma implementação similar e siga a estratégia da stack. Crie testes na camada mais barata que prove o comportamento e adicione testes de integração ou end-to-end apenas quando forem necessários para validar contratos reais.

Mantenha a separação:

- Gherkin descreve comportamento e critérios de aceite;
- testes automatizados fornecem a verificação executável;
- fixtures e mocks representam dependências sem esconder contratos importantes.

Não altere o código de produção para fazer o teste passar sem investigar a causa. Se a implementação estiver incorreta, registre o achado e encaminhe a correção conforme o fluxo do projeto.

### 4. Executar as validações

Execute os comandos documentados pelo projeto, na ordem mais estreita para a mais ampla:

1. teste ou cenário diretamente afetado;
2. suíte da camada;
3. suíte de integração ou end-to-end;
4. cobertura, typecheck, lint, build e demais gates obrigatórios.

Registre no arquivo de testes ou no relatório da execução:

- comando executado;
- resultado;
- cenário `CT-*` coberto;
- teste automatizado correspondente;
- cenários aprovados, falhos e pendentes;
- limitações do ambiente;
- cobertura, quando houver meta definida.

### 5. Encaminhar resultados

- Se todos os critérios estiverem cobertos e as validações passarem, marque a funcionalidade como testada.
- Se houver falha de comportamento, encaminhe para correção e repita os testes.
- Se faltar um cenário por ambiguidade de requisito, encaminhe para planejamento ou especificação.
- Se a ferramenta ou ambiente impedir a execução, registre a limitação sem declarar aprovação.

Ao final, informe o arquivo de cenários, os testes criados ou alterados, os comandos executados, os resultados e as pendências.