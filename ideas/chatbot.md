# Chatbot — Shared Conversation Platform

## Architecture overview

The default architecture is a **shared multi-tenant SaaS platform**. Each provider creates and owns its official application/bot identity directly on the relevant platform, authorizes this application to use that identity, and adds it to the provider-controlled destinations it wants connected. There is no internal bot account and no separate provider-group onboarding dashboard. The platform stores provider connections, credentials, configuration, canonical conversations, history and context under strict tenant boundaries.

The shared TypeScript library, Vercel EVE runtime integration and PostgreSQL persistence serve many provider tenants. Tenant identity is resolved from trusted authentication and verified connection metadata. Every read, write, session, tool, task, file and outbound action is tenant-scoped. A provider may add or replace a platform identity without losing application-owned context or history. A replacement identity must be authorized separately and its platform-side memberships/permissions established again; external memberships and conversation identifiers do not automatically transfer.

Shared PostgreSQL and shared runtime infrastructure are the default. Dedicated databases, namespaces or Kubernetes deployments are optional stronger-isolation profiles, not the default SaaS architecture.

~~~mermaid
flowchart TB
  subgraph Providers["Provider-owned platform identities"]
    IdentityA["Provider A official application identity"]
    IdentityB["Provider B official application identity"]
    IdentityN["Additional provider identities"]
  end
  subgraph SaaS["Shared multi-tenant chatbot SaaS"]
    Auth["Tenant authentication and connection authorization"]
    Registry["Provider integration / credential registry"]
    Gateway["Unified gateway and adapters"]
    Conversation["Canonical conversations and context"]
    Runtime["Shared EVE runtime pool<br/>tenant-scoped agent sessions"]
    Scheduler["Durable scheduler and task workers"]
    Outbox["Authorized outbound dispatcher / outbox"]
    Tools["Common agent tools"]
    DB[("Shared PostgreSQL<br/>tenant-scoped records")]
    Blob[("Tenant-scoped attachment storage")]
    Auth --> Registry
    Registry --> Gateway
    Gateway --> Conversation
    Conversation <--> Runtime
    Runtime <--> Tools
    Runtime --> Outbox
    Scheduler <--> Runtime
    Gateway <--> DB
    Conversation <--> DB
    Runtime <--> DB
    Scheduler <--> DB
    Outbox <--> DB
    Conversation <--> Blob
  end
  IdentityA <-->|"authorized events and sends"| Gateway
  IdentityB <-->|"authorized events and sends"| Gateway
  IdentityN <-->|"authorized events and sends"| Gateway
~~~

**Normative separation rule:** common requirements are specified here once. Provider-specific behavior and authorization are supplied through tenant-scoped extension points; they must not create competing shared implementations or weaken the common isolation contract.

## 1. Architecture: shared SaaS with reusable library boundaries

Implement the common chatbot layer as a **private, reusable TypeScript library/package** used by the shared SaaS application. The package is the reusable implementation unit; it is not a separately operated service per provider. The default deployment is one shared multi-tenant application, shared EVE runtime pool and shared PostgreSQL persistence layer. Optional dedicated deployments may be offered for providers requiring stronger contractual or regulatory isolation, but they are not the default.

Each provider tenant supplies its authorization policy, agent configuration, domain capabilities and data scope through explicit tenant-scoped extension points. Dedicated databases, namespaces or deployments remain available when a provider's isolation requirements justify the operational cost.

The chatbot/ directory in this repository contains the specifications. It is not itself the implementation package. The eventual package can be versioned and consumed by the shared SaaS application and optional dedicated deployments.

### 1.1 Package boundaries

The shared package should expose these modules/interfaces:

| Module or interface | Shared responsibility |
|---|---|
| **Provider integration registry** | Maintain tenant-owned connections to provider-created platform identities, authorization state, credential references, capabilities and lifecycle. It does not create an internal bot identity. |
| **Principal and identity mapping** | Map authenticated application principals and external platform identities to a tenant and its authorization policy without treating platform IDs as global identities. |
| **Conversation service and repository** | Create and resolve canonical conversations, associate messages and participants, persist messages, maintain isolated context and provide indexed history retrieval/search. |
| **Shared web application** | Own the reusable browser chat application, including conversation history, transcript/composer, streaming, voice, file attachments and progress/events. |
| **Unified gateway** | Provide common ingress/egress for browser and supported platform connections; authenticate, normalize, route and dispatch tenant-authorized messages. |
| **Vercel EVE runtime integration** | Run tenant-scoped agent sessions on the shared EVE runtime baseline, including lifecycle, tool calling, workflows, delegation and events. |
| **Outbound dispatcher and durable outbox** | Send owner-authorized or tenant-workflow-authorized messages through a connected provider identity; persist delivery intent and retry safely. |
| **Attachment receipt and retrieval** | Enforce attachment-ingestion settings, store tenant-scoped metadata/content references and provide authorized retrieval. |
| **Scheduler service and repository** | Persist schedules and durable executions in shared PostgreSQL with tenant ownership, quotas and fair scheduling. |
| **Task/work service and repository** | Provide tenant-scoped asynchronous task/run identities, lifecycle, dependencies, cancellation, checkpoints, leases, retries and recovery. |
| **Linked interaction service and repository** | Correlate related conversations, questions, deferred work and replies through scoped, authorized interaction records without merging transcripts. |
| **Product action registry** | Register tenant/product-owned handlers for scheduled work, asynchronous tasks and interaction events under tenant authorization. |
| **Common agent tools** | Expose conversation-history search and authorized common task/schedule/interaction operations to the connected agent runtime. |

The common platform standardizes on Vercel EVE for agent execution. The conversation library, scheduler, task lifecycle, linked-interaction mechanism and gateway remain modular TypeScript components around that runtime; none embeds provider-specific business logic.

### 1.2 Tenant and product extension points

Each provider tenant supplies its authenticated principal and authorization resolver, human-role definitions, tenant data scope, agent instructions, capabilities, tools and domain handlers. The common layer maps verified external identities to tenant principals; the tenant defines role meaning and permissions.

Tenant-specific context and configuration are stored as access-controlled tenant data and composed with deployment-wide defaults. Context must be appended without replacing existing system instructions. The tenant supplies usage instructions and input/output contracts for its domain capabilities. Attachment access and processing remain tenant-authorized.

Tenant-owned handlers for schedules, asynchronous tasks and linked interactions are registered through in-process modules/providers or authenticated APIs. The agent can invoke these capabilities when relevant, but context does not grant permission: the owning service must still enforce authentication, data scope and business rules. All chat turns enter through the unified gateway; non-chat resource operations such as listing history, attachment transfer and settings retrieval remain separate authorized application services. No separate API server, universal API or common tool protocol/registry is required for every tenant capability.

### 1.3 Shared EVE runtime and unified gateway

All tenants use the same **shared Vercel EVE runtime baseline and runtime pool**. The platform may host many tenant-scoped agent instances/sessions on that shared runtime; it does not create a separate runtime deployment for every provider. Each request is bound to a tenant using trusted server-side authentication and a verified integration connection. An EVE session ID is never accepted as proof of tenant ownership.

EVE session state is execution context, not the canonical record of conversations or business state. Persist messages, task/business records, schedules, checkpoints and delivery intents in shared PostgreSQL with tenant ownership and enforce authorization in repository/service boundaries. Agent instructions, tools, capabilities, delegation policy and domain workflows remain tenant-owned; the common platform provides EVE's generic runtime primitives.

The **unified gateway** is a shared library module and service interface, not an internal bot account or per-provider gateway. It receives authenticated browser requests and verified platform events, resolves the tenant and authorized provider connection from trusted server-side context, normalizes messages into the common contract, routes turns to tenant-scoped EVE sessions and returns replies through the originating connection.

An authorized owner can explicitly initiate outbound messages through a selected connected provider identity. Any other background delivery must be tied to a tenant-authorized workflow and eligible destination; the platform does not independently select prospects or perform autonomous cold outreach. Persist messages and delivery intents before acknowledging durable receipt.

The gateway and runtime are shared, while provider identities, credentials, connection metadata, conversation data, context and authorization remain tenant-isolated. Do not assume built-in adapters support arbitrary per-tenant credential resolution; validate each adapter and use a custom adapter/connection registry where needed.

## 2. Logical agents, platform identities and conversations

### 2.1 Provider-owned platform identities

Each provider creates and owns its own official application/bot identity directly on the relevant platform. The provider authorizes this application to receive events and send messages through that identity using the platform's supported credential or authorization mechanisms. The application does not create a separate internal bot account and does not own the provider's platform identity.

The provider adds its identity to the groups, rooms, workspaces or other supported destinations it wants connected and grants the necessary permissions directly on the platform. There is no separate provider-group onboarding dashboard or internal bot-management conversation.

The platform stores a tenant-scoped connection record containing the provider, platform, external identity, authorization/credential reference, effective capabilities and lifecycle status. Secrets are held in a secrets manager or encrypted credential store, never in prompts, browser state or ordinary logs. A provider may authorize multiple identities and may revoke, replace or add connections at any time.

Replacing an identity does not delete or reset application-owned context, canonical conversation history, tasks, schedules or configuration. The replacement identity must be separately authorized, and its memberships/permissions must be established on the platform. Do not assume external group membership, platform conversation identifiers or message history transfer to the replacement identity. Retain application-owned history; associate newly received conversations only through explicit, authorized mapping rules.

A platform identity is a transport identity for a tenant's logical agent, not a separate agent runtime. One connection may serve multiple conversations where the platform permits it. Platform membership, access and message visibility remain controlled by the platform's own installation, invite and permission mechanisms.

### 2.2 Provider connection lifecycle and authorization

1. The provider creates its official application/bot identity on the platform.
2. The provider authorizes the SaaS application to use that identity through the platform's supported credential or authorization flow.
3. The provider adds the identity to the destinations it wants connected and grants the required permissions directly on the platform.
4. The application verifies authorization, records a tenant-scoped connection and its effective capabilities, and begins accepting verified events.
5. The provider can revoke authorization, replace the identity or add another identity. The application updates only affected connection records and credential references; tenant-owned history, context and business data remain intact.

There is no internal bot account or ID, management conversation, provider sign-in to an application-owned bot, or separate group-registration dashboard. The platform identity itself is the provider's messaging identity.

The application maps external sender identities to authenticated tenant principals only through trusted identity claims or an explicit tenant-controlled linking process. A display name or matching username is never sufficient proof. Changing a person's external platform identity updates only the relevant identity mapping after re-authentication; it does not change tenant ownership or reset application-owned context.

Each connection has its own credentials, permissions, capability record and external destinations. Connections belonging to one tenant can access only that tenant's conversations and context. A connection must not read or send through another tenant's identity.

An owner or other authorized tenant principal may explicitly initiate a message through a selected connected identity. The application verifies the principal's permission, destination eligibility, connection status and platform rules before queuing delivery. This is owner-directed operation, not autonomous prospecting or cold outreach.

If a provider revokes or replaces a connection, stop accepting events and sending through revoked credentials, mark the connection unavailable, and retain application-owned history. Do not silently rebind old external conversations to a replacement identity.

### 2.3 Canonical conversation mapping

The application creates a stable internal conversation ID and maps it to the external platform context. The mapping must preserve tenant ID, authorized provider-connection ID, platform/channel type, external identity, destination/thread identifiers, external sender/message identifiers scoped to the connection, timestamps and reply/reference metadata.

A direct message, web conversation, group/channel thread or other independent context receives its own canonical conversation mapping. Context is never mixed across unrelated groups, DMs, tenants or web conversations. A web conversation maps the authenticated tenant principal and client conversation identifier to a canonical conversation ID and EVE session. Adapter thread identifiers are routing identifiers, not the public conversation contract.

A runtime session ID is an execution identifier mapped to the canonical conversation ID and tenant. It is not the source of truth for message history or authorization. Cross-conversation retrieval requires an explicit reference and an independent authorization check.

### 2.4 Direct, group and channel interaction

Ordinary natural language is the normal interaction method. Slash commands may be supplied as shortcuts, but users must not need a command to make an ordinary request.

- **Provider-side groups:** connected groups are the normal provider communication surface. A provider-owned identity may participate in multiple destinations the provider has connected; the agent can use only context received through authorized connections. A one-to-one provider DM is not required.
- **Mention-required group mode:** when `CHATBOT_REQUIRE_MENTION=true`, the bot responds in group conversations only to messages that mention/address it (including supported platform-native reply-to-bot behavior). Unaddressed group messages are ignored.
- **Mention-not-required group mode:** when `CHATBOT_REQUIRE_MENTION=false`, the bot may respond to ordinary group messages without an @mention, subject to platform permissions and the conversation's access policy.
- **Replies and threads:** replying to an agent message or another message can continue or refer to that exchange. Preserve the referenced message and thread context.

The deployment environment variable `CHATBOT_REQUIRE_MENTION` controls whether the bot must be mentioned before it responds in a connected group and must be explicitly set to `true` or `false`. Platform privacy settings and permissions still determine which group messages reach the bot; disabling the mention requirement cannot override them. The shared library must not impose a universal mention-only, public-only or private-only restriction.

**Initial messaging targets are Discord, WhatsApp Business and Telegram.** The shared platform supports these three adapters in its initial integration baseline; each tenant configures the subset it uses. Additional adapters can follow the same common contract.

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

The application authenticates the event, resolves tenant and connection scope from trusted server-side context, checks access and deduplicates it before dispatching the turn to the shared Vercel EVE runtime. Before any attachment bytes are fetched from a messaging platform, uploaded by a web client, proxied to the application, persisted or exposed to the agent, the ingress path must enforce the deployment environment variable `CHATBOT_ATTACHMENTS_ENABLED` (`true` or `false`). Each deployment must set this value explicitly. When `false`, do not fetch or download attachment content, accept or proxy web-upload bytes, store attachment content or durable attachment references, or include attachments in the EVE/agent turn; ignore the attachment and continue processing any accompanying text. The client should disable its upload control when possible, but the backend must enforce the setting even if a client or platform still sends attachment metadata or an upload request. A lightweight indication that an attachment was ignored may be shown to the sender, but the attachment itself must not be transferred into backend storage or AI processing. This prevents unnecessary backend storage and attachment-related AI/token usage. When `true`, the normal authorized attachment receipt, storage, retrieval and agent-capability rules apply. Do not trust a model-supplied tenant, customer, owner or conversation identifier.

### 3.2 Agent and application messages

Agent replies are persisted as canonical messages and delivered through the adapter associated with the originating conversation. Preserve message ordering and reply/thread destination where the platform supports it.

The shared web client submits chat turns through the Chat SDK Web adapter and receives the agent response over its AI SDK UI message-stream/SSE connection. Messaging clients receive platform-native messages through their respective adapters. A live browser connection is not required for the service to continue a task or deliver a later message; persist resulting messages and work state independently of the connection. The current Web adapter provides request-scoped streaming, not durable push to disconnected browsers or other open tabs.

The common project owns the standalone shared web application and its reusable chat experience. Products configure its identity, roles, authorization and product actions. All clients that submit chat turns use the same gateway and adapter contract; a product or third party may build a separate client against the documented client and conversation contracts without creating a second chat-message ingress or streaming path.

### 3.3 Proactive outbound text — independent of an incoming user turn

An authorized owner **must be able to explicitly initiate a text message through a selected connected provider identity without an active inbound turn**. This is an owner-directed outbound capability, not permission for autonomous prospecting or cold outreach.

The owner-triggered operation must support an explicitly selected eligible destination and a clear message body. Separately, tenant-authorized workflows may deliver necessary service notifications or continue an already established interaction when configured and eligible. The platform must not independently choose new recipients or initiate unsolicited outreach.

The outbound operation accepts the authenticated owner/tenant principal, selected provider connection, authorized destination, content, cause/reference and idempotency key. It must not require an inbound message object or an open web client.

Use a durable outbound queue/outbox. Persist intent before delivery, record attempts and outcomes, retry transient failures, and deduplicate retries so a message is not sent repeatedly because of a timeout or restart. Keep message delivery separate from the domain action that triggered it; a delivery failure must not undo a completed booking, task or other product operation.

Owner-initiated sending is still subject to platform rules. Some destinations require prior interaction, consent or an eligible messaging window. Send only to a destination the provider has connected and the platform currently permits. An explicit owner action does not bypass recipient eligibility, platform policy or required opt-in.

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

The shared client does not implement product-specific approval policies, compute controls, booking workflows or provider/customer operations. It renders authorized product-supplied actions through extension points. Each tenant supplies its own domain actions and approval semantics. A product must not expose an action unless its server-side authorization permits it.

### 4.5 Client reliability and security

- The client is replaceable and must not own agent execution, durable task state, schedules, canonical message storage or artifact storage.
- Closing, refreshing or replacing the browser client must not interrupt background work or lose messages.
- Reconnect and resynchronize from the canonical server state after a network interruption; do not rely solely on transient browser memory or an open streaming connection.
- Authenticate browser chat at the Web adapter boundary through `getUser`; use separate authenticated host-product resource operations only for non-chat reads and attachment/artifact transfer. Use server-issued upload/download/access references. Never expose channel tokens, database credentials, speech-provider secrets or unrestricted internal service endpoints.
- Reconnect the Web adapter stream to the canonical conversation state after interruption. Because Web v1 does not provide multi-tab/disconnected-browser proactive push, persist background replies and refresh the canonical transcript when the client reconnects or reopens.
- Resolve expiring artifact and product-access URLs at click time through the host product service.

## 5. Durable scheduling, asynchronous tasks and linked interactions

The common platform provides reusable mechanisms for durable schedules, asynchronous work and linked interactions. Each product persists these records in its own authoritative database and registers authorized handlers. The common layer determines when work is due, tracks its lifecycle and routes related events; it does not implement tenant-domain behavior.

### 5.1 Shared scheduler

Support:
- One-time date/time schedules and calendar recurrences (daily, weekly, monthly and yearly).
- Interval schedules such as hourly checks.
- Event- or condition-triggered schedules and scheduled reports.
- Explicit timezone handling, daylight-saving transitions, next-run calculation, missed-run policy, overlap rules, bounded retries and duplicate prevention.
- Editing, pausing, resuming and deactivating a schedule while retaining its execution history.
- Enforce the deployment-wide active-schedule limit and minimum recurring interval configured through the environment variables in §7.3. These limits are shared scheduler policy, not product-specific agent instructions.
- Count each active schedule definition once, including one-time schedules, recurring schedules and reminders, across all tenants in that product deployment. Paused, deactivated, cancelled and completed one-time schedules do not count. Schedule runs/tasks are not additional schedules.
- Check the active-schedule limit atomically when creating, activating or resuming a schedule so concurrent requests cannot exceed the configured limit. Reject a request that would exceed the limit with a clear explanation; do not silently discard or replace another schedule.
- Apply the minimum interval to time-based recurring schedules, including recurring reminders, when they are created, edited or resumed. Validate the recurrence cadence, not the lead time of a one-time reminder. Event- or condition-triggered schedules have no fixed recurrence interval and are governed by the active-schedule limit instead.

Persist schedule definitions and each schedule run. A due schedule creates a durable task/run for a tenant-registered handler; do not rely on an in-memory timer, an active agent session or a connected client as the source of truth. Recheck authorization and relevant business state when the handler executes. The scheduler owns timing, recurrence and deployment-wide schedule limits; the product handler owns the actual action and its domain rules.

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

The SaaS runs a shared application, EVE runtime pool, gateway and PostgreSQL persistence layer for many provider tenants. Tenant isolation is enforced in application services and repositories. The only technology differences belong to product-specific capabilities that the agent operates or integrates with; they do not create separate chatbot, runtime or deployment foundations.

| Area | Common technical baseline | Product-specific boundary |
|---|---|---|
| **Agent runtime** | **Vercel EVE**, integrated through the shared TypeScript platform. All product deployments use the same runtime integration and lifecycle model. | Agent identity/instructions, tools, capability access, policies and workflows are product-specific. |
| **Application language** | TypeScript for the common library, EVE integration, gateway and application services. | Product domain services extend the same typed contracts. |
| **Web client** | Preferred frontend options are **Cloudflare Vinext** and **Vite+**; **Next.js** is also supported. Use TypeScript and the Chat SDK Web adapter client integration (React, Vue or Svelte as appropriate). Browser chat turns and response streaming use the same Chat SDK gateway as messaging platforms; canonical history and non-chat resources remain in product-owned services. | Roles, branding, page configuration and product actions. |
| **Persistence** | PostgreSQL with Drizzle ORM, managed inside Kubernetes by the CloudNativePG operator in every environment. Use shared PostgreSQL by default with tenant-scoped records, strict row/service authorization and least-privilege service credentials. Dedicated databases or clusters are optional stronger-isolation profiles. Use persistent volumes from each cluster's configured StorageClass. Configure independent base/WAL backups and tested recovery; a PVC alone is not a backup. | Tenant records remain logically isolated in shared PostgreSQL. Dedicated databases/clusters may be selected for contractual or regulatory isolation; the same PostgreSQL operator/configuration model is used locally and in production. |
| **Unified bot gateway** | One shared TypeScript Chat SDK `Chat` instance and gateway contract for all bot/channel ingress and egress, including browser chat through `@chat-adapter/web`; bridge normalized turns into EVE through `chatSdkChannel`. | Provider-owned identities and credentials are stored per tenant connection; authorization and delivery policies are tenant-scoped. |
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
- Chat SDK adapter state may be used for adapter-specific subscriptions, locks, deduplication or caching. It must not become the canonical product transcript or replace the application's full message store. Configure Web adapter message-history caching deliberately: set `persistMessageHistory: false` when the handler relies on EVE/canonical history and does not need `thread.messages`; otherwise treat the cache only as adapter-local history. The canonical transcript remains in the product PostgreSQL database.
- The shared EVE runtime integration consumes turns from the common Chat SDK gateway through `chatSdkChannel`; inbound/outbound messages must still pass through shared persistence, identity, authorization, history-search and delivery contracts.
- The Web adapter is the browser's chat-message ingress and response-stream transport, not the whole browser application or a replacement for product services. Its current v1 feature set is deliberately limited: text/Markdown and request-scoped SSE streaming are supported; native file uploads, buttons/cards/modals, thread listing, a client-facing full-history API and multi-tab/disconnected-browser proactive push are not. Do not claim these capabilities are supplied by the adapter.
- Keep canonical conversation listing/history in the product conversation service. Expose only the required authenticated non-chat resource operations (for example, read-only history queries and signed attachment/artifact transfer) through the host product; do not recreate a parallel chat-turn submission or agent-response streaming API.
- Browser-targeted background messages must be durably stored. They can be displayed after the client reconnects or reloads from canonical history; do not promise real-time browser push while the client is disconnected.

References: [Chat SDK repository and feature overview](https://github.com/vercel/chat), [Chat SDK documentation](https://chat-sdk.dev/docs/getting-started), [Chat SDK Web adapter documentation and v1 feature matrix](https://chat-sdk.dev/adapters/official/web), [EVE Chat SDK channel](https://github.com/vercel/eve/blob/main/docs/channels/chat-sdk.mdx), [Vercel guidance on Chat SDK and EVE](https://vercel.com/kb/guide/chat-sdk-and-eve), [Chat SDK adapter catalog](https://chat-sdk.dev/docs/create-chat-sdk).

### 7.2 Persistence ownership

Conversation records, messages, enabled attachment metadata/references, provider-connection mappings, credential references and outbound delivery records are persisted in shared PostgreSQL and tenant-scoped storage interfaces, using Drizzle ORM. Every record carries an immutable tenant ownership key. When `CHATBOT_ATTACHMENTS_ENABLED=false`, no attachment content or durable attachment reference is persisted.

The library provides Drizzle schema definitions, migrations, repository implementations and indexes for shared chat records. The SaaS applies and operates them in shared PostgreSQL with tenant-aware keys, constraints and repository guards. Dedicated databases are optional stronger-isolation deployments. Do not allow Chat SDK's short-lived state/cache to be the only copy of a message.

Tenant business data remains owned by the tenant's application or domain services. The EVE runtime's session state is execution context, not proof that a message or business action was persisted successfully. In local development and CI, run CloudNativePG-managed PostgreSQL inside the local Kubernetes test cluster, using a persistent volume. Retain the PVC and its backing storage across Pod restarts and normal application redeployments; deleting the cluster or losing the host can still destroy local data. CI environments may intentionally be ephemeral. Production deployments use the same CloudNativePG model with environment-appropriate persistent storage, independent backups and tested recovery. Do not use SQLite or a separate PostgreSQL emulator for accepted durable application state.

### 7.3 Environment configuration and secrets

All application-level and runtime configuration is supplied through environment variables only. This includes channel and integration settings, model/provider settings, feature flags, runtime options, limits, voice settings, attachment-ingestion controls, group mention behavior and shared scheduler limits. Do not use database rows, conversational commands or a separate settings UI as an alternative source for application/runtime configuration. Each deployment must receive its configured values through its environment; do not bake deployment-specific values into container images or source code. This rule does not turn tenant identity mappings, provider connections, conversation history/context or business records into environment variables: these are durable application data stored in the shared tenant-scoped database.

The following named common settings must be explicitly configured for the shared deployment:

| Environment variable | Required value | Purpose and enforcement |
|---|---|---|
| `CHATBOT_ATTACHMENTS_ENABLED` | `true` or `false` | Controls whether attachment bytes may enter backend storage and agent processing. When `false`, attachment bytes are not fetched, uploaded, proxied, persisted or passed to the agent. |
| `CHATBOT_CONTEXT` | Required, non-secret text | Deployment-provided additional system context. Load at application startup and append it to the agent's existing system-role context, preserving existing system instructions; do not replace or overwrite them. Send the added content with role `system`, not as user content, a developer-role message or ordinary retrieved context. |
| `CHATBOT_REQUIRE_MENTION` | `true` or `false` | Controls whether a supported mention/reply is required for the bot to respond in group conversations. Direct messages do not require a mention; platform permissions and privacy rules still apply. |
| `CHATBOT_MAX_ACTIVE_SCHEDULES` | Positive integer | Maximum number of active schedule definitions across the shared deployment; tenant-level quotas/fairness are enforced separately. Includes one-time schedules, recurring schedules and reminders; excludes paused, deactivated, cancelled and completed one-time schedules. |
| `CHATBOT_MIN_SCHEDULE_INTERVAL_SECONDS` | Non-negative integer | Minimum recurrence interval for time-based recurring schedules and reminders. For example, `86400` imposes a 24-hour minimum, preventing hourly or every-four-hour recurrence. Set to `0` to disable this minimum-interval restriction. It does not restrict the lead time of a one-time schedule/reminder or the event cadence of event-/condition-triggered schedules. |
| `CHATBOT_AI_CONFIG` | Required JSON object | Configures the AI-compatible API endpoint in one object. `base_url` is required; `model` is optional and is supplied only when the endpoint requires an explicit model name; `api_key` is optional and is supplied only when the endpoint requires API-key authentication. No provider selector or provider-specific default endpoint is assumed. |

Set **CHATBOT_CONTEXT** as shared deployment defaults. Tenant-specific additions are stored as tenant-owned, access-controlled configuration and composed with the base context without replacing it. Load its value from the application process environment at startup and append its text to the system-role context already configured for the agent. Preserve the existing system prompt/instructions and their content; `CHATBOT_CONTEXT` adds deployment-specific instructions and must never replace or overwrite the existing system-role context. Include the added text with role `system`, alongside the product agent's configured instructions and before turn-specific conversation messages; it does not replace conversation history. Supply this non-secret value through a ConfigMap or equivalent deployment environment configuration. Do not put credentials or other secrets in it. Updating the value takes effect after a workload restart or rollout.

Enable and configure the common scheduler and periodic-worker functionality through deployment environment variables (for example, ConfigMaps for non-sensitive controls). These are common platform features; each product supplies the scheduled handlers and work to execute. Keep feature controls and operational settings in deployment configuration, not in product agent instructions.

The active-schedule cap is enforced atomically on schedule creation, activation and resumption. When the cap is reached, requests that would add another active schedule are rejected with a clear explanation; existing schedules are not silently removed. The minimum recurrence interval is checked when a time-based recurring schedule is created, changed or resumed. The recurrence minimum is a shared deployment setting. The active-schedule value is a deployment safety cap; enforce tenant quotas/fairness separately.

Application workloads receive environment variables through Kubernetes resources: use ConfigMaps for non-sensitive values and Secrets for sensitive values. The attachment, mention and scheduler-limit settings are non-sensitive and should be supplied through ConfigMaps. Supply `CHATBOT_AI_CONFIG` through a Kubernetes ConfigMap when its object does not contain an API key; when `api_key` is present, supply the entire JSON object through a Kubernetes Secret so the credential is not exposed. Omit `api_key` for endpoints that do not require API-key authentication.

- Use **ConfigMaps** for non-sensitive settings such as log level, feature flags, public service endpoints, allowed origins and non-secret runtime options. Consume individual keys through `env[].valueFrom.configMapKeyRef`, or a controlled set through `envFrom.configMapRef`.
- Use **Kubernetes Secrets** for credentials and sensitive values such as database passwords, signing keys, OAuth/client secrets, channel tokens, webhook secrets and provider API keys. Reference only the required keys through `secretKeyRef` or a narrowly scoped `envFrom.secretRef`; mount files instead when a library expects certificate/key files or when file-based rotation is preferable.
- Never commit plaintext Secret manifests, credentials, connection strings containing passwords, or private keys. Use Flux CD with SOPS-encrypted Secret manifests (or a separately selected external secret manager integration) so Git stores encrypted values and Flux decrypts them only during reconciliation. Keep decryption keys outside the Git repository and protect them independently.
- Kubernetes Secret values are base64-encoded, not automatically encrypted at rest. Enable cluster secret encryption at rest where supported, restrict Secret read permissions with namespace-scoped RBAC, restrict Pod creation/exec privileges, and avoid broad service-account access. Do not log or expose secret values to the browser, agent/model context, task transcripts or diagnostics.
- Use shared platform namespaces and service accounts with least privilege. Isolate tenants through scoped application authorization and per-connection credential references; per-tenant namespaces/deployments are optional stronger-isolation profiles. Crossplane provider credentials are held in appropriately scoped Kubernetes Secrets and referenced by the relevant ProviderConfig; they are not copied into application Pods unless the application itself needs them.
- Prefer Kubernetes Service DNS names for in-cluster dependencies. `localhost` inside an application Pod refers to that same Pod, not PostgreSQL or another service. Use the CloudNativePG read/write Service name for database connections; use port-forwarding only for deliberate host-side development access.
- Environment variables sourced from ConfigMaps or Secrets are fixed when a container starts. A change to their values requires a workload rollout/restart to update the process environment. Use mounted files or a dedicated reload mechanism when a workload must observe rotations without restart; do not assume env vars update live.

### 7.4 Security and operational requirements

- Resolve tenant scope, provider connection and sender identity from trusted server-side authentication and verified platform events. Never accept tenant identity from model output, browser input or an untrusted external identifier.
- Enforce tenant isolation in every repository query, service operation, cache key, object-storage path, EVE session mapping, task, schedule, event cursor, attachment, search result and outbound dispatch. Use tenant-aware keys/constraints and deny access when tenant context is absent or ambiguous.
- Recheck authorization for inbound events, agent tool calls, callbacks, background work, attachment retrieval, owner-initiated sends and workflow notifications. A valid platform credential is not by itself authorization to access another tenant's data.
- Store each provider's credentials separately using encrypted secret storage or a secrets manager. Never expose tokens, webhook secrets, database credentials or storage credentials to the browser, model context, task transcripts or logs.
- A provider may add or replace its platform identity without losing tenant-owned history/context. Reauthorize replacement identities and establish platform-side memberships/permissions again; do not assume platform conversation IDs or memberships transfer.
- Deduplicate source events and handle retries/out-of-order delivery. Persist accepted messages and outbound intents before acknowledging durable receipt; recover pending delivery after restarts without repeating business effects.
- Keep logs and diagnostics useful but redact credentials, private attachment content and unnecessary personal information.
- Record effective capabilities and connection status so unsupported or revoked operations fail explicitly rather than appearing successful.
- Verify CHATBOT_AI_CONFIG parsing and keep any API key in a Kubernetes Secret, never in model context, browser state or logs.
- Enforce both per-tenant scheduler quotas/fairness and the deployment-wide active-schedule safety cap atomically, including under concurrent requests.
- Test cross-tenant isolation for conversations, history search, attachments, runtime sessions, tasks, schedules, event replay, credentials and outbound delivery.
- Keep the shared library and gateway maintainable and replaceable. Do not let platform SDK identifiers or EVE runtime session IDs become the public tenant/conversation contract.

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

The shared platform defines reusable technical contracts. Each tenant remains responsible for its own domain behavior and authorization:

- **Identity and integrations:** the shared layer manages tenant-scoped connection records, credential references, capabilities and mappings. Providers create and own their official platform identities and manage their platform-side memberships/permissions.
- **Conversation history:** the shared layer owns canonical message persistence, context isolation and history retrieval in shared PostgreSQL. Each tenant defines the authorized conversation scope available to its agent.
- **Outbound actions:** the shared outbox handles durable delivery, retries and idempotency. An authorized owner can explicitly initiate a send through a selected connected identity. Tenant workflows may send eligible service notifications under tenant-defined authorization; autonomous cold outreach is not a platform capability.
- **Attachments:** the shared layer receives, persists/references and authorizes retrieval. Each tenant decides how its agent processes supported files.
- **Authorization:** the shared layer provides identity and policy interfaces; each tenant supplies its principals, roles, tenant scope and authorization decisions.
- **Agent and client configuration:** the shared EVE integration and browser client are common. Agent instructions, tools, actions, approval semantics, views and branding remain tenant-owned.
- **Scheduling and tasks:** the shared scheduler, task lifecycle and linked-interaction primitives are common. Each tenant supplies domain handlers, business records and rules for human-dependent work.
- **Infrastructure:** shared PostgreSQL and shared runtime infrastructure are the default. Dedicated databases, namespaces or deployments are optional isolation choices, not per-tenant defaults.

The common library must not own tenant business records, business workflows, approval policies, domain-specific tools or tenant-specific deployment topology.

## 10. Shared acceptance checks

The shared implementation is complete only when these checks pass:

- A user can start and continue a direct conversation using ordinary text without a slash command.
- Each provider creates and owns its official platform identity; the application never provisions a separate internal bot account.
- A provider authorizes the application to use its platform identity and adds that identity to selected destinations directly on the platform; there is no separate group-onboarding dashboard.
- Multiple provider tenants can use the shared EVE runtime pool and shared PostgreSQL database without cross-tenant access to conversations, history, credentials, context, attachments, tasks, schedules or outbound actions.
- A tenant can authorize multiple provider-owned platform identities; each connection has distinct credentials, permissions and capability metadata.
- Replacing or adding a provider-owned identity preserves tenant-owned context, history, tasks and data. Replacement requires fresh authorization and platform-side membership/permissions; external memberships and conversation identifiers are not assumed to transfer.
- External sender IDs are scoped to their provider connection and are never treated as globally unique human identities.
- A supported platform identity maps multiple DMs/groups/threads to distinct canonical conversations without cross-conversation or cross-tenant context leakage.
- When a customer asks a question outside the available context, the agent uses only authorized context and does not invent a recipient or handoff.
- With CHATBOT_REQUIRE_MENTION=true, the bot ignores unaddressed group messages and responds to supported mentions/replies; with false, it can respond to ordinary group messages without an @mention, subject to platform permissions. Direct messages work without a mention in either mode.
- A web conversation enters through the authenticated web adapter on the same unified gateway, maps to an isolated canonical conversation/EVE session and cannot supply its own tenant identity.
- All tenants use the same shared EVE runtime integration, unified gateway, web client and tenant-scoped PostgreSQL/Drizzle persistence contracts.
- An authorized owner can explicitly initiate a text message through a selected connected provider identity without an active inbound turn, provided the destination is eligible and platform rules/consent requirements are met. The platform does not autonomously select recipients or perform cold outreach.
- One-time, recurring, interval and event-triggered schedules produce durable runs in the configured timezone and remain due/recoverable while the agent or client is offline.
- Accepted asynchronous tasks survive client disconnect and process restart; cancellation is recorded and stale workers are fenced from committing.
- Replayed or duplicate source events resume from a durable cursor and do not repeat an accepted business effect.
- A transient delivery failure or duplicate event does not lose a message or repeat its associated business action.
- search_conversation_history finds an exact older message from before context compaction and returns its source reference and timestamp.
- The history tool cannot retrieve messages from an unrelated conversation or tenant scope.
- A restart does not erase canonical messages, enabled attachment metadata/references or pending outbound delivery.
- A platform-specific unsupported capability is rejected or reported explicitly, not silently treated as successful.

