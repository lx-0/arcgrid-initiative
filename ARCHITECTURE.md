# ArcGrid Initiative — System Architecture

## Overview

ArcGrid is a distributed clean-energy platform that simulates and manages networks of arc-reactor-inspired power nodes. The system models energy generation, routing, and consumption across grid topologies — serving both civilian infrastructure and defense installations.

This document defines the major components, their boundaries, data flow, and technology choices.

## System Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Dashboard UI (React)                      │
│              Real-time grid visualization                    │
│              Telemetry panels & alerts                       │
├──────────────────────┬──────────────────────────────────────┤
│        REST API      │         WebSocket                     │
│      (read/write)    │     (live telemetry)                  │
├──────────────────────┴──────────────────────────────────────┤
│                  Grid Management API                         │
│           Express/Fastify · OpenAPI spec                     │
│      Node CRUD · Grid ops · Auth · Telemetry                 │
├─────────────────────────────────────────────────────────────┤
│                  Simulation Engine                            │
│    ┌──────────┐  ┌───────────┐  ┌──────────────────┐        │
│    │  Node    │  │  Grid     │  │  Power Routing   │        │
│    │  Model   │  │  Topology │  │  Algorithm       │        │
│    └──────────┘  └───────────┘  └──────────────────┘        │
│    ┌──────────────────────────────────────────────┐          │
│    │           Tick Engine (time-step loop)       │          │
│    └──────────────────────────────────────────────┘          │
├─────────────────────────────────────────────────────────────┤
│                   Data Layer                                 │
│           PostgreSQL · Time-series store                     │
└─────────────────────────────────────────────────────────────┘
```

### 1. Simulation Engine

The core of ArcGrid. Runs discrete time-step simulations of energy grids.

**Sub-components:**

- **Node Model** — Represents a single power node: capacity (MW), current output, operational status (online/offline/degraded), location, and connections. Nodes are the atomic unit of the grid.
- **Grid Topology** — Manages the graph structure of nodes and edges (power lines). Supports adding/removing nodes and connections, querying neighbors, and detecting disconnected segments.
- **Power Routing Algorithm** — Determines how energy flows from sources to consumers. Uses load-aware shortest-path routing. Handles rerouting when nodes fail or segments are isolated.
- **Tick Engine** — Drives simulation forward in discrete time steps. Each tick: updates node states, runs the routing algorithm, records telemetry, and emits events for state changes (failures, overloads, reroutes).

**API boundary:** The simulation engine exposes a programmatic TypeScript API (not HTTP). The Grid Management API layer wraps it for external access.

```typescript
// Simulation Engine public interface (conceptual)
interface SimulationEngine {
  createNode(config: NodeConfig): Node;
  removeNode(nodeId: string): void;
  connect(nodeA: string, nodeB: string, capacity: number): Edge;
  disconnect(edgeId: string): void;
  tick(): TickResult;
  getGridState(): GridState;
  getNodeTelemetry(nodeId: string): Telemetry;
}
```

### 2. Grid Management API

REST API that wraps the simulation engine for external consumers (dashboard, CLI tools, integrations).

**Responsibilities:**
- Node CRUD — Register, update, decommission power nodes
- Grid operations — Connect/disconnect nodes, trigger simulation ticks, set operating modes
- Telemetry — Query current and historical grid metrics
- Auth — API key authentication, role-based access (operator vs. observer)

**Key endpoint groups:**

| Group | Endpoints | Description |
|-------|-----------|-------------|
| Nodes | `GET/POST/PATCH/DELETE /api/nodes` | CRUD for power nodes |
| Edges | `GET/POST/DELETE /api/edges` | Manage connections between nodes |
| Grid | `GET /api/grid/state`, `POST /api/grid/tick` | Grid-wide state and simulation control |
| Telemetry | `GET /api/telemetry/nodes/:id`, `GET /api/telemetry/grid` | Current and historical metrics |
| Alerts | `GET /api/alerts` | Active alerts (overloads, failures) |
| Auth | `POST /api/auth/token` | API key exchange |
| Defense | `POST /api/grid/mode`, `POST /api/grid/isolate` | Hardened mode, segment isolation |

**API boundary:** All external access goes through this layer. The simulation engine is never accessed directly by the dashboard or external systems.

### 3. Dashboard UI

Web-based interface for grid operators to visualize, monitor, and control the energy grid.

**Capabilities:**
- **Grid Map** — Visual graph of nodes and power flow, with node status indicators (color-coded: green/yellow/red)
- **Telemetry Panels** — Real-time charts for capacity, load, efficiency, and output per node and grid-wide
- **Alert Feed** — Live feed of node failures, overload warnings, and rerouting events
- **Control Panel** — Trigger simulation ticks, toggle hardened mode, isolate grid segments

**Real-time updates:** The dashboard connects via WebSocket to receive live telemetry and event streams. REST API is used for on-demand queries and mutations.

### 4. Defense & Resilience Layer

Cross-cutting capabilities for defense-site deployments. Not a separate service — these are features embedded in the simulation engine and exposed through the API.

- **N+1 Redundancy** — Ensures critical grid segments have backup capacity. The routing algorithm factors in redundancy requirements when computing paths.
- **Threat-Aware Rerouting** — Isolates compromised grid segments and reroutes power around them automatically.
- **Hardened Mode** — Locks the grid to essential-only routing, disabling non-critical paths to minimize attack surface.

## Data Flow

```
User/Operator ──→ Dashboard UI
                      │
              REST / WebSocket
                      │
                      ▼
            Grid Management API ──→ Auth check
                      │
                      ▼
              Simulation Engine
                 │         │
           ┌─────┘         └─────┐
           ▼                     ▼
     Grid Topology         Tick Engine
     (graph state)         (step loop)
           │                     │
           └──────┬──────────────┘
                  ▼
        Power Routing Algorithm
                  │
                  ▼
            State Updates
           ┌──────┴──────┐
           ▼              ▼
      PostgreSQL    WebSocket push
    (persistence)   (live events)
```

**Write path:** Operator creates/modifies nodes via Dashboard → REST API → Simulation Engine mutates grid state → persisted to PostgreSQL.

**Simulation path:** Tick triggered (manual or automated) → Tick Engine iterates all nodes → Power Routing computes flows → state deltas persisted → telemetry emitted via WebSocket.

**Read path:** Dashboard queries REST API for current state/history → API reads from engine in-memory state (current) or PostgreSQL (historical).

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Language | TypeScript | Unified language across all layers. Strong typing catches grid model bugs at compile time. Large ecosystem for web and API development. |
| Runtime | Node.js | Async event-driven model fits the simulation's event-emitting architecture. Fast iteration for a new project. |
| API Framework | Fastify | Faster than Express with comparable DX. Built-in schema validation via JSON Schema pairs well with OpenAPI. |
| API Spec | OpenAPI 3.1 | Industry standard. Generates client SDKs, documentation, and request validation from a single source of truth. |
| Dashboard | React + Vite | React for component model, Vite for fast builds. Widely understood, large component ecosystem for data visualization. |
| Visualization | D3.js or React Flow | Graph visualization for grid topology. D3 for custom rendering, React Flow if interactive node editing is prioritized. |
| Real-time | WebSocket (ws) | Native browser support, low-latency push for telemetry. Simple protocol for event streaming. |
| Database | PostgreSQL | Reliable, well-understood. JSONB columns for flexible node metadata. PostGIS extension available if geospatial queries are needed later. |
| Telemetry Storage | TimescaleDB (PostgreSQL extension) | Time-series queries on telemetry data without introducing a separate database. Runs as a PostgreSQL extension — single database to operate. |
| Testing | Vitest | Fast, native TypeScript support, compatible with the Vite toolchain. |
| Monorepo | pnpm workspaces | Package management for multi-package repo (engine, api, dashboard). Efficient disk usage via content-addressable storage. |

**Why TypeScript end-to-end:** A single language reduces context-switching cost and lets us share types between engine, API, and dashboard. If simulation performance becomes a bottleneck (unlikely for v1 grid sizes), the engine's hot path can be extracted to a Rust/WASM module without changing the API boundary.

## Repository Structure

```
arcgrid-initiative/
├── packages/
│   ├── engine/           # Simulation engine (Node model, Topology, Routing, Tick)
│   │   ├── src/
│   │   │   ├── node.ts
│   │   │   ├── topology.ts
│   │   │   ├── routing.ts
│   │   │   ├── tick-engine.ts
│   │   │   └── index.ts
│   │   └── package.json
│   ├── api/              # Grid Management API (Fastify + OpenAPI)
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   ├── middleware/
│   │   │   └── server.ts
│   │   └── package.json
│   └── dashboard/        # Dashboard UI (React + Vite)
│       ├── src/
│       │   ├── components/
│       │   ├── hooks/
│       │   └── App.tsx
│       └── package.json
├── ARCHITECTURE.md
├── README.md
├── ROADMAP.md
├── package.json          # Root workspace config
├── pnpm-workspace.yaml
└── tsconfig.base.json    # Shared TypeScript config
```

## Key Design Decisions

1. **Simulation engine as a library, not a service.** The engine runs in-process with the API server. This avoids IPC overhead and keeps the architecture simple for v1. If we need to scale simulation independently, we extract it to a separate process later — the TypeScript API boundary makes this straightforward.

2. **Discrete time-step simulation, not continuous.** Ticks are explicit (triggered via API or on interval). This is simpler to reason about, test, and debug than a continuous event loop. Grid sizes for v1 (hundreds to low thousands of nodes) don't require real-time physics simulation.

3. **PostgreSQL for everything.** One database for grid state, telemetry (via TimescaleDB extension), and configuration. Operational simplicity over specialized databases. We can revisit if telemetry volume exceeds what TimescaleDB handles well.

4. **WebSocket for live data, REST for everything else.** The dashboard needs sub-second telemetry updates. WebSocket handles that. All mutations and queries go through REST for simplicity and cacheability.

5. **Monorepo with shared types.** Engine, API, and dashboard share TypeScript interfaces. A node type change in the engine is immediately visible in the API and dashboard at compile time. Prevents the drift that happens with separate repos.
