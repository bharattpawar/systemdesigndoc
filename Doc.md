# System Design Validation (Backend) — simple

This is the validation that runs when frontend calls:

- `POST /v1/system-design/analyze`

Code lives in:

- `src/modules/systemDesign/systemDesign.service.ts`
- `src/modules/systemDesign/graphRuleChecks.ts`

---

## What is already implemented

When `SystemDesignService.analyzeArchitecture(...)` runs, it does **Stage A validation first** (before calling AI):

1. **At least 1 edge required**
   - error `400`: `Architecture graph must include at least one connection`
2. **Edges must point to real nodes**
   - if an edge references missing `sourceNodeId/targetNodeId`
   - error `400`: `Some components are not connected/ never used by the system`
3. **No totally-disconnected components**
   - node has **0 incoming** AND **0 outgoing**
   - error `400`: `Disconnected components found: ...`
4. **No forbidden direct connections (rule engine)**
   - example: `Client -> Database` is forbidden
   - error `400`: `Invalid architecture connections: <source> -> <target>: <reason>`
5. **Client + Database cannot exist without an app/compute service**
   - catches designs like `Client -> Cache -> Database` with no service layer
   - error `400`: `No application service is present in the architecture`

---

## Where each check happens

- Stage A entrypoint: `src/modules/systemDesign/systemDesign.service.ts` → `validateGraphStructure(...)`
- Graph conversion: `src/modules/systemDesign/systemDesign.service.ts` → `buildGraphAdjacencyList(...)`
- Forbidden edges: `src/modules/systemDesign/graphRuleChecks.ts` → `checkForbiddenArchitectureEdges(...)`

---

## What is NOT implemented yet

- Stage B (requirement coverage scoring from rubric JSON)
- Deeper “arrows meaningful” semantics (beyond forbidden-edge rules)
- Returning a full `violations[]` array to frontend (right now it returns one combined error message)
- Persisting/using C4 diagram level or per-level problem formats on backend (frontend supports it locally; backend support pending)

---

## How to add a new forbidden connection

Edit `src/modules/systemDesign/graphRuleChecks.ts`:

- Add patterns to `SOURCE` / `TARGET` (if needed)
- Add a new entry to `RULES` with a clear `reason`

---

# Frontend additions (System Design UI)

These are **frontend-only changes** added to the System Design UI. Backend is **not** updated yet for these fields.

## B: User picks the diagram level (C4 Abstraction)

When a user selects/works on a System Design problem, the UI shows a diagram-level selector with 3 options:

- **Beginner** → **System Context Diagram**
- **Intermediate** → **Container Diagram**
- **Advanced** → **Component Diagram**

Current implementation details:

- Stored in browser `localStorage` **per problem**
- Also sent in `POST /v1/system-design/analyze` request body under `meta.diagramLevel`
  - Values: `beginner | intermediate | advanced`
  - Backend currently ignores this (until backend is updated)

## Admin panel: 3 formats per question (local-only)

In Admin → System Design Problems, the create/edit form has 3 text areas to add different formats for the same question:

- Beginner (System Context)
- Intermediate (Container)
- Advanced (Component)

Current implementation details:

- Saved to browser `localStorage` (keyed by `problemId`)
- Not included in create/update payload yet (backend support pending)


## Stage 2: 

## 1. Schema Changes
A new optional field has been added to the **SystemDesignProblem** model to support structured grading:

* **Field:** `evaluationSpec`
* **Type:** `Json?` (Optional)

---

## 2. Backend Validation
The backend now includes strict validation for Stage B specifications and analysis levels:

* **Evaluation Spec:** Validates the structure of `evaluationSpec` during problem creation and updates.
* **Abstraction Level:** Validates the `abstractionLevel` parameter within analyze requests.

---

## 3. Deterministic Stage B Evaluator
A new module has been implemented to handle Stage B scoring with high predictability.

### Capability Extraction
The evaluator parses the submitted design to infer capabilities based on:
* **Nodes:** Type, labels/names, and configuration.
* **Edges:** Relationships and edge-specific configurations.
* **Context:** Optional text metadata provided with the design.

### Requirement Evaluation
Detected capabilities are compared against the selected level rubric using logical operators:
* `allOf`: Requires all specified capabilities.
* `anyOf`: Requires at least one of the specified capabilities.
* `noneOf`: Penalizes the presence of specific capabilities. 
