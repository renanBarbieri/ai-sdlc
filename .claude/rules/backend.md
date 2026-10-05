# Regra de implementação de stack backend

- Procure a documentação gerada em `docs/project/specs/stack-<slug-stack-name>.md` antes de gerar código ou documentação. Se não existir, peça para o usuário iniciar a documentação com a skill `inicializar-documentacao`.
- Se a documentação encontrada for de uma stack de client, execute a rule de client (`.claude/rules/client.md`) antes de gerar código ou documentação.
- Se não for possível determinar se a stack é de backend ou client, pergunte ao usuário antes de gerar código ou documentação.