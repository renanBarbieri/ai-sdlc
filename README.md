# claude-sdlc

Casca externa e reutilizável para aplicar o [AI-Native SDLC Playbook](https://claude.com/blog/the-ai-native-sdlc-playbook) a qualquer projeto usando Claude Code — sem copiar regras, skills ou documentação para dentro do projeto analisado.

## Ideia central

O ciclo de desenvolvimento tem 7 etapas — **Plan → Design → Build → Review → Test → Deploy → Maintain** — e cada uma se apoia em um de dois mecanismos:

- **Rules** (`.claude/rules/`): padrões genéricos da casca, sempre ativos durante a sessão. Elas orientam como explorar e alterar o projeto-alvo, sem fingir conhecer sua stack.
- **Skills** (`.claude/skills/*/SKILL.md`): procedimentos sob demanda para plan, design, build, review, test, deploy e maintain.
- **Contexto do alvo** (`.claude/project.json` + `docs/project/` + `docs/.ia/`): configuração e documentação gerada ficam na casca. O código do projeto-alvo é somente lido e alterado quando uma tarefa de desenvolvimento pedir isso explicitamente.

| Etapa | Artefato | Mecanismo |
|---|---|---|
| 1. Plan | documento de plano | Skill |
| 2. Design | seção de conformidade no mesmo documento | Skill (aplica as rules como restrição) |
| 3. Build | código | Rule (sempre-on + por stack) + Skill |
| 4. Review | achados de code review e decisão de aprovação | Skill |
| 5. Test | testes + CI | Skill + pipeline |
| 6. Deploy | PR + release | Skill |
| 7. Maintain | achados de auditoria → novo plano | Skill agendável |

Detalhes completos em [`docs/.ia/sdlc/sdlc.md`](./docs/.ia/sdlc/sdlc.md).

**Onde entra especialidade de stack** (Node, React, Android, React Native, etc.): nunca dentro das rules/skills genéricas deste repo. Ela entra em dois lugares, criados na casca para o projeto configurado:

- Regra escopada por stack: `.claude/rules/<stack>.md`, com `paths:` relativos ao projeto-alvo.
- Conhecimento arquitetural detalhado: `docs/project/specs/stack-<slug-stack-name>.md`.

## Como usar

1. Copie ou clone este repositório em uma pasta de trabalho, por exemplo `nomeProjetoAI`.
2. Abra Claude Code na raiz da casca, não na raiz do projeto-alvo.
3. Rode a skill [`inicializar-documentacao`](./.claude/skills/inicializar-documentacao/SKILL.md) (ou peça para "inicializar a documentação do projeto"). Na primeira execução, informe o caminho relativo do projeto-alvo, por exemplo `../meu-projeto`.
4. O caminho é salvo em `.claude/project.json` (ignorado pelo Git; veja [`project.example.json`](./.claude/project.example.json)). As próximas sessões devem reutilizá-lo e perguntar novamente apenas quando ele estiver ausente, inválido ou quando você pedir para trocar de projeto.
5. O Claude explora o alvo e gera/atualiza `docs/project/` e `docs/.ia/sdlc/sdlc.md` dentro da casca. As rules em `.claude/rules/` e as skills em `.claude/skills/` já vêm prontas neste repositório. Nada é criado no projeto-alvo durante a inicialização.

Para trocar de alvo, edite `.claude/project.json` ou remova o arquivo e rode a skill novamente.

## Origem

Extraído e generalizado a partir da instância aplicada no projeto Kakeboo.
