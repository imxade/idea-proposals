# Chatbot — Shared Conversation Platform

## Architecture overview

The default architecture is a **shared multi-tenant SaaS platform** with a provider dashboard. Each provider tenant can create and manage multiple logical bots. Each bot owns its configuration, optional system instructions, persistent context and conversation history, together with its platform connections, credentials and linked destinations. The provider creates and owns each official application/bot identity on the relevant platform; the SaaS does not create or own those external identities. The dashboard manages bot records and connection setup, and supports linking/unlinking channels, groups and destinations through platform APIs where available. Where a platform requires an administrator to complete an action on the platform itself, the dashboard guides that step and verifies the resulting connection. All bot and connection data remains tenant-scoped.

**Exclusive deployment constraint:** Kubernetes is the only supported deployment platform for the complete application. Every deployment profile—including local development, CI, a self-managed VPS or local machine, and managed cloud—must deploy the complete application stack into a Kubernetes cluster. The frontend, backend, EVE runtime and runtime services, Chat SDK channel/gateway, workers, schedulers and application data services must run as workloads inside that deployment's target cluster. PostgreSQL must run inside the same cluster under CloudNativePG management. Do not support a split deployment with application components or the database hosted outside the target cluster.

The shared chatbot application, provider dashboard, Vercel EVE runtime integration and PostgreSQL persistence serve many provider tenants. Each logical bot is a durable application-owned record within its provider tenant; its connections are replaceable transport resources. Tenant and bot scope are resolved from trusted authentication and verified connection metadata. Every read, write, session, tool, task, file and outbound action is scoped to the owning tenant and bot. A provider may edit credentials, replace a platform identity, or link/unlink destinations without losing that bot's application-owned context, instructions, configuration or history. A replacement identity must be authorized separately and its platform-side memberships/permissions established again; external memberships and conversation identifiers do not automatically transfer.

Shared PostgreSQL and shared runtime infrastructure are the default. Dedicated databases/CloudNativePG clusters, namespaces or isolated application workloads may be used when stronger isolation is required; they do not change the shared SaaS default.

~~~mermaid
flowchart TB
  subgraph Providers["Provider-owned platform identities (connected per logical bot)"]
    IdentityA["Bot A provider-owned application identity"]
    IdentityB["Bot B provider-owned application identity"]
    IdentityN["Additional bot connections"]
  end
  subgraph SaaS["Shared multi-tenant chatbot SaaS"]
    Dashboard["Provider dashboard<br/>bot and connection management"]
    Auth["Tenant authentication and authorization"]
    Bots["Tenant-scoped logical bots<br/>bot configuration and optional system instructions"]
    Registry["Bot-scoped platform connections<br/>credentials, capabilities and destinations"]
    EveChatSDK["EVE Chat SDK channel<br/>platform adapters, per-connection auth"]
    Gateway["Unified gateway<br/>tenant/bot authorization and routing"]
    Conversation["Bot-scoped canonical conversations and context"]
    Runtime["Shared EVE runtime pool<br/>tenant- and bot-scoped agent sessions"]
    Scheduler["Durable scheduler and task workers"]
    Outbox["Authorized outbound dispatcher / outbox"]
    Tools["Common agent tools"]
    DB[("Shared PostgreSQL<br/>tenant-scoped records")]
    Blob[("Tenant-scoped attachment storage")]
    Auth --> Dashboard
    Dashboard --> Bots
    Bots --> Registry
    Registry --> EveChatSDK
    EveChatSDK <--> Gateway
    Gateway --> Conversation
    Conversation <--> Runtime
    Runtime <--> Tools
    Runtime --> Outbox
    Scheduler <--> Runtime
    Dashboard <--> DB
    Bots <--> DB
    Gateway <--> DB
    Conversation <--> DB
    Runtime <--> DB
    Scheduler <--> DB
    Outbox <--> DB
    Outbox --> Gateway
    Conversation <--> Blob
  end
  IdentityA <-->|"adapter-managed events and delivery"| EveChatSDK
  IdentityB <-->|"adapter-managed events and delivery"| EveChatSDK
  IdentityN <-->|"adapter-managed events and delivery"| EveChatSDK
~~~

**Normative separation rule:** common requirements are specified here once. Provider-specific behavior and authorization are supplied through tenant-scoped extension points; they must not create competing shared implementations or weaken the common isolation contract.

**Required messaging integration rule:** all supported browser and messaging-platform chat ingress and egress must use EVE's provided Chat SDK channel integration, `chatSdkChannel` from `eve/channels/chat-sdk`, with the appropriate Vercel Chat SDK adapters. Do not build a parallel direct-platform API, webhook or message-delivery path for capabilities handled by a registered adapter. The adapter owns platform-specific authentication, webhook verification, event handling and delivery; the EVE channel hands accepted turns to the EVE runtime. The shared application gateway remains responsible for tenant/bot authorization, canonical persistence, routing, policy and durable delivery coordination around that integration.

## 1. Architecture: modular shared chatbot application

Implement the chatbot as a **modular shared multi-tenant SaaS application** with stable internal interfaces, rather than a separate chatbot implementation for each product. The default deployment is one shared application, shared EVE runtime pool and shared PostgreSQL persistence layer. Optional dedicated deployments may be offered for providers requiring stronger contractual or regulatory isolation, but they are not the default.

Each provider tenant supplies its authorization policy, agent configuration, domain capabilities and data scope through explicit tenant-scoped extension points. Dedicated databases, namespaces or deployments remain available when a provider's isolation requirements justify the operational cost.

This document specifies the shared application and its contracts, not an implementation repository layout.

### 1.1 Application component boundaries

The shared application should maintain these modules/interfaces:

| Module or interface | Shared responsibility |
|---|---|
| **Provider dashboard and bot management** | Let authenticated provider users create and manage multiple logical bots and edit each bot's settings, optional system instructions, API capability references, secrets, platform connections and linked destinations. Enforce tenant and bot authorization for every operation. |
| **Bot-scoped secret vault** | Let authorized bot administrators save, update and remove individually named secret entries for each logical bot, including multiline `KEY=VALUE` input through the dashboard. Bot secrets are distinct from deployment-wide environment variables and other bots' secrets. |
| **Application adapter registry** | Install and register Chat SDK-compatible official, vendor or third-party messaging adapters directly in the chatbot application behind a common interface, with capability and lifecycle metadata. |
| **Provider integration registry** | Maintain bot-scoped connections to provider-created platform identities, authorization state, encrypted credential references, linked destinations, capabilities and lifecycle. Resolve only adapters registered in the chatbot application, with the correct instance and credential reference for each verified tenant/bot connection. It does not create or own external platform identities. |
| **Principal and identity mapping** | Map authenticated application principals and external platform identities to a tenant and its authorization policy without treating platform IDs as global identities. |
| **Conversation service and repository** | Create and resolve canonical conversations, associate messages and participants, persist messages, maintain isolated context and provide indexed history retrieval/search. |
| **Shared web application** | Own the reusable browser chat and authenticated provider dashboard, including bot management, connection settings, conversation history, transcript/composer, streaming, voice, file attachments and progress/events. |
| **EVE Chat SDK channel and unified gateway** | Use EVE's `chatSdkChannel` with the selected Chat SDK adapters as the required browser/platform message ingress and egress path. The application gateway adds trusted tenant/bot resolution, authorization, canonical persistence, routing and durable-delivery contracts; it is not a parallel transport implementation. |
| **Vercel EVE runtime integration** | Run tenant-scoped agent sessions on the shared EVE runtime baseline, including lifecycle, tool calling, workflows, delegation and events. |
| **Outbound dispatcher and durable outbox** | Send owner-authorized or tenant-workflow-authorized messages through the selected tenant-/bot-scoped Chat SDK adapter registered with EVE's channel; persist delivery intent and retry safely. |
| **Attachment receipt and retrieval** | Enforce attachment-ingestion settings, store tenant-scoped metadata/content references and provide authorized retrieval. |
| **Scheduler service and repository** | Persist schedules and durable executions in shared PostgreSQL with tenant ownership, quotas and fair scheduling. |
| **Task/work service and repository** | Provide tenant-scoped asynchronous task/run identities, lifecycle, dependencies, cancellation, checkpoints, leases, retries and recovery. |
| **Linked interaction service and repository** | Correlate related conversations, questions, deferred work and replies through scoped, authorized interaction records without merging transcripts. |
| **Product action registry** | Register tenant/product-owned handlers for scheduled work, asynchronous tasks and interaction events under tenant authorization. |
| **Common agent tools** | Expose conversation-history search and authorized common task/schedule/interaction operations to the connected agent runtime. |

The common platform standardizes on Vercel EVE for agent execution. Conversation persistence, scheduler, task lifecycle, linked-interaction mechanism and gateway remain modular TypeScript components around that runtime; none embeds provider-specific business logic.

### 1.2 Tenant and product extension points

Each provider tenant supplies its authenticated principal and authorization resolver, human-role definitions, tenant data scope, capabilities, tools and domain handlers. A tenant can create multiple logical bots; each bot owns its configuration, credentials/connections, linked destinations, optional system-level instructions, context and conversation history. The common layer maps verified external identities to tenant principals and bot connections; the tenant defines role meaning and permissions.

Tenant defaults and bot-specific context/configuration are stored as access-controlled application data and composed with deployment-wide defaults. Bot-specific settings are always associated with the owning logical bot, not inferred from the connected platform identity. A bot may have optional provider-authored system instructions; when configured, include them in the system-role instruction stack ahead of ordinary conversation/user messages, while preserving higher-priority application safety and security instructions. Context must not be promoted into instructions unless explicitly configured as bot instructions. The tenant supplies usage instructions and input/output contracts for its domain capabilities. Attachment access and processing remain tenant- and bot-authorized.

The chatbot application installs and registers its supported Chat SDK adapters directly. Provider users configure only adapters available in that application; dashboard input does not install executable adapter packages. Each logical bot may also have API names, documentation links or endpoint/contract references and instructions describing preferred external capabilities. Its agent may use any available code or network execution capability to make API requests, including Python or Node.js, without a mandated HTTP tool or per-service chatbot SDK. External services remain authoritative for their own operations and authorization. Tenant-owned handlers for schedules, asynchronous tasks and linked interactions may be registered through in-process modules/providers or authenticated APIs. The agent can invoke these capabilities when relevant, but context does not grant permission: the owning service must still enforce authentication, data scope and business rules. All browser and platform chat turns enter through EVE's Chat SDK channel integration and the unified gateway; non-chat resource operations such as listing history, attachment transfer and settings retrieval remain separate authorized application services. No separate universal API or common tool protocol/registry is required for every tenant capability.

### 1.3 Shared EVE runtime and unified gateway

All tenants use the same **shared Vercel EVE runtime baseline and runtime pool**. The platform may host many tenant- and bot-scoped agent instances/sessions on that shared runtime; it does not create a separate runtime deployment for every provider or bot. Each request is bound to a tenant and logical bot using trusted server-side authentication and a verified integration connection. An EVE session ID is never accepted as proof of tenant or bot ownership.

EVE session state is execution context, not the canonical record of conversations or business state. Persist bot configuration, optional system instructions, messages, bot context, task/business records, schedules, checkpoints and delivery intents in shared PostgreSQL with tenant and bot ownership, and enforce authorization in repository/service boundaries. Each bot's instructions, tools, capabilities, delegation policy and domain workflows remain bot-/tenant-owned; the common platform provides EVE's generic runtime primitives.

EVE's provided `chatSdkChannel` integration is the required transport bridge between the Vercel Chat SDK adapters and the shared EVE runtime. The chatbot application registers its supported messaging adapters and `@chat-adapter/web` for browser chat with this channel. Accepted adapter events are handed to EVE from the Chat SDK handler through the channel's `send` operation; replies return through the originating adapter/thread. Do not add direct platform message-send calls, a second webhook/event pipeline or a separate chat-turn API for a capability already provided by the adapter.

The **unified gateway** is the shared application-level authorization and routing boundary around that channel, not an alternative messaging transport, internal platform identity or per-provider gateway. It resolves the tenant, logical bot and authorized provider connection from trusted server-side context; applies policy; coordinates canonical persistence and idempotency; maps the accepted turn to the correct bot-scoped EVE session; and ensures the reply is delivered through the originating connection. The gateway must not bypass the EVE Chat SDK channel to call platform messaging APIs directly.

The connection registry must select the credential and adapter configuration belonging to the verified tenant/bot connection. Validate whether each selected adapter supports multiple independently credentialed identities and dynamic per-connection resolution. Do not assume a built-in adapter can safely serve every tenant or multiple bots on one static configuration. Where required, use isolated adapter instances or a custom adapter that implements the Chat SDK adapter contract and is registered through `chatSdkChannel`; keep all such instances within the shared application/runtime architecture and enforce tenant/bot scope on every event and send.

An authorized owner can explicitly initiate outbound messages through a selected connected provider identity. Any other background delivery must be tied to a tenant-authorized workflow and eligible destination; the platform does not independently select prospects or perform autonomous cold outreach. Persist messages and delivery intents before acknowledging durable receipt, then deliver through the selected Chat SDK adapter.

Use direct platform APIs only for narrowly scoped administrative operations that the relevant adapter does not expose, such as parts of OAuth setup, destination discovery, membership or permission administration. Keep those operations in separate authenticated integration services; they must not become an alternate path for chat-message ingress, processing or delivery.

The gateway and runtime are shared, while logical bot records, provider identities, credentials, connection metadata, linked destinations, conversation data, context, instructions and authorization remain tenant- and bot-isolated. A custom adapter is an extension of the Chat SDK integration, not a reason to bypass it.

## 2. Logical bots, platform identities and conversations

### 2.1 Provider-owned platform identities

A provider can create multiple logical bots in the SaaS dashboard. Each logical bot is an application-owned record with its own configuration, optional system instructions, persistent context, canonical conversation history and one or more platform connections. The logical bot is distinct from the external platform identity: the provider creates and owns the official application/bot identity directly on the relevant platform, then authorizes the SaaS to receive events and send messages through it using the platform's supported credential or authorization mechanisms. The SaaS does not create or own the provider's external platform identity.

From the dashboard, the provider selects an adapter made available by the deployed product and can link or unlink its supported channels, groups, rooms, workspaces and other destinations to a selected logical bot. Where the platform exposes suitable APIs, the dashboard performs the operation; where platform policy requires an administrator to act directly on the platform, the dashboard provides the required instructions and confirms the resulting membership/permissions. The platform remains authoritative for membership, access and permissions. There is no internal bot-management conversation.

The application stores each connection against both its tenant and logical bot, including the platform, external identity, authorization/credential reference, linked destinations, effective capabilities and lifecycle status. The dashboard lets an authorized provider user add, edit/rotate, validate, revoke, replace and remove connection credentials as supported by the platform. Secrets are held in a secrets manager or encrypted credential store, never in prompts, browser state or ordinary logs. A bot may have multiple connections where supported; a provider may manage multiple bots independently.

Editing, rotating or replacing a bot's credentials or external platform identity does not create a new logical bot and does not delete or reset that bot's context, system instructions, canonical conversation history, tasks, schedules or other configuration. The replacement identity must be separately authorized, and its memberships/permissions must be established or linked again on the platform. Do not assume external group membership, platform conversation identifiers or message history transfer to the replacement identity. Retain application-owned history; associate newly received conversations only through explicit, authorized mapping rules.

A platform identity is a transport identity connected to a logical bot, not the bot's durable identity or a separate agent runtime. One logical bot may have multiple connections, and one connection may serve multiple conversations where the platform permits it. Platform membership, access and message visibility remain controlled by the platform's own installation, invite and permission mechanisms.

### 2.2 Provider connection lifecycle and authorization

1. The provider creates a logical bot in the SaaS dashboard and configures its bot-level settings and optional system instructions.
2. The provider selects an application-registered adapter, creates and owns the corresponding official application/bot identity on that platform, then authorizes the SaaS application through the platform's supported credential or authorization flow. The dashboard records the credential/authorization against the selected logical bot; it does not install the adapter package.
3. The provider links the desired channels, groups or destinations to that bot through the dashboard where supported; otherwise, the dashboard guides the provider through the platform-side action and verifies the resulting access.
4. The application verifies authorization and destination access, records the bot-scoped connection and effective capabilities, and begins accepting verified events.
5. The provider can edit/rotate credentials, unlink destinations, revoke authorization, replace an external identity or add another connection. Only the affected connection records change; the logical bot's instructions, configuration, context, conversation history and business data remain intact.

There is no internal platform-owned messaging identity or management conversation. The SaaS does have its own logical bot records and provider dashboard; those records organize the bot's configuration and replaceable external platform connections. The platform identity remains the provider's messaging identity.

The application maps external sender identities to authenticated tenant principals only through trusted identity claims or an explicit tenant-controlled linking process. A display name or matching username is never sufficient proof. Changing a person's external platform identity updates only the relevant identity mapping after re-authentication; it does not change tenant ownership or reset application-owned context.

Each connection has its own credentials, permissions, capability record and linked destinations and belongs to exactly one logical bot within one tenant. A connection can access only that bot's authorized conversations and context. A connection must not read or send through another bot or tenant's identity/data.

An owner or other authorized tenant principal may explicitly initiate a message through a selected connected identity. The application verifies the principal's permission, destination eligibility, connection status and platform rules before queuing delivery. This is owner-directed operation, not autonomous prospecting or cold outreach.

If a provider revokes or replaces a connection, stop accepting events and sending through revoked credentials, mark the connection unavailable, and retain application-owned history. Do not silently rebind old external conversations to a replacement identity.

### 2.3 Canonical conversation mapping

The application creates a stable internal conversation ID and maps it to the external platform context. The mapping must preserve tenant ID, logical bot ID, authorized provider-connection ID, platform/channel type, external identity, destination/thread identifiers, external sender/message identifiers scoped to the connection, timestamps and reply/reference metadata.

A direct message, web conversation, group/channel thread or other independent context receives its own canonical conversation mapping. Context is never mixed across unrelated groups, DMs, tenants or web conversations. A web conversation maps the authenticated tenant principal, selected logical bot and client conversation identifier to a canonical conversation ID and bot-scoped EVE session. Adapter thread identifiers are routing identifiers, not the public conversation contract.

A runtime session ID is an execution identifier mapped to the canonical conversation ID, tenant and logical bot. It is not the source of truth for message history or authorization. Cross-conversation retrieval requires an explicit reference and an independent authorization check.

### 2.4 Direct, group and channel interaction

Ordinary natural language is the normal interaction method. Slash commands may be supplied as shortcuts, but users must not need a command to make an ordinary request.

- **Provider communication destinations:** provider/staff communication may use authorized groups or channels, or one-to-one conversations with an authorized provider/staff member, according to the selected adapter's capabilities and the provider's configured destinations. No mode is mandatory. When group communication is unavailable or unsupported, use an eligible one-to-one conversation if configured; when it is supported and authorized, a group/channel may be used. The agent can use only context received through authorized connections.
- **Mention-required group mode:** when `CHATBOT_REQUIRE_MENTION=true`, the bot responds in group conversations only to messages that mention/address it (including supported platform-native reply-to-bot behavior). Unaddressed group messages are ignored.
- **Mention-not-required group mode:** when `CHATBOT_REQUIRE_MENTION=false`, the bot may respond to ordinary group messages without an @mention, subject to platform permissions and the conversation's access policy.
- **Replies and threads:** replying to an agent message or another message can continue or refer to that exchange. Preserve the referenced message and thread context.

The deployment environment variable `CHATBOT_REQUIRE_MENTION` controls whether the bot must be mentioned before it responds in a connected group and must be explicitly set to `true` or `false`. Platform privacy settings and permissions still determine which group messages reach the bot; disabling the mention requirement cannot override them. The chatbot application must not impose a universal mention-only, public-only or private-only restriction.

**Initial messaging integration priorities:** Discord and WhatsApp Business are the first external messaging platforms targeted for implementation and validation. Browser chat through `@chat-adapter/web` is a core capability of the chatbot application. Additional compatible messaging adapters may be integrated later through the same modular interface. No separate MVP application or isolated integration product is required.

Actual access and visibility remain controlled by platform installation, membership, privacy settings, permissions, intents, user consent and policies. The application must not promise that every platform can expose all messages in all spaces. A platform adapter must report its real capabilities rather than silently pretending that an unsupported mode works.


### 2.5 Platform-specific capability constraints

The interaction model is shared, but actual access and delivery differ by platform. The application must record the effective capabilities of each adapter and connection rather than promising identical behaviour everywhere. Discord and WhatsApp Business are the initial external messaging priorities; Telegram and Matrix below document possible later extension constraints, not additional initial implementation obligations.

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

The application authenticates the event, resolves tenant and connection scope from trusted server-side context, checks access and deduplicates it before dispatching the turn to the shared Vercel EVE runtime. Before any attachment bytes are fetched from a messaging platform, uploaded by a web client, proxied to the application, persisted or exposed to the agent, the ingress path must enforce the deployment environment variable `CHATBOT_ATTACHMENTS_ENABLED` (`true` or `false`). Each deployment must set this value explicitly. When `false`, do not fetch or download attachment content, accept or proxy web-upload bytes, store attachment content or durable attachment references, or include attachments in the EVE/agent turn; ignore the attachment and continue processing any accompanying text. The client should disable its upload control when possible, but the backend must enforce the setting even if a client or platform still sends attachment metadata or an upload request. A lightweight indication that an attachment was ignored may be shown to the sender, but the attachment itself must not be transferred into backend storage or AI processing. This prevents unnecessary backend storage and attachment-related AI/token usage. When `true`, the normal authorized attachment receipt, storage, retrieval and agent-capability rules apply. Do not trust a model-supplied tenant, customer, owner or conversation identifier.

### 3.2 Agent and application messages

Agent replies are persisted as canonical messages and delivered through the adapter associated with the originating conversation. Preserve message ordering and reply/thread destination where the platform supports it.

The shared web client submits chat turns through the Web adapter registered with EVE's `chatSdkChannel` integration and receives the agent response over its AI SDK UI message-stream/SSE connection. Messaging clients use their registered Chat SDK adapters through the same EVE channel. A live browser connection is not required for the service to continue a task or deliver a later message; persist resulting messages and work state independently of the connection. The current Web adapter provides request-scoped streaming, not durable push to disconnected browsers or other open tabs.

The common project owns the standalone shared web application and its reusable chat experience. Products configure its identity, roles, authorization and product actions. All clients that submit chat turns use the same gateway and adapter contract; a product or third party may build a separate client against the documented client and conversation contracts without creating a second chat-message ingress or streaming path.

### 3.3 Owner-initiated outbound text — independent of an incoming user turn

An authorized owner **must be able to explicitly initiate a text message through a selected connected provider identity without an active inbound turn**. This is an owner-directed outbound capability, not permission for autonomous prospecting or cold outreach.

The owner-triggered operation must support an explicitly selected eligible destination and a clear message body. Separately, tenant-authorized workflows may deliver necessary service notifications or continue an already established interaction when configured and eligible. The platform must not independently choose new recipients or initiate unsolicited outreach.

The outbound operation accepts the authenticated owner/tenant principal, selected provider connection, authorized destination, content, cause/reference and idempotency key. It must not require an inbound message object or an open web client.

Use a durable outbound queue/outbox. Persist intent before delivery, record attempts and outcomes, retry transient failures, and deduplicate retries so a message is not sent repeatedly because of a timeout or restart. Keep message delivery separate from the domain action that triggered it; a delivery failure must not undo a completed booking, task or other product operation.

Owner-initiated sending is still subject to platform rules. Some destinations require prior interaction, consent or an eligible messaging window. Send only to a destination the provider has connected and the platform currently permits. An explicit owner action does not bypass recipient eligibility, platform policy or required opt-in.

### 3.4 Answering questions when provider handoff is unavailable

The agent should first answer the user's question directly using the active conversation, the bot's authorized context and knowledge, and available tenant/product tools. It should not treat provider handoff as the default response to an unanswered question. When a question is general and can be answered responsibly from general model knowledge, the agent may provide that general answer; it must distinguish general information from verified provider-specific, tenant-specific, transactional, policy or time-sensitive facts. For those facts, use authorized product sources and do not present an unsupported inference as confirmed.

When the agent cannot establish a reliable answer, or the question requires a provider decision or action, it may use provider handoff only when the application confirms that the current logical bot has an enabled, authorized and currently usable provider connection with an eligible destination for that request. A saved connection alone is insufficient: revoked or invalid credentials, missing destination links, missing permissions, unsupported capabilities or an unavailable adapter make that route unavailable. Check eligibility and connection status at the time of the handoff, because they can change after the conversation starts.

When no eligible provider channel/destination is configured or available, the agent must not say or imply that it has contacted the provider, forwarded or escalated the issue, is waiting for a provider response, or will get back to the user. It must not create a false pending request. Instead, it should answer any part it can support and clearly say that it cannot verify the remaining detail from the information currently available and that no provider contact route is configured or available for this bot. It must not guess to fill the gap. For example: “I can provide general guidance, but I can't verify the provider-specific answer from the information available here. This bot doesn't currently have an available provider contact channel, so I haven't forwarded your question.”

The application, not the model, is authoritative for handoff availability. Derive the capability from the current bot's persisted connection and destination records, authorization, permissions and adapter health. Expose a handoff operation only when an eligible route exists, or have the operation return a structured unavailable result; validate again when the operation executes. Provider-authored bot instructions may explain the behavior, but they cannot enable a missing route or override application policy.

The agent may say a request was forwarded only after the application has durably recorded the interaction and the outbound dispatch has been accepted by the configured channel/adapter. Wording must reflect the actual state: queued or accepted for delivery is not proof that a provider read it or responded. The agent must not promise a later response or follow-up unless a durable pending interaction, an eligible responder route and a follow-up/notification workflow have actually been created. If delivery fails or the route becomes unavailable, report that truthfully rather than implying a successful escalation.

## 4. Shared web client

The shared application includes both the reusable browser-based chat client and its Chat SDK Web adapter integration. The common project owns the standalone shared web application; `@chat-adapter/web` is its browser channel adapter and must be registered through EVE's `chatSdkChannel`, not exposed as a parallel message-ingress path. All product deployments can use the shared client and configure product identity, authorization, capabilities, branding and visible workspaces; they may also build another client using the same gateway contract.

### 4.1 Framework and application boundary

- Implement the shared web client in TypeScript using one of the supported frontend options listed in the common technical stack. Keep the UI framework replaceable; use the matching Chat SDK Web client integration (React, Vue or Svelte) and preserve the common canonical conversation and event contracts.
- Submit every browser chat turn through `@chat-adapter/web` registered with EVE's `chatSdkChannel` integration. Use that channel to dispatch the turn to the product's EVE runtime; do not create a parallel platform API path, separate authenticated chat-message API or independent browser streaming endpoint.
- Configure the Web adapter's `getUser` callback to resolve the authenticated user from the host product's verified server-side session. Derive the adapter thread ID from that authenticated identity and the client conversation identifier, then map it to the canonical conversation ID and EVE session. Enforce product/tenant scope and authorization on the server for every turn.
- The common project owns and ships the shared web application. It may be a separate frontend workload from the backend, but both belong to the same deployment cluster; a separately hosted frontend or Web adapter endpoint is not supported.
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

The shared client does not implement product-specific approval policies, compute controls, booking workflows or provider/customer operations. It renders authorized product-supplied actions through extension points. Each tenant supplies its own domain actions and approval semantics. A product must not expose an action unless its server-side authorization permits it.

### 4.5 Client reliability and security

- The client is replaceable and must not own agent execution, durable task state, schedules, canonical message storage or artifact storage.
- Closing, refreshing or replacing the browser client must not interrupt background work or lose messages.
- Reconnect and resynchronize from the canonical server state after a network interruption; do not rely solely on transient browser memory or an open streaming connection.
- Authenticate browser chat at the Web adapter boundary through `getUser`; use separate authenticated host-product resource operations only for non-chat reads and attachment/artifact transfer. Use server-issued upload/download/access references. Never expose channel tokens, database credentials, speech-provider secrets or unrestricted internal service endpoints.
- Reconnect the Web adapter stream to the canonical conversation state after interruption. Because Web v1 does not provide multi-tab/disconnected-browser proactive push, persist background replies and refresh the canonical transcript when the client reconnects or reopens.
- Resolve expiring artifact and product-access URLs at click time through the host product service.

## 5. Durable scheduling, asynchronous tasks and linked interactions

The common platform provides reusable mechanisms for durable schedules, asynchronous work and linked interactions. It persists tenant-owned records in shared PostgreSQL and invokes only handlers registered and authorized for that tenant. The common layer determines when work is due, tracks its lifecycle and routes related events; it does not implement tenant-domain behavior.

### 5.1 Shared scheduler

Support:
- One-time date/time schedules and calendar recurrences (daily, weekly, monthly and yearly).
- Interval schedules such as hourly checks.
- Event- or condition-triggered schedules and scheduled reports.
- Explicit timezone handling, daylight-saving transitions, next-run calculation, missed-run policy, overlap rules, bounded retries and duplicate prevention.
- Editing, pausing, resuming and deactivating a schedule while retaining its execution history.
- Enforce the deployment-wide active-schedule limit and minimum recurring interval configured through the environment variables in §7.3. These limits are shared scheduler policy, not product-specific agent instructions.
- Count active schedule definitions across the shared deployment while enforcing per-tenant quotas and fair scheduling. Paused, deactivated, cancelled and completed one-time schedules do not count. Schedule runs/tasks are not additional schedules.
- Check the active-schedule limit atomically when creating, activating or resuming a schedule so concurrent requests cannot exceed the configured limit. Reject a request that would exceed the limit with a clear explanation; do not silently discard or replace another schedule.
- Apply the minimum interval to time-based recurring schedules, including recurring reminders, when they are created, edited or resumed. Validate the recurrence cadence, not the lead time of a one-time reminder. Event- or condition-triggered schedules have no fixed recurrence interval and are governed by the active-schedule limit instead.

Persist schedule definitions and each schedule run. A due schedule creates a durable task/run for a tenant-registered handler; do not rely on an in-memory timer, an active agent session or a connected client as the source of truth. Recheck authorization and relevant business state when the handler executes. The scheduler owns timing, recurrence and deployment-wide schedule limits; the tenant handler owns the actual action and its domain rules.

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

The common interfaces must support creating, reading, updating, pausing/deactivating and listing schedules; querying schedule runs; creating, inspecting, cancelling and resuming eligible tasks; recording checkpoints and worker leases; recording/replaying event cursors; and opening, listing/inspecting, assigning/routing, responding to, clarifying, resuming, resolving, closing and cancelling interactions. Link interaction records to their originating conversations/messages/participants and dependent task IDs. Persist schedule definitions/runs, task/run state, event cursors/deduplication keys, interaction/request records, approved context-projection references, participant/responder references, lifecycle state and event/delivery references in shared PostgreSQL with tenant ownership keys. These are functional data/API requirements, not a fixed schema or exhaustive endpoint list.


## 6. Search the complete persisted conversation history

The shared chatbot application must expose an agent tool named **`search_conversation_history`**.

Its purpose is to let the agent recover an earlier detail when the current model context no longer contains it because of context-window limits, summarization, compaction or a long-running conversation.

The tool must:

- Search the full persisted message history of the current canonical conversation, beginning with its first stored message, not just the currently loaded prompt, recent-message cache or compaction summary.
- Search stored text across the conversation's user messages, agent messages and other persisted textual conversation entries.
- Support ordinary text/term and phrase queries, and return relevant matching message content or excerpts.
- Return stable message IDs, timestamps, speaker/author and enough conversation/thread metadata to identify the source. Preserve the original message so the agent can inspect the exact wording rather than relying on a generated summary.
- Be used when a user refers to an older statement, decision, instruction, preference or detail that is not clearly present in the active context. The agent should search before asking the user to repeat something that may already be in the stored conversation.
- Return the exact stored text for each selected result, or allow the agent to retrieve the full message by its returned message ID when an excerpt is truncated.
- Support bounded result counts and pagination or subsequent retrieval so a large conversation does not produce an oversized tool response.
- Be available to all tenant-scoped agent sessions running on the shared Vercel EVE runtime.

A compaction summary is an optimization for model context, not a replacement for the original messages. Do not delete or overwrite the source history merely because a summary has been created. The search tool must still find a phrase or fact from before compaction.

By default the tool searches **only the current conversation**. Searching another conversation requires an explicit reference and a separate authorization check. It must never search another tenant's, provider's, owner's or user's messages merely because they share the same database.

Implement this as indexed text search over the persisted database (PostgreSQL full-text search is the natural implementation for the current shared database choice). Semantic/embedding search is not required for this capability. Search results must be grounded in stored messages; the agent must not invent a historical quote or claim a match when none exists.

## 7. Common technical and deployment foundation

The SaaS runs a shared application, EVE runtime pool, gateway and PostgreSQL persistence layer for many provider tenants. Tenant isolation is enforced in application services and repositories. The only technology differences belong to product-specific capabilities that the agent operates or integrates with; they do not create separate chatbot, runtime or deployment foundations.

| Area | Common technical baseline | Product-specific boundary |
|---|---|---|
| **Agent runtime** | **Vercel EVE**, integrated through the shared TypeScript platform. All provider tenants use the same shared runtime integration and lifecycle model. | Agent identity/instructions, tools, capability access, policies and workflows are product-specific. |
| **Application language** | TypeScript for the chatbot application, EVE integration, gateway and shared services. | External product services can use any language and expose documented APIs; they do not have to import chatbot code. |
| **Web client** | Preferred frontend options are **DaisyUI**, **Cloudflare Vinext** and **Vite+**. Only if Vinext gets uneasy we'll switch to Next.JS is also supported. Use TypeScript and the Chat SDK Web adapter client integration (React, Vue or Svelte as appropriate), registered through EVE's `chatSdkChannel`. Browser chat turns and response streaming use the same EVE Chat SDK channel and application gateway as messaging platforms; canonical history and non-chat resources remain in product-owned services. | Roles, branding, page configuration and product actions. |
| **Persistence** | PostgreSQL with Drizzle ORM and the CloudNativePG operator. Use shared PostgreSQL by default with tenant-scoped records, strict row/service authorization and least-privilege credentials. Dedicated databases or additional CloudNativePG clusters are optional stronger-isolation profiles. Use persistent volumes from the configured StorageClass and configure independent base/WAL backups and tested recovery; a PVC alone is not a backup. | Tenant records remain logically isolated in shared PostgreSQL. Dedicated databases may be selected for contractual or regulatory isolation. |
| **Application-managed messaging adapters** | The chatbot application directly installs and registers the prioritized Discord and WhatsApp Business adapters, and includes `@chat-adapter/web` as its core browser-chat integration, behind a common adapter interface. | Additional compatible official, vendor or third-party adapters can be installed and registered in the application. Dashboard users configure available adapters and credentials; they do not install npm code. |
| **Unified bot gateway** | Use EVE's `chatSdkChannel` integration with application-registered Chat SDK adapters as the required message ingress/egress path, including browser chat through `@chat-adapter/web`. Keep one shared application gateway contract; instantiate or isolate adapter configurations as needed for independently credentialed bot connections rather than assuming one static adapter configuration can serve all tenants. | Provider-owned identities and credentials are stored per bot connection; authorization and delivery policies are tenant- and bot-scoped. Direct platform APIs are reserved for separate administrative operations not exposed by the adapter. |
| **Application deployment** | Deploy all application workloads into one configured Kubernetes cluster per environment. This applies to local development, CI, self-managed VPS/local-machine and managed-cloud profiles. | Environment-specific overlays supply storage, identity, networking, sizing and secret references. |
| **Cluster provisioning and bootstrap** | Use an existing Kubernetes cluster or provision one through the selected environment's infrastructure workflow. Provisioning and access to the cluster control plane precede the initial GitOps bootstrap; the provisioning tool is not prescribed by the application architecture. | Managed-cloud and self-managed environments supply their own cluster lifecycle, identity, networking and access configuration. |
| **Desired state and reconciliation** | Flux CD reconciles Git-hosted Kustomize resources and Helm releases to install, upgrade and manage Kubernetes operators (including CloudNativePG), application workloads and supporting configuration. Apply operator custom resources only after their CRDs and controllers are ready; each operator reconciles the resources it owns. | Product-specific manifests and capability resources use the same GitOps workflow. |
| **Local/CI cluster parity** | Use a local Kubernetes test cluster for development and workload integration tests. Deploy the application and CloudNativePG using the same portable Kubernetes resource contracts used for self-managed and managed production clusters. | Product test data, capabilities and validation cases remain separate. |
| **Kubernetes distribution stack** | K3s for local testing and self-managed VPS/local-machine clusters; AKS, EKS or GKE for managed cloud clusters, with other conformant distributions supported after validation. | Validate required Kubernetes APIs, storage, networking and workload capabilities before supporting another distribution. |
| **Autoscaling** | Configure Kubernetes Horizontal Pod Autoscalers for eligible stateless APIs/workers with metrics, resource requests and explicit minimum/maximum replicas.  | Do not assume autoscaling is enabled by default. PostgreSQL scaling and replication use database-aware policies, not generic HPA. |




### 7.1 EVE Chat SDK channel and adapter responsibilities

EVE provides the `chatSdkChannel` integration from `eve/channels/chat-sdk`. This is the required bridge between the Vercel Chat SDK and the shared EVE agent runtime. The Vercel Chat SDK supplies the common TypeScript `Chat` API and adapter contract; EVE's channel integration connects registered adapter handlers to EVE sessions. Use the EVE integration rather than wiring a separate direct platform messaging implementation alongside it. The Chat SDK is a transport/adapter layer, not the chatbot's database, canonical conversation store, domain backend or agent runtime.

**Application-managed adapter scope:**

- The chatbot application registers `@chat-adapter/discord` and `@chat-adapter/whatsapp` as its first external messaging integrations, alongside the core `@chat-adapter/web` integration. It may install other compatible official, vendor or third-party Chat SDK adapters behind the same common interface. Provider users can configure only adapters made available in the application, not dynamically install npm packages.
- Keep contract tests independent of live credentials and test platform authentication, delivery, credential rotation, reconnection, bot/tenant isolation and unsupported capabilities before enabling each adapter.

- The chatbot application registers its selected messaging-platform adapters with EVE's `chatSdkChannel`; the shared browser application registers `@chat-adapter/web`. Call the channel's `send` operation from the Chat SDK handler with the originating thread so EVE receives the accepted turn and can return the response through that adapter.
- The adapter is responsible for platform-specific authentication, webhook/signature verification, event receipt/normalization and message delivery. EVE's channel is responsible for handing accepted turns to the EVE runtime and providing the EVE channel behavior supported by the integration. The application gateway remains responsible for trusted tenant/bot resolution, authorization, canonical persistence, idempotency, policy and durable-delivery coordination.
- Every supported platform/browser chat message must enter and leave through this EVE Chat SDK integration and its registered adapter. Do not implement a parallel direct platform API send path, webhook/event-consumer path or separate chat-turn/streaming endpoint for functionality already supplied by the adapter. The same rule applies to custom adapters: implement the Chat SDK adapter contract and register them through `chatSdkChannel`.
- Resolve the adapter instance and credential reference from the trusted tenant, logical bot and provider-connection mapping. Configure `getUser` (where supported, including the Web adapter) as a mandatory authentication boundary: resolve the caller from a verified server-side product session and return no user for unauthenticated requests. Never accept tenant, bot, connection or principal scope from message text, model output or an untrusted client field.
- Validate each adapter's support for multiple independently credentialed identities, dynamic per-connection credential resolution and the required hosting/event lifecycle. Do not assume one static adapter configuration can serve all tenants or multiple logical bots. Where necessary, use isolated adapter instances or a custom Chat SDK adapter that implements the supported contract and is registered through EVE's channel. If the required identity/credential isolation cannot be implemented safely, mark that adapter/connection mode unsupported rather than bypassing the SDK.
- Use direct platform APIs only for separate, narrowly scoped administrative operations that the adapter does not expose, such as parts of OAuth setup, destination discovery, membership changes or permission administration. Keep those operations in a separate authenticated integration service; they must not receive, process or deliver chat messages and must not become an alternate transport path.
- Run each adapter using a process model that matches its event lifecycle. Discord slash commands/button interactions can arrive through webhooks, but regular messages and reactions use a Gateway WebSocket listener. Matrix requires a long-running sync/event process and cannot be implemented as a webhook-only function. Validate reconnection, event delivery, scaling and process lifecycle for each selected adapter.
- Chat SDK adapter state may be used for adapter-specific subscriptions, locks, deduplication or caching. It must not become the canonical product transcript or replace the application's full message store. Configure Web adapter message-history caching deliberately: set `persistMessageHistory: false` when the handler relies on EVE/canonical history and does not need `thread.messages`; otherwise treat the cache only as adapter-local history. The canonical transcript remains in the product PostgreSQL database.
- The Web adapter is the browser's chat-message ingress and response-stream transport, not the whole browser application or a replacement for product services. Its current v1 feature set is deliberately limited: text/Markdown and request-scoped SSE streaming are supported; native file uploads, buttons/cards/modals, thread listing, a client-facing full-history API and multi-tab/disconnected-browser proactive push are not. Do not claim these capabilities are supplied by the adapter.
- Keep canonical conversation listing/history in the product conversation service. Expose only authenticated non-chat resource operations (for example, read-only history queries and signed attachment/artifact transfer) through the host product; do not recreate a parallel chat-turn submission or agent-response streaming API.
- Browser-targeted background messages must be durably stored. They can be displayed after the client reconnects or reloads from canonical history; do not promise real-time browser push while the client is disconnected.
References: [Chat SDK repository and feature overview](https://github.com/vercel/chat), [Chat SDK documentation](https://chat-sdk.dev/docs/getting-started), [Chat SDK Web adapter documentation and v1 feature matrix](https://chat-sdk.dev/adapters/official/web), [EVE Chat SDK channel](https://github.com/vercel/eve/blob/main/docs/channels/chat-sdk.mdx), [Vercel guidance on Chat SDK and EVE](https://vercel.com/kb/guide/chat-sdk-and-eve), [Chat SDK adapter catalog](https://chat-sdk.dev/docs/create-chat-sdk).

### 7.2 Persistence ownership

Logical bot records and their configuration, optional system instructions, bot context, conversation records, messages, enabled attachment metadata/references, bot-scoped provider-connection mappings, credential references and outbound delivery records are persisted in shared PostgreSQL and tenant-scoped storage interfaces, using Drizzle ORM. Every record carries an immutable tenant ownership key and bot-owned records also carry an immutable bot ownership key. When `CHATBOT_ATTACHMENTS_ENABLED=false`, no attachment content or durable attachment reference is persisted.

The application owns Drizzle schema definitions, migrations, repository implementations and indexes for shared chat records, and applies them in shared PostgreSQL with tenant-aware keys, constraints and repository guards. Dedicated databases are optional stronger-isolation profiles. Do not allow Chat SDK's short-lived state/cache to be the only copy of a message.

Tenant business data remains owned by the tenant's application or domain services. The EVE runtime's session state is execution context, not proof that a message or business action was persisted successfully. Use CloudNativePG-managed PostgreSQL with persistent volume storage in every environment. Retain the PVC and its backing storage across Pod restarts and normal application redeployments; deleting the cluster or losing the host can still destroy local data. CI environments may intentionally be ephemeral. Production uses the same CloudNativePG model with environment-appropriate persistent storage, independent backups and tested recovery. Do not use SQLite or a PostgreSQL emulator for accepted durable application state.

### 7.3 Environment configuration and secrets

Deployment-wide application/runtime defaults are supplied through environment variables. This includes integration defaults, model/provider settings, feature flags, runtime options, limits, voice settings, attachment-ingestion controls, mention behavior and shared scheduler limits. Provider- and bot-specific settings are managed through the authenticated dashboard and persisted as tenant-/bot-owned application data with authorization; they are not deployment environment variables. This includes bot-scoped API capability references, secret-vault entries and connection credential references, linked destinations, bot configuration, optional system instructions and persistent context. Provider-handoff availability is also bot-specific runtime state: derive it from the current logical bot's persisted connections, linked destinations, authorization, effective permissions and adapter availability. Do not add a deployment-wide environment variable such as `CHATBOT_PROVIDER_CHANNELS_ENABLED` to represent whether an individual bot has a provider channel; a global operational kill switch, if ever required, is separate from per-bot availability checks and cannot replace them. Do not use conversational commands or an unprotected settings UI as an alternative source for deployment configuration. The shared deployment receives global defaults through its environment; do not bake deployment-specific values into container images or source code.

**Per-bot secret vault:** Each logical bot has a dashboard section for secret or environment-style key-value entries. Accept multiple newline-separated `KEY=VALUE` assignments and store them as individually named bot-scoped secrets. Authorized bot administrators can create, update/rotate and delete entries. Saved values are masked rather than displayed again. These are bot settings, not deployment-wide environment variables. Bot instructions may refer to secret names when describing external API capabilities; secret values are not inserted into the instructions.

The following named common settings must be explicitly configured for the shared deployment:

| Environment variable | Required value | Purpose and enforcement |
|---|---|---|
| `CHATBOT_ATTACHMENTS_ENABLED` | `true` or `false` | Controls whether attachment bytes may enter backend storage and agent processing. When `false`, attachment bytes are not fetched, uploaded, proxied, persisted or passed to the agent. |
| `CHATBOT_CONTEXT` | Required, non-secret text | Deployment-provided additional system context. Load at application startup and append it to the agent's existing system-role context, preserving existing system instructions; do not replace or overwrite them. Send the added content with role `system`, not as user content, a developer-role message or ordinary retrieved context. |
| `CHATBOT_REQUIRE_MENTION` | `true` or `false` | Controls whether a supported mention/reply is required for the bot to respond in group conversations. Direct messages do not require a mention; platform permissions and privacy rules still apply. |
| `CHATBOT_MAX_ACTIVE_SCHEDULES` | Positive integer | Maximum number of active schedule definitions across the shared deployment; tenant-level quotas/fairness are enforced separately. Includes one-time schedules, recurring schedules and reminders; excludes paused, deactivated, cancelled and completed one-time schedules. |
| `CHATBOT_MIN_SCHEDULE_INTERVAL_SECONDS` | Non-negative integer | Minimum recurrence interval for time-based recurring schedules and reminders. For example, `86400` imposes a 24-hour minimum, preventing hourly or every-four-hour recurrence. Set to `0` to disable this minimum-interval restriction. It does not restrict the lead time of a one-time schedule/reminder or the event cadence of event-/condition-triggered schedules. |
| `CHATBOT_AI_CONFIG` | Required JSON object | Configures the AI-compatible API endpoint in one object. `base_url` is required; `model` is optional and is supplied only when the endpoint requires an explicit model name; `api_key` is optional and is supplied only when the endpoint requires API-key authentication. No provider selector or provider-specific default endpoint is assumed. |

Set **CHATBOT_CONTEXT** as a shared deployment default. Tenant- and bot-specific additions are stored as access-controlled application data and composed with the base context without replacing it. A bot's optional provider-authored system instructions are stored with that bot and included in the system-role instruction stack before ordinary conversation/user messages; they must not override higher-priority application safety/security instructions. Preserve all existing system instructions and conversation history. `CHATBOT_CONTEXT` remains non-secret deployment configuration supplied through a ConfigMap or equivalent environment configuration; do not put credentials or other secrets in it. Updating this deployment default takes effect after a workload restart or rollout; bot-level changes follow the application's persisted configuration update path.

Enable and configure common scheduler and periodic-worker functionality through deployment environment variables (for example, ConfigMaps for non-sensitive controls). These are common platform features; each tenant supplies its authorized handlers and work to execute. Keep feature controls and operational settings in deployment configuration, not in product agent instructions.

The active-schedule cap is enforced atomically on schedule creation, activation and resumption. When the cap is reached, requests that would add another active schedule are rejected with a clear explanation; existing schedules are not silently removed. The minimum recurrence interval is checked when a time-based recurring schedule is created, changed or resumed. The recurrence minimum is a shared deployment setting. The active-schedule value is a deployment safety cap; enforce tenant quotas and fair scheduling separately.

Application workloads receive environment variables through Kubernetes resources: use ConfigMaps for non-sensitive values and Secrets for sensitive values. The attachment, mention and scheduler-limit settings are non-sensitive and should be supplied through ConfigMaps. Supply `CHATBOT_AI_CONFIG` through a Kubernetes ConfigMap when its object does not contain an API key; when `api_key` is present, supply the entire JSON object through a Kubernetes Secret so the credential is not exposed. Omit `api_key` for endpoints that do not require API-key authentication.

- Use **ConfigMaps** for non-sensitive settings such as log level, feature flags, public service endpoints, allowed origins and non-secret runtime options. Consume individual keys through `env[].valueFrom.configMapKeyRef`, or a controlled set through `envFrom.configMapRef`.
- Use **Kubernetes Secrets** for credentials and sensitive values such as database passwords, signing keys, OAuth/client secrets, channel tokens, webhook secrets and provider API keys. Reference only the required keys through `secretKeyRef` or a narrowly scoped `envFrom.secretRef`; mount files instead when a library expects certificate/key files or when file-based rotation is preferable.
- Never commit plaintext Secret manifests, credentials, connection strings containing passwords, or private keys. Use Flux CD with SOPS-encrypted Secret manifests (or a separately selected external secret manager integration) so Git stores encrypted values and Flux decrypts them only during reconciliation. Keep decryption keys outside the Git repository and protect them independently.
- Kubernetes Secret values are base64-encoded, not automatically encrypted at rest. Enable cluster secret encryption at rest where supported, restrict Secret read permissions with namespace-scoped RBAC, restrict Pod creation/exec privileges, and avoid broad service-account access. Do not log or expose secret values to the browser, agent/model context, task transcripts or diagnostics.
- Use shared platform namespaces and service accounts with least privilege. Isolate tenants through scoped application authorization and per-connection credential references; per-tenant namespaces/deployments are optional stronger-isolation profiles.
- Prefer Kubernetes Service DNS names for in-cluster dependencies. `localhost` inside an application Pod refers to that same Pod, not PostgreSQL or another service. Use the CloudNativePG read/write Service name for database connections; use port-forwarding only for deliberate host-side development access.
- Environment variables sourced from ConfigMaps or Secrets are fixed when a container starts. A change to their values requires a workload rollout/restart to update the process environment. Use mounted files or a dedicated reload mechanism when a workload must observe rotations without restart; do not assume env vars update live.

### 7.4 Security and operational requirements

- Resolve tenant scope, logical bot, provider connection and sender identity from trusted server-side authentication and verified platform events. Never accept tenant or bot identity from model output, browser input or an untrusted external identifier.
- Enforce tenant and bot isolation in every repository query, dashboard operation, service operation, cache key, object-storage path, EVE session mapping, task, schedule, event cursor, attachment, search result and outbound dispatch. Use tenant-/bot-aware keys and constraints and deny access when either scope is absent or ambiguous.
- Recheck authorization for inbound events, agent tool calls, callbacks, background work, attachment retrieval, owner-initiated sends and workflow notifications. A valid platform credential is not by itself authorization to access another tenant's data.
- Store each bot connection's credentials separately using encrypted secret storage or a secrets manager. The dashboard may accept credentials and display non-secret connection status/metadata, but must never return stored secret values after submission. Never expose tokens, webhook secrets, database credentials or storage credentials to the model context, task transcripts or logs.
- A provider may edit credentials, replace a bot's platform identity or link/unlink destinations without losing that bot's instructions, configuration, history or context. Reauthorize replacement identities and establish or verify platform-side memberships/permissions again; do not assume platform conversation IDs or memberships transfer.
- Deduplicate source events and handle retries/out-of-order delivery. Persist accepted messages and outbound intents before acknowledging durable receipt; recover pending delivery after restarts without repeating business effects.
- Keep logs and diagnostics useful but redact credentials, private attachment content and unnecessary personal information.
- Record effective capabilities and connection status so unsupported or revoked operations fail explicitly rather than appearing successful.
- Verify CHATBOT_AI_CONFIG parsing and keep any API key in a Kubernetes Secret, never in model context, browser state or logs.
- Enforce both per-tenant scheduler quotas/fairness and the deployment-wide active-schedule safety cap atomically, including under concurrent requests.
- Test cross-tenant and cross-bot isolation for conversations, history search, attachments, runtime sessions, tasks, schedules, event replay, credentials and outbound delivery.
- Keep internal application modules and the gateway maintainable and replaceable. Do not let platform SDK identifiers or EVE runtime session IDs become the public tenant/conversation contract.

### 7.5 Common deployment and parity workflow

This section defines how clusters are provisioned or selected, bootstrapped and workloads are configured, reconciled and validated across local development, CI, self-managed infrastructure and managed cloud. All profiles use the same Kubernetes resource contracts, CloudNativePG operator, Kustomize bases and Flux reconciliation; environment overlays supply storage classes, replica counts, resource budgets, endpoints and credentials. The shared chatbot is distribution-agnostic: deployment tooling uses the standard Kubernetes API, while the application runtime does not create clusters, administer nodes or implement a multi-cluster control plane.

Cluster provisioning, initial GitOps bootstrap and steady-state workload reconciliation are separate stages:

- **Local development and CI:** create or use a local Kubernetes test cluster and apply the same Kubernetes resource contracts used for production. The test cluster is a deployment profile, not a separate application architecture.
- **Self-managed VPS or local-machine deployment:** the operator checks host capacity and prerequisites, configures secrets, hostname, networking and storage, then bootstraps a Kubernetes cluster on the host and installs the initial GitOps components. This uses the same Kubernetes API and workload model as a cloud-managed cluster. A single-node host is a single failure domain, not a highly available cluster; production-grade use requires appropriate topology, durable storage, off-host backups, tested recovery, monitoring, security and upgrade/maintenance procedures.
- **Managed cloud deployment:** AKS, EKS and GKE are supported target profiles after their required Kubernetes APIs, storage, networking, identity and workload capabilities pass validation.
- **Cluster provisioning:** the cluster may be pre-existing or created through provider-native tooling or another separately selected infrastructure workflow. This document does not prescribe a cluster-provisioning product; cluster availability and authorized access are prerequisites for GitOps bootstrap.
- **GitOps bootstrap:** once the cluster is available, install and configure Flux through a documented initial bootstrap workflow, run by CI/CD or an operator. Flux cannot install itself into a cluster that does not yet exist.
- **Workload configuration:** after bootstrap, Flux reconciles Kustomize bases/overlays and Helm releases for namespaces, policies, CloudNativePG, application services, workers, storage configuration, ingress and monitoring. Apply an operator's custom resources only after its CRDs and controllers are ready; operators reconcile their own resources. Avoid imperative workload deployment paths outside the documented initial bootstrap.

**Local development and CI**

1. Provide one documented command to start the complete local development environment, including the local Kubernetes test cluster, initial Flux bootstrap, required supporting services, CloudNativePG, the application and the shared browser client. It must wait for readiness and leave the environment usable without requiring a sequence of manual startup commands.
2. Use Flux CD and the local Kustomize overlay to install the CloudNativePG operator and deploy the application and supporting services. Create the PostgreSQL Cluster resource through GitOps only after the operator's CRDs and controllers are ready, and use a persistent volume from the cluster StorageClass. Retain local backing storage across normal restarts and redeployments; CI may destroy its disposable environment after tests.
3. Build prebuilt immutable images, deploy through the same Kustomize/Flux manifests and application configuration contracts, and promote the same image digests through subsequent validation.
4. Test application, database, gateway, frontend, scheduler, task recovery, storage and product capabilities. Include Pod restart/rescheduling, PostgreSQL restart and recovery, persistence across workload restart, upgrade/rollback, event replay, task cancellation and end-to-end checks. Test local persistence separately from backup/restore and machine-loss recovery.

**One-machine VPS deployment**

1. Use the same immutable application images, Kustomize manifests and Flux CD reconciliation flow validated locally and in CI. Bootstrap a self-managed Kubernetes cluster on the VPS or local machine, then perform the initial Flux bootstrap against the environment's Git configuration. After bootstrap, Flux reconciles the application, PostgreSQL operator and supporting services as workloads inside that cluster within the host's resource budget.
2. Deploy CloudNativePG-managed PostgreSQL with persistent volumes from the cluster's configured StorageClass. Configure backups outside the primary database volume and test recovery. Do not substitute SQLite or a separate external PostgreSQL service for the standard deployment.

A successful unit-test run is not evidence of deployment parity. Validate workload deployment, persistent-data behavior and recovery using the same cluster resource contracts used for production.


## 8. Illustrative organizational use case

A community, company or project organization can use the shared chatbot as a common front door for members, contributors and maintainers. The organization supplies its own membership rules, knowledge, domain actions and meeting policies; the shared platform supplies channel normalization, identity mapping, canonical conversations, linked interactions, durable scheduling and outbound delivery.



### Example flow

1. An organization administrator connects the bot to the authorized workspace, configures enabled channels and eligible roles, and chooses whether group responses require a mention. Platform identity and channel permissions remain authoritative.
2. Maintainers provide organization-scoped instructions, policies, project information, event details and meeting availability through the organization's application tools. The agent receives only information applicable to the active organization and conversation.
3. A contributor asks a question in an enabled channel, thread or direct message. The shared gateway resolves the sender and channel, then routes the message to the correct canonical conversation. The agent answers what it can establish from authorized context.
4. For a missing or ambiguous detail, the agent first answers any part supported by the organization's authorized knowledge and context. If the application confirms that an authorized, usable provider-side destination is configured for that bot and is eligible for the issue, it records a durable interaction and routes only the necessary approved context. The agent says the request was forwarded only after dispatch has been accepted; it does not claim that a provider has read or answered it without evidence. If no eligible destination is available, the agent explains that it cannot verify the remaining detail from current sources and has not contacted the provider; it does not imply escalation or promise a follow-up. When a provider response is received, the agent evaluates it and returns only information authorized for the originating conversation. Private member context is not exposed to other members.
5. A member can ask to arrange a contributor/mentor meeting. The organization's service determines participants and availability; the shared scheduler persists confirmations and reminders in the configured organization timezone. Other questions can continue while a response or meeting detail is pending.
6. Organization-wide announcements or onboarding messages require an authorized administrator's direction and an explicitly eligible audience. The shared outbound mechanism enforces channel permissions, delivery policy and opt-in requirements; it does not independently decide whom the organization should contact.

This is an illustrative shared-platform use case, not a built-in organization-management product. Membership, role meaning, knowledge scope, meeting rules, audience selection and domain authorization remain organization/product responsibilities.

## 9. Common versus product-specific boundary

The shared platform defines reusable technical contracts. Each tenant remains responsible for its own domain behavior and authorization:

- **Bots, identity and integrations:** the shared application provides logical bot records, dashboard management and its own extensible Chat SDK adapter registry. The chatbot application installs and registers platform adapters, prioritizing Discord and WhatsApp Business initially while retaining browser chat as a core capability. Each bot owns its configuration, optional system instructions, context, history and platform connections. The connection registry manages bot-scoped credential references, capabilities and mappings; providers create and own their official platform identities and retain platform-side authority over memberships/permissions.
- **Conversation history:** the shared layer owns canonical message persistence, context isolation and history retrieval in shared PostgreSQL. Each tenant defines the authorized conversation scope available to its agent.
- **Outbound actions:** the shared outbox handles durable delivery, retries and idempotency. An authorized owner can explicitly initiate a send through a selected connected identity. Tenant workflows may send eligible service notifications under tenant-defined authorization; autonomous cold outreach is not a platform capability.
- **Attachments:** the shared layer receives, persists/references and authorizes retrieval. Each tenant decides how its agent processes supported files.
- **Authorization:** the shared layer provides identity and policy interfaces; each tenant supplies its principals, roles, tenant scope and authorization decisions.
- **Agent and client configuration:** the shared EVE integration and browser client are common. Bot-level settings, optional system instructions, external API references and secret-vault entries are managed per logical bot; tools, actions, approval semantics, views and branding remain tenant-/product-owned.
- **Scheduling and tasks:** the shared scheduler, task lifecycle and linked-interaction primitives are common. Each tenant supplies domain handlers, business records and rules for human-dependent work.
- **Infrastructure:** shared PostgreSQL and shared runtime infrastructure are the default. Dedicated databases, namespaces or deployments are optional isolation choices, not per-tenant defaults.

The shared chatbot application must not own tenant business records, business workflows, approval policies, domain-specific tools or tenant-specific deployment topology.

### Optional library packaging

Keep stable internal module/interface boundaries. Common components may later be extracted into reusable libraries/packages if useful, but packaging them separately is not required for the initial application implementation or for external API-based product capabilities.

## 10. Shared acceptance checks

The shared implementation is complete only when these checks pass:

- A user can start and continue a direct conversation using ordinary text without a slash command.
- Each provider can create and manage multiple logical bots in the SaaS dashboard; the application never provisions or owns the external platform identities.
- Each logical bot has independently managed configuration, optional system instructions, API capability references, a bot-scoped secret vault, persistent context, conversation history and one or more provider-owned platform connections.
- The provider dashboard can add/edit/rotate/revoke credentials and link/unlink supported channels, groups and destinations. Platform-side actions that cannot be performed through an API are guided and verified rather than falsely represented as complete.
- The dashboard can create, replace and delete bot-scoped secret entries from newline-separated `KEY=VALUE` input, with masked saved values and isolation between bots.
- Bot instructions can refer to documented external APIs; agents can use available code or network capabilities to make API requests without a prescribed programming language, custom SDK or specific HTTP tool.
- Multiple provider tenants can use the shared EVE runtime pool and shared PostgreSQL database without cross-tenant access; multiple bots within one tenant cannot access one another's bot-scoped conversations, history, credentials, context, attachments, tasks, schedules or outbound actions.
- A logical bot can have multiple provider-owned platform connections; each connection has distinct credentials, permissions, linked destinations and capability metadata.
- Editing, rotating or replacing credentials or an external identity preserves the same bot's instructions, configuration, context, history, tasks and data. Replacement requires fresh authorization and platform-side membership/permissions; external memberships and conversation identifiers are not assumed to transfer.
- Linking or unlinking a channel/group changes only the selected bot's destination configuration and does not reset bot context or conversation history.
- External sender IDs are scoped to their provider connection and are never treated as globally unique human identities.
- A supported platform identity maps multiple DMs/groups/threads to distinct canonical conversations without cross-conversation or cross-tenant context leakage.
- When a question can be answered from authorized bot/tenant knowledge or safe general knowledge, the agent answers directly rather than escalating by default; it distinguishes general information from verified provider-specific facts.
- When the agent cannot verify an answer and the current bot has no enabled, authorized, usable provider connection and eligible destination, it says it cannot verify the answer and has no available provider contact route. It must not claim that it contacted, forwarded to or escalated to a provider, is waiting for a provider, or will follow up.
- Handoff availability is derived and revalidated from persisted tenant-/bot-scoped connection, destination, authorization, permission and adapter state; it is not represented by a deployment-wide environment variable or trusted solely because a bot prompt says a provider is available.
- The agent claims a request was forwarded only after a durable interaction has been recorded and dispatch is accepted by the selected adapter. It distinguishes queued/accepted delivery from confirmed delivery, provider receipt or provider response, and promises follow-up only when a durable follow-up workflow is actually established.
- Provider handoff works through an authorized group/channel when supported and configured, or through an eligible one-to-one conversation with the provider/staff member when group delivery is unsupported or unavailable. The receiving destination must be verified for the selected bot and connection; absence of an eligible destination must not be reported as a successful handoff.
- When a customer asks a question outside the available context, the agent uses only authorized context and does not invent a recipient or handoff.
- With CHATBOT_REQUIRE_MENTION=true, the bot ignores unaddressed group messages and responds to supported mentions/replies; with false, it can respond to ordinary group messages without an @mention, subject to platform permissions. Direct messages work without a mention in either mode.
- Every supported browser and platform chat ingress/egress path uses EVE's `chatSdkChannel` integration and its registered Chat SDK adapter; no direct-platform message-send, webhook/event-processing or parallel chat-turn path exists for adapter-supported functionality.
- The chatbot application registers Discord and WhatsApp Business as its initial external messaging integrations and `@chat-adapter/web` as a core browser-chat integration; adapter contract tests can use a fake adapter while integration tests verify real adapter lifecycle and tenant/bot isolation.
- The chatbot application can register another compatible official or third-party Chat SDK adapter without changing its common messaging interfaces. Dashboard users configure only application-registered adapters; entering an npm package name does not trigger installation or execution in the shared SaaS.
- User-facing interaction uses portable plain-text messages across all supported platforms. Do not depend on or emit platform-specific interactive buttons, menus, cards, modals, callback controls, slash-command UI, rich-link controls or other platform-specific response elements; users must be able to understand and complete supported interactions through ordinary text messages. Do not rely on clickable links or platform-specific rendering as an interaction mechanism; when a URL is genuinely necessary, include it as ordinary text and keep the interaction usable without link-specific UI.
- The adapter/connection registry selects the correct tenant- and bot-scoped adapter configuration and credentials, including when multiple bots use the same platform; unsupported dynamic-credential configurations are rejected or handled by a verified custom Chat SDK adapter, never by a direct messaging bypass.
- A web conversation enters through the authenticated Web adapter registered with EVE's `chatSdkChannel`, maps to an isolated canonical conversation/EVE session and cannot supply its own tenant identity.
- All tenants use the same shared EVE runtime integration, unified gateway, web client and tenant-scoped PostgreSQL/Drizzle persistence contracts.
- Every deployment profile runs the complete application stack inside one target Kubernetes cluster; no frontend, backend, EVE runtime, Chat SDK adapter/gateway, worker, scheduler or application service is hosted outside that cluster.
- After a cluster is provisioned or selected and Flux is bootstrapped, Flux reconciles Kubernetes operator installations and upgrades, their dependent custom resources, and application workloads from Git; operator custom resources are applied only after the required CRDs and controllers are ready.
- PostgreSQL runs inside the same target Kubernetes cluster and is managed by CloudNativePG in every environment; external managed PostgreSQL and SQLite are not supported.
- Docker Compose, standalone host/container services, serverless application components and split frontend/backend/database hosting are not supported deployment alternatives.
- An authorized owner can explicitly initiate a text message through a selected connected provider identity without an active inbound turn, provided the destination is eligible and platform rules/consent requirements are met. The platform does not autonomously select recipients or perform cold outreach.
- One-time, recurring, interval and event-triggered schedules produce durable runs in the configured timezone and remain due/recoverable while the agent or client is offline.
- Accepted asynchronous tasks survive client disconnect and process restart; cancellation is recorded and stale workers are fenced from committing.
- Replayed or duplicate source events resume from a durable cursor and do not repeat an accepted business effect.
- A transient delivery failure or duplicate event does not lose a message or repeat its associated business action.
- search_conversation_history finds an exact older message from before context compaction and returns its source reference and timestamp.
- The history tool cannot retrieve messages from an unrelated conversation or tenant scope.
- A restart does not erase canonical messages, enabled attachment metadata/references or pending outbound delivery.
- A platform-specific unsupported capability is rejected or reported explicitly, not silently treated as successful.

