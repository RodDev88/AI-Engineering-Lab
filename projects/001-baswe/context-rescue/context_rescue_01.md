# AI ENGINEERING LAB — CONTEXT RESCUE / UPDATE
## Baswe Project 001 — Knowledge Graph RAG
***FECHA: 08-10-26***

### ESTADO ACTUAL
```
Phase 1 → Graph concepts ✅
Phase 2 → Vector Index ✅
Phase 3 → Router ✅
Phase 4 → Merge + Grounding ✅
Phase 5 → Evaluation ✅ CERRADA
Phase 6 → Implementation ⏭️ SIGUIENTE
```
### LO QUE APRENDIMOS

1. GRAPH
- Nodes + relationships + paths.
- Cypher.
- Hops.
- Entity canonicalization / resolution.
- Node IDs.
- Graph ≠ Vector.

2. VECTOR
- Chunks → embeddings → pgvector.
- Semantic similarity.
- HNSW / ef_search.
- Recall@k.
- Graph y Vector parten de los mismos chunks; no es obligatorio construir uno antes que otro.

3. ROUTER
- Decide VECTOR vs GRAPH.
- Few-shot + enum.
- Confidence.
- Low confidence → fallback VECTOR + GRAPH.
- Entity extraction → entity resolution → node ID.
- Query templates + parameterized Cypher.
- Logging / observability.

4. MERGE + GROUNDING
- Vector → passages/context.
- Graph → paths/relationships.
- Filtrar/seleccionar → Evidence Set.
- Evidence Set → grounding prompt → LLM.
- LLM genera lenguaje; evidencia fundamenta.
- Si evidencia insuficiente → reconocerlo, no inventar.

5. EVALUATION
Evaluamos por capas:
```
RETRIEVAL
→ ¿encontró la evidencia correcta?
→ Precision / Recall / Recall@k.

CORRECTNESS
→ ¿la respuesta coincide con el Ground Truth?

FAITHFULNESS / GROUNDEDNESS
→ ¿las afirmaciones de la respuesta están respaldadas por la evidencia?

- Claims permiten evaluar cada afirmación individual.
- Evaluation Dataset contiene:
  Question + Ground Truth + Expected Evidence.
- Exact Match no basta.
- Semantic Similarity tampoco basta.
- Regression testing permite comparar versiones.
- LLM-as-a-Judge automatiza parte de la evaluación.
- Human Calibration valida al Judge frente a evaluadores humanos.
```
### ARQUITECTURA CONCEPTUAL
```
Documents
   ↓
Chunks
   ↓
Graph + Vector
   ↓
Router
   ↓
Retrieval
   ↓
Evidence Set
   ↓
LLM
   ↓
Answer
   ↓
Evaluation
   ├── Retrieval
   ├── Correctness
   └── Faithfulness
```
### SIGUIENTE ETAPA

PHASE 6 — IMPLEMENTATION

Método:
```
concepto
↓
código pequeño
↓
ejecutar
↓
probar
↓
entender
↓
siguiente componente
```
Primer objetivo:
```
→ estructura mínima del proyecto
→ ingestion pipeline
```
Stack previsto:
```
Python + Neo4j + LangChain + pgvector + FastAPI + Claude API.
```
IMPORTANTE:
>No comenzar instalando/configurando todo de golpe.

>No entregar grandes bloques de código.
>Claude todavía no requiere suscripción/configuración prematura.

### RESUME PHRASE

“Retomemos Baswe Project 001 desde Phase 6,
comenzando por la estructura mínima del proyecto
y el ingestion pipeline.”

### NOTA DE NOMENCLATURA

Baswe Project 001 = Project 001 del AI Engineering Lab.

“Baswe Project 004” = cuarto proyecto de la cartera original de Baswe
(LLM-as-a-Judge + Human Calibration).
NO confundir ambos números.