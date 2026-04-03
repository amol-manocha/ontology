# Auto Insurance Claims Ontology — Microsoft Fabric IQ

A complete, end-to-end example of building an **ontology-backed Data Agent** in Microsoft Fabric. This project demonstrates how to model an auto insurance claims domain as a Fabric Ontology — defining entity types, properties, relationships, and data bindings — then wire it up as the semantic foundation for a conversational Data Agent that answers natural-language business questions over lakehouse data.

---

## Why Ontology?

Enterprise AI agents that query raw SQL tables struggle with semantic ambiguity — the same column name means different things in different tables, join paths are implicit, and business logic lives in undocumented stored procedures. **Ontologies solve this** by providing a formal semantic layer between human intent and table-level data.

The research is compelling:

| Metric | SQL-Only | With Ontology | Source |
|--------|----------|---------------|--------|
| LLM accuracy on enterprise questions | 16% | 72% | [Sequeda et al. 2023](https://arxiv.org/abs/2311.07509) |
| Schema-intensive question accuracy | 0% | 76% | [Allemang & Sequeda 2024](https://arxiv.org/html/2405.11706v1) |
| RAG retrieval quality (MRR) | baseline | +77.6% | [Xu et al., LinkedIn](https://arxiv.org/html/2404.17723v1) |

The ontology gives the agent a "dictionary" of how business concepts relate, which tables contain what, and what the valid reasoning paths are — improving accuracy by **4.2x** over SQL-only approaches.

---

## What's in This Repo

```
├── data/                    # Sample dimension & fact tables (CSV)
│   ├── customer_dim.csv     # 9 customers with demographics
│   ├── policy_dim.csv       # Insurance policies
│   ├── vehicle_dim.csv      # Insured vehicles
│   ├── agent_dim.csv        # Insurance agents
│   ├── adjuster_dim.csv     # Claims adjusters
│   ├── coverage_dim.csv     # Coverage types & limits
│   ├── incident_dim.csv     # Incident records
│   ├── claims_fact.csv      # Claims with all FK relationships
│   ├── payments_fact.csv    # Claim payment transactions
│   └── repair_shop_dim.csv  # Certified repair shops
│
├── notebooks/
│   ├── create_lakehouse_tables.ipynb      # Fabric notebook — loads CSVs into managed Delta tables
│   └── autoclaims_ontology_setup.ipynb   # Fabric notebook — creates the full ontology via REST API
│
├── docs/
│   ├── ontology-backed-ai-agents-report.md   # Deep-dive research report with citations
│   ├── autoclaims_demo.pptx                  # Presentation deck
│   └── autoclaims_demo.pdf                   # Presentation (PDF)
│
├── visualization/
│   └── auto-ontology.html   # Interactive ontology graph explorer (standalone HTML)
│
├── .gitignore
└── README.md
```

---

## Domain Model

The ontology models the full **auto insurance claims lifecycle** with 10 entity types and 11 relationships:

```
                    ┌──────────┐
                    │  Agent   │
                    └────┬─────┘
                         │ manages
                    ┌────▼─────┐        ┌──────────┐
 ┌──────────┐       │  Policy  │◄───────│ Coverage │
 │ Customer │◄──────┤          ├───────►│          │
 └────┬─────┘ held  └────┬─────┘ under  └──────────┘
      │ filed by          │ covers
      │              ┌────▼─────┐
      └─────────────►│  Vehicle │
                     └──────────┘
                          │ involves
      ┌──────────┐  ┌────▼─────┐  ┌────────────┐
      │ Incident │◄─┤  Claim   ├─►│  Adjuster  │
      └──────────┘  └────┬─────┘  └────────────┘
        results from      │ sent to
                    ┌─────▼──────┐
                    │ RepairShop │
                    └────────────┘
                          │ settles
                    ┌─────▼──────┐
                    │  Payment   │
                    └────────────┘
```

### Entity Types

| Entity | Source Table | Key Column | Description |
|--------|------------|------------|-------------|
| Customer | `customer_dim` | `customer_id` | Policyholders with demographics and contact info |
| Policy | `policy_dim` | `policy_id` | Insurance policies with terms, premiums, and status |
| Vehicle | `vehicle_dim` | `vehicle_id` | Insured vehicles (VIN, make, model, market value) |
| Agent | `agent_dim` | `agent_id` | Licensed insurance agents managing policies |
| Adjuster | `adjuster_dim` | `adjuster_id` | Claims adjusters with specialization and region |
| Coverage | `coverage_dim` | `coverage_id` | Coverage types, limits, deductibles, and premiums |
| Incident | `incident_dim` | `incident_id` | Incident records (type, location, weather, police report) |
| Claim | `claims_fact` | `claim_id` | Central hub — links policy, customer, vehicle, incident, adjuster, and repair shop |
| Payment | `payments_fact` | `payment_id` | Claim settlement payments with method and status |
| RepairShop | `repair_shop_dim` | `shop_id` | Certified repair facilities with network tier |

### Relationships

| Relationship | Source → Target | Binding Table |
|-------------|----------------|---------------|
| PolicyHeldByCustomer | Policy → Customer | `policy_dim` |
| PolicyManagedByAgent | Policy → Agent | `policy_dim` |
| VehicleCoveredByPolicy | Vehicle → Policy | `vehicle_dim` |
| CoverageUnderPolicy | Coverage → Policy | `coverage_dim` |
| ClaimUnderPolicy | Claim → Policy | `claims_fact` |
| ClaimFiledByCustomer | Claim → Customer | `claims_fact` |
| ClaimInvolvesVehicle | Claim → Vehicle | `claims_fact` |
| ClaimResultsFromIncident | Claim → Incident | `claims_fact` |
| ClaimAssignedToAdjuster | Claim → Adjuster | `claims_fact` |
| ClaimSentToRepairShop | Claim → RepairShop | `claims_fact` |
| PaymentSettlesClaim | Payment → Claim | `payments_fact` |

---

## Getting Started

### Prerequisites

- A **Microsoft Fabric** workspace with capacity (F16 or higher)
- A **Lakehouse** created in the workspace
- Fabric **Ontology** feature enabled (preview)

### Step 1 — Load Sample Data

1. Upload the CSV files from [data/](data/) into your Fabric Lakehouse **Files/** area (drag-and-drop in the Lakehouse explorer)
2. Open [notebooks/create_lakehouse_tables.ipynb](notebooks/create_lakehouse_tables.ipynb) in your Fabric workspace
3. Set `LAKEHOUSE_NAME` in the configuration cell to match your Lakehouse name
4. Run all cells — the notebook discovers the CSVs, creates managed Delta tables with explicit schemas, and verifies row counts

The notebook is **idempotent** — re-running it overwrites existing tables.

### Step 2 — Create the Ontology

Open [notebooks/autoclaims_ontology_setup.ipynb](notebooks/autoclaims_ontology_setup.ipynb) in your Fabric workspace. Update the configuration cell:

```python
WORKSPACE_NAME = "your-workspace-name"
LAKEHOUSE_NAME = "your-lakehouse-name"
ONTOLOGY_NAME  = "AutoClaimsOntology"
```

Run all cells. The notebook:
1. Resolves workspace and lakehouse IDs via the Fabric REST API
2. Reads table schemas and maps Spark types to ontology value types
3. Builds entity type definitions with properties and data bindings
4. Creates relationship types with contextualizations (FK bindings)
5. Submits the full ontology definition via `POST /v1/workspaces/{id}/ontologies`

The notebook is **idempotent** — re-running it deletes any existing ontology with the same name before recreating it.

### Step 3 — Build a Data Agent

1. In your Fabric workspace, create a new **Data Agent**
2. Add the ontology you just created as a **knowledge source**
3. Add the following **agent instructions** — these guide the agent's behavior, output formatting, and analytical defaults:

```
### Role & Objective
- You are an auto insurance analytics agent.
- Your purpose is to analyze auto claims data to identify trends, patterns, comparisons, and potential business insights.
- Default to descriptive and diagnostic analytics unless the user explicitly asks for forecasting or predictions.

### Data & Analysis Assumptions
- If no time range is specified, default to the most recent complete period available.
- Aggregate data at a level appropriate to the question (e.g., by month, product, customer, or region).
- Clearly indicate when results are based on limited or incomplete data.
- If someone references a claim or claims with a dollar amount such as "claims over $20,000", this is referencing claim_amount.
- Unless otherwise mentioned, always use ClaimDate for any date sent by the user.

### Output Formatting (Strict)
- Always return results in a clearly formatted table.
- Use human-readable column names and include units where applicable (e.g., $, %, quantity).
- If no data is available, return an empty table with column headers and note this in the summary.

### Summary Requirements
- Always include up to 2 concise sentences summarizing the most important insight from the results.
- The summary should highlight trends, notable changes, comparisons, or outliers when present.

### Follow-Up Questions
- Provide 2–3 relevant follow-up questions that could deepen insight or support decision-making.
- Follow-up questions should be actionable and related to:
  - Time trends
  - Product or customer performance
  - Regional or category comparisons
- Do not repeat the original question in a different form.

### Clarity & Integrity
- Do not make assumptions beyond the available data.
- If a question cannot be answered with the data provided, clearly state the limitation and suggest a related question that can be answered.

Support GROUP BY in GQL
```

4. The agent will use the ontology to resolve natural-language questions into GQL graph traversals

### Step 4 — Ask Questions

Once the agent is configured, try these sample questions:

**1-hop queries** (single relationship traversal):
- "Who is the adjuster on claim CLM004?"
- "What coverage types does Policy POL003 include?"
- "Show me all claims for customer David Kim"

**2-hop queries** (multi-relationship traversal):
- "What vehicles are covered under policies managed by agent Laura Chen?"
- "Find all payments for claims involving 2023 Honda CR-Vs"
- "Which repair shops handled claims from customers in Atlanta?"

---

## Interactive Visualization

Before diving into a live demo, use [visualization/auto-ontology.html](visualization/auto-ontology.html) as a **client-facing walkthrough** to communicate the value of an ontology-backed approach. Open it in any browser — no dependencies or login required — to illustrate how the auto claims domain is modeled, how entities relate to each other, and why this semantic foundation enables more accurate AI agents.

Use it to:

- **Explain the business case** — Show stakeholders how an ontology bridges the gap between raw database tables and natural-language questions, and why that matters for accuracy and governance
- **Walk through the domain model** — Click any entity node to inspect its properties, data types, and connected relationships; click any relationship edge to see the underlying key bindings and contextualization details
- **Demonstrate multi-hop reasoning** — Highlight how the agent traverses defined relationships (e.g., Customer → Policy → Claim → Payment) rather than guessing SQL join paths
- **Tailor the conversation** — Drag nodes to rearrange the layout and focus on the entities most relevant to your audience; entities are color-coded by domain group (Parties, Policy & Coverage, Claims Lifecycle, Service Providers)

This visualization pairs well with the [research report](docs/ontology-backed-ai-agents-report.md) for audiences that want to see the benchmarks behind the approach before seeing it in action.

---

## Research Report

The [docs/ontology-backed-ai-agents-report.md](docs/ontology-backed-ai-agents-report.md) provides an in-depth analysis of why ontology-backed AI agents outperform SQL-only approaches, covering:

- **Accuracy benchmarks** — The Sequeda-Allemang studies showing 4.2x accuracy improvement
- **GQL vs. SQL** — Why graph traversal outperforms multi-JOIN SQL for relationship-heavy queries
- **Hallucination reduction** — How explicit graph structures reduce ungrounded LLM responses
- **Governance** — Single semantic layer eliminating conflicting business definitions
- **Microsoft Fabric IQ** — Architecture of Ontology, Graph, Data Agents, and NL2Ontology
- **Real-world cases** — Insurance fraud detection, LinkedIn customer service, financial anti-fraud

All claims are sourced from 21 references including 4 peer-reviewed papers.

---

## Key Concepts

### What is a Fabric Ontology?

A Fabric Ontology is a **semantic layer** that sits on top of Lakehouse tables. It translates raw data into named, typed business entities and relationships — the same concepts used in everyday business conversations. Each entity maps a table to a business concept (`claims_fact` → `Claim`), and each relationship wires entities together via foreign keys with a business name (`ClaimFiledByCustomer`).

### Why Not Just Use SQL?

SQL schemas encode structure but not meaning. Column names like `cust_rev_ytd_adj` are opaque to both humans and AI. Join paths are implicit in foreign keys that the LLM must discover. Without formal semantics, the agent is guessing — and on complex enterprise schemas, it guesses wrong 84% of the time ([Sequeda et al.](https://arxiv.org/abs/2311.07509)).

### How the Data Agent Uses the Ontology

1. User asks a natural-language question
2. Agent resolves business terms against the ontology's named entities and relationships
3. Agent generates a GQL graph query that traverses the defined relationships
4. Results are returned in business language — no GQL or schema knowledge required

The ontology is the bridge between **human intent** and **table-level data**.

---

Microsoft Fabric Ontology is currently in preview. Features and capability may change before general availability.



