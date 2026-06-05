# Desafio Técnico — Dev Sênior (Front + Manutenção de Backend)

Olá! Este desafio simula o que você vai fazer no dia a dia: construir uma interface em Vue consumindo uma API que **outro time** entregou, e, no processo, encontrar e corrigir problemas nesse backend.

Entregamos para você um backend Spring Boot funcional (pasta `omni-challenge-backend`), com README e API documentada. Ele simula, de forma simplificada, uma plataforma de atendimento omnichannel: mensagens chegam de diferentes canais, são agrupadas em conversas e o atendente responde.

> **Importante:** este backend **não** é perfeito. Ele tem alguns problemas plantados de propósito — alguns você percebe usando a tela, outros só lendo o código. Faz parte do desafio encontrá-los.

Use as ferramentas que preferir. Depois da entrega faremos uma conversa onde você apresenta a solução e discutimos suas decisões.

---

## Stack

- **Front:** Vue 3 (Composition API preferível). Use Vite. Gerência de estado e libs ficam a seu critério.
- **Backend (fornecido):** Java + Spring Boot, banco H2 em memória. Sobe com `mvn spring-boot:run` em `http://localhost:8080`. Veja o README dele.

---

## Parte 1 — Frontend (Vue)

Construa uma caixa de entrada de atendente, enxuta mas funcional, consumindo a API fornecida:

- **Lista de conversas** à esquerda, com nome do contato, canal, prévia da última mensagem e indicador de não-lidas.
- **Thread da conversa** selecionada à direita, com as mensagens em ordem.
- **Enviar resposta** pela thread.
- **Status de entrega** das mensagens enviadas pelo atendente (enviando / enviada / falhou) visível na interface.
- A tela deve **refletir mensagens novas** que chegam enquanto está aberta. Como fazer isso (polling, SSE, WebSocket) é decisão sua — justifique o trade-off no README.

Capricho visual **não** é o que avaliamos. Avaliamos organização do estado, estrutura de componentes, tratamento de erros/loading e clareza do código.

Endpoints principais (detalhes no README do backend):

```
GET  /api/conversations
GET  /api/conversations/{id}/messages
POST /api/conversations/{id}/messages   { "text": "..." }
POST /api/webhooks/inbound              (injetar uma mensagem recebida)
```

O webhook existe para você simular mensagens chegando enquanto desenvolve. Há um `requests.http` com exemplos prontos.

## Parte 2 — Manutenção do backend (caçar e corrigir bugs)

O backend que entregamos tem **6 problemas plantados**. Encontre quantos conseguir.

Para cada bug que encontrar:

1. **Descreva o sintoma** — o que está errado e como reproduzir.
2. **Aponte a causa raiz** — onde está, no código, e por que acontece.
3. **Corrija** — faça a correção no backend Spring.
4. **Justifique** — por que sua correção resolve de fato (e não só "esconde") o problema.

Não vamos dizer onde estão. Encontrar 4 bem corrigidos e bem explicados vale mais do que listar 6 superficialmente. Se suspeitar de um problema mas não conseguir corrigir, **documente mesmo assim** — o raciocínio conta.

---

## Escopo — o que NÃO precisa fazer

Para manter o esforço razoável (~4 a 8 horas), **não** implemente: autenticação/login, mídia além de texto, gestão de atendentes, ou deploy. Mocks e simplificações são esperados — só documente o que simplificou.

Se faltar tempo, **prefira profundidade a cobertura**: melhor um front sólido com alguns bugs bem resolvidos do que tudo pela metade. Diga no README o que priorizou.

---

## Entregáveis

1. Repositório (Git) com o **front** e o **backend já com suas correções** (pode ser um fork/cópia do backend fornecido).
2. **README do seu projeto** cobrindo:
   - Como rodar o front.
   - Decisões de arquitetura do front e principais trade-offs (incl. a estratégia de atualização em tempo real).
   - **Relatório de bugs:** para cada um — sintoma, causa raiz, correção e justificativa.
   - O que priorizou / simplificou / deixou de fora.
3. Testes automatizados onde você considerar crítico (não precisa de cobertura total).

---

## Como será avaliado

Numa conversa de ~45–60 min você vai apresentar a solução e vamos:

- Ver o front rodando e revisar sua organização de código.
- Discutir cada bug: como achou, por que acontece, por que sua correção é a correta.
- **Simular cenários** (mensagem duplicada, fora de ordem, provedor caindo, dois atendentes) e perguntar como o sistema — já corrigido — se comporta.
- Pedir uma **extensão ao vivo** (ex.: adicionar um terceiro canal, ou suporte a um tipo de mídia).

O que mais valorizamos: capacidade de entender código alheio, diagnóstico de causa raiz, qualidade das correções, clareza do front e honestidade sobre o que ficou de fora. Correção que mascara o sintoma sem resolver a causa conta **contra**.

Boa sorte — e divirta-se. Dúvidas sobre o enunciado, pode perguntar.
