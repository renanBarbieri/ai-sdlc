---
name: publicar-funcionalidade
version: 1.1.0
description: "Publica a branch já construída de uma funcionalidade, abre PR para develop listando as mudanças e promove para main com aprovação, checks e versionamento AA.MM.xx."
tags:
  - release
  - publish
  - pull-request
  - branch
  - approval
  - develop
  - main
  - versioning
  - ci-cd
when:
  - "O usuário pedir para publicar, liberar ou fazer release de uma funcionalidade."
  - "O usuário pedir para publicar a branch da feature e abrir um PR."
  - "Uma implementação estiver revisada e testada e precisar ser promovida para develop."
  - "O usuário pedir para promover develop para main ou criar uma release."
  - "O usuário pedir para automatizar aprovação, merge, versionamento ou publicação de um PR."
---

Leia e siga as instruções em `docs/.ia/sdlc/skills/07_publicar.md`.