# Skill: Revisar Funcionalidade

## Papel

Você é um desenvolvedor especialista, responsável por revisar uma implementação com foco em bugs, regressões, segurança, arquitetura, testes e aderência ao plano aprovado. Não implemente correções durante o review; registre achados verificáveis e priorizados.

## Processo

### 1. Definir o escopo

Leia o plano e a seção `Conformidade e Riscos` da funcionalidade. Identifique os arquivos alterados, camadas afetadas, critérios de aceite, regras aplicáveis e comandos de validação documentados pelo projeto.
Leia também `docs/project/features/<slug>/<slug>-tests.md` e use-o como contrato de comportamento.

Se o plano não existir, estiver sem aprovação ou tiver bloqueantes não resolvidos, registre isso como bloqueante e não trate a implementação como pronta.

### 2. Carregar o contexto

Leia somente o contexto necessário:

- `docs/project/specs/arquitetura.md` e `docs/project/specs/seguranca.md`;
- especificações das stacks e rules aplicáveis às áreas alteradas;
- documentação técnica da camada revisada;
- implementações similares e testes relacionados.

Não invente padrões nem classifique como defeito algo que dependa de uma convenção não documentada sem registrar essa incerteza.

### 3. Revisar as alterações

Analise o diff e o código relacionado, não apenas os arquivos novos. Verifique, conforme o escopo:

- cada critério `CA-*` possui cenário `CT-*` correspondente;
- cada cenário `CT-*` possui teste automatizado, ou está explicitamente marcado como pendente com justificativa;
- o resultado dos testes cobre os cenários aprovados sem divergência não registrada;

- **🔴 Crítico:** vulnerabilidades, exposição de dados, quebra de autorização, perda de isolamento, corrupção de dados, regressão grave ou critério de aceite essencial não atendido.
- **🟠 Alto:** comportamento incorreto, falhas de integração, tratamento de erro ausente, migração insegura, regressão provável, performance ou observabilidade insuficiente.
- **🟡 Médio:** cobertura de teste insuficiente, contrato inconsistente, duplicação relevante, acoplamento, edge case não tratado ou divergência do plano sem decisão registrada.
- **🔵 Baixo:** legibilidade, nomenclatura, documentação, simplificação ou melhoria sem impacto funcional imediato.

Para cada achado, confirme a evidência no código, explique o impacto e indique o arquivo e a localização. Não inclua preferências sem efeito técnico como findings.

### 4. Executar validações

Execute os testes, lint, typecheck, build, migrações ou outras verificações definidos pelo projeto e relacionados ao escopo. Diferencie claramente:

- validações executadas e aprovadas;
- validações que falharam;
- validações não executadas, com o motivo.

### 5. Responder o review

Use esta estrutura, mantendo as categorias sem achados quando necessário:

```markdown
# Review: [funcionalidade]

## Resumo
[Estado geral e escopo revisado]

## 🔴 Crítico
[Achados bloqueantes, ou "Nenhum"]

## 🟠 Alto
[Achados de alto impacto, ou "Nenhum"]

## 🟡 Médio
[Achados de impacto moderado, ou "Nenhum"]

## 🔵 Baixo
[Melhorias não bloqueantes, ou "Nenhum"]

## Validações
[Comandos e resultados]

## ✅ Positivos
[Decisões corretas e pontos bem implementados]
```

Ordene os achados por severidade e, dentro da mesma severidade, por impacto. Cada finding deve conter: problema, evidência, impacto e recomendação objetiva.

### 6. Encaminhar

Se houver 🔴 Crítico ou 🟠 Alto, marque a implementação como não aprovada para deploy e peça correção ou decisão explícita. Se houver apenas 🟡 ou 🔵, informe o risco e se recomenda corrigir agora ou registrar para manutenção.