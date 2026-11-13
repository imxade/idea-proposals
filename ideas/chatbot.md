# Chatbot — Shared Conversation Platform

## Architecture overview

The diagram shows the shared library embedded in a product application deployed to Kubernetes. Local development uses K3s; production uses the deployment target defined in the technical baseline. Each deployment has an isolated application namespace, database identity, credentials, authorization scope and product-owned handlers. Deployments may share a cluster where isolation controls are adequate; separate clusters remain an option for stronger isolation.

~~~mermaid
flowchart TB
  subgraph Clients["Shared interaction surfaces"]
    Browser["Browser user"]
    Web["Standalone shared Next.js web application<br/>Local: Kubernetes workload / Azure: Static Web Apps"]
    Discord["Discord"]
    WhatsApp["WhatsApp Business"]
    Other["Other supported channel adapters"]
    Browser <--> Web
  end

  subgraph Provisioning["Infrastructure and desired-state management"]
    Crossplane["Crossplane<br/>Supported infrastructure provisioning"]
    Flux["Flux CD + Kustomize<br/>Git-based reconciliation"]
  end

  subgraph Cluster["Kubernetes cluster — local K3s / production target"]
    subgraph ProductHost["Product namespace — isolated application instance"]
      subgraph Common["Reusable TypeScript chatbot library embedded in product application"]
        Gateway["Unified bot gateway and channel adapters"]
        WebAPI["Authenticated application API and stream endpoint"]
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
        WebAPI --> Conversation
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
  Web <-->|"Authenticated API + streaming"| WebAPI
  WebAPI <--> Outbound
  Discord <-->|"Platform events / outbound messages"| Gateway
  WhatsApp <-->|"Platform events / outbound messages"| Gateway
  Other <-->|"Platform events / outbound messages"| Gateway
  Outbound --> Gateway
~~~

**Normative separation rule:** a common requirement is specified here once. The product documents may configure or constrain that requirement for their own use, but must not restate the shared implementation. A product-specific requirement belongs only in that product's document.

## 1. Architecture decision: a reusable library, not a separate service

Implement the common chatbot layer as a **private, reusable TypeScript library/package** embedded in each product's application. It is not an independently deployed chatbot microservice.

The common platform combines a reusable TypeScript conversation library with one shared Vercel EVE runtime integration. Each product hosts the same platform baseline in its own application/deployment and supplies its own authorization, domain capabilities, agent definitions and product-data scope.

This is preferable to a separate service because:

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
| **Shared web application** | Own and provide the standalone reusable Next.js browser chat application, including conversation history, transcript/composer, streaming, voice, file attachments, progress/events and authenticated API integration. |
| **Unified bot gateway** | Provide the common ingress/egress path for configured bot channels: authenticate and normalize events, route them to the correct product agent and dispatch authorized outbound messages. |
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
- Its deployment-provided baseline system prompt, exposed through the `CHATBOT_CONTEXT` environment variable. The common chatbot loads this value at startup and injects it into model requests as a message with role `system`; it is not user content, a developer-role message or ordinary retrieved context. The product owns the prompt content.
- Its domain-specific tools/capabilities and tool authorization. The product also supplies the usage instructions, context and request/response contracts the deployed agent needs to use those capabilities.
- Its attachment-processing capabilities. The common environment setting controls whether attachment content enters the backend at all; when enabled, the common layer delivers supported attachments to the agent subject to authorization and product capabilities.
- Product-owned handlers for scheduled work, asynchronous tasks and linked interactions, plus product-specific events and reasons for proactive messages.

These shared extension interfaces and product-owned lifecycle handlers are in-process modules/providers registered with the host application. This is not a plugin marketplace or a mandatory plugin microservice. This registration requirement does not impose a common tool protocol or registry on every product capability: a product may expose its own tools through in-process integrations, APIs, CLI commands or installed dependencies and provide the deployed agent with the corresponding usage instructions and contracts. A separate API server accepting documented JSON requests and returning asynchronous results is one possible implementation, not a prescribed architecture. A product capability must not gain unrestricted database or platform access merely because it is available to the agent.

### 1.3 Shared EVE runtime and unified bot gateway

All product deployments use the same **Vercel EVE runtime baseline**, integrated through the common TypeScript platform. The shared runtime layer owns the common agent execution lifecycle, runtime sessions, tool invocation, workflow execution, delegation primitives, streaming/events and runtime-to-conversation mapping. A product defines its own agent identity/instructions, capabilities, tools, authorization and domain workflows on top of that foundation.

EVE session state is execution context, not the canonical record of conversations or product business state. Persist messages, task/business records, schedules, checkpoints and delivery intents through the product's authoritative data services. Product-specific specialist agents, worker definitions, selection, delegation policy and result evaluation remain part of the product agent; the common chatbot provides EVE's generic runtime primitives but does not define a shared specialist registry or product-neutral subagent orchestration. All product deployments use the same EVE integration and version baseline, not one shared cross-product agent instance. Each product has its own deployment, agent instances, runtime sessions, identities and data; capability-specific tools and workloads remain product-owned.

The **unified bot gateway** is also common. It is a shared library module and service interface, not a separately operated cross-product bot service. It is the single ingress/egress path for supported messaging platforms and:
- Receives platform events through configured channel adapters and verifies authenticity.
- Resolves the product deployment, bot account, bot connection, participant and canonical conversation scope.
- Normalizes messages, attachments, replies, threads and platform metadata.
- Routes accepted user turns to that product's shared EVE runtime integration.
- Returns agent/application messages and asynchronous events through the correct platform adapter.
- Supports durable, idempotent proactive outbound delivery through the shared outbox.

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
4. A person who will manage or provide context to the bot connects their platform identity and is assigned the relevant product-defined role from within that bot account's management conversation. The common flow does not require a separate registration website, pasted token, additional identity-verification step or external account-linking ceremony.

The bot account's management conversation is the control surface for adding, changing and disconnecting linked human platform identities. A platform identity cannot be attached to or switched for the bot account through an unrelated customer conversation or an unrelated bot account.

When a linked person changes their platform user ID, they use the bot account's management conversation to disconnect the previous ID and connect the replacement. The application updates the human-to-platform identity mapping only. It does not create a new bot account, change the bot account ID, reset or migrate the bot's context, or alter its conversation history, tasks, linked interactions, configuration or other account data. The existing bot account continues as before under the replacement platform identity.

Changing the bot's own platform identity or replacing one of its channel connections is also done from the same bot-account management conversation. It updates only the affected platform-connection mapping; the internal bot account and all of its context and data remain unchanged.

The common layer supports human-provided context and role-aware handling of knowledge gaps. A linked person can contact the bot through their associated platform identity from within the bot account and provide new information or context; the application associates that contribution with the bot account and its source.

**When a provider or other linked human-role participant asks the agent a question**, the agent answers from the context and capabilities already available to it. If the required information is missing, it tells the participant that it does not have enough context and is unsure. It does not forward the question back to that same provider, ask the provider to answer their own question, or repeatedly request the missing answer. If the provider then supplies information, a clarification or a correction, the agent records that contribution as text in the bot account's durable context store/database, with its source and relevant scope, so it can be retrieved for future conversations.

**When a customer asks a question**, the agent first checks its authorized context. If the answer is unavailable and a relevant provider role is linked, the common linked-interaction service forwards the customer's question to that provider. When the provider responds, the agent stores the response as text in the bot account's durable context store/database, associated with its source and relevant scope, and then sends the answer back to the originating customer conversation. The customer should not have to repeat the question.

If no relevant provider or other human role is linked, the agent does not create a handoff with no recipient or invent missing information. It tells the customer that it cannot determine the answer from the available context. The same rule applies to provider-initiated questions: the agent states that it lacks the context rather than asking a nonexistent person or asking the provider to answer their own question.

### 2.3 Canonical conversation mapping

The application creates a stable internal conversation ID and maps it to the external platform context. The mapping must preserve enough information to route messages back to the right place:

- Product/deployment, bot account ID and any product-specific tenant/owner scope.
- Channel type and channel-connection identity.
- External platform account, room, group, channel or DM identifier.
- External thread/topic identifier where applicable.
- External sender identity and message identifier.
- Message timestamp, reply/reference target and relevant platform metadata.

A direct message, web conversation, group/channel thread or other independent conversation context receives its own canonical conversation mapping. A group's channel/thread context is not mixed with unrelated groups, DMs or web conversations.

A web conversation connects directly to the product's application API. It does not invite or create a messaging bot. Each new web conversation has its own canonical conversation and agent-runtime session/context; reopening an existing conversation resumes its existing context. The shared logical agent can therefore serve many isolated conversations without creating a separate agent identity or process for each one.

A runtime session ID is an execution identifier mapped to the canonical conversation ID. It is not the source of truth for message history. If a runtime is replaced, the canonical conversation and its persisted history remain available.

A previous conversation is not silently added to the current one. Cross-conversation retrieval requires an explicit reference and an independent authorization check.

### 2.4 Direct, group and channel interaction

Ordinary natural language is the normal interaction method. Slash commands may be supplied as shortcuts, but users must not need a command to make an ordinary request.

- **Direct messages:** messages are addressed to the agent by virtue of being sent to its identity. A mention or command is not required.
- **Mention-required group mode:** when `CHATBOT_REQUIRE_MENTION=true`, the bot responds in group conversations only to messages that mention/address it (including supported platform-native reply-to-bot behavior). Unaddressed group messages are ignored.
- **Mention-not-required group mode:** when `CHATBOT_REQUIRE_MENTION=false`, the bot may respond to ordinary group messages without an @mention, subject to platform permissions and the conversation's access policy.
- **Replies and threads:** replying to an agent message or another message can continue or refer to that exchange. Preserve the referenced message and thread context.

The deployment environment variable `CHATBOT_REQUIRE_MENTION` controls this behavior for group conversations and must be explicitly set to `true` or `false`. Direct messages do not require a mention. A platform may still restrict which group messages reach the bot, so disabling the mention requirement cannot override platform privacy settings or permissions. The shared library must not impose a universal DM-only, mention-only, public-only or private-only restriction.

**Initial messaging targets are Discord, WhatsApp Business and Telegram.** The shared platform supports these three adapters in its initial integration baseline; each product configures the subset it uses. Additional adapters can follow the same common contract.

Actual access and visibility remain controlled by platform installation, membership, privacy settings, permissions, intents, user consent and policies. The library must not promise that every platform can expose all messages in all spaces. A platform adapter must report its real capabilities rather than silently pretending that an unsupported mode works.


### 2.5 Platform-specific capability constraints

The interaction model is shared, but actual access and delivery differ by platform. The library and product configuration must record the effective capabilities of each adapter and connection rather than promising identical behaviour everywhere.

| Platform | Identity and access model | Constraints to preserve |
|---|---|---|
| **Discord** | A Discord application/bot identity is installed into a server or authorized for supported user contexts. | Server and channel permissions govern access. Reading ordinary message content in guild channels depends on the applicable Message Content Intent and Discord's approval/configuration requirements. Installing the bot does not authorize unsolicited DMs to every server member. Use a permitted welcome/onboarding surface and an opt-in path to DM rather than an assumed bulk-DM campaign. |
| **Telegram** | A Telegram bot has its own bot identity and username. A person can start a private chat or add the bot to a group. | A bot cannot initiate an ordinary private conversation with a person who has not started it. Group privacy mode limits which messages the bot receives; full group participation requires the appropriate privacy/admin configuration. A mention or reply works only where the platform delivers that event to the bot. |
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

The shared web client receives messages and status/progress events through the product's authenticated application API and a streaming transport appropriate to the client. Messaging clients receive platform-native messages. A live browser connection is not required for the service to continue a task or deliver a later message.

The common project owns the shared standalone Next.js web application and its reusable chat experience. Products configure its identity, roles, authorization and product actions. The common conversation/API contracts also allow a product or third party to build a separate client of its own; that does not transfer ownership of the shared web application.

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

The shared package includes the reusable browser-based chat client, not merely a web transport adapter or API contract. The common project owns the standalone Next.js application. All product deployments can use it and configure product identity, authorization, capabilities, branding and visible workspaces; they may also build another client against the common API.

### 4.1 Framework and application boundary

- Implement the shared web client using **Next.js with TypeScript and file-based routing**. Do not prescribe a specific Next.js routing mode.
- The browser client connects directly to the host product's authenticated application API. Each web conversation maps to its own canonical conversation and agent-runtime session/context; it does not create a messaging bot or a separate agent process.
- The common project owns and ships the standalone shared Next.js web application. It connects to the host product's authenticated application API and can be deployed separately from the product backend.
- Products may use the shared web application as-is or build their own web/mobile client against the documented common conversation and event APIs. A custom client does not replace or redefine the shared application.
- The same client must support both desktop and mobile browser layouts. Do not create a native desktop application as part of this shared scope.

### 4.2 Shared chat experience

The common client provides:
- A conversation list/history view and the ability to create, open, resume and switch between conversations without mixing their contexts.
- A message transcript that distinguishes user and agent messages, preserves chronological order and renders supported text/Markdown content.
- A composer for ordinary text messages, send/submit state, multiline input and appropriate keyboard behavior.
- Streaming agent responses and live status/progress events, with clear sending, receiving, reconnecting and failure states.
- Correct handling of long conversations, including loading older persisted messages and preserving the current scroll/reading position where practical.
- Message timestamps and supported reply/reference information.
- Responsive layouts, accessible controls, keyboard navigation, visible focus, readable contrast and clear empty/loading/error states.
- A settings surface for preferences supported by the host product.
- Safe rendering of message content; do not execute arbitrary HTML or active content supplied in messages or attachments.

### 4.3 Voice interaction

The common web client supports speech-to-text (STT) and text-to-speech (TTS). When voice input is enabled, the user explicitly starts microphone capture; the client continuously listens while active and detects a configurable silence/pause interval before finalizing the utterance and submitting it for a response. The deploying operator configures this interval, with a sensible default. Show a clear listening state and provide an immediate stop/mute control. Handle microphone permission denial, unsupported browsers and recognition errors. TTS provides visible playback state, pause/stop controls and interruption when the user starts a new interaction or selects another response.

Voice interaction uses the same canonical conversation and message APIs. Text interaction remains available if voice permissions, browser APIs or a speech provider are unavailable. Make the speech-processing method and provider/processing location visible to the user. Raw microphone audio is not retained as canonical conversation data and must not be retained by default; persist the resulting text message, and retain audio only when the user explicitly attaches or requests its retention under the host product's policy. Keep provider credentials and speech-service secrets on the server; do not expose them in browser code. The host product configures the speech provider, allowed languages, voice, retention policy and disclosure.

### 4.4 Shared client actions and product extension points

The common client provides reusable presentation and action contracts for:
- Task/status/progress events supplied by the host product.
- Focused requests for additional user input and confirmations, rendered as actionable messages/cards using a generic interaction contract.
- Display of attachments and authorized references only when attachment ingestion is enabled by the deployment environment; expose them to the agent through the common attachment-retrieval contract. When disabled, disable web upload controls and rely on mandatory backend enforcement so attachment bytes are not uploaded or processed.
- Artifact/file references and click-time download resolution supplied by the host product.
- External access links that open in a separate browser tab/window when the host product supplies them.

The shared client does not implement product-specific approval policies, compute controls, booking workflows or provider/customer operations. It renders authorized product-supplied actions through extension points. Each product supplies its own domain actions and approval semantics. A product must not expose an action unless its server-side authorization permits it.

### 4.5 Client reliability and security

- The client is replaceable and must not own agent execution, durable task state, schedules, canonical message storage or artifact storage.
- Closing, refreshing or replacing the browser client must not interrupt background work or lose messages.
- Reconnect and resynchronize from the canonical server state after a network interruption; do not rely solely on transient browser memory or an open streaming connection.
- Use authenticated, authorized API requests and server-issued upload/download/access references. Never expose channel tokens, database credentials, speech-provider secrets or unrestricted internal service endpoints.
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

Provide a durable interaction ID that can link an originating conversation, related participant conversations, a question or case, deferred task(s), provider/owner response and follow-up delivery. Track lifecycle such as open, waiting, resolved and closed.

The interaction service provides scoped context projections and event-based handoff/routing; it does not merge complete transcripts or grant participants access to one another's private messages. Check authorization whenever context is shared, a participant is added, a response is attached or a message is sent. Product handlers define participant roles, what information may be shared and how a response resolves the originating request. This mechanism supports product-specific pending questions and owner-dependent work without making any product's approval or business rules common.

### 5.5 Persistence and API requirements

The common interfaces must support creating, reading, updating, pausing/deactivating and listing schedules; querying schedule runs; creating, inspecting, cancelling and resuming eligible tasks; recording checkpoints and worker leases; recording/replaying event cursors; and opening, linking, routing, responding to and resolving interactions. Persist the corresponding schedule definitions/runs, task/run state, event cursors/deduplication keys, interaction links/lifecycle and outbound delivery records in the product's database. These are functional data/API requirements, not a fixed schema or exhaustive endpoint list.


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
| **Web client** | The common project owns a standalone Next.js application with TypeScript and file-based routing; products and third parties may also build clients against the common API. | Roles, branding, page configuration and product actions. |
| **Persistence** | PostgreSQL with Drizzle ORM, managed inside Kubernetes by the CloudNativePG operator in every environment. Use separate databases and credentials for each product deployment. Local K3s and production use persistent volumes from their respective StorageClasses. Configure independent base/WAL backups and tested recovery; a PVC alone is not a backup. | Product databases and records remain isolated. The same PostgreSQL operator/configuration model is used locally and in production; size, replica count and StorageClass are environment-specific. |
| **Unified bot gateway** | One shared TypeScript gateway implementation and contract for all bot/channel ingress and egress. Vercel Chat SDK is the preferred adapter layer where its support is adequate. | Separate bot identities, credentials, channel connections, opt-in rules and policies per product. |
| **Agent/application workloads** | Kubernetes is the common workload environment: K3s locally. Deploy the application, EVE-backed services, workers, Jobs and in-cluster supporting services into declared namespaces. | Capability-specific workloads, permissions and resource profiles remain product-owned. |
| **Infrastructure provisioning** | Retain Crossplane to provision supported infrastructure resources, including supported Azure resources through a management/bootstrap plane. No Terraform is part of this architecture. | Product-specific resource compositions and permissions. Crossplane does not replace the initial bootstrap needed to create the cluster on which it runs. |
| **Desired state and reconciliation** | Kustomize bases/overlays define cluster, operator and application configuration; Flux CD reconciles Git-hosted desired state, including Crossplane resources and the CloudNativePG operator. | Product-specific manifests and capability resources use the same GitOps workflow. |
| **Local/CI cluster parity** | Use K3s for local development and Kubernetes workload integration tests. Deploy the application and CloudNativePG into the local cluster using the same Kubernetes resource contracts used for production. | Product test data, capabilities and validation cases remain separate. |
| **Autoscaling** | Configure Kubernetes Horizontal Pod Autoscalers for eligible stateless APIs/workers with metrics, resource requests and explicit minimum/maximum replicas.  | Do not assume autoscaling is enabled by default. PostgreSQL scaling and replication use database-aware policies, not generic HPA. |
| **Deployment target** | Local development and integration testing use K3s. Production deployment targets AKS; validate Azure infrastructure provisioning through Crossplane before accepting the target. | Environment-specific overlays supply storage, identity, networking, sizing and secret references. |



### 7.1 Chat SDK's role

Vercel Chat SDK is the preferred shared **messaging transport/adapter layer**, not the chatbot's database, domain backend or agent runtime. It supplies platform-oriented adapters and message/thread/event abstractions. The unified bot gateway wraps it behind the shared channel contract so it can be replaced or supplemented without changing product behavior. Its catalog distinguishes Vercel-maintained official adapters from vendor-official and community adapters. Discord, Telegram and WhatsApp Business have official adapters; Matrix currently has a vendor-official Beeper Matrix adapter. Select an adapter by its real feature coverage and hosting requirements, not just by its presence in the catalog.

- Use the appropriate Chat SDK adapter for a platform when it supports the required inbound events, identity, thread behaviour, message formats and outbound delivery.
- The deployment must support the adapter's event lifecycle. Discord slash commands/button interactions can arrive through webhooks, but regular messages and reactions use a Gateway WebSocket listener. Matrix requires a long-running sync/event process; it cannot run in a webhook-only serverless runtime. Do not assume all adapters can be hosted as short-lived webhook functions. Validate reconnection, event delivery, scaling and process lifecycle for the selected hosting environment.
- Use a custom adapter when no suitable adapter exists or a platform feature is missing. Do not claim support solely because a generic Chat SDK interface exists.
- Chat SDK adapter state may be used for adapter-specific subscriptions, locks, deduplication or caching. It must not become the canonical product transcript or replace the application's full message store.
- The shared EVE runtime integration consumes normalized turns from the common gateway. EVE's Chat SDK channel may be used as an internal bridge where appropriate, but inbound/outbound messages must still pass through shared persistence, identity, authorization, history-search and delivery contracts.
- The Chat SDK web adapter is a transport option, not the shared browser application. The shared Next.js client uses the host product API, durable conversation store, attachment endpoints and outbound event/outbox contracts; do not assume the adapter alone provides uploads, complete history, durable proactive push or product-specific actions.

References: [Chat SDK repository and feature overview](https://github.com/vercel/chat), [Chat SDK documentation](https://chat-sdk.dev/docs/getting-started), [EVE Chat SDK channel](https://github.com/vercel/eve/blob/main/docs/channels/chat-sdk.mdx), [Vercel guidance on Chat SDK and EVE](https://vercel.com/kb/guide/chat-sdk-and-eve), [Chat SDK web adapter details](https://github.com/vercel/chat/blob/main/packages/adapter-web/README.md), [Chat SDK adapter catalog](https://chat-sdk.dev/docs/create-chat-sdk).

### 7.2 Persistence ownership

Conversation records, messages, enabled attachment metadata/references, channel identity mappings, and outbound delivery records are persisted through the host product's PostgreSQL database and storage interfaces, using Drizzle ORM. When `CHATBOT_ATTACHMENTS_ENABLED=false`, no attachment content or durable attachment reference is persisted.

The library may provide Drizzle schema definitions, migrations, repository implementations and indexes for these shared chat records. Each product applies and operates them in its own database. Do not run a separate shared chatbot database or allow Chat SDK's short-lived state/cache to be the only copy of a message.

Product business data remains owned by the product application. The EVE runtime's session state is execution context, not proof that a message or business action was persisted successfully. In local development and CI, run CloudNativePG-managed PostgreSQL inside K3s, using a persistent volume. The PVC and the local cluster's backing data volume must be retained across Pod restarts and normal application redeployments; deleting the cluster or losing the host can still destroy local data. CI environments may intentionally be ephemeral. Production deployments use the same CloudNativePG model with environment-appropriate persistent storage, independent backups and tested recovery. Do not use SQLite or a separate PostgreSQL emulator for accepted durable application state.

### 7.3 Environment configuration and secrets

All application-level and runtime configuration is supplied through environment variables only. This includes channel and integration settings, model/provider settings, feature flags, runtime options, limits, voice settings, attachment-ingestion controls, group mention behavior and shared scheduler limits. Do not use database rows, conversational commands or a separate settings UI as an alternative source for application/runtime configuration. Each deployment must receive its configured values through its environment; do not bake deployment-specific values into container images or source code. This rule does not turn bot-account identity mappings, conversation history/context or product business records into environment variables: those are durable application data and remain in the product database.

The following named common settings must be explicitly configured for each deployment:

| Environment variable | Required value | Purpose and enforcement |
|---|---|---|
| `CHATBOT_ATTACHMENTS_ENABLED` | `true` or `false` | Controls whether attachment bytes may enter backend storage and agent processing. When `false`, attachment bytes are not fetched, uploaded, proxied, persisted or passed to the agent. |
| `CHATBOT_CONTEXT` | Required, non-secret text | Deployment-provided baseline system prompt. Load at application startup and include in model requests with role `system`; do not pass it as user content, a developer-role message or ordinary retrieved context. |
| `CHATBOT_REQUIRE_MENTION` | `true` or `false` | Controls whether a supported mention/reply is required for the bot to respond in group conversations. Direct messages do not require a mention; platform permissions and privacy rules still apply. |
| `CHATBOT_MAX_ACTIVE_SCHEDULES` | Positive integer | Maximum number of active schedule definitions in the deployment, counted across all bot accounts. Includes one-time schedules, recurring schedules and reminders; excludes paused, deactivated, cancelled and completed one-time schedules. |
| `CHATBOT_MIN_SCHEDULE_INTERVAL_SECONDS` | Non-negative integer | Minimum recurrence interval for time-based recurring schedules and reminders. For example, `86400` imposes a 24-hour minimum, preventing hourly or every-four-hour recurrence. Set to `0` to disable this minimum-interval restriction. It does not restrict the lead time of a one-time schedule/reminder or the event cadence of event-/condition-triggered schedules. |
| `CHATBOT_AI_CONFIG` | Required JSON object | Configures the AI-compatible API endpoint in one object. `base_url` is required; `model` is optional and is supplied only when the endpoint requires an explicit model name; `api_key` is optional and is supplied only when the endpoint requires API-key authentication. No provider selector or provider-specific default endpoint is assumed. |

Set **`CHATBOT_CONTEXT`** independently for each product deployment. Load its value from the application process environment at startup and include it in model requests as the deployment's baseline message with role `system`. Combine it with the product agent's configured instructions and the conversation's turn-specific messages; it does not replace conversation history. Supply this non-secret value through a ConfigMap or equivalent deployment environment configuration. Do not put credentials or other secrets in it. Updating the value takes effect after a workload restart or rollout.

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

The application, database and supporting services run as Kubernetes workloads. Local development, CI and production use the same Kubernetes resource contracts, CloudNativePG operator, Kustomize bases and Flux reconciliation; environment overlays supply storage classes, replica counts, resource budgets, endpoints and credentials.

Cluster bootstrap and infrastructure provisioning are distinct from workload deployment:

- **Local K3s:** local development uses the same Kubernetes resource contracts as production and does not depend on an Azure emulator.
- **One-machine VPS:** a host bootstrap checks capacity/prerequisites, configures secrets and hostname, installs K3s and establishes the initial GitOps/bootstrap components. This is a self-managed deployment profile, not a highly available cluster.
- **Infrastructure provisioning:** bootstrap a management plane before installing Crossplane. Use supported provider resources/compositions to provision infrastructure; Crossplane cannot bootstrap itself onto a cluster that does not yet exist.
- **Workload configuration:** Flux reconciles Kustomize bases/overlays for namespaces, policies, CloudNativePG, application services, workers, storage configuration, ingress and monitoring. Operators reconcile their own custom resources. Avoid imperative one-off deployment paths except for documented bootstrap operations.

**Local development and CI**

1. Provide one documented command to start the complete local development environment, including K3s, required supporting services, CloudNativePG, the application and the shared Next.js client. It must wait for readiness and leave the environment usable without requiring a sequence of manual startup commands.
2. Use Flux CD and the local Kustomize overlay to deploy the application, supporting services and CloudNativePG operator. Create the PostgreSQL Cluster resource through GitOps and use a persistent volume from the cluster StorageClass. Retain local backing storage across normal restarts and redeployments; CI may destroy its disposable environment after tests.
3. Build prebuilt immutable images, deploy through the same Kustomize/Flux manifests and application configuration contracts, and promote the same image digests through subsequent validation.
4. Test application, database, gateway, frontend, scheduler, task recovery, storage and product capabilities. Include Pod restart/rescheduling, PostgreSQL restart and recovery, persistence across workload restart, upgrade/rollback, event replay, task cancellation and end-to-end checks. Test local persistence separately from backup/restore and machine-loss recovery.

**One-machine VPS deployment**

1. Use the same immutable application images, Kustomize manifests, Crossplane resources and Flux CD reconciliation flow validated locally and in CI. Install K3s on the host and run the application, PostgreSQL and supported services inside that cluster within the host's resource budget.
2. Deploy CloudNativePG-managed PostgreSQL with persistent K3s volumes. Configure backups outside the primary database volume and test recovery. Do not substitute SQLite or a separate external PostgreSQL service for the standard deployment.
3. The shared Next.js application may run inside K3s for this deployment profile.

A successful unit-test run is not evidence of deployment parity. Do not substitute an ad-hoc Compose deployment or host-run service for this workflow.


## 8. Common versus product-specific boundary

The shared platform defines reusable technical contracts. Each product remains responsible for its own domain behavior and authorization:

- **Messaging and identity:** the shared layer normalizes channel identities, conversations, DMs, groups, threads, mentions and replies. Each product configures its own platform accounts, channel connections and access policies.
- **Conversation history:** the shared layer owns canonical message persistence, context isolation and history retrieval. Each product defines the authorized conversation scope available to its agent.
- **Proactive delivery:** the shared outbox handles durable delivery, retries and idempotency. Each product defines its own business triggers, consent requirements and message policy.
- **Attachments:** the shared layer receives, persists/references and authorizes retrieval. Each product decides how its agent processes supported files.
- **Bot accounts and human roles:** the shared layer owns bot-account identity, platform-identity mappings and the control-conversation mechanism for linking or switching humans. Each product defines its role names, role permissions and business meaning.
- **Authorization:** the shared layer provides identity and policy interfaces; each product supplies its own principals, roles, tenant/owner scope and authorization decisions.
- **Agent and client configuration:** the shared EVE integration and Next.js client are common. Agent instructions, tools, actions, approval semantics, views and branding remain product-owned.
- **Scheduling and tasks:** the shared scheduler, task lifecycle and linked-interaction primitives are common. Each product supplies domain handlers, business records and rules for human-dependent work.
- **Kubernetes access:** Kubernetes is deployment infrastructure, not a default agent capability. A product may expose narrowly scoped cluster operations only through its own authorized service boundary.

The common library must not own product business records, business workflows, approval policies, domain-specific tools or product-specific deployment topology.

## 9. Shared acceptance checks

The shared implementation is complete only when these checks pass independently in all product deployments:

- A user can start and continue a direct conversation using ordinary text without a slash command.
- A supported platform identity maps multiple DMs/groups/threads to distinct canonical conversations without cross-conversation context leakage.
- A bot account can be provisioned and connected to a platform installation without confusing the bot account ID with a platform bot ID or a human user ID.
- A human role can be attached or its platform identity switched from the bot account's management conversation; switching the ID leaves the bot account's context, history, tasks and data unchanged.
- Switching a bot's platform identity or channel connection also leaves its internal bot account, context, history, tasks and data unchanged.
- When a provider asks a question that is outside the available context, the agent states that it lacks context and does not ask that provider to answer their own question; information the provider subsequently supplies is stored as text in the bot account's durable context/database with source attribution.
- When a customer asks a question outside the available context and a relevant provider is linked, the question is forwarded to that provider; the provider's response is stored as text in the bot account's durable context/database with source attribution and returned to the originating customer conversation.
- When no relevant provider or other human role is linked, the agent does not invent a recipient or create a handoff; it clearly states that it cannot determine the answer from available context.
- With `CHATBOT_REQUIRE_MENTION=true`, the bot ignores unaddressed group messages and responds to supported mentions/replies; with `false`, it can respond to ordinary group messages without an @mention, subject to platform delivery and permissions. Direct messages work without a mention in either mode.
- A web conversation connects directly to the product application and has an isolated canonical conversation/session; it does not create or invite a messaging bot.
- All product deployments use the same Vercel EVE runtime integration, unified bot gateway, Next.js web client, PostgreSQL/Drizzle persistence contracts and deployment/parity workflow; product-specific actions remain separately authorized.
- A proactive text message can be queued and delivered without an active or immediately preceding user turn, provided the platform permits that destination and the required consent/opt-in exists.
- One-time, recurring, interval and event-triggered schedules produce durable runs in the configured timezone and remain due/recoverable while the agent or client is offline.
- Accepted asynchronous tasks survive client disconnect and process restart; cancellation is recorded and stale workers are fenced from committing.
- Replayed or duplicate source events resume from a durable cursor and do not repeat an accepted business effect.
- A linked interaction routes an authorized provider/owner response to the originating conversation without merging private transcripts or requiring the customer to repeat the question.
- With `CHATBOT_ATTACHMENTS_ENABLED=true`, a file attached through a supported web or messaging client is received, linked to the correct canonical message/conversation and made available to the authorized agent; unsupported file processing is not assumed.
- With `CHATBOT_ATTACHMENTS_ENABLED=false`, attachment bytes are not fetched, uploaded, proxied, stored or passed to the agent; the text portion of the message still works, and attachment-related storage and AI processing are avoided.
- A transient delivery failure or duplicate event does not lose a message or repeat its associated business action.
- `search_conversation_history` finds an exact older message from before context compaction and returns its source reference and timestamp.
- The history tool cannot retrieve messages from an unrelated conversation or product scope.
- A restart does not erase canonical messages, enabled attachment metadata/references or pending outbound delivery.
- A platform-specific unsupported capability is rejected or reported explicitly, not silently treated as successful.
