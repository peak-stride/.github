# PeakStride AI: Multi-Agent Triathlon Coaching System

An intelligent, multi-agent AI system providing personalized triathlon coaching through coordinated training and nutrition guidance. Built with Google's Agent Development Kit (ADK) and powered by Gemini, PeakStride demonstrates how specialized AI agents can collaborate to deliver expert-level coaching.

> **Deployment Note:** This system is designed as a modular Go library. It functions as a standalone runner for
> development [`peak-stride/ai`](https://github.com/peak-stride/ai) or as the intelligence engine embedded directly within the [`peak-stride/api`](https://github.com/peak-stride/api)
> web platform.

-----

## Problem Statement

Triathlon training is inherently complex, requiring athletes to balance three disciplines, nutrition, and recovery. Current solutions effectively force athletes to choose between **generic, static plans** that ignore individual biometrics or **expensive personal coaching** ($100-300/mo).

**The Core Challenge:** Creating an assistant that is **personalized, integrated, and accessible.**

We solve this by unifying distinct domains:

* **Data Integration:** Transforming raw data (Strava, wearables) into actionable insights.
* **Contextualization:** Moving beyond generic advice to specific guidance based on age, weight, and fitness level.
* **Holistic Coordination:** Ensuring training load and nutritional fueling are calculated together, not in silos.

-----

## Why Agents?

Traditional monolithic LLM chains struggle to maintain the context required for multi-domain coaching. We utilize a multi-agent architecture because:

1. **Specialization:** A single LLM cannot be an expert in everything. We decompose the problem into a **Training Agent** (workout logic) and a **Nutrition Agent** (dietary science), improving accuracy and reducing hallucinations.
2. **Hub-and-Spoke Delegation:** The **Orchestrator** acts as a manager. It doesn't need to know *how* to write a plan; it simply delegates to the expert, synthesizes the result, and ensures the user gets a coherent answer.
3. **Scalable Tooling:** Agents use tools independently. The **Training Agent** accesses Strava via our custom MCP server, while the **Nutrition Agent** accesses the web. This prevents a monolithic dependency graph.
4. **Stateful Reasoning:** By leveraging the ADK's memory management, agents maintain session state (e.g., "The user is injured") across interactions, enabling long-term coaching rather than one-off answers.

-----

## Architecture & Components

The system follows a hierarchical delegation pattern.

### Core Components

1. **Orchestrator Agent** (`internal/agents/orchestrator.go`)

    * **Role:** The user's primary point of contact.
    * **Logic:** Manages the athlete's profile in session memory. It routes intent to sub-agents and synthesizes their
      outputs.
    * **Tools:** `update_athlete_profile`.

2. **Training Agent** (`internal/agents/training.go`)

    * **Role:** Performance analysis and workout creation.
    * **Logic:** Analyzes historical data to ensure plans align with current fitness.
    * **Tools:** `StravaMCPTool` (fetches activities via JSON-RPC).

3. **Nutrition Agent** (`internal/agents/nutrition.go`)

    * **Role:** Dietary strategy and hydration planning.
    * **Logic:** Provides evidence-based advice tailored to the training intensity.
    * **Tools:** `GoogleSearch` (Gemini built-in).

4. **Strava MCP Server** (`pkg/strava-mcp/`)

    * **Role:** A stateless bridge between the agents and the Strava API.
    * **Tech:** Compiled as a single binary with zero runtime dependencies. Communicates via JSON-RPC 2.0 over Standard I/O (stdio).
    * **Why:** Allows the agents to safely execute code to fetch data without embedding API logic directly into the LLM prompts.

-----

## The Build

We chose **Go** and **Google ADK** for performance and type safety in a domain that requires reliability.

### Technology Stack

| Component         | Technology       | Reasoning                                                  |
|:------------------|:-----------------|:-----------------------------------------------------------|
| **Language**      | Go 1.25+         | Type-safe, high concurrency, single-binary distribution.   |
| **Framework**     | Google ADK 0.2.0 | Native support for state management and agent primitives.  |
| **Intelligence**  | Gemini 2.5 Flash | Low latency and high reasoning capability for tool use.    |
| **Protocol**      | JSON-RPC 2.0     | Standardized, language-agnostic communication for the MCP. |
| **Observability** | Uber Zap         | Structured logging for tracing complex multi-agent flows.  |

### Implementation Details

* **Stateless MCP:** The Strava MCP server does not store tokens. The `access_token` is injected into the context by the main application (or API) and passed to the binary during execution. This allows one server instance to scale to thousands of users.
* **Zero-Dependency Tools:** The MCP tools are built as standalone binaries. This decoupling means the agent runner can effectively "spawn" capabilities on demand.
* **In-Memory Services:** Currently uses the ADK's in-memory session and memory services for simplicity, designed with interfaces that allow a seamless swap to Vertex AI services for production.

-----

## Future Roadmap

If we had more time, we would expand the system from a "Smart Assistant" to a "Proactive Coach":

1. **Production Persistence:** Migrate from in-memory storage to **Vertex AI Session & Memory Bank Services** to support long-term athlete history.
2. **Deep Data Analysis:** Expand the Strava MCP to parse activity streams (Heart Rate/Power zones) and calculate fatigue/fitness scores (TSS).
3. **New Specialist Agents:**
    * **Injury Prevention Agent:** To analyze load spikes.
    * **Race Strategy Agent:** To predict finish times based on weather and course profile.
4. **Proactive Notifications:** Move beyond "Chat" to "Push"—alerting the athlete if they are undertraining or overtraining based on background data analysis.
