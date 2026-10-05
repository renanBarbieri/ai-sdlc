# Ciclo de Vida de Desenvolvimento (SDLC)

Mapeamento das 7 etapas do processo de desenvolvimento para os mecanismos usados — baseado no [AI-Native SDLC Playbook](https://claude.com/blog/the-ai-native-sdlc-playbook).

| Etapa | Artefato | Mecanismo | Onde está |
|---|---|---|---|
| **1. Plan** | Plano + critérios de aceite + cenários Gherkin (`docs/project/features/<slug>/<slug>-plan.md`, `<slug>-tests.md`) | Skill | `planejar-funcionalidade` |
| **2. Design** | Seção "Conformidade e Riscos" no mesmo documento | Skill (aplica as rules como restrição) | `especificar-funcionalidade` |
| **3. Build** | Código + branch `feature/<slug>` com commits pequenos | Rule (sempre-on + por stack) + Skill + Hook | `.claude/rules/`, `construir-funcionalidade`, hook de lint pós-edição |
| **4. Review** | Achados de code review e decisão de aprovação | Skill | `revisar-funcionalidade` (core review) |
| **5. Test** | Testes + CI | Skill + pipeline | `testar-funcionalidade`, pipeline documentado pelo projeto |
| **6. Deploy** | PR para `develop`, promoção para `main` e release versionada `AA.MM.xx` | Skill | `publicar-funcionalidade` |
| **7. Maintain** | Achados de auditoria → correção direta posterior ou novo plano | Skill agendável | `auditar-funcionalidade` |

## Onde entra especialidade de stack

Este documento e as skills acima são propositalmente agnósticos de stack. Convenções específicas de uma tecnologia (ex.: Node/NestJS, React, Android, React Native) entram em dois lugares:

- **Regra sempre-ativa por stack**: `.claude/rules/<stack>.md`, escopada por `paths:` no frontmatter (ex.: `.claude/rules/backend.md` com `paths: ["<pasta-do-backend-no-alvo>/**"]`).
- **Conhecimento arquitetural detalhado por stack**: `docs/project/specs/stack-<slug-stack-name>.md` (uma stack nova replica o mesmo padrão de arquivo).

Nenhuma skill ou rule desta casca deve conter conteúdo de uma stack específica fora desses dois locais.

## Referências

- Regras sempre-on: `CLAUDE.md`, `.claude/rules/`
- Instruções de cada etapa: `docs/.ia/sdlc/skills/`
