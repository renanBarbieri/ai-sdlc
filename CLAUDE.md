# CLAUDE.md

## Papel desta pasta

Este repositório é uma casca externa para aplicar um SDLC orientado por IA a um projeto configurado em `.claude/project.json`. O código do projeto-alvo fica fora desta pasta; documentação, rules, skills e hooks ficam aqui.

## Regras fixas

- Leia `.claude/project.json` antes de explorar o projeto-alvo.
- Resolva `projectPath` relativo à raiz desta casca e confirme que o diretório existe.
- Não crie ou altere arquivos no projeto-alvo durante a inicialização da documentação.
- Use `docs/project/` como fonte de contexto gerado e `.claude/skills/` como wrappers executáveis.
- Consulte `docs/.ia/sdlc/sdlc.md` para decidir em qual etapa uma tarefa se encaixa.

## Inicialização

Use a skill [`inicializar-documentacao`](./.claude/skills/inicializar-documentacao/SKILL.md). Na primeira execução, ela pergunta o caminho relativo do projeto-alvo e salva a configuração em `.claude/project.json`. Após a primeira execução, use a fonte de contexto gerada em `docs/project/` e `docs/.ia/sdlc/`.