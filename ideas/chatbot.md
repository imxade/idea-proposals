# Chatbot — Shared Conversation Platform

## Architecture overview

The diagram shows the integrated/library deployment: the shared chatbot package runs inside a product application deployed to an operator-provisioned Kubernetes cluster. A product may instead deploy the chatbot application separately from its frontend or domain services and connect them through authenticated APIs; that is an optional deployment choice, not a different common-platform contract. The common platform targets standard Kubernetes APIs and portable Kubernetes resource contracts rather than distribution-specific APIs; the selected distribution is deployment configuration, not a chatbot runtime dependency. Each deployed product instance uses an isolated application namespace, database identity, credentials, authorization scope and product-owned handlers. Multiple product deployments may share a cluster where isolation controls are adequate; separate clusters remain an option for stronger isolation.

~~~mermaid
flowchart TB
  subgraph Clients["Shared interaction surfaces"]
    Browser["Browser user"]
    Web["Standalone shared web application<br/>Local: Kubernetes workload / Azure: Static Web Apps"]
    Discord["Discord"]
    WhatsApp["WhatsApp Business"]
    Other["Other supported channel adapters"]
    Browser <--> Web
  end

  subgraph Provisioning["Infrastructure and desired-state management"]
    Crossplane["Crossplane<br/>Supported infrastructure provisioning"]
    Flux["Flux CD + Kustomize<br/>Git-based reconciliation"]
  end

  subgraph Cluster["Operator-provisioned Kubernetes cluster"]
    subgraph ProductHost["Product namespace — isolated application instance"]
      subgraph Common["Reusable TypeScript chatbot library embedded in product application"]
        Gateway["Unified bot gateway<br/>Vercel Chat SDK Chat instance"]
        WebAdapter["Chat SDK Web adapter<br/>Authenticated HTTP / SSE"]
        WebAdapter --> Gateway
        Conversation["Conversation service and repository"]
        Attachments["Environment-gated attachment receipt and authorized retrieval"]
        Runtime["Vercel EVE runtime integration"]
        Scheduler["Durable scheduler"]
        Tasks["Asynchronous task/work lifecycle"]
        Events["Durable event cursors and replay"]
        Interactions["Linked interaction / mediation service"]
        Outbound["Outbound dispatcher and durable outbox"]
        Tools["Common agent tools"]
        Accounts["Bot account and human-role registry"]
        Accounts --> Gateway
        Accounts --> Conversation
        Accounts <--> Interactions
        Gateway --> Conversation
        Conversation <--> Attachments
        Conversation <--> Runtime
        Scheduler --> Tasks
        Events --> Tasks
        Tasks <--> Runtime
        Interactions <--> Tasks
        Runtime <--> Tools
        Runtime --> Outbound
        Tasks --> Outbound
        Interactions --> Outbound
      end

      subgraph ProductExtension["Product-owned extension points"]
        Policy["Authenticated principal and authorization resolver"]
        Agent["Product agent definition and instructions"]
        Handlers["Product action handlers and domain services"]
      end

      DB[("Product-specific PostgreSQL database/user<br/>CloudNativePG-managed")]
      Blob[("Product-scoped attachment storage interface / durable backend")]

      Policy --> Gateway
      Policy --> Accounts
      Policy --> Conversation
      Policy --> Attachments
      Agent <--> Runtime
      Handlers <--> Runtime
      Handlers <--> Tasks
      Handlers <--> Scheduler
      Handlers <--> Interactions
      Conversation <--> DB
      Scheduler <--> DB
      Tasks <--> DB
      Events <--> DB
      Interactions <--> DB
      Outbound <--> DB
      Attachments <--> Blob
    end

    CNPG["CloudNativePG operator"]
    PGStorage[("PostgreSQL persistent volume(s)<br/>provided by cluster StorageClass")]
    CNPG --> DB
    DB --> PGStorage
  end

  Crossplane -. "Provisions supported infrastructure" .-> Cluster
  Flux -. "Reconciles operators, services and applications" .-> Cluster
  Web <-->|"Authenticated Web adapter / SSE"| WebAdapter
  Discord <-->|"Platform events / outbound messages"| Gateway
  WhatsApp <-->|"Platform events / outbound messages"| Gateway
  Other <-->|"Platform events / outbound messages"| Gateway
  Outbound --> Gateway
~~~

**Normative separation rule:** a common requirement is specified here once. The product documents may configure or constrain that requirement for their own use, but must not restate the shared implementation. A product-specific requirement belongs only in that product's document.

## 1. Architecture: reusable library with flexible deployment

Implement the common chatbot layer as a **private, reusable TypeScript library/package**. The package is the reusable unit; it does not have to run as an independently deployed service. The recommended integrated mode imports it into each product's backend, but a product may deploy a chatbot application separately when that better fits its application or operational boundaries.

The common platform combines a reusable TypeScript conversation library with one shared Vercel EVE runtime integration. Each product supplies its own authorization, domain capabilities, agent definitions and product-data scope, whether those are hosted in the same application or reached through a separately deployed service.

The integrated/library mode is the recommended default because:

- Each product deployment needs substantially the same message normalization, identity mapping, conversation storage, context retrieval and outbound delivery.
- A network service would introduce another deployment, API boundary, availability dependency and potential competing owner of conversation history.
- Each product already has its own application database and different security and domain rules. The shared library can use the host application's database without combining product data.
- A library lets consuming products evolve independently while reusing the same tested conversation contracts and adapters.

The `chatbot/` directory in this repository contains the three specifications. It is not itself the implementation package. The eventual package can be versioned and consumed by both application codebases.

### 1.1 Package boundaries

The shared package should expose these modules/interfaces:

| Module or interface | Shared responsibility |
|---|---|
| **Channel adapters** | Receive platform/web events, verify and normalize them, preserve platform references, and deliver outbound messages. |
| **Bot account and human-role registry** | Maintain each bot account's stable application identity, connect its platform installations, associate human platform identities with product-defined roles, and manage those associations from the bot account's control conversation. |
| **Conversation service and repository** | Create and resolve canonical conversations, associate messages and participants, persist messages, maintain isolated context and provide indexed history retrieval/search. |
| **Shared web application** | Own and provide the standalone reusable browser chat application, including conversation history, transcript/composer, streaming, voice, file attachments, progress/events and integration with the Chat SDK Web adapter. All browser chat turns use the same gateway path as messaging platforms. |
| **Unified bot gateway** | Provide the common ingress/egress path for all configured channels, including browser chat through `@chat-adapter/web`: authenticate and normalize events, route them to the correct product agent and dispatch authorized outbound messages. |
| **Vercel EVE runtime integration** | Run each product deployment on the same EVE runtime baseline, including agent lifecycle, sessions, tool calling, workflows, delegation and agent events. |
| **Outbound message dispatcher** | Send agent- or application-originated messages without an active inbound turn; persist delivery intent and retry safely. |
| **Attachment receipt and retrieval** | When enabled by the deployment environment, receive attachments delivered by supported clients/channels, persist their metadata/content references and provide authorized access to the agent. When disabled, stop attachment bytes before backend ingestion, storage or agent processing. |
| **Scheduler service and repository** | Persist one-time, recurring, interval and event/condition schedules; calculate due times and create durable executions using each product's database. |
| **Task/work service and repository** | Provide shared asynchronous task/run identities, lifecycle, dependencies, cancellation, checkpoints, leases, retries and recovery. Product handlers implement the work itself. |
| **Linked interaction service and repository** | Correlate related conversations, questions, deferred work and replies through scoped, authorized interaction records without merging full transcripts. |
| **Product action registry** | Register product-owned handlers for scheduled work, asynchronous tasks and interaction events under product authorization. |
| **Common agent tools** | Expose the shared `search_conversation_history` tool and authorized common task/schedule/interaction operations to the connected agent runtime. |

The common platform standardizes on Vercel EVE for agent execution. The conversation library, scheduler, task lifecycle, linked-interaction mechanism and bot gateway remain modular TypeScript components around that runtime; none embeds product-specific business logic.


### 1.2 Product extension points

Each product supplies the following through explicit interfaces:

- Its authenticated principal and authorization/policy resolver.
- Its human-role definitions, role-specific permissions and human-context scope. The common layer manages account-scoped identity associations; the product defines what each role can do.
- Its own database connection and product-specific tenant/owner scope.
- Its product-specific agent definition, instructions, capabilities and communication policy.
- Its deployment-provided additional system context, exposed through the `CHATBOT_CONTEXT` environment variable. The common chatbot loads this value at startup and appends it to the system-role context already configured for the agent, preserving all existing system instructions rather than replacing or overwriting them. The added content is sent with role `system`; it is not user content, a developer-role message or ordinary retrieved context. The product owns the value.
- Its domain-specific tools/capabilities and tool authorization. The product also supplies the usage instructions, context and request/response contracts the deployed agent needs to use those capabilities.
- Its attachment-processing capabilities. The common environment setting controls whether attachment content enters the backend at all; when enabled, the common layer delivers supported attachments to the agent subject to authorization and product capabilities.
- Product-owned handlers for scheduled work, asynchronous tasks and linked interactions, plus product-specific events and reasons for proactive messages.

These shared extension interfaces and product-owned lifecycle handlers can be registered as in-process modules/providers when the library is embedded; this is not a plugin marketplace or a mandatory plugin microservice. Product capabilities may be exposed to the agent through in-process handlers, same-application endpoints, CLI commands, installed dependencies or authenticated APIs provided by independently deployed product services. The agent can invoke those product-owned capabilities when relevant, using the tool definitions, usage instructions, input/output contracts and product-specific context supplied to it (including through its configured agent context). The normal direction for a product capability is therefore the chatbot/agent calling the product service; the product service does not have to call the chatbot's APIs. All user chat turns, including browser turns, enter through the unified gateway using the configured Chat SDK adapter; the browser uses `@chat-adapter/web` rather than a separate chatbot conversation API or stream endpoint. The shared web application may still use host-product resource operations for non-chat functions such as listing persisted conversations, retrieving older history, obtaining authorized attachment uploads/downloads and loading product settings. Those operations do not accept chat turns or provide a parallel agent-response stream. When the chatbot is embedded as a library, these capabilities can run within the same application/server, alongside endpoints implemented with the product's chosen framework. When the chatbot is deployed separately, it can call authenticated APIs exposed by the product service. Both are supported deployment options, not requirements. Context makes capabilities discoverable and usable by the agent; it does not grant permission, so the owning service must still enforce authentication, data scope and business rules. No separate API server, universal API or common tool protocol/registry is required for every product capability.

### 1.3 Shared EVE runtime and unified bot gateway

All product deployments use the same **Vercel EVE runtime baseline**, integrated through the common TypeScript platform. The shared runtime layer owns the common agent execution lifecycle, runtime sessions, tool invocation, workflow execution, delegation primitives, streaming/events and runtime-to-conversation mapping. A product defines its own agent identity/instructions, capabilities, tools, authorization and domain workflows on top of that foundation.

EVE session state is execution context, not the canonical record of conversations or product business state. Persist messages, task/business records, schedules, checkpoints and delivery intents through the product's authoritative data services. Product-specific specialist agents, worker definitions, selection, delegation policy and result evaluation remain part of the product agent; the common chatbot provides EVE's generic runtime primitives but does not define a shared specialist registry or product-neutral subagent orchestration. All product deployments use the same EVE integration and version baseline, not one shared cross-product agent instance. Each product has its own deployment, agent instances, runtime sessions, identities and data; capability-specific tools and workloads remain product-owned.

The **unified bot gateway** is also common. It is a shared library module and service interface, not a separately operated cross-product bot service. It is the single ingress/egress path for browser chat and supported messaging platforms. Use one Vercel Chat SDK `Chat` instance with the relevant adapters, including `@chat-adapter/web`, and bridge accepted turns into the shared EVE runtime through EVE's Chat SDK channel integration:
- Receives browser requests through the authenticated Chat SDK Web adapter and platform events through the configured messaging adapters; each adapter performs its applicable request authentication, webhook verification and event parsing.
- Resolves the product deployment, bot account, authenticated participant, channel connection and canonical conversation scope. The Web adapter's `getUser` callback must resolve the caller from the host product's verified server-side session; never trust a browser-supplied user or tenant identity.
- Normalizes messages, attachments, replies, threads and platform metadata into the common message contract.
- Routes accepted turns from every adapter to that product's shared EVE runtime integration using the EVE Chat SDK channel bridge.
- Returns agent replies through the originating adapter. Browser replies stream through the Web adapter's AI SDK UI message-stream/SSE transport; messaging replies use their platform adapter.
- Supports durable, idempotent proactive outbound delivery through the shared outbox where the destination adapter supports it. The current Web adapter does not provide multi-tab or disconnected-browser proactive push; browser-targeted background messages remain persisted in the canonical conversation and are shown when the client next synchronizes.

The gateway implementation and contracts are common, but **bot identities, credentials, channel installations, configuration, conversation data and authorization remain isolated per product and deployment**. Each bot account has a stable internal account ID and its own context, conversations, data and configured channel connections. Separate product deployments do not share bot accounts or conversation stores. The gateway must enforce each product's own policy and platform capability limits.

## 2. Logical agents, platform identities and conversations

### 2.1 Logical agent and platform identities

Each product has its own logical agent. For each configured messaging integration, that product provisions the corresponding platform identity or addressable account:

- Discord: a Discord application/bot user.
- Telegram: a Telegram bot.
- Matrix: a Matrix bot account.
- WhatsApp: a WhatsApp Business account/number, which is not the same identity model as a conventional username-based bot.
- Other supported messaging platforms: their native application, bot or business identity.

These are channel identities for reaching the product's logical agent; they are not separate agents. Each product has separate platform identities, credentials, conversation database and authorization scope.

One configured platform identity can serve multiple direct messages, groups, channels and threads. Do not create a new bot identity or a new agent process for each conversation.

The identity must be installed, invited, joined or connected through the platform's own mechanism. A username mention alone does not grant membership, room access or permission to read messages.

### 2.2 Bot account onboarding and human identity management

Each service has a **bot account** with a stable internal account ID. This account is the durable owner of the bot's configuration, conversation history, context, linked interactions, tasks and other service data. It is distinct from the bot's Discord/Telegram/WhatsApp/Matrix identity and from any human's platform user ID. A bot account can have multiple channel connections, each mapped to its own platform identity and installation context.

The onboarding flow is:

1. Create or provision the bot account for the service.
2. Install, invite or connect the bot through the chosen platform's normal mechanism and associate that channel connection with the bot account.
3. Open the bot account's own management conversation and use ordinary language to configure it. Commands may be supplied as shortcuts, but are not required for ordinary onboarding.
4. Providers connect their provider-side groups to the bot account by signing in to the bot account (using its stable bot account ID) and authorizing the bot to access the selected groups. The bot account can be connected to multiple groups, including groups used by different providers or provider helpers. The common flow does not require a separate registration website or a separate direct-message relationship with each provider.

The bot account's management conversation is the control surface for managing its platform connections and authorized provider-side groups. Group membership and access are established through the platform's normal sign-in, invite and permission mechanisms. A group cannot be attached to an unrelated bot account or customer conversation.

When a linked person changes their platform user ID, they use the bot account's management conversation to disconnect the previous ID and connect the replacement. The application updates the human-to-platform identity mapping only. It does not create a new bot account, change the bot account ID, reset or migrate the bot's context, or alter its conversation history, tasks, linked interactions, configuration or other account data. The existing bot account continues as before under the replacement platform identity.

Changing the bot's own platform identity or replacing one of its channel connections is also done from the same bot-account management conversation. It updates only the affected platform-connection mapping; the internal bot account and all of its context and data remain unchanged.

The common layer treats connected provider-side groups as communication and context sources for the bot account. Providers and their helpers contribute through those groups rather than through a required one-to-one provider DM workflow. The bot account can connect to several groups; incoming questions, answers and supporting context are collected from the authorized groups, with their source and group scope preserved. The agent decides which information is relevant, what may be shared, and which response should be returned to the customer.

**Provider-side communication happens in connected groups.** A provider or helper can ask a question, add context or correct an answer in any group connected to the bot account. The agent uses the available group context and capabilities to respond where appropriate, and records useful contributions with their source and group scope. It does not need a separate direct-message channel to each provider.

**When a customer asks a question**, the agent first checks its authorized context. If more information is needed, it forwards the question and the necessary context to the connected provider-side groups. One provider or several providers/helpers may respond in those groups. The agent evaluates the replies and supporting context, decides what is relevant and permitted for the customer's scope, records the source, and returns an appropriate answer to the originating customer conversation. The customer should not have to repeat the question.

If no relevant provider-side group is connected, the agent does not invent a recipient or missing information. It tells the customer that it cannot determine the answer from the available context.

### 2.3 Canonical conversation mapping

The application creates a stable internal conversation ID and maps it to the external platform context. The mapping must preserve enough information to route messages back to the right place:

- Product/deployment, bot account ID and any product-specific tenant/owner scope.
- Channel type and channel-connection identity.
- External platform account, room, group, channel or DM identifier.
- External thread/topic identifier where applicable.
- External sender identity and message identifier.
- Message timestamp, reply/reference target and relevant platform metadata.

A direct message, web conversation, group/channel thread or other independent conversation context receives its own canonical conversation mapping. A group's channel/thread context is not mixed with unrelated groups, DMs or web conversations.

A web conversation enters through the Chat SDK Web adapter and the unified gateway; it does not connect to a separate chatbot conversation API, invite or create a messaging bot. Map the authenticated user plus the Web adapter conversation/thread identifier to a canonical conversation ID and EVE session. Each new web conversation has its own canonical conversation and agent-runtime session/context; reopening an existing conversation resumes its existing context. Treat Chat SDK thread identifiers as adapter-level routing identifiers, not as the public product conversation contract. The shared logical agent can therefore serve many isolated conversations without creating a separate agent identity or process for each one.

A runtime session ID is an execution identifier mapped to the canonical conversation ID. It is not the source of truth for message history. If a runtime is replaced, the canonical conversation and its persisted history remain available.

A previous conversation is not silently added to the current one. Cross-conversation retrieval requires an explicit reference and an independent authorization check.

### 2.4 Direct, group and channel interaction

Ordinary natural language is the normal interaction method. Slash commands may be supplied as shortcuts, but users must not need a command to make an ordinary request.

- **Provider-side groups:** connected groups are the normal provider communication surface. The bot account may connect to multiple groups, and the agent can forward questions/context to the relevant connected groups and evaluate responses from them. A one-to-one provider DM is not required.
- **Mention-required group mode:** when `CHATBOT_REQUIRE_MENTION=true`, the bot responds in group conversations only to messages that mention/address it (including supported platform-native reply-to-bot behavior). Unaddressed group messages are ignored.
- **Mention-not-required group mode:** when `CHATBOT_REQUIRE_MENTION=false`, the bot may respond to ordinary group messages without an @mention, subject to platform permissions and the conversation's access policy.
- **Replies and threads:** replying to an agent message or another message can continue or refer to that exchange. Preserve the referenced message and thread context.

The deployment environment variable `CHATBOT_REQUIRE_MENTION` controls whether the bot must be mentioned before it responds in a connected group and must be explicitly set to `true` or `false`. Platform privacy settings and permissions still determine which group messages reach the bot; disabling the mention requirement cannot override them. The shared library must not impose a universal mention-only, public-only or private-only restriction.

**Initial messaging targets are Discord, WhatsApp Business and Telegram.** The shared platform supports these three adapters in its initial integration baseline; each product configures the subset it uses. Additional adapters can follow the same common contract.

Actual access and visibility remain controlled by platform installation, membership, privacy settings, permissions, intents, user consent and policies. The library must not promise that every platform can expose all messages in all spaces. A platform adapter must report its real capabilities rather than silently pretending that an unsupported mode works.


### 2.5 Platform-specific capability constraints

The interaction model is shared, but actual access and delivery differ by platform. The library and product configuration must record the effective capabilities of each adapter and connection rather than promising identical behaviour everywhere.

| Platform | Identity and access model | Constraints to preserve |
|---|---|---|
| **Discord** | A Discord application/bot identity is installed into a server. | The server owner/admin decides which channels the bot can access and use. The bot communicates only in channels allowed by those permissions and intents; an administrator can authorize one channel or several. Installing the bot does not grant access to every channel or authorize unsolicited DMs. |
| **Telegram** | A Telegram bot has its own bot identity and username and can be added to authorized groups. | Provider-side communication can take place in groups the bot has joined. Group privacy mode and admin settings determine which messages the bot receives; the adapter must respect those platform controls. |
| **WhatsApp Business** | The integration uses a WhatsApp Business account/number, not a conventional username-based bot identity. | Business-initiated delivery is subject to opt-in, approved message templates and applicable messaging windows. Group capabilities and eligibility are restricted; do not treat the integration as permission to join any arbitrary existing group. |
| **Matrix** | The bot uses a Matrix account/user ID and participates in rooms. | The bot must join the room and follow that room's membership and power-level permissions. Select an adapter and hosting mode that support the required sync/event lifecycle; some integrations require a persistent process rather than webhook-only execution. |

For each configured adapter, record support for DMs, group/channel events, mentions, replies, threads, full-message subscription, edits, reactions, attachments, outbound initiation and delivery receipts. If a capability is absent, use a supported fallback or explain that the operation is unavailable; never silently pretend it succeeded.

Platform references: [Discord Message Content Intent](https://discord.com/developers/docs/events/gateway#message-content-intent), [Discord application installation](https://discord.com/developers/docs/resources/application), [Telegram bot privacy mode](https://core.telegram.org/bots/features#privacy-mode), [WhatsApp Business Policy](https://whatsappbusiness.com/policy/) and [Matrix Client-Server API](https://spec.matrix.org/latest/client-server-api/).


## 3. Common message behaviour

### 3.1 Inbound message processing

All clients and messaging integrations normalize inbound content into the shared message contract before product logic or the agent processes it. Preserve:

- Canonical conversation and participant identity.
- Original platform, channel connection and external message identifiers.
- Text and supported rich-message parts.
- Reply/thread/reference information.
- Attachment metadata and references only when attachment ingestion is enabled for that deployment.
- Original event time and relevant platform interaction metadata.

The application authenticates the event, resolves the product/account scope, checks access and deduplicates it before dispatching the turn to the shared Vercel EVE runtime. Before any attachment bytes are fetched from a messaging platform, uploaded by a web client, proxied to the application, persisted or exposed to the agent, the ingress path must enforce the deployment environment variable `CHATBOT_ATTACHMENTS_ENABLED` (`true` or `false`). Each deployment must set this value explicitly. When `false`, do not fetch or download attachment content, accept or proxy web-upload bytes, store attachment content or durable attachment references, or include attachments in the EVE/agent turn; ignore the attachment and continue processing any accompanying text. The client should disable its upload control when possible, but the backend must enforce the setting even if a client or platform still sends attachment metadata or an upload request. A lightweight indication that an attachment was ignored may be shown to the sender, but the attachment itself must not be transferred into backend storage or AI processing. This prevents unnecessary backend storage and attachment-related AI/token usage. When `true`, the normal authorized attachment receipt, storage, retrieval and agent-capability rules apply. Do not trust a model-supplied tenant, customer, owner or conversation identifier.

### 3.2 Agent and application messages

Agent replies are persisted as canonical messages and delivered through the adapter associated with the originating conversation. Preserve message ordering and reply/thread destination where the platform supports it.

The shared web client submits chat turns through the Chat SDK Web adapter and receives the agent response over its AI SDK UI message-stream/SSE connection. Messaging clients receive platform-native messages through their respective adapters. A live browser connection is not required for the service to continue a task or deliver a later message; persist resulting messages and work state independently of the connection. The current Web adapter provides request-scoped streaming, not durable push to disconnected browsers or other open tabs.

The common project owns the standalone shared web application and its reusable chat experience. Products configure its identity, roles, authorization and product actions. All clients that submit chat turns use the same gateway and adapter contract; a product or third party may build a separate client against the documented client and conversation contracts without creating a second chat-message ingress or streaming path.

### 3.3 Proactive outbound text — independent of an incoming user turn

The agent/application **must be able to send a text message without a preceding or currently active user message**. This is a first-class outbound capability, not merely a reply operation.

It must support messages initiated by:
- An agent continuing or completing background work.
- A product event or state change.
- A scheduled task or reminder.
- A follow-up that becomes ready later.
- A product-specific workflow that needs to contact an already known participant or conversation.

The outbound operation accepts an authorized target (canonical conversation or previously resolved recipient/channel destination), content, cause/reference and idempotency key. It must not require an inbound message object or an open web client.

Use a durable outbound queue/outbox. Persist intent before delivery, record attempts and outcomes, retry transient failures, and deduplicate retries so a message is not sent repeatedly because of a timeout or restart. Keep message delivery separate from the domain action that triggered it; a delivery failure must not undo a completed booking, task or other product operation.

Proactive sending is still subject to platform rules. Some platforms require a user to initiate a DM, grant consent, or fall within a messaging window; some contexts cannot be contacted without an existing membership or thread. The product may send only where it has a valid destination and the platform permits it. This capability does not grant permission for unsolicited bulk DMs or bypass user opt-in.

## 4. Shared web client

The shared package includes both the reusable browser-based chat client and its Chat SDK Web adapter integration. The common project owns the standalone shared web application; `@chat-adapter/web` is its browser channel adapter, not a separate application or a second gateway. All product deployments can use the shared client and configure product identity, authorization, capabilities, branding and visible workspaces; they may also build another client using the same gateway contract.

### 4.1 Framework and application boundary

- Implement the shared web client in TypeScript using one of the supported frontend options listed in the common technical stack. Keep the UI framework replaceable; use the matching Chat SDK Web client integration (React, Vue or Svelte) and preserve the common canonical conversation and event contracts.
- Submit every browser chat turn through `@chat-adapter/web` on the same Chat SDK `Chat` instance used by the messaging adapters. Use the EVE Chat SDK channel bridge to dispatch the turn to the product's EVE runtime; do not add a separate authenticated chat-message API or independent browser streaming endpoint.
- Configure the Web adapter's `getUser` callback to resolve the authenticated user from the host product's verified server-side session. Derive the adapter thread ID from that authenticated identity and the client conversation identifier, then map it to the canonical conversation ID and EVE session. Enforce product/tenant scope and authorization on the server for every turn.
- The common project owns and ships the standalone shared web application. It may be deployed separately from the product backend, provided its Web adapter endpoint is reachable over an authenticated, appropriately protected origin.
- Products may use the shared web application as-is or build their own web/mobile client against the same Web adapter and documented conversation contracts. A custom client must not create a parallel chat ingress or response-stream implementation.
- The same client must support both desktop and mobile browser layouts. Do not create a native desktop application as part of this shared scope.

### 4.2 Shared chat experience

The common client provides:
- A conversation list/history view and the ability to create, open, resume and switch between conversations without mixing their contexts. The canonical conversation repository remains the source of truth. Chat SDK Web v1 does not list threads or provide a client-facing full-history API, so list/history loading uses authenticated host-product read operations; these are not alternate chat-turn submission or streaming endpoints.
- A message transcript that distinguishes user and agent messages, preserves chronological order and renders supported text/Markdown content.
- A composer for ordinary text messages, send/submit state, multiline input and appropriate keyboard behavior.
- Streaming agent responses through the Web adapter's AI SDK UI message-stream/SSE protocol, with clear sending, receiving, reconnecting and failure states. Use the client's standard sending/streaming status for request lifecycle; do not assume Chat SDK Web v1 transports arbitrary durable task/plan events.
- Correct handling of long conversations, including loading older persisted messages from the canonical conversation read model and preserving the current scroll/reading position where practical. Do not treat the Web adapter's state-cache history as the canonical transcript.
- Message timestamps and supported reply/reference information.
- Responsive layouts, accessible controls, keyboard navigation, visible focus, readable contrast and clear empty/loading/error states.
- A settings surface for preferences supported by the host product.
- Safe rendering of message content; do not execute arbitrary HTML or active content supplied in messages or attachments.

### 4.3 Voice interaction

The common web client supports speech-to-text (STT) and text-to-speech (TTS). When voice input is enabled, the user explicitly starts microphone capture; the client continuously listens while active and detects a configurable silence/pause interval before finalizing the utterance and submitting it for a response. The deploying operator configures this interval, with a sensible default. Show a clear listening state and provide an immediate stop/mute control. Handle microphone permission denial, unsupported browsers and recognition errors. TTS provides visible playback state, pause/stop controls and interruption when the user starts a new interaction or selects another response.

Voice interaction submits recognized text through the same Chat SDK Web adapter and canonical conversation mapping. Text interaction remains available if voice permissions, browser APIs or a speech provider are unavailable. Make the speech-processing method and provider/processing location visible to the user. Raw microphone audio is not retained as canonical conversation data and must not be retained by default; persist the resulting text message, and retain audio only when the user explicitly attaches or requests its retention under the host product's policy. Keep provider credentials and speech-service secrets on the server; do not expose them in browser code. The host product configures the speech provider, allowed languages, voice, retention policy and disclosure.

### 4.4 Shared client actions and product extension points

The common client provides reusable presentation and action contracts for:
- Task/status/progress events supplied by the host product.
- Focused requests for additional user input and confirmations using the generic linked-interaction contract. Chat SDK Web v1 renders Markdown but does not provide native buttons, cards or modals; the baseline web flow must therefore support a clear text-reply fallback. Richer controls require an explicit shared-client extension and server-side authorization, not an assumption that the Web adapter supplies those widgets. Where enabled, the client may present a pending-request view/inbox using authenticated host-product read operations.
- Display of attachments and authorized references only when attachment ingestion is enabled by the deployment environment; expose them to the agent through the common attachment-retrieval contract. Chat SDK Web v1 does not implement file uploads, so browser file transfer requires a separate authenticated, server-issued upload mechanism owned by the host product/storage integration; the resulting authorized reference is associated with the message through a documented extension. This upload mechanism is not a second chat-message or streaming endpoint. When attachments are disabled, disable web upload controls and enforce the setting on the server so bytes are not uploaded or processed.
- Artifact/file references and click-time download resolution supplied by the host product.
- External access links that open in a separate browser tab/window when the host product supplies them.

The shared client does not implement product-specific approval policies, compute controls, booking workflows or provider/customer operations. It renders authorized product-supplied actions through extension points. Each product supplies its own domain actions and approval semantics. A product must not expose an action unless its server-side authorization permits it.

### 4.5 Client reliability and security

- The client is replaceable and must not own agent execution, durable task state, schedules, canonical message storage or artifact storage.
- Closing, refreshing or replacing the browser client must not interrupt background work or lose messages.
- Reconnect and resynchronize from the canonical server state after a network interruption; do not rely solely on transient browser memory or an open streaming connection.
- Authenticate browser chat at the Web adapter boundary through `getUser`; use separate authenticated host-product resource operations only for non-chat reads and attachment/artifact transfer. Use server-issued upload/download/access references. Never expose channel tokens, database credentials, speech-provider secrets or unrestricted internal service endpoints.
- Reconnect the Web adapter stream to the canonical conversation state after interruption. Because Web v1 does not provide multi-tab/disconnected-browser proactive push, persist background replies and refresh the canonical transcript when the client reconnects or reopens.
- Resolve expiring artifact and product-access URLs at click time through the host product service.

## 5. Durable scheduling, asynchronous tasks and linked interactions

The common platform provides reusable mechanisms for durable schedules, asynchronous work and linked interactions. Each product persists these records in its own authoritative database and registers authorized handlers. The common layer determines when work is due, tracks its lifecycle and routes related events; it does not implement product-domain behavior.

### 5.1 Shared scheduler

Support:
- One-time date/time schedules and calendar recurrences (daily, weekly, monthly and yearly).
- Interval schedules such as hourly checks.
- Event- or condition-triggered schedules and scheduled reports.
- Explicit timezone handling, daylight-saving transitions, next-run calculation, missed-run policy, overlap rules, bounded retries and duplicate prevention.
- Editing, pausing, resuming and deactivating a schedule while retaining its execution history.
- Enforce the deployment-wide active-schedule limit and minimum recurring interval configured through the environment variables in §7.3. These limits are shared scheduler policy, not product-specific agent instructions.
- Count each active schedule definition once, including one-time schedules, recurring schedules and reminders, across all bot accounts in that product deployment. Paused, deactivated, cancelled and completed one-time schedules do not count. Schedule runs/tasks are not additional schedules.
- Check the active-schedule limit atomically when creating, activating or resuming a schedule so concurrent requests cannot exceed the configured limit. Reject a request that would exceed the limit with a clear explanation; do not silently discard or replace another schedule.
- Apply the minimum interval to time-based recurring schedules, including recurring reminders, when they are created, edited or resumed. Validate the recurrence cadence, not the lead time of a one-time reminder. Event- or condition-triggered schedules have no fixed recurrence interval and are governed by the active-schedule limit instead.

Persist schedule definitions and each schedule run. A due schedule creates a durable task/run for a product-registered handler; do not rely on an in-memory timer, an active agent session or a connected client as the source of truth. Recheck authorization and relevant business state when the handler executes. The scheduler owns timing, recurrence and deployment-wide schedule limits; the product handler owns the actual action and its domain rules.

### 5.2 Shared asynchronous task lifecycle

Provide common task/run IDs, conversation and interaction references, dependencies, status, timestamps, attempts, idempotency keys, checkpoints where supported, and result/evidence references. Support queued, ready, running, waiting-for-human, waiting-for-external-event, waiting-for-schedule, retrying, succeeded, failed, expired and cancelled states.

Workers use renewable leases/heartbeats and fencing so an expired or stale worker cannot commit over a newer execution. Retries are bounded and use backoff; uncertain external side effects are reconciled before retry. Independent tasks continue while another task waits.

A user or authorized product operation can request cancellation. Cancellation is durable and propagated to the handler as a cooperative cancellation signal; the service records whether work stopped, completed before cancellation, or could not be interrupted safely. Fence further commits from a cancelled/stale worker. Accepted work survives client disconnects and process restarts, and resumes from a committed checkpoint where the handler supports it.

### 5.3 Durable event processing

Persist event-source cursors and event identities for channel and product event streams. Acknowledge an event only after its accepted work is durably recorded. On reconnect/restart, resume from the last committed cursor where the source supports replay; deduplicate by stable source event ID and handle out-of-order delivery without repeating business effects. If a source cannot replay, record that limitation and use its supported recovery mechanism.

### 5.4 Linked interactions and mediation

Provide a durable interaction ID that can link an originating canonical conversation and message, related participant conversations, a pending question or case, dependent task/run(s), responder, response and follow-up delivery.

Each pending request is linked to its originating conversation/message, relevant participant(s) and any dependent task/run. Support one independent request per independently actionable question or human action, so several requests can remain open without blocking unrelated messages or work. Store a concise request/summary and only the context projection approved by the host product; refer to canonical messages rather than copying complete private transcripts into interaction records.

The common lifecycle supports, as applicable, open, waiting-for-response, response-received, resumed, resolved, closed and cancelled states. It supports assignment/routing to an authorized responder, notification through common outbound/client mechanisms, recording a response or clarification, linking that response to the originating request, and emitting a durable event or resuming the dependent task/handler. Interaction state and task/run state remain related but distinct: resolving a question must not incorrectly mark a larger task complete, and a task may continue while another independent request remains open.

The common layer owns generic interaction records, lifecycle transitions, correlation, notification/delivery coordination and response routing. It does not decide whether a matter is a booking question, complaint, owner approval, payment confirmation or another product-specific case. The product handler supplies the request category, priority/severity, intended responder role/recipient, response instructions, context allowed for sharing and the business rule for resolving the request. The host product's authenticated principal/scope resolver and product handler decide who may open, view, respond to or receive each request; the shared interaction service enforces those checks through the host-provided authorization contract. Do not introduce a separate shared role or policy-management framework.

The interaction service must never merge complete transcripts or grant participants access to one another's private messages. Check authorization and scope whenever a request is created or listed, context is projected, a responder is assigned, a response is attached, a task resumes or a message is sent. A generic client/inbox surface may present pending requests, but product policy determines which users see and act on them. A response must be correlated to the exact request; do not resolve unrelated requests merely because they concern similar topics.

Expose authorized common agent-facing operations to open a pending interaction, inspect an individual request, list requests visible to the current principal, assign or route a request, submit a response or clarification, resume dependent work and resolve, close or cancel a request where permitted. These are generic lifecycle operations; products supply the business trigger, category/priority, responder selection, context projection and resolution meaning.

### 5.5 Persistence and API requirements

The common interfaces must support creating, reading, updating, pausing/deactivating and listing schedules; querying schedule runs; creating, inspecting, cancelling and resuming eligible tasks; recording checkpoints and worker leases; recording/replaying event cursors; and opening, listing/inspecting, assigning/routing, responding to, clarifying, resuming, resolving, closing and cancelling interactions. Link interaction records to their originating conversations/messages/participants and dependent task IDs. Persist schedule definitions/runs, task/run state, event cursors/deduplication keys, interaction/request records, approved context-projection references, participant/responder references, lifecycle state and event/delivery references in the host product's database. These are functional data/API requirements, not a fixed schema or exhaustive endpoint list.


## 6. Search the complete persisted conversation history

The shared library must expose an agent tool named **`search_conversation_history`**.

Its purpose is to let the agent recover an earlier detail when the current model context no longer contains it because of context-window limits, summarization, compaction or a long-running conversation.

The tool must:

- Search the full persisted message history of the current canonical conversation, beginning with its first stored message, not just the currently loaded prompt, recent-message cache or compaction summary.
- Search stored text across the conversation's user messages, agent messages and other persisted textual conversation entries.
- Support ordinary text/term and phrase queries, and return relevant matching message content or excerpts.
- Return stable message IDs, timestamps, speaker/author and enough conversation/thread metadata to identify the source. Preserve the original message so the agent can inspect the exact wording rather than relying on a generated summary.
- Be used when a user refers to an older statement, decision, instruction, preference or detail that is not clearly present in the active context. The agent should search before asking the user to repeat something that may already be in the stored conversation.
- Return the exact stored text for each selected result, or allow the agent to retrieve the full message by its returned message ID when an excerpt is truncated.
- Support bounded result counts and pagination or subsequent retrieval so a large conversation does not produce an oversized tool response.
- Be available to all product deployments' agent instances running on the shared Vercel EVE runtime.

A compaction summary is an optimization for model context, not a replacement for the original messages. Do not delete or overwrite the source history merely because a summary has been created. The search tool must still find a phrase or fact from before compaction.

By default the tool searches **only the current conversation**. Searching another conversation requires an explicit reference and a separate authorization check. It must never search another tenant's, provider's, owner's or user's messages merely because they share the same database.

Implement this as indexed text search over the persisted database (PostgreSQL full-text search is the natural implementation for the current shared database choice). Semantic/embedding search is not required for this capability. Search results must be grounded in stored messages; the agent must not invent a historical quote or claim a match when none exists.

## 7. Common technical and deployment foundation

All product deployments use the same application, agent-runtime, data, bot-gateway and deployment-validation baseline. The only technology differences belong to product-specific capabilities that the agent operates or integrates with; they do not create separate chatbot, runtime or deployment foundations.

| Area | Common technical baseline | Product-specific boundary |
|---|---|---|
| **Agent runtime** | **Vercel EVE**, integrated through the shared TypeScript platform. All product deployments use the same runtime integration and lifecycle model. | Agent identity/instructions, tools, capability access, policies and workflows are product-specific. |
| **Application language** | TypeScript for the common library, EVE integration, gateway and application services. | Product domain services extend the same typed contracts. |
| **Web client** | Preferred frontend options are **Cloudflare Vinext** and **Vite+**; **Next.js** is also supported. Use TypeScript and the Chat SDK Web adapter client integration (React, Vue or Svelte as appropriate). Browser chat turns and response streaming use the same Chat SDK gateway as messaging platforms; canonical history and non-chat resources remain in product-owned services. | Roles, branding, page configuration and product actions. |
| **Persistence** | PostgreSQL with Drizzle ORM, managed inside Kubernetes by the CloudNativePG operator in every environment. Use separate databases and credentials for each product deployment. Use persistent volumes from each cluster's configured StorageClass. Configure independent base/WAL backups and tested recovery; a PVC alone is not a backup. | Product databases and records remain isolated. The same PostgreSQL operator/configuration model is used locally and in production; size, replica count and StorageClass are environment-specific. |
| **Unified bot gateway** | One shared TypeScript Chat SDK `Chat` instance and gateway contract for all bot/channel ingress and egress, including browser chat through `@chat-adapter/web`; bridge normalized turns into EVE through `chatSdkChannel`. | Separate bot identities, credentials, channel connections, opt-in rules and policies per product. |
| **Agent/application workloads** | Kubernetes is the common workload environment. Deploy the application, EVE-backed services, workers, Jobs and in-cluster supporting services into declared namespaces using portable Kubernetes resources and standard APIs. | Capability-specific workloads, permissions and resource profiles remain product-owned. |
| **Infrastructure provisioning** | Use Crossplane to provision supported infrastructure resources through an already bootstrapped management plane. Cluster creation and initial control-plane bootstrap remain operator/platform responsibilities; Crossplane cannot bootstrap itself onto a cluster that does not yet exist. | Product-specific resource compositions and permissions. Crossplane does not replace the initial bootstrap needed to create the cluster on which it runs. |
| **Desired state and reconciliation** | Kustomize bases/overlays define cluster, operator and application configuration; Flux CD reconciles Git-hosted desired state, including Crossplane resources and the CloudNativePG operator. | Product-specific manifests and capability resources use the same GitOps workflow. |
| **Local/CI cluster parity** | Use a local Kubernetes test cluster for development and workload integration tests. Deploy the application and CloudNativePG using the same portable Kubernetes resource contracts used for self-managed and managed production clusters. | Product test data, capabilities and validation cases remain separate. |
| **Kubernetes distribution stack** | K3s for local testing and self-managed VPS/local-machine clusters; AKS, EKS or GKE for managed cloud clusters, with other conformant distributions supported after validation. | Validate required Kubernetes APIs, storage, networking and workload capabilities before supporting another distribution. |
| **Autoscaling** | Configure Kubernetes Horizontal Pod Autoscalers for eligible stateless APIs/workers with metrics, resource requests and explicit minimum/maximum replicas.  | Do not assume autoscaling is enabled by default. PostgreSQL scaling and replication use database-aware policies, not generic HPA. |
| **Deployment target** | Each deployment targets one configured Kubernetes cluster. Production may use a self-managed cluster or a managed target such as AKS, EKS or GKE after the required APIs, storage, networking and workload capabilities pass validation. | Environment-specific overlays supply storage, identity, networking, sizing and secret references. |



### 7.1 Chat SDK's role

Vercel Chat SDK is the shared **channel transport/adapter layer** for both browser chat and messaging platforms, not the chatbot's database, canonical conversation store, domain backend or agent runtime. Use one Chat SDK `Chat` instance in the unified gateway and register `@chat-adapter/web` alongside the selected messaging adapters. The Web adapter accepts authenticated browser requests and speaks the AI SDK UI message-stream protocol; the same Chat SDK handlers process browser and messaging turns. Bridge those handlers to the shared EVE runtime with EVE's `chatSdkChannel` integration. Chat SDK remains replaceable behind the common channel contract. Its catalog distinguishes Vercel-maintained official adapters from vendor-official and community adapters. Discord, Telegram and WhatsApp Business have official adapters; Matrix currently has a vendor-official Beeper Matrix adapter. Select adapters by actual feature coverage and hosting requirements, not just by catalog presence.

- Use `@chat-adapter/web` for browser chat as well as the appropriate Chat SDK adapters for messaging platforms. Configure `getUser` as a mandatory authentication boundary: resolve the caller from a verified server-side product session and return no user for unauthenticated requests. Map the authenticated user and adapter conversation identifier to the canonical conversation and EVE session.
- The EVE Chat SDK channel bridge receives accepted messages through Chat SDK handlers and calls `send` with the originating thread. The gateway owns the shared normalization, canonical persistence, authorization and delivery contracts; EVE owns agent execution and session continuation.
- The deployment must support the adapter's event lifecycle. Discord slash commands/button interactions can arrive through webhooks, but regular messages and reactions use a Gateway WebSocket listener. Matrix requires a long-running sync/event process; it cannot run in a webhook-only serverless runtime. Do not assume all adapters can be hosted as short-lived webhook functions. Validate reconnection, event delivery, scaling and process lifecycle for the selected hosting environment.
- Use a custom adapter when no suitable adapter exists or a platform feature is missing. Do not claim support solely because a generic Chat SDK interface exists.
- Chat SDK adapter state may be used for adapter-specific subscriptions, locks, deduplication or caching. It must not become the canonical product transcript or replace the application's full message store. Configure Web adapter message-history caching deliberately; the canonical transcript remains in the product PostgreSQL database.
- The shared EVE runtime integration consumes turns from the common Chat SDK gateway through `chatSdkChannel`; inbound/outbound messages must still pass through shared persistence, identity, authorization, history-search and delivery contracts.
- The Web adapter is the browser's chat-message ingress and response-stream transport, not the whole browser application or a replacement for product services. Its current v1 feature set is deliberately limited: text/Markdown and request-scoped SSE streaming are supported; native file uploads, buttons/cards/modals, thread listing, a client-facing full-history API and multi-tab/disconnected-browser proactive push are not. Do not claim these capabilities are supplied by the adapter.
- Keep canonical conversation listing/history in the product conversation service. Expose only the required authenticated non-chat resource operations (for example, read-only history queries and signed attachment/artifact transfer) through the host product; do not recreate a parallel chat-turn submission or agent-response streaming API.
- Browser-targeted background messages must be durably stored. They can be displayed after the client reconnects or reloads from canonical history; do not promise real-time browser push while the client is disconnected.

References: [Chat SDK repository and feature overview](https://github.com/vercel/chat), [Chat SDK documentation](https://chat-sdk.dev/docs/getting-started), [Chat SDK Web adapter documentation and v1 feature matrix](https://chat-sdk.dev/adapters/official/web), [EVE Chat SDK channel](https://github.com/vercel/eve/blob/main/docs/channels/chat-sdk.mdx), [Vercel guidance on Chat SDK and EVE](https://vercel.com/kb/guide/chat-sdk-and-eve), [Chat SDK adapter catalog](https://chat-sdk.dev/docs/create-chat-sdk).

### 7.2 Persistence ownership

Conversation records, messages, enabled attachment metadata/references, channel identity mappings, and outbound delivery records are persisted through the host product's PostgreSQL database and storage interfaces, using Drizzle ORM. When `CHATBOT_ATTACHMENTS_ENABLED=false`, no attachment content or durable attachment reference is persisted.

The library may provide Drizzle schema definitions, migrations, repository implementations and indexes for these shared chat records. Each product applies and operates them in its own database. Do not run a separate shared chatbot database or allow Chat SDK's short-lived state/cache to be the only copy of a message.

Product business data remains owned by the product application. The EVE runtime's session state is execution context, not proof that a message or business action was persisted successfully. In local development and CI, run CloudNativePG-managed PostgreSQL inside the local Kubernetes test cluster, using a persistent volume. Retain the PVC and its backing storage across Pod restarts and normal application redeployments; deleting the cluster or losing the host can still destroy local data. CI environments may intentionally be ephemeral. Production deployments use the same CloudNativePG model with environment-appropriate persistent storage, independent backups and tested recovery. Do not use SQLite or a separate PostgreSQL emulator for accepted durable application state.

### 7.3 Environment configuration and secrets

All application-level and runtime configuration is supplied through environment variables only. This includes channel and integration settings, model/provider settings, feature flags, runtime options, limits, voice settings, attachment-ingestion controls, group mention behavior and shared scheduler limits. Do not use database rows, conversational commands or a separate settings UI as an alternative source for application/runtime configuration. Each deployment must receive its configured values through its environment; do not bake deployment-specific values into container images or source code. This rule does not turn bot-account identity mappings, conversation history/context or product business records into environment variables: those are durable application data and remain in the product database.

The following named common settings must be explicitly configured for each deployment:

| Environment variable | Required value | Purpose and enforcement |
|---|---|---|
| `CHATBOT_ATTACHMENTS_ENABLED` | `true` or `false` | Controls whether attachment bytes may enter backend storage and agent processing. When `false`, attachment bytes are not fetched, uploaded, proxied, persisted or passed to the agent. |
| `CHATBOT_CONTEXT` | Required, non-secret text | Deployment-provided additional system context. Load at application startup and append it to the agent's existing system-role context, preserving existing system instructions; do not replace or overwrite them. Send the added content with role `system`, not as user content, a developer-role message or ordinary retrieved context. |
| `CHATBOT_REQUIRE_MENTION` | `true` or `false` | Controls whether a supported mention/reply is required for the bot to respond in group conversations. Direct messages do not require a mention; platform permissions and privacy rules still apply. |
| `CHATBOT_MAX_ACTIVE_SCHEDULES` | Positive integer | Maximum number of active schedule definitions in the deployment, counted across all bot accounts. Includes one-time schedules, recurring schedules and reminders; excludes paused, deactivated, cancelled and completed one-time schedules. |
| `CHATBOT_MIN_SCHEDULE_INTERVAL_SECONDS` | Non-negative integer | Minimum recurrence interval for time-based recurring schedules and reminders. For example, `86400` imposes a 24-hour minimum, preventing hourly or every-four-hour recurrence. Set to `0` to disable this minimum-interval restriction. It does not restrict the lead time of a one-time schedule/reminder or the event cadence of event-/condition-triggered schedules. |
| `CHATBOT_AI_CONFIG` | Required JSON object | Configures the AI-compatible API endpoint in one object. `base_url` is required; `model` is optional and is supplied only when the endpoint requires an explicit model name; `api_key` is optional and is supplied only when the endpoint requires API-key authentication. No provider selector or provider-specific default endpoint is assumed. |

Set **`CHATBOT_CONTEXT`** independently for each product deployment. Load its value from the application process environment at startup and append its text to the system-role context already configured for the agent. Preserve the existing system prompt/instructions and their content; `CHATBOT_CONTEXT` adds deployment-specific instructions and must never replace or overwrite the existing system-role context. Include the added text with role `system`, alongside the product agent's configured instructions and before turn-specific conversation messages; it does not replace conversation history. Supply this non-secret value through a ConfigMap or equivalent deployment environment configuration. Do not put credentials or other secrets in it. Updating the value takes effect after a workload restart or rollout.

Enable and configure the common scheduler and periodic-worker functionality through deployment environment variables (for example, ConfigMaps for non-sensitive controls). These are common platform features; each product supplies the scheduled handlers and work to execute. Keep feature controls and operational settings in deployment configuration, not in product agent instructions.

The active-schedule cap is enforced atomically on schedule creation, activation and resumption. When the cap is reached, requests that would add another active schedule are rejected with a clear explanation; existing schedules are not silently removed. The minimum recurrence interval is checked when a time-based recurring schedule is created, changed or resumed. These are deployment-wide scheduler limits, not per-bot settings. Both settings apply to every product using the shared scheduler in that deployment.

Application workloads receive environment variables through Kubernetes resources: use ConfigMaps for non-sensitive values and Secrets for sensitive values. The attachment, mention and scheduler-limit settings are non-sensitive and should be supplied through ConfigMaps. Supply `CHATBOT_AI_CONFIG` through a Kubernetes ConfigMap when its object does not contain an API key; when `api_key` is present, supply the entire JSON object through a Kubernetes Secret so the credential is not exposed. Omit `api_key` for endpoints that do not require API-key authentication.

- Use **ConfigMaps** for non-sensitive settings such as log level, feature flags, public service endpoints, allowed origins and non-secret runtime options. Consume individual keys through `env[].valueFrom.configMapKeyRef`, or a controlled set through `envFrom.configMapRef`.
- Use **Kubernetes Secrets** for credentials and sensitive values such as database passwords, signing keys, OAuth/client secrets, channel tokens, webhook secrets and provider API keys. Reference only the required keys through `secretKeyRef` or a narrowly scoped `envFrom.secretRef`; mount files instead when a library expects certificate/key files or when file-based rotation is preferable.
- Never commit plaintext Secret manifests, credentials, connection strings containing passwords, or private keys. Use Flux CD with SOPS-encrypted Secret manifests (or a separately selected external secret manager integration) so Git stores encrypted values and Flux decrypts them only during reconciliation. Keep decryption keys outside the Git repository and protect them independently.
- Kubernetes Secret values are base64-encoded, not automatically encrypted at rest. Enable cluster secret encryption at rest where supported, restrict Secret read permissions with namespace-scoped RBAC, restrict Pod creation/exec privileges, and avoid broad service-account access. Do not log or expose secret values to the browser, agent/model context, task transcripts or diagnostics.
- Use separate namespaces, ConfigMaps, Secrets, service accounts and credentials for each product deployment, even when they share a cluster. Crossplane provider credentials are held in appropriately scoped Kubernetes Secrets and referenced by the relevant ProviderConfig; they are not copied into application Pods unless the application itself needs them.
- Prefer Kubernetes Service DNS names for in-cluster dependencies. `localhost` inside an application Pod refers to that same Pod, not PostgreSQL or another service. Use the CloudNativePG read/write Service name for database connections; use port-forwarding only for deliberate host-side development access.
- Environment variables sourced from ConfigMaps or Secrets are fixed when a container starts. A change to their values requires a workload rollout/restart to update the process environment. Use mounted files or a dedicated reload mechanism when a workload must observe rotations without restart; do not assume env vars update live.

### 7.4 Security and operational requirements

- Resolve product/account scope and sender identity from server-side channel and bot-account context. External sender IDs are scoped to their platform connection. Human identity associations are created and changed only through the relevant bot account's management conversation; do not infer a link from a matching name or username. No separate identity-verification step is required by this common linking flow.
- The product's authorization layer decides what a participant and agent can read or change. Recheck authorization for inbound events, agent tool calls, callbacks, background work, attachment retrieval and proactive messages.
- Do not expose channel tokens, webhook secrets, database credentials or storage credentials to the browser or model.
- Keep each conversation's history and attachment access isolated. For multi-tenant products, enforce tenant scoping in repository queries and application services, not just in prompts.
- Deduplicate webhook events and handle duplicates, retries and out-of-order delivery. Use external event/message identifiers where available and application idempotency keys for outbound effects.
- Persist accepted messages and outbound intents before acknowledging durable receipt. Recover pending delivery after restarts without repeating product actions.
- Keep logs and diagnostics useful but redact credentials, private attachment content and unnecessary personal information.
- Record platform-specific capability/configuration so the system can explain why a message, mention, attachment or proactive delivery cannot be processed on a given channel.
- Verify that `CHATBOT_AI_CONFIG` is parsed as a JSON object with required `base_url` and optional `model` and `api_key` fields. A local/self-hosted endpoint must work with `model` and `api_key` omitted when it does not require them; an endpoint requiring an explicit model or API-key authentication receives those values through configuration, with the API key kept in a Kubernetes Secret and never exposed to model context, the browser or logs.
- Verify that the scheduler rejects creation, activation or resumption when the deployment-wide active-schedule cap would be exceeded, including under concurrent requests.
- Verify that recurring schedules and reminders below the configured minimum recurrence interval are rejected on creation, edit and resumption, while one-time schedules and event-/condition-triggered schedules remain governed by their applicable rules.
- Keep the shared library and gateway maintainable and replaceable. Do not let a platform SDK's internal identifiers or EVE's runtime session model become the public product contract.

### 7.5 Common deployment and parity workflow

The application, database and supporting services run as Kubernetes workloads. Local development, CI, self-managed non-cloud deployments and managed cloud deployments use the same Kubernetes API/resource contracts, CloudNativePG operator, Kustomize bases and Flux reconciliation; environment overlays supply storage classes, replica counts, resource budgets, endpoints and credentials. The shared chatbot is distribution-agnostic: deployment tooling uses the standard Kubernetes API, while the application runtime does not create clusters, administer nodes or implement a multi-cluster control plane.

Cluster bootstrap and infrastructure provisioning are distinct from workload deployment:

- **Local development and CI:** create or use a local Kubernetes test cluster and apply the same Kubernetes resource contracts used for production. The test cluster is a deployment profile, not a separate application architecture.
- **Self-managed VPS or local-machine deployment:** the operator checks host capacity and prerequisites, configures secrets, hostname, networking and storage, then bootstraps a Kubernetes cluster on the host and installs the initial GitOps components. This uses the same Kubernetes API and workload model as a cloud-managed cluster. A single-node host is a single failure domain, not a highly available cluster; production-grade use requires appropriate topology, durable storage, off-host backups, tested recovery, monitoring, security and upgrade/maintenance procedures.
- **Managed cloud deployment:** provision or select the managed cluster through the cloud/provider infrastructure process. AKS, EKS and GKE are supported target profiles after their required Kubernetes APIs, storage, networking, identity and workload capabilities pass validation.
- **Infrastructure provisioning:** bootstrap a management plane before installing Crossplane. Use supported provider resources/compositions to provision infrastructure; Crossplane cannot bootstrap itself onto a cluster that does not yet exist.
- **Workload configuration:** Flux reconciles Kustomize bases/overlays for namespaces, policies, CloudNativePG, application services, workers, storage configuration, ingress and monitoring. Operators reconcile their own custom resources. Avoid imperative one-off deployment paths except for documented bootstrap operations.

**Local development and CI**

1. Provide one documented command to start the complete local development environment, including the local Kubernetes test cluster, required supporting services, CloudNativePG, the application and the shared browser client. It must wait for readiness and leave the environment usable without requiring a sequence of manual startup commands.
2. Use Flux CD and the local Kustomize overlay to deploy the application, supporting services and CloudNativePG operator. Create the PostgreSQL Cluster resource through GitOps and use a persistent volume from the cluster StorageClass. Retain local backing storage across normal restarts and redeployments; CI may destroy its disposable environment after tests.
3. Build prebuilt immutable images, deploy through the same Kustomize/Flux manifests and application configuration contracts, and promote the same image digests through subsequent validation.
4. Test application, database, gateway, frontend, scheduler, task recovery, storage and product capabilities. Include Pod restart/rescheduling, PostgreSQL restart and recovery, persistence across workload restart, upgrade/rollback, event replay, task cancellation and end-to-end checks. Test local persistence separately from backup/restore and machine-loss recovery.

**One-machine VPS deployment**

1. Use the same immutable application images, Kustomize manifests, Crossplane resources and Flux CD reconciliation flow validated locally and in CI. Bootstrap a self-managed Kubernetes cluster on the VPS or local machine, then run the application, PostgreSQL and supported services as workloads inside that cluster within the host's resource budget.
2. Deploy CloudNativePG-managed PostgreSQL with persistent volumes from the cluster's configured StorageClass. Configure backups outside the primary database volume and test recovery. Do not substitute SQLite or a separate external PostgreSQL service for the standard deployment.
3. The shared browser application may run inside the self-managed cluster for this deployment profile.

A successful unit-test run is not evidence of deployment parity. Do not substitute an ad-hoc Compose deployment or host-run service for this workflow.


## 8. Illustrative organizational use case

A community, company or project organization can use the shared chatbot as a common front door for members, contributors and maintainers. The organization supplies its own membership rules, knowledge, domain actions and meeting policies; the shared platform supplies channel normalization, identity mapping, canonical conversations, linked interactions, durable scheduling and outbound delivery.



### Example flow

1. An organization administrator connects the bot to the authorized workspace, configures enabled channels and eligible roles, and chooses whether group responses require a mention. Platform identity and channel permissions remain authoritative.
2. Maintainers provide organization-scoped instructions, policies, project information, event details and meeting availability through the organization's application tools. The agent receives only information applicable to the active organization and conversation.
3. A contributor asks a question in an enabled channel, thread or direct message. The shared gateway resolves the sender and channel, then routes the message to the correct canonical conversation. The agent answers what it can establish from authorized context.
4. For a missing or ambiguous detail, the agent forwards the question and only the necessary context to the organization's connected provider-side group or groups. Providers and helpers can discuss and respond there; the agent evaluates the replies and returns only information authorized for the originating conversation. Private member context is not exposed to other members.
5. A member can ask to arrange a contributor/mentor meeting. The organization's service determines participants and availability; the shared scheduler persists confirmations and reminders in the configured organization timezone. Other questions can continue while a response or meeting detail is pending.
6. Organization-wide announcements or onboarding messages require an authorized administrator's direction and an explicitly eligible audience. The shared outbound mechanism enforces channel permissions, delivery policy and opt-in requirements; it does not independently decide whom the organization should contact.

This is an illustrative shared-platform use case, not a built-in organization-management product. Membership, role meaning, knowledge scope, meeting rules, audience selection and domain authorization remain organization/product responsibilities.

## 9. Common versus product-specific boundary

The shared platform defines reusable technical contracts. Each product remains responsible for its own domain behavior and authorization:

- **Messaging and identity:** the shared layer normalizes channel identities, conversations, DMs, groups, threads, mentions and replies. Each product configures its own platform accounts, channel connections and access policies.
- **Conversation history:** the shared layer owns canonical message persistence, context isolation and history retrieval. Each product defines the authorized conversation scope available to its agent.
- **Proactive delivery:** the shared outbox handles durable delivery, retries and idempotency. Each product defines its own business triggers, consent requirements and message policy.
- **Attachments:** the shared layer receives, persists/references and authorizes retrieval. Each product decides how its agent processes supported files.
- **Bot accounts and provider groups:** the shared layer owns bot-account identity, platform connections and the association of authorized provider-side groups. Each product defines provider roles, group access policy and the business meaning of information shared from each group.
- **Authorization:** the shared layer provides identity and policy interfaces; each product supplies its own principals, roles, tenant/owner scope and authorization decisions.
- **Agent and client configuration:** the shared EVE integration and browser client are common. Agent instructions, tools, actions, approval semantics, views and branding remain product-owned.
- **Scheduling and tasks:** the shared scheduler, task lifecycle and linked-interaction primitives are common. Each product supplies domain handlers, business records and rules for human-dependent work.
- **Kubernetes access:** Kubernetes is deployment infrastructure, not a default agent capability. A product may expose narrowly scoped cluster operations only through its own authorized service boundary.

The common library must not own product business records, business workflows, approval policies, domain-specific tools or product-specific deployment topology.

## 10. Shared acceptance checks

The shared implementation is complete only when these checks pass independently in all product deployments:

- A user can start and continue a direct conversation using ordinary text without a slash command.
- A bot account can connect to multiple authorized provider-side groups; questions and context can be shared with relevant groups, and the agent evaluates replies and enforces the originating customer's information scope.
- A supported platform identity maps multiple DMs/groups/threads to distinct canonical conversations without cross-conversation context leakage.
- A bot account can be provisioned and connected to a platform installation without confusing the bot account ID with a platform bot ID or a human user ID.
- A provider can sign in to a bot account using its ID and authorize one or more groups as provider-side communication surfaces without creating a separate provider-DM workflow.
- Switching a bot's platform identity or channel connection also leaves its internal bot account, context, history, tasks and data unchanged.
- Provider contributions and corrections received in connected groups are stored with source and group scope; the agent can use them in later conversations only when the originating product's authorization and information scope permit.
- When a customer asks a question outside the available context, the agent forwards it with necessary context to relevant connected provider groups, evaluates available replies, and returns an authorized answer to the originating conversation.
- When no relevant provider-side group is connected, the agent does not invent a recipient or handoff; it clearly states that it cannot determine the answer from available context.
- With `CHATBOT_REQUIRE_MENTION=true`, the bot ignores unaddressed group messages and responds to supported mentions/replies; with `false`, it can respond to ordinary group messages without an @mention, subject to platform delivery and permissions. Direct messages work without a mention in either mode.
- A web conversation enters through `@chat-adapter/web` on the same unified Chat SDK gateway as messaging platforms, is authenticated by the server-side `getUser` resolver, and maps to an isolated canonical conversation/EVE session; it does not create or invite a messaging bot or use a separate chat-message API.
- All product deployments use the same Vercel EVE runtime integration, unified Chat SDK gateway (including the Web adapter), shared web client, PostgreSQL/Drizzle persistence contracts and deployment/parity workflow; product-specific actions remain separately authorized.
- A proactive text message can be queued and delivered without an active or immediately preceding user turn, provided the platform permits that destination and the required consent/opt-in exists.
- One-time, recurring, interval and event-triggered schedules produce durable runs in the configured timezone and remain due/recoverable while the agent or client is offline.
- Accepted asynchronous tasks survive client disconnect and process restart; cancellation is recorded and stale workers are fenced from committing.
- Replayed or duplicate source events resume from a durable cursor and do not repeat an accepted business effect.
- Provider-group replies are evaluated against source, relevance and customer scope before being returned; group discussions do not merge unrelated customer conversations or expose private transcripts.
- With `CHATBOT_ATTACHMENTS_ENABLED=true`, a file attached through a supported web or messaging client is received, linked to the correct canonical message/conversation and made available to the authorized agent; unsupported file processing is not assumed.
- With `CHATBOT_ATTACHMENTS_ENABLED=false`, attachment bytes are not fetched, uploaded, proxied, stored or passed to the agent; the text portion of the message still works, and attachment-related storage and AI processing are avoided.
- A transient delivery failure or duplicate event does not lose a message or repeat its associated business action.
- `search_conversation_history` finds an exact older message from before context compaction and returns its source reference and timestamp.
- The history tool cannot retrieve messages from an unrelated conversation or product scope.
- A restart does not erase canonical messages, enabled attachment metadata/references or pending outbound delivery.
- A platform-specific unsupported capability is rejected or reported explicitly, not silently treated as successful.
