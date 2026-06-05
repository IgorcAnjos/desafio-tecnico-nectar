# Omni Challenge — Igor Catulo

Plataforma de atendimento omnichannel: backend Spring Boot corrigido + frontend Vue 3 com caixa de entrada para atendentes.

---

## Como rodar

### Opção 1 — Docker (recomendado)

**Pré-requisito:** Docker e Docker Compose instalados.

```bash
docker compose up --build
```

- Frontend: **http://localhost:5173**
- Backend: **http://localhost:8080**

O compose aguarda o backend estar saudável antes de subir o frontend. O primeiro build baixa dependências Maven e npm — leva alguns minutos. Builds subsequentes são rápidos graças ao cache de layers.

```bash
# Parar
docker compose down
```

---

### Opção 2 — Direto no terminal

**Pré-requisitos:** Java 17+, Maven 3.8+ e Node.js 18+.

**Terminal 1 — backend:**

```bash
cd omni-challenge-backend
mvn spring-boot:run
```

O backend estará disponível em **http://localhost:8080**. O banco H2 é recriado a cada boot com seed de 3 conversas de exemplo.

**Terminal 2 — frontend:**

```bash
cd omni-challenge-frontend
npm install
npm run dev
```

O frontend estará disponível em **http://localhost:5173**. CORS já está habilitado no backend para essa origem.

> Inicie o backend antes do frontend. O frontend aguarda conexão SSE com o backend ao montar; se o backend ainda não estiver de pé, a conexão vai reconectar automaticamente após o boot.

---

## Arquitetura do frontend

Interface em duas colunas: lista de conversas à esquerda, thread da conversa selecionada à direita. Construída com Vue 3 + Vite + TypeScript.

| Camada | Escolha | Motivo |
|---|---|---|
| Estado global | Pinia (setup stores) | API reativa sem boilerplate |
| HTTP | Axios via `src/api/client.ts` | Instância centralizada, fácil de mockar nos testes |
| Ícones | lucide-vue-next | SVG flat, sem component library pesada |
| Testes | Vitest + @vue/test-utils | Integração nativa com Vite, zero config |

Nenhuma component library. CSS vanilla com `<style scoped>` por componente.

### Estrutura de componentes

```
App.vue                    ← grid duas colunas
├── ConversationList.vue   ← painel esquerdo
│   └── ConversationItem.vue  ← item: nome, canal, prévia, badge não-lidas
└── ConversationThread.vue ← painel direito
    ├── MessageBubble.vue  ← bolha INBOUND/OUTBOUND + ícone de status
    └── ReplyInput.vue     ← campo de texto + botão enviar
```

### Stores Pinia

**`conversations.ts`** — lista de conversas, conversa ativa, conexão SSE.
**`messages.ts`** — mensagens da thread, envio com update otimista, status de entrega.

---

## Atualização em tempo real — decisão e trade-offs

Estratégia adotada: **SSE (Server-Sent Events)** via `EventSource` nativo do browser.

O backend expõe `GET /api/events` com dois eventos nomeados:

| Evento | Payload | Quando |
|---|---|---|
| `conversation-updated` | `ConversationSummary` da conversa afetada | Após inbound, reply ou markAsRead |
| `message-added` | `{ conversationId, message }` | Após inbound ou reply |

O frontend abre uma única conexão SSE ao montar `ConversationList`. Os handlers atualizam o estado Pinia sem requisições HTTP adicionais (fat events — payload cirúrgico por evento, sem round-trip extra).

### Por que SSE e não polling ou WebSocket

**Polling** foi a primeira implementação (descartada): gera requisições a cada 3s mesmo sem dados novos, desperdiça recursos e atrasa a entrega de eventos em até 3s.

**WebSocket** foi descartado por overengineering: a comunicação é majoritariamente server→client. O POST normal já cobre client→server. Bidirecionalidade não agrega valor nesse escopo.

**SSE** é a escolha correta: conexão persistente unidirecional, reconexão automática pelo protocolo, suporte nativo no browser sem biblioteca, e Spring Boot suporta via `SseEmitter` sem dependências extras. A migração de polling para SSE foi localizada nos stores — nenhum componente foi alterado.

### Infraestrutura Docker

O nginx proxy encaminha `/api/events` para o backend com `proxy_buffering off` — obrigatório para SSE funcionar atrás de proxy reverso (sem isso os eventos ficam presos no buffer e nunca chegam ao browser).

---

## Testes

### Frontend — 20 testes (Vitest)

```bash
cd omni-challenge-frontend && npm install && npm run test
```

Cobre os casos críticos dos stores: deduplicação de mensagens, transição de status PENDING→SENT/FAILED, merge de eventos SSE sem duplicata, ordenação por `sentAt`, update otimista.

### Backend — 32 testes (JUnit 5 + MockMvc)

```bash
cd omni-challenge-backend && mvn test
```

Testes de integração via HTTP com `@SpringBootTest` + `MockMvc` + H2 em memória. Cobre todos os 6 bugs corrigidos.

---

## Relatório de bugs

### B1 — DeliveryStatus nunca fica FAILED

**Sintoma:** `POST /api/conversations/{id}/messages` sempre retorna `deliveryStatus: "SENT"` mesmo quando o provedor falha. O atendente nunca sabe que a mensagem não foi entregue.

**Causa raiz:** em `InboxService.sendReply()`, o status era definido como `SENT` *antes* de chamar o provedor. O bloco `catch` estava vazio — a exceção era silenciada e o status permanecia `SENT` mesmo com falha.

```java
// antes — bugado
message.setDeliveryStatus(DeliveryStatus.SENT);  // definido antes do envio
messages.save(message);
try {
    providerClient.send(...);
} catch (Exception e) {
    // catch vazio — falha silenciada
}
```

**Correção:** status inicial `PENDING`; `SENT` definido após sucesso do provedor; `FAILED` no `catch`. Como a entidade é gerenciada pelo contexto JPA dentro da transação `@Transactional`, o dirty check persiste a mudança automaticamente no commit — sem segundo `save()`.

```java
// depois — correto
message.setDeliveryStatus(DeliveryStatus.PENDING);
messages.save(message);
try {
    providerClient.send(...);
    message.setDeliveryStatus(DeliveryStatus.SENT);
} catch (Exception e) {
    message.setDeliveryStatus(DeliveryStatus.FAILED);
}
```

**Justificativa:** a correção resolve a causa raiz (sequência errada + catch vazio), não mascara o sintoma. O status reflete o resultado real do envio ao provedor.

---

### B2 — Thread de mensagens fora de ordem cronológica

**Sintoma:** `GET /api/conversations/{id}/messages` retorna mensagens na ordem de inserção no banco (PK `id` ASC), não pela ordem de envio (`sentAt` ASC). Na conversa de seed, "numero do pedido: 4421" (sentAt `base+2min`) aparece após "Podem verificar?" (sentAt `base+10min`).

**Causa raiz:** `MessageRepository` declarava `findByConversationIdOrderByIdAsc` — o Spring Data gerava `ORDER BY id ASC`. O `id` é autoincremento de inserção, não reflete a ordem cronológica.

**Correção:** renomear o método para `findByConversationIdOrderBySentAtAsc` e atualizar a chamada em `InboxService.getMessages()`. O Spring Data gera automaticamente `ORDER BY sent_at ASC`.

**Justificativa:** `sentAt` representa a intenção real do remetente, independente de latência de rede ou ordem de chegada ao servidor. Ordenar por PK é um detalhe de implementação do banco, não um contrato semântico.

---

### B3 — Webhook sem idempotência duplica mensagens

**Sintoma:** `POST /api/webhooks/inbound` com o mesmo `externalId` cria uma nova mensagem a cada chamada. Dois POSTs idênticos geram duas entradas no banco e incrementam `unreadCount` duas vezes.

**Causa raiz:** o campo `externalId` existia na entidade `Message` mas sem constraint `unique = true` no banco e sem verificação em `ingest()` antes de persistir.

**Correção:** três mudanças coordenadas:
1. `Model.java` — `@Column(name = "external_id", unique = true)` — constraint no banco
2. `Repositories.java` — `Optional<Message> findByExternalId(String externalId)` — busca antes de inserir
3. `InboxService.ingest()` — verificação de idempotência como primeiro bloco: se `externalId` já existe, retorna o `MessageView` existente sem criar nova mensagem

**Justificativa:** idempotência silenciosa (mesma entrada → mesma saída) é o contrato correto para webhooks. Retornar erro seria incorreto — o provedor pode reenviar o mesmo evento por política de retry. O `unique = true` é a segunda camada de defesa contra race conditions concorrentes.

---

### B4 — `lastMessageAt` retroage com mensagens atrasadas

**Sintoma:** quando uma mensagem chega com `sentAt` no passado (mensagem atrasada do provedor), a conversa cai para baixo na lista ordenada por `lastMessageAt DESC`, mesmo sendo a conversa mais recente.

**Causa raiz:** `InboxService.ingest()` chamava `conversation.setLastMessageAt(message.getSentAt())` incondicionalmente — qualquer mensagem sobrescrevia o timestamp, mesmo com `sentAt` anterior ao valor atual.

**Correção:** atualização condicional — `lastMessageAt` só é atualizado se o `sentAt` da nova mensagem for *posterior* ao valor atual:

```java
if (conversation.getLastMessageAt() == null
        || message.getSentAt().isAfter(conversation.getLastMessageAt())) {
    conversation.setLastMessageAt(message.getSentAt());
    conversation.setLastMessagePreview(message.getText());
}
```

**Justificativa:** `lastMessageAt` deve refletir o momento da mensagem mais recente conhecida, não da última processada. A condição `== null` cobre conversas novas sem histórico.

---

### B5 — N+1 queries em `listConversations`

**Sintoma:** `GET /api/conversations` executa N+1 queries SQL — 1 para listar conversas + 1 por conversa para buscar o preview da última mensagem. Com N conversas, o tempo de resposta cresce linearmente.

**Causa raiz:** `listConversations()` iterava sobre todas as conversas e chamava `messages.findFirstByConversationIdOrderBySentAtDesc(c.getId())` para cada uma — uma query individual por conversa.

**Correção:** denormalização — campo `lastMessagePreview` (nullable) adicionado diretamente na entidade `Conversation`. O campo é atualizado em `ingest()` e `sendReply()` sempre que a mensagem mais recente muda. `listConversations()` lê `c.getLastMessagePreview()` diretamente — sem query adicional. Resultado: exatamente 1 query SQL independentemente do número de conversas.

**Justificativa:** denormalização controlada é o padrão de plataformas de messaging (Slack, Intercom, Zendesk) para preview de última mensagem. A consistência é garantida pelos pontos de escrita controlados (`ingest` e `sendReply`) — não há caminho para atualizar mensagens sem passar por esses métodos.

---

### B6 — Conversa inexistente retorna 500 em vez de 404

**Sintoma:** `GET /api/conversations/{id}/messages` e `POST /api/conversations/{id}/messages` com `id` inexistente retornam `500 Internal Server Error`. O frontend recebe um erro opaco sem conseguir tratar adequadamente.

**Causa raiz:** `InboxService` lança `IllegalArgumentException` para IDs inexistentes, mas `InboxController` não tinha `@ExceptionHandler` para essa exceção. O Spring usa o handler padrão que mapeia para 500.

**Correção:** adicionar em `InboxController`:

```java
@ExceptionHandler(IllegalArgumentException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
void handleNotFound() {}
```

**Justificativa:** a exceção e sua semântica (argumento inválido = recurso não encontrado) estavam corretas no service. A correção pertence ao controller, que é o responsável pelo mapeamento de exceções de domínio para códigos HTTP — sem alterar o service.

---

## Inconsistências menores corrigidas

Além dos 6 bugs principais, dois problemas menores foram identificados e corrigidos durante a análise.

### G1 — Webhook sem validação retorna 500 em vez de 400

**Sintoma:** `POST /api/webhooks/inbound` com campos obrigatórios nulos ou em branco causava `DataIntegrityViolationException` no banco, retornando `500`. O comportamento esperado é `400 Bad Request`.

**Causa raiz:** `InboundRequest` não tinha constraints de validação e `inbound()` não aplicava `@Valid` — assimetria em relação a `SendReplyRequest`, que já usava `@NotBlank` + `@Valid`.

**Correção:** `@NotBlank` em `provider`, `channelAccount`, `contactIdentity` e `text` em `InboundRequest`; `@Valid` no parâmetro do controller. Campos opcionais (`externalId`, `contactName`, `sentAt`) sem alteração.

---

### G2 — `GET /messages` zerava `unreadCount` como efeito colateral de leitura

**Sintoma:** `GET /api/conversations/{id}/messages` modificava o banco a cada chamada, zerando `unreadCount`. Uma operação GET com side-effect de escrita viola semântica REST — qualquer retry ou prefetch do cliente alterava o estado silenciosamente.

**Causa raiz:** `getMessages()` executava `conversations.save()` com `unreadCount = 0` dentro de um método de leitura, sem endpoint dedicado para marcar como lida.

**Correção:** `getMessages()` passou a ser leitura pura. Endpoint `PATCH /api/conversations/{id}/read` adicionado como operação explícita e idempotente de marcação de leitura, com broadcast SSE de `conversation-updated`.

---

## O que foi simplificado / deixado de fora

| Item | Decisão |
|---|---|
| Autenticação | Fora do escopo do enunciado |
| Mídia além de texto | Fora do escopo do enunciado |
| Múltiplos atendentes | Fora do escopo do enunciado |
| Paginação | A API não pagina; lista completa a cada atualização |
| Responsividade mobile | Layout two-column descrito no enunciado é desktop-first |
| Retry automático em falha de rede | SSE reconecta automaticamente; `onopen` dispara refresh completo |
| Testes de componentes Vue | Enunciado avalia comportamento e organização, não cobertura de template |
| Deploy | Fora do escopo do enunciado |
