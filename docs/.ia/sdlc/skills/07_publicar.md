# Skill: Publicar Funcionalidade

## Papel

Você é responsável por publicar uma funcionalidade já construída, com rastreabilidade, revisão por PR e promoção controlada entre `develop` e `main`. A branch dedicada e os commits pequenos já foram produzidos durante a etapa de build (`construir-funcionalidade`); esta skill não cria commits novos de implementação. Não faça merge ou publicação em produção sem os gates de aprovação definidos pelo projeto.

## Fluxo de branches

Use esta convenção, salvo se o projeto documentar outra:

```text
feature/<slug-da-feature> -> develop -> main
```

- A branch da funcionalidade nasce da `develop` atualizada e recebe os commits da feature durante o build.
- A branch da feature deve ser publicada no remoto e originar um PR para `develop`.
- Após aprovação e checks obrigatórios, o PR da feature pode ser mergeado em `develop`.
- A publicação em `main` ocorre por um segundo PR de `develop` para `main`, com seus próprios checks e aprovação.
- Nunca faça push direto em `develop` ou `main`, salvo política explícita do projeto.

## Pré-condições

Antes de publicar:

1. Confirme que a branch `feature/<slug-da-feature>` existe localmente, foi criada a partir de `develop` atualizada e contém apenas commits pequenos, coerentes e sem segredos, produzidos durante o build.
2. Confirme o plano, a especificação, o review core e os testes da feature.
3. Confirme que não há bloqueantes de segurança, review ou testes pendentes.
4. Leia as regras de contribuição, release, CI/CD e proteção de branches do projeto.
5. Confirme o slug da feature, o responsável pelo PR, os reviewers e o ambiente de destino.
6. Confirme que o diretório de trabalho não contém mudanças não relacionadas. Não descarte mudanças do usuário.

Se qualquer pré-condição falhar, pare e reporte o bloqueio — inclusive se a branch/commits não existirem, encaminhando de volta para a skill `construir-funcionalidade`.

## Processo

### 1. Validar antes do push

Execute os comandos reais documentados pelo projeto e confirme:

- critérios `CA-*` cobertos por cenários `CT-*`;
- review core aprovado;
- testes, lint, typecheck, build e demais checks obrigatórios aprovados;
- documentação e cenários atualizados;
- diff e lista de arquivos revisados;
- branch sem mudanças não relacionadas.

Registre os comandos e resultados no relatório ou PR. Não declare a branch pronta apenas porque o commit foi criado.

### 2. Publicar a branch e abrir o PR para `develop`, listando as mudanças

Faça push da branch dedicada e abra um PR com:

- título seguindo a convenção do projeto;
- resumo da motivação e da solução;
- link para o plano, overview e `-tests.md` da feature;
- lista das mudanças por commit/camada;
- critérios `CA-*` e cenários `CT-*` cobertos;
- lista de validações executadas;
- migrações, flags, riscos, rollback e impacto operacional;
- reviewers e labels adequados.

O PR deve ter `develop` como base. Aguarde os checks e a aprovação exigida antes do merge.

### 3. Automatizar aprovação e merge com segurança

A automação recomendada não deve aprovar o próprio código sem uma política explícita. Configure no provedor de Git:

- proteção de `develop` e `main`;
- PR obrigatório, branch atualizada e checks obrigatórios;
- número mínimo de aprovações por pessoas ou equipes autorizadas;
- dismiss de aprovação quando novos commits alterarem o PR;
- proibição de autoaprovação pelo autor, bot ou conta que criou o PR;
- CODEOWNERS, quando suportado;
- merge automático somente depois de aprovação válida e checks verdes;
- auditoria de quem aprovou, quem mergeou e qual commit foi promovido.

Um bot pode abrir PRs, solicitar reviewers, acompanhar checks e habilitar auto-merge depois dos gates. A aprovação pode ser automatizada apenas se o projeto aceitar formalmente uma aprovação por serviço, com identidade separada, permissões mínimas e trilha de auditoria. Nunca simule aprovação humana para contornar proteção de branch.

### 4. Promover `develop` para `main`

Depois do merge em `develop`:

1. Atualize `develop` e confirme o commit que será promovido.
2. Calcule a próxima versão no formato `AA.MM.xx`:
	- `AA`: dois últimos dígitos do ano;
	- `MM`: mês de publicação;
	- `xx`: sequência de duas posições dentro do mês, iniciando em `01`.
3. Verifique tags/releases existentes para evitar duplicidade.
4. Crie um PR de `develop` para `main` com a versão, changelog, mudanças incluídas e plano de rollback.
5. Aguarde checks e aprovação independentes do PR de produção.
6. Após o merge aprovado em `main`, crie a tag `vAA.MM.xx` conforme a convenção do projeto e publique a release.
7. Valide o commit, a tag, o artefato publicado, migrations, health checks e métricas pós-release.

Não crie a tag antes da confirmação do commit em `main`. Se o projeto usar outro mecanismo de release, documente a exceção no PR.

### 5. Reportar a publicação

Registre:

- branch da feature e commits publicados;
- URL e estado dos PRs para `develop` e `main`;
- aprovadores, checks e timestamps relevantes;
- versão `AA.MM.xx`, tag e release;
- comandos executados e resultados;
- migrações, flags, incidentes, rollback e validação pós-release.

Nunca registre tokens, senhas ou strings de conexão no relatório.
