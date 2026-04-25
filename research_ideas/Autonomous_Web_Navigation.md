# Universal Agentic Sidecar: The Master Blueprint for Autonomous Web Navigation & Action

## 1. Executive Summary
This blueprint defines a framework-agnostic architecture for deploying high-order AI agents on any web application with minimal code intrusion. By shifting from **Code-Level Integration** to **Runtime Semantic Observation**, the system enables an AI chatbot to navigate complex Micro-Frontends (MFEs), understand deep application states (tabs, modals, dialogs), and perform multi-step actions (form pre-filling) on behalf of the user.

---

## 2. The Core Architecture: The Semantic Sidecar
The "Sidecar" is a single, unified JavaScript bundle injected into the application. It acts as the "Nervous System" of the agent.

### 2.1 The Network Proxy (Service Worker)
Instead of monitoring framework-specific state (Redux/Vuex), we intercept the universal language of the web: **HTTP Requests**.
* **Mechanism:** A Service Worker intercepts all `fetch` and `XHR` calls.
* **Goal:** To correlate user actions (clicks) with API endpoints. This creates a "Semantic Fingerprint" for every action (e.g., "Clicking this button calls `/api/v1/orders`").

### 2.2 The DOM Observer (MutationObserver)
Since many modern applications do not change the URL for deep navigation (tabs/modals), we use **Virtual Routing**.
* **Mechanism:** A `MutationObserver` tracks large-scale DOM shifts.
* **A11y Tree Hashing:** The system generates a unique hash of the **Accessibility Tree** (Roles, Labels, and States). 
* **Virtual Routes:** Even if the URL remains `/dashboard`, a change in the A11y hash signals to the AI that it has transitioned from the "Summary View" to the "Task Detail View."

---

## 3. Autonomous Discovery Engine (The Cartographer)
To avoid manual configuration, the system "learns" the application via an automated probing phase.

### 3.1 Headless Probing (Staging/Dev)
* **Strategy:** A background process (Playwright/Puppeteer) performs a Breadth-First Search (BFS) on the UI.
* **Discovery Loop:** 1. Identify all interactive elements (`role="button"`, `role="link"`, `role="tab"`).
    2. Execute "Intent Probing": Click the element and observe the Network + DOM response.
    3. Map the Result: If a modal opens, record the link between the button and the modal state.

### 3.2 Semantic Correlation
The LLM (Gemini 3 Flash) analyzes the raw traces:
* **Trace:** `[Click: "View Billing"] -> [Network: GET /api/finance/invoices] -> [UI: Table with "Invoice #" appears]`.
* **Tool Generation:** The LLM generates a tool definition: `open_billing_invoices()`.

---

## 4. Action Execution & Semantic Injection
The most challenging part of an Agentic UI is performing actions within isolated MFEs with manual Redux state.

### 4.1 The Synthetic Event Loop
To force the application state (Redux/React/Vue) to update without touching the code, the Sidecar mimics a human user.
* **Focus:** Triggering `focus()` on the element.
* **Value Injection:** Using native property descriptors to set the value.
* **Event Dispatching:** Sending `input`, `change`, and `blur` events with `bubbles: true`. This ensures the framework's internal listeners are notified.

### 4.2 Handling Complex Elements
* **Dropdowns/Comboboxes:** The Agent clicks the trigger, waits for the DOM mutation (the list opening), finds the list item matching the semantic intent, and clicks it.
* **Forms:** The Agent maps user data to fields using **Heuristic Label Matching** (Aria-labels > Labels > Placeholders > Proximity Text).

---

## 5. Model Context Protocol (MCP) Integration
We use **MCP** as the standard communication layer between the LLM and the Browser Sidecar.

### 5.1 The Schema
The MCP Server hosts the `semantic-map.json` discovered during the probing phase. It exposes:
* **Tools:** `navigate_to(view_id)`, `fill_form(intent, data)`, `execute_action(action_id)`.
* **Resources:** `current_view_context` (The A11y tree of what the user sees right now).

### 5.2 Real-time Tool Discovery
The Chatbot queries the MCP server: "What can I do on this page?"
The Server responds: "You can 'Add a Task', 'Filter by Date', or 'Switch to the Analytics Tab'."

---

## 6. Implementation Roadmap

### Phase 1: Instrumentation (Day 1)
* Deploy the Sidecar script via the global HTML template.
* Serve the script from the application's domain to bypass CSP restrictions.
* Register the Service Worker.

### Phase 2: Autonomous Map Generation (Day 2)
* Run the "Explorer Bot" in a staging environment.
* Capture the `POST` payloads of real user form submissions to learn the API schemas.
* Generate the `manifest.json` (The Semantic Map).

### Phase 3: The Protocol Bridge (Day 3)
* Host a lightweight Node.js MCP server.
* Connect the Chatbot (Gemini) to the MCP server.
* Configure the "Confirmation UI": A safety layer where the bot shows a summary of its intended actions before execution.

### Phase 4: Production Deployment & Feedback
* Enable the Chatbot for users.
* Use "Failed Action" logs to re-probe specific routes that have changed due to MFE updates.

---

## 7. Security & Governance
* **Auth Guard:** The Sidecar only executes actions if the `fetch` interceptor confirms the user has a valid session token.
* **Data Privacy:** PII (Personally Identifiable Information) is never stored in the Semantic Map; the map only stores field labels and API patterns.
* **Execution Logs:** Every action performed by the AI is logged with a "Triggered by AI" flag for auditing.
