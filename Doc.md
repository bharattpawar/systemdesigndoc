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

---

## How to add a new forbidden connection

Edit `src/modules/systemDesign/graphRuleChecks.ts`:

- Add patterns to `SOURCE` / `TARGET` (if needed)
- Add a new entry to `RULES` with a clear `reason`
