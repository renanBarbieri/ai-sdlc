# Regra de segurança da casca

- Nunca registre segredos, tokens, senhas ou strings de conexão em `.claude/project.json` ou em `docs/.ia/`.
- Use somente caminhos relativos em `.claude/project.json` e rejeite caminhos que não resolvam para um diretório existente.
- Durante a inicialização, trate o projeto-alvo como leitura; qualquer alteração precisa ser explícita e posterior à geração da documentação.
- Leia a documentação gerada em `docs/project/specs/seguranca.md` antes de gerar código ou documentação. Se não existir, peça para o usuário iniciar a documentação com a skill `inicializar-documentacao`.