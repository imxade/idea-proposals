# Static Product Demo and Video Generator

## Overview

The project is a system that takes an existing authorized web product and converts it into a self-contained, backend-free, playable demo.

The main goal is not just to record a website.

The system should:

* Accept a website URL.
* Analyze the product and its user-facing flows.
* Reproduce the frontend and static assets.
* Remove dependencies on the production backend.
* Replace backend actions with deterministic mock behavior.
* Generate realistic dummy data where required.
* Build complete product workflows.
* Turn those workflows into self-playing demonstrations.
* Display contextual subtitles, hints, and explanations while the demo is playing.
* Record the self-playing demo as a video.
* Export semantic information that can later be used by an AI model for voiceover, editing, localization, showcase generation, or tutorial generation.

The primary output is therefore a **self-playing static version of the product**.

Video generation is one possible output built on top of that demo.

---

# Core Concept

The original product may depend on:

* Authentication
* APIs
* Databases
* Payment providers
* User accounts
* Server-generated data
* External services
* Backend business logic

The generated demo removes those dependencies.

Instead, the system creates a self-contained version containing:

* Frontend
* Static assets
* Local application state
* Mock data
* Mock backend behavior
* Generated workflows
* Demo playback runtime
* Tutorial overlays
* Transcript metadata

The generated application should behave similarly to the expected product experience without requiring the original production backend.

---

# Example

Consider an application with the following workflow:

1. User visits the landing page.
2. User clicks Login.
3. Authentication happens on the backend.
4. The dashboard loads.
5. User creates a project.
6. Project information is stored in a database.
7. User purchases something.
8. Payment is processed by a payment provider.
9. A success screen appears.

The generated demo would reproduce the expected experience without using those real services.

The generated version could:

* Automatically create a demo user when Login is selected.
* Open the dashboard immediately.
* Generate example projects locally.
* Store changes using local demo state.
* Simulate payment success when the Pay button is selected.
* Continue naturally to the payment success screen.

The user experiences the expected product flow even though no real authentication, database, payment processor, or backend is involved.

---

# Product Architecture

The system can be divided into several major components.

## 1. Product Explorer

The Product Explorer opens the supplied website and analyzes the application.

Its job is to understand:

* Available pages
* Navigation
* Buttons
* Forms
* Modals
* Menus
* Important controls
* Application states
* User workflows
* Network requests
* API behavior
* State transitions

Instead of blindly clicking everything, the Explorer should try to understand meaningful product actions.

For example, it should understand that:

* Login leads to a dashboard.
* New Project starts a creation flow.
* Checkout leads to payment.
* Settings opens configuration options.
* Publish changes the project into a published state.

The result should be a model of how the product behaves.

---

# 2. Product State Graph

The system should convert what it discovers into a graph of meaningful product states and transitions.

An example graph might include:

* Landing Page

  * Login

    * Dashboard

      * Create Project

        * Project Workspace

          * Edit
          * Share
          * Publish
      * Settings
      * Billing
  * Pricing

    * Checkout

      * Payment Success

This graph becomes the foundation for generating product demonstrations.

The system should focus on **meaningful workflows** rather than attempting to enumerate every theoretically possible state.

Web applications can contain effectively unlimited states because of:

* Search
* Free-form text
* Pagination
* Dates
* Generated content
* Filters
* User-created objects
* Repeated actions

The practical goal should therefore be:

> Discover and demonstrate the meaningful product workflows within a defined exploration budget.

---

# 3. Static Product Generator

After understanding the application, the system generates a self-contained version of the product.

It should preserve:

* Layout
* Styles
* Components
* Navigation
* Animations where practical
* Images
* Icons
* Forms
* Product interactions

Production dependencies should be removed or replaced.

The generated product should not require:

* Production APIs
* Production authentication
* Production databases
* Production payment providers
* Private credentials
* Real customer data

The generated application can still use JavaScript and local state.

It does not need to be pure static HTML.

A better description is:

> A self-contained static application.

---

# 4. Mock Backend Runtime

Backend-dependent behavior should be replaced by a deterministic local runtime.

For example:

### Authentication

When the user selects Login:

* A demo user is created or loaded.
* The user becomes authenticated locally.
* The expected dashboard appears.

### Project Creation

When the user creates a project:

* A project object is generated locally.
* The project becomes available inside the generated application.
* The user is taken to the expected workspace.

### Payments

When the user selects Pay:

* A successful demo transaction is generated.
* The order becomes paid.
* The expected success screen appears.

### Other APIs

API responses can be replaced with generated fixtures and local state transitions.

The frontend should behave as naturally as possible without knowing that the production backend has been removed.

---

# 5. Dummy Data Generator

Many workflows require data.

The system should automatically generate realistic synthetic data such as:

* User names
* Email addresses
* Projects
* Orders
* Products
* Messages
* Notifications
* Analytics
* Billing information
* Activity history
* Team members
* Configuration values

The generated data should be deterministic when required so that recordings remain consistent.

The same demo flow should ideally produce the same visible result every time.

---

# 6. Flow Generator

Once the product has been modeled, an agent generates useful product flows.

Possible flows might include:

* Complete Product Tour
* Signup
* Login
* Create Project
* Configure Project
* Invite Team Member
* Checkout
* Upgrade Plan
* Publish Project
* Settings
* Reporting
* Complete End-to-End Workflow

Each flow should be described as semantic steps rather than only low-level browser interactions.

For example, several individual actions might be combined into one presentation step:

### Configure Your Project

Enter the basic project details and continue to create the workspace.

The underlying automation may perform several clicks and form operations, but the viewer only needs one meaningful explanation.

---

# 7. Demo Playback Runtime

The generated product should contain its own playback system.

A normal URL can allow manual interaction with the generated product.

A playback route can automatically execute a flow.

For example, the conceptual structure could be:

* Normal product demo
* Play complete flow
* Play signup flow
* Play checkout flow
* Play project creation flow

Additional options could control:

* Playback speed
* Automatic playback
* Captions
* Hints
* Cursor visibility
* Highlighting
* Tutorial mode

The important idea is that the generated application itself knows how to play the workflow.

Playwright can still be used for:

* Exploration
* Validation
* Testing
* Recording
* Automated screenshots
* Ensuring generated flows work correctly

But the final product demonstration should ideally be executable directly by the generated frontend.

---

# 8. Tutorial Overlay

A tutorial presentation layer should be built directly into the generated frontend.

While a flow is playing, a dedicated region should explain what is happening.

The overlay could appear:

* At the bottom
* At the top
* In a side panel
* As a floating card
* Near the active element

A bottom presentation region may contain:

* Step number
* Step title
* Short explanation
* Progress
* Optional subtitle

For example:

### Step 4 of 10

**Complete Checkout**

Confirm the order to finish the purchase.

The overlay should update automatically as the workflow progresses.

---

# 9. Three Levels of Explanation

Every semantic step should ideally contain three types of text.

## Title

Very short description of the step.

Example:

**Create Your Project**

## Hint

A concise explanation suitable for displaying directly inside the frontend.

Example:

Enter the project details and select Create.

## Narration

A richer explanation intended for voiceover or transcript generation.

Example:

Let's create your first project. Enter the basic information and then continue to open the new project workspace.

The generated frontend may display only the title and hint.

The narration can remain available as metadata for later AI processing.

---

# 10. Smart Presentation Agent

The presentation system should understand the difference between automation and storytelling.

A browser flow may internally perform:

* Focus input
* Type project name
* Open dropdown
* Select category
* Change another value
* Click Create
* Wait for navigation

A tutorial should not describe every individual action.

Instead, the Presentation Agent might convert all of those actions into:

### Configure Your Project

Add the basic project details and create the workspace.

This produces a much cleaner tutorial.

The system should therefore maintain two levels of information.

## Execution Timeline

Contains all low-level browser actions.

## Presentation Timeline

Contains meaningful human-readable product steps.

This separation is important for producing professional-looking demonstrations.

---

# 11. Visual Presentation Features

The generated player should support presentation effects such as:

* Animated cursor
* Smooth cursor movement
* Click indicators
* Element highlighting
* Spotlight effects
* Automatic zoom
* Focus on important controls
* Step transitions
* Progress indicators
* Section titles
* Contextual hints
* Subtitles
* Success messages

These effects should make the generated demo feel intentionally produced rather than simply being browser automation.

---

# 12. Transcript Generation

Every generated flow should automatically produce a transcript.

The transcript should be synchronized with the demo timeline.

It should describe:

* What screen is being shown
* What action is taking place
* Why the action matters
* What the user should notice
* What result was produced

The transcript becomes useful for several downstream applications.

---

# 13. Video Generation

The playable static demo can then be recorded.

The recording process can:

1. Launch the generated demo.
2. Start the selected workflow.
3. Run the workflow automatically.
4. Display tutorial hints and subtitles.
5. Capture the browser window.
6. Produce the final video.

Because the demo is deterministic, recordings should be reproducible.

The recording does not need to interact with production systems.

---

# 14. Voiceover Generation

The transcript and semantic flow information can be provided together with the recorded video to another AI agent.

The voiceover agent can then generate different narration styles.

Examples include:

* Product showcase
* SaaS launch video
* Beginner tutorial
* Technical walkthrough
* Sales demonstration
* Feature announcement
* Customer onboarding
* Help-center tutorial
* Social media product demo

The same recording could therefore produce several different videos without recreating the product workflow.

---

# 15. Why Semantic Metadata Matters

A video-only AI model can attempt to understand what is happening visually.

However, the system already knows what every interaction means.

It should therefore preserve that information.

Instead of providing only a video, downstream systems can receive:

* Video
* Product description
* Flow description
* Step titles
* Hints
* Transcript
* Timestamps
* Important UI elements
* Semantic actions

This makes later AI generation significantly more reliable.

---

# 16. Other Outputs

The generated demo should not be limited to video.

The same underlying product representation could produce:

* Interactive product demo
* Tutorial video
* Product showcase video
* Voiceover script
* Help-center article
* Step-by-step documentation
* Screenshots
* GIF demonstrations
* Sales material
* Onboarding guides
* Product walkthroughs
* Localized tutorials
* Training material

This significantly increases the value of the system.

---

# 17. Suggested Generated Project Structure

Each generated product should logically contain several sections.

## Application

Contains the copied or reconstructed frontend and its assets.

## Mock Runtime

Contains:

* Local product state
* Generated API behavior
* Synthetic fixtures
* Backend replacement logic

## Demo Runtime

Contains:

* Flow player
* Cursor system
* Highlight system
* Tutorial overlay
* Presentation controls

## Flows

Contains generated product workflows such as:

* Full Tour
* Signup
* Checkout
* Project Creation
* Publishing
* Settings

## Transcript Data

Contains:

* Step descriptions
* Narration
* Captions
* Timing information
* Presentation metadata

---

# 18. Agent Responsibilities

The main agent can be divided into four conceptual roles.

## Product Explorer

Understands what the application contains and how it works.

## Demo Compiler

Transforms backend-dependent functionality into deterministic local behavior.

## Flow Generator

Identifies useful product workflows and generates executable demonstrations.

## Presentation Director

Decides:

* What should be shown
* What should be skipped
* What should be highlighted
* What should be explained
* How long to pause
* Where subtitles should appear
* What narration should say

The Presentation Director is what turns browser automation into a useful tutorial.

---

# 19. Different Fidelity Levels

The system should support multiple levels of product access.

## URL-Only Mode

The system receives only a public URL.

It can accurately reproduce publicly accessible parts of the application.

For inaccessible functionality, it may generate synthetic or inferred states.

This mode provides the lowest fidelity for authenticated features.

## URL With Demo Credentials

The system receives authorized test credentials.

The agent can explore authenticated areas and reproduce workflows much more accurately.

This provides significantly better product coverage.

## Repository or API Access

The system receives authorized access to:

* Source code
* API schema
* Routes
* Components
* Product definitions

This allows the highest fidelity version of the generated demo.

---

# 20. Important Limitation

A system cannot accurately reproduce information it has never been able to observe.

For example, if a dashboard only exists after authentication and the system only receives the public landing page, it cannot know exactly what the private dashboard contains.

In that situation, it can:

* Infer likely behavior
* Generate placeholder states
* Create synthetic demonstrations

However, these should be treated as generated approximations rather than exact copies.

Authorized access to test accounts or source information provides much better results.

---

# 21. Recommended Product Positioning

The product should not primarily be described as a screen recorder.

It is also broader than a simple video generator.

A better description is:

> A system that converts an existing web product into a deterministic, self-playing, backend-free product demonstration.

From that generated demonstration, the system can automatically produce:

* Product tutorials
* Showcase videos
* Voiceovers
* Documentation
* Interactive walkthroughs
* Localized demonstrations
* Sales demos

---

# Final Workflow

The complete concept can be summarized as:

**Website URL**

↓

**Product Exploration**

↓

**Product State and Workflow Discovery**

↓

**Frontend Reproduction**

↓

**Backend Dependency Removal**

↓

**Mock Data and Local State Generation**

↓

**Generated Product Flows**

↓

**Tutorial and Presentation Metadata**

↓

**Self-Playing Static Product**

↓

**Automatic Playback**

↓

**Video Recording**

↓

**Video + Transcript + Semantic Metadata**

↓

**AI Voiceover / Showcase / Tutorial / Localization / Editing**

---

# Core Product Idea

The central idea is:

> Take the static and observable part of an authorized product, reproduce its expected backend-driven behavior using deterministic local state and mock data, and transform it into a self-playing product experience.

A playback route starts the complete tutorial automatically.

During playback, the generated frontend:

* Performs the user workflow.
* Moves through expected product states.
* Shows contextual hints.
* Displays subtitles.
* Highlights important UI elements.
* Tracks progress.
* Maintains a synchronized transcript.

The self-playing experience can then be recorded directly into a video.

That video, together with its transcript and semantic metadata, can be given to an AI model to create polished product showcases, tutorials, voiceovers, localized content, documentation, and other derivative material.
