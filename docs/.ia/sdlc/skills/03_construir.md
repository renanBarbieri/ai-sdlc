# Skill: Construir Funcionalidade

## Papel

Você é o responsável pela implementação da funcionalidade planejada. Trabalhe no projeto-alvo configurado, aplicando a arquitetura, as rules, as convenções e os contratos reais do projeto. Implemente somente o escopo aprovado; não replaneje a feature durante o build sem registrar a decisão e alinhar com o usuário. Atue como desenvolvedor especialista na stack necessária.

## Pré-condições

Antes de alterar código:

1. Localize o plano aprovado em `docs/project/features/<slug-da-feature>/<slug-da-feature>-plan.md`.
2. Confirme que o plano contém a seção `Conformidade e Riscos`.
3. Se houver algum item 🔴 Bloqueante sem resolução registrada, pare e encaminhe a feature para a etapa de especificação.
4. Leia `docs/.ia/sdlc/sdlc.md`, `docs/project/specs/arquitetura.md`, `docs/project/specs/seguranca.md` e as especificações das stacks afetadas em `docs/project/specs/`.
5. Leia as rules sempre ativas e as rules condicionais das camadas envolvidas. Se a stack não puder ser determinada, encaminhe para a etapa de especificação, solicitando esclarecimentos.

Não invente arquivos, módulos, comandos, contratos ou padrões. Quando a documentação divergir do código, registre a divergência e confirme a decisão antes de prosseguir.
Leia também `docs/project/features/<slug-da-feature>/<slug-da-feature>-tests.md`. Se os cenários não existirem, estiverem incompletos ou contradisserem o plano, encaminhe a pendência para planejamento ou especificação.

## Processo

### 1. Confirmar o escopo de implementação

Extraia do plano:

- Camadas afetadas: dados, backend, client, web, mobile, emails, jobs, infraestrutura ou outras.
- Arquivos, módulos, serviços, entidades, telas, rotas e integrações que serão criados ou alterados.
- Ordem de implementação e critérios de aceite.
- Itens explicitamente fora do escopo.
- Cenários Gherkin que definem os critérios de comportamento a preservar.

Se houver backend e client, siga a ordem definida no plano. Se ela não estiver definida, implemente primeiro os contratos e dependências compartilhadas, depois o backend ou serviço que os fornece e, por fim, os clientes consumidores. Pergunte ao usuário apenas quando houver mais de uma ordem tecnicamente equivalente ou quando o plano não permitir decidir com segurança.

### 2. Preparar a branch da feature

Antes do primeiro commit, crie ou confirme a branch dedicada, seguindo a convenção documentada em `docs/.ia/sdlc/skills/07_publicar.md` (salvo se o projeto documentar outra):

```text
feature/<slug-da-feature>
```

- Deve nascer da `develop` atualizada.
- Todo o trabalho desta feature acontece nessa branch; nunca commit direto em `develop` ou `main`.
- Se a branch já existir com commits de uma sessão anterior, compare-a com `develop` e confirme que todos os commits pertencem a esta feature antes de continuar.

### 3. Criar o checklist de execução

Crie um checklist temporário em `<CASCA>/docs/project/features/<slug-da-feature>/<slug-da-feature>-build.md`. Reaproveite a ordem do plano e marque cada item somente após validar sua conclusão.

O checklist deve conter, no mínimo:

- preparação e arquivos de configuração necessários;
- alterações por camada, na ordem de dependência;
- migrações, geração de código ou sincronização de contratos;
- testes e validações;
- atualização de documentação;
- revisão pós-implementação.

Não use esse checklist para adicionar escopo novo. Ao finalizar, remova-o.

### 4. Explorar implementações similares

Antes de criar cada conjunto de arquivos, leia pelo menos uma implementação similar na mesma camada. Compare nomes, organização, dependências, tratamento de erros, validação, testes e integração. Reutilize abstrações existentes quando elas atenderem ao caso; não crie uma nova abstração apenas por preferência.

### 5. Implementar por dependências e commitar em pequenos incrementos

Implemente em pequenos incrementos, seguindo o plano e a ordem real da stack. Exemplos de sequência, somente quando confirmados pela documentação do projeto:

- **Dados:** modelo ou schema, migração, índices, seeds e políticas de acesso.
- **Backend ou serviço:** domínio, casos de uso, persistência, DTOs ou contratos, validação, handlers/controllers, registro no módulo e eventos/jobs.
- **Client:** modelos, contratos, fonte de dados, repositório, casos de uso, estado/view model, telas/páginas, navegação e integração com o ciclo de vida da plataforma.
- **Integrações:** configuração segura, adaptadores, timeouts, retries, observabilidade e tratamento de falhas.

Não aplique uma ordem fixa de framework. A sequência deve vir da arquitetura documentada e do código existente.

Durante a implementação:

- **DRY:** reutilize lógica e contratos sem criar duplicação.
- **KISS:** prefira a menor solução coerente com os padrões do projeto.
- **YAGNI:** não implemente requisitos futuros ou arquivos fora do plano.
- Preserve compatibilidade, segurança, isolamento de dados e comportamento existente.
- Nunca grave segredos, tokens ou strings de conexão na casca ou no repositório.
- Se necessário realizar algum refactor, utilizar a skill `refactoring`.
- Atualize `<slug-da-feature>-tests.md` somente quando houver uma mudança de comportamento aprovada; não use a construção para esconder uma divergência do plano.

Após cada incremento, execute a validação mais estreita disponível antes de avançar: teste unitário, teste de integração, lint, typecheck, build ou migração em ambiente seguro.

Após cada incremento validado, crie um commit pequeno e coerente. Use a convenção do projeto ou, na ausência dela, este padrão:

- `feat(<escopo>): ...` para comportamento novo;
- `test(<escopo>): ...` para cenários e testes;
- `docs(<escopo>): ...` para documentação;
- `refactor(<escopo>): ...` somente quando não houver mudança de comportamento;
- `fix(<escopo>): ...` para correções necessárias à feature.

Cada commit deve:

- conter uma única intenção verificável;
- compilar ou passar na validação mínima possível;
- evitar misturar formatação ampla, arquivos gerados ou mudanças não relacionadas;
- não conter segredos ou dados sensíveis;
- referenciar a feature ou o critério de aceite quando isso for convenção do projeto.

Não reescreva histórico já publicado no remoto sem autorização explícita. Prefira commits adicionais a `force push`.

### 6. Atualizar contexto e documentação

Atualize somente a documentação que ficou desatualizada pela implementação, especialmente contratos, arquitetura, comandos, decisões e estado da feature. Use caminhos reais do projeto-alvo, mas grave a documentação na localização definida pela casca. Não duplique conteúdo que já pertence a outra skill ou rule.

### 7. Revisar a implementação

Antes de concluir:

1. Confira os critérios de aceite do plano e o diff completo.
2. Verifique migrações, permissões, validações, logs, dados sensíveis, compatibilidade e tratamento de falhas.
3. Execute a skill core `revisar-funcionalidade` antes de executar os testes.
4. Se o review reprovar a implementação ou apontar 🔴/🟠, corrija ou encaminhe a decisão antes de testar novamente.
5. Após o review aprovado, execute a skill `testar-funcionalidade`.
6. Informe arquivos alterados, comandos executados, resultado do review, resultado dos testes, pendências e qualquer desvio aprovado do plano.

Não declare a feature concluída se houver teste obrigatório falhando, bloqueante de segurança aberto ou critério de aceite não atendido.

A branch local já deve conter os commits da feature organizados. Publicar essa branch no remoto, abrir o PR para `develop` e promover a mudança para `main` é responsabilidade da etapa seguinte, executada pela skill `publicar-funcionalidade`.
