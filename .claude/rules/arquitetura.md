# Regra de arquitetura

- A casca controla contexto e automação; o projeto-alvo controla o código de produto.
- Documentação gerada deve usar caminhos reais do alvo, mas ser gravada somente na casca: `docs/project/` para a documentação do projeto-alvo (overview, specs, features) e `docs/.ia/` para o framework de SDLC (mapeamento de etapas e instruções de skill).
- Não invente stack, módulos, comandos ou contratos: marque como desconhecido até verificar no alvo.
- Skills devem referenciar a documentação gerada em vez de duplicá-la.
- Leia a arquitetura do projeto-alvo em `docs/project/specs/arquitetura.md` antes de gerar documentação ou código. Se não existir, peça para o usuário iniciar a documentação com a skill `inicializar-documentacao`.