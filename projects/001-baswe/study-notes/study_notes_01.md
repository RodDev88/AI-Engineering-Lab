# Project 001 — Baswe
# Study Notes 01 — From Vector RAG to Knowledge Graph RAG

> AI Engineering Lab · Project 001
> 
> Este documento consolida el primer bloque conceptual trabajado en Project 001 — Baswe.
> Su objetivo es fijar comprensión, no documentar todavía la implementación definitiva del proyecto.

---

## Índice
## FASE 01
1. [Punto de partida: RAG](#1-punto-de-partida-rag)
2. [Vector RAG](#2-vector-rag)
3. [Embeddings](#3-embeddings)
4. [La limitación del vector retrieval](#4-la-limitación-del-vector-retrieval)
5. [Knowledge Graph](#5-knowledge-graph)
6. [Hops](#6-hops)
7. [Graph Database y Neo4j](#7-graph-database-y-neo4j)
8. [Cypher: hablarle al grafo](#8-cypher-hablarle-al-grafo)
9. [MATCH y RETURN](#9-match-y-return)
10. [Entidades, relaciones y properties](#10-entidades-relaciones-y-properties)
11. [Del documento al grafo](#11-del-documento-al-grafo)
12. [Entity Resolution](#12-entity-resolution)
13. [Chunks](#13-chunks)
14. [Source Chunk ID y grounding](#14-source-chunk-id-y-grounding)
15. [Neo4j + Vector Store](#15-neo4j--vector-store)
16. [Router: elegir el camino de retrieval](#16-router-elegir-el-camino-de-retrieval)
17. [El sistema que estamos construyendo](#17-el-sistema-que-estamos-construyendo)
18. [Vocabulario nuevo](#18-vocabulario-nuevo)
19. [Conexión con nuestros estudios previos](#19-conexión-con-nuestros-estudios-previos)
20. [Repaso Vector vs Graph](#20-repaso-vector-vs-graph)

## FASE 02

21. [Objetivo Fase 02](#21-objetivo-fase-02)
22. [Chunks y embeddings](#22-chunks-y-embeddings)
23. [¿Qué significa "embed the same chunks"?](#23-qué-significa-embed-the-same-chunks)
24. [pgvector](#24-pgvector)
25. [Metadata](#25-metadata)
26. [entity_ids: puente entre vector y graph](#26-entity_ids-puente-entre-vector-y-graph)
27. [chunk_id: la clave compartida](#27-chunk_id-la-clave-compartida)
28. [ Vector → Graph y Graph → Texto](#28-vector--graph-y-graph--texto)
29. [HNSW](#29-hnsw)
30. [ef_search](#30-ef_search)
31. [recall@k](#31-recallk)
32. [Integración de fase 02](#32-integración-de-phase-2)
33. [Idea clave](#33-idea-clave)

## FASE 03

34. [Objetivo fase 03](#34objetivo-de-fase-3)
35. [El problema del routing](#35-el-problema-del-routing)
36. [Vector vs Graph](#36-vector-vs-graph)
37. [El Router](#37-el-router)
38. [Few-shot y Enum](#38-few-shot-y-enum)
39. [Confidence](#39-confidence)
40. [Fallback: ejecutar ambos caminos](#40-fallback-ejecutar-ambos-caminos)
41. [Entity Extraction](#41-entity-extraction)
42. [Entity Resolution y Node ID](#42-entity-resolution-y-node-id)
43. [Query Templates](#43-query-templates)
44. [Cypher parametrizado](#44-cypher-parametrizado)
45. [Logging y observabilidad](#45-logging-y-observabilidad)
46. [Integración de Phase 3](#46-integración-de-phase-3)
47. [Idea clave](#47-idea-clave)

## Fase 04

48. [Objetivo de Phase 4](#48-objetivo-de-phase-4)
49. [Vector y Graph aportan evidencias diferentes](#49-vector-y-graph-aportan-evidencias-diferentes)
50. [Passage vs Path](#50-passage-vs-path)
51. [Merge: combinar las evidencias](#51-merge-combinar-las-evidencias)
52. [Grounded Answer](#52-grounded-answer)
53. [Filtrar la evidencia antes del LLM](#53-filtrar-la-evidencia-antes-del-llm)
54. [Evidence Set](#54-evidence-set)
55. [El papel del LLM](#55-el-papel-del-llm)
56. [Grounding Prompt](#56-grounding-prompt)
57. [Evidencia insuficiente](#57-evidencia-insuficiente)
58. [Ejemplo completo](#58-ejemplo-completo)
59. [Arquitectura fase 04](#59-arquitectura-de-phase-4)
60. [Idea central de Phase 4](#60-idea-central-de-phase-4)

## FASE 05

61. [Objetivo de Phase 5](#61-objetivo-de-phase-5)
62. [Por qué evaluar por capas](#62-por-qué-evaluar-por-capas)
63. [Evaluación del Retrieval](#63-evaluación-del-retrieval)
64. [Precision y Recall](#64-precision-y-recall)
65. [Recall@k](#65-recallk)
66. [Evaluation Dataset](#66-evaluation-dataset)
67. [Ground Truth](#67-ground-truth)
68. [Exact Match vs Semantic Similarity](#68-exact-match-vs-semantic-similarity)
69. [Correctness](#69-correctness)
70. [Faithfulness / Groundedness](#70-faithfulness--groundedness)
71. [Claims](#71-claims)
72. [Correctness vs Faithfulness](#72-correctness-vs-faithfulness)
73. [Expected Evidence](#73-expected-evidence)
74. [Diagnóstico por capas](#74-diagnóstico-por-capas)
75. [Regression Testing](#75-regression-testing)
76. [LLM-as-a-Judge](#76-llm-as-a-judge)
77. [Human Calibration](#77-human-calibration)
78. [Pipeline completo de evaluación](#78-pipeline-completo-de-evaluación)
79. [Idea central de Phase 5](#79-idea-central-de-phase-5)
80. [Cierre y paso siguiente](#80-cierre-y-paso-siguiente)


---
---
ACTUALIZAR AL FINAL

20. [20. Estado de Project 001](#20-estado-de-project-001)
21. [21. Grafo con estado actual](#21-️-grafo-compacto-de-nuestro-avance)
22. [22. Texto para retomar trabajo](#22--texto-para-retomar-trabajo)

---

# 1. Punto de partida: RAG

RAG significa **Retrieval-Augmented Generation**.

La idea general es:

```text
Pregunta
   ↓
Retrieval
   ↓
información relevante
   ↓
LLM
   ↓
respuesta
```
El sistema no depende exclusivamente del conocimiento aprendido por el modelo.

Primero recupera información relevante y luego utiliza esa información para generar la respuesta.

- [Volver arriba](#)

---

# 2. Vector RAG

Un Vector RAG utiliza representaciones vectoriales para buscar fragmentos de información semánticamente relacionados con una pregunta.

Conceptualmente:
```
Documento
   ↓
chunks
   ↓
embeddings
   ↓
vector database

```

Cuando llega una pregunta:
```
Pregunta
   ↓
embedding
   ↓
búsqueda por similitud
   ↓
chunks relevantes
   ↓
LLM
```
La idea fundamental es:

Encontrar contenido que sea semánticamente similar a lo que estamos preguntando.

Un Vector RAG funciona especialmente bien para preguntas como:

- definiciones;
- información puntual;
- búsquedas semánticas;
- recuperación de un fragmento relevante.

- [Volver arriba](#)

---

# 3. Embeddings

Un embedding es una representación numérica de información.

Conceptualmente:
```
"Acme Corporation"
        ↓
   embedding
        ↓
[0.12, -0.37, 0.81, ...]
```
El vector contiene muchos números que permiten representar semánticamente el contenido.

La idea importante para este proyecto es que textos con significados relacionados pueden quedar representados de manera que podamos comparar su similitud.

Los embeddings no son simplemente una "traducción" palabra → número.

Son una representación matemática de información semántica generada por un modelo.


- [Volver arriba](#)

---

# 4. La limitación del vector retrieval

El vector retrieval responde muy bien a preguntas del tipo:
```
"¿Qué es Kubernetes?"
```
Pero puede ser menos natural para preguntas que requieren seguir una cadena explícita de relaciones.

Por ejemplo:
```
"¿Qué producto desarrolló la empresa adquirida por Acme?"
```
La estructura lógica es:
```
Acme
 ↓ acquired
Empresa B
 ↓ developed
Producto X
```
Aquí necesitamos seguir relaciones.

Por eso aparece la idea de `Knowledge Graph RAG.`


- [Volver arriba](#)

---

# 5. Knowledge Graph

Un Knowledge Graph representa conocimiento mediante:

nodes + relationships + properties

Ejemplo:
```
Acme
  │
  │ ACQUIRED
  ↓
Beta
  │
  │ DEVELOPED
  ↓
Product X
```
Los nodos representan entidades.

Las relaciones representan conexiones entre entidades.

Las properties contienen información adicional sobre nodos o relaciones.

La ventaja conceptual es que podemos recorrer explícitamente las conexiones.

- [Volver arriba](#)

---

# 6. Hops

Un hop es un salto de una entidad/nodo a otra atravesando una relación.

Regla:
```
1 relación atravesada = 1 hop.
```
Ejemplo:
```
Acme ──ACQUIRED──> Beta
```
Tiene:
```
1 hop
```
Mientras:
```
Acme ──ACQUIRED──> Beta ──DEVELOPED──> Product X
```
tiene:
```
2 hops
```
Porque atravesamos dos relaciones.

Importante:
```
relationship ≠ hop
```
>La relación describe qué conecta.

>El hop describe cuánto avanzamos en el recorrido.

- [Volver arriba](#)

---

# 7. Graph Database y Neo4j

Una Graph Database es una base de datos diseñada para trabajar naturalmente con:
```
nodes
relationships
properties
```
> Neo4j es una graph database.

En Project 001, Neo4j será el lugar donde podremos representar entidades y relaciones y realizar recorridos sobre ellas.

Conceptualmente:
```
Documentos
   ↓
Entidades + relaciones
   ↓
Neo4j
```

- [Volver arriba](#)

---

# 8. Cypher: hablarle al grafo

> Cypher es el lenguaje de consultas de Neo4j.

Una característica interesante es que su sintaxis intenta representar visualmente los patrones del grafo.

Ejemplo conceptual:
```
(A)-[:ACQUIRED]->(B)
```
Esto representa:
```
A ──ACQUIRED──> B
```
Elementos básicos:
```
(A)             → nodo / variable de nodo

[:ACQUIRED]     → tipo de relación

->              → dirección

[r]             → variable que representa una relación
```

- [Volver arriba](#)

---

# 9. MATCH y RETURN

Dos palabras fundamentales de Cypher:
```
MATCH
RETURN
```
> MATCH indica el patrón que queremos encontrar.

> RETURN indica qué queremos obtener del patrón encontrado.

Ejemplo:
```
MATCH (a)-[:ACQUIRED]->(b)-[:DEVELOPED]->(p)
RETURN p
```
Interpretación:
```
Encuentra un recorrido donde a adquirió b y b desarrolló p. Devuélveme p.
```
Si:
```
Acme → ACQUIRED → Beta → DEVELOPED → Product X
```
entonces:
```
RETURN a → Acme
RETURN b → Beta
RETURN p → Product X
```
Las letras son variables que nos permiten referirnos a los nodos encontrados.

- [Volver arriba](#)

---

# 10. Entidades, relaciones y properties

Un nodo puede tener un tipo y properties.

Ejemplo:
```
(:Company {
    name: "Acme"
})
```
Podemos interpretarlo como:
```
Un nodo de tipo Company cuyo nombre es Acme.
```
Una relación puede tener un tipo:
```
[:ACQUIRED]
```
También puede tener properties.

Conceptualmente:

```
Acme
  │
  │ ACQUIRED
  │ {year: 2020}
  ↓
Beta
```
Distinciones importantes:
```
Company
   → tipo de nodo

name: "Acme"
   → property

ACQUIRED
   → tipo de relación
```
- [Volver arriba](#)

---

# 11. Del documento al grafo

El proyecto parte de documentos, no de un grafo ya construido.

Ejemplo:
```
"OpenAI developed GPT-4."
```
Queremos extraer:
```
Entidad 1 → OpenAI
Entidad 2 → GPT-4
Relación  → DEVELOPED
```
Y transformarlo conceptualmente en:
```
OpenAI
   │
   │ DEVELOPED
   ↓
GPT-4
```
El brief propone utilizar un LLM para realizar esta extracción.

Conceptualmente:
```
Documento
   ↓
LLM
   ↓
entidades + relaciones
   ↓
Knowledge Graph
```
> El LLM es una pieza del pipeline.

> No es el Knowledge Graph.

- [Volver arriba](#)

---


# 12. Entity Resolution

Una misma entidad puede aparecer con distintos nombres.

Ejemplo:
```
Acme Corp
Acme Corporation
ACME
```
Podrían representar la misma entidad.

Si las tratamos como entidades independientes:
```
Acme Corp

Acme Corporation

ACME
```
podemos terminar con nodos duplicados.

Entity resolution es el proceso mediante el cual intentamos determinar si diferentes menciones representan la misma entidad.

Una vez resuelto, podemos utilizar una representación canónica:
```
Acme Corp ─┐
Acme       ├──→ Acme Corporation
ACME       ┘
```
Distinción:
```
Entity Resolution
→ proceso para determinar identidad

Canonical Name
→ representación elegida para la entidad
```

- [Volver arriba](#)

---

# 13. Chunks

Un chunk es un fragmento de un documento tratado como una unidad de información.

Ejemplo:
```
Documento
   ↓
Chunk 001
Chunk 002
Chunk 003
...
```
Los chunks son fundamentales en RAG porque permiten trabajar y recuperar partes específicas de documentos.

Ejemplo:
```
chunk_002

"In 2020, Acme Corporation acquired Beta Inc."
```

Los chunks también pueden utilizarse durante la extracción de entidades y relaciones.

El proyecto asigna un ID a cada chunk.

Ejemplo:
```
chunk_002
```
- [Volver arriba](#)

---

# 14. Source Chunk ID y grounding

Una relación del grafo puede guardar el ID del chunk que proporciona la evidencia de esa relación.

Ejemplo:
```
Microsoft ──ACQUIRED──> OpenAI
               │
               └── source_chunk_id: chunk_007
```
El chunk_007 contiene la evidencia:
```
"Microsoft acquired OpenAI in 2028."
```
Esto permite volver desde la afirmación del grafo hacia la evidencia original.

Conceptualmente:
```
Relación
   ↓
source_chunk_id
   ↓
chunk original
   ↓
evidencia
```
Esto está relacionado con `grounding`:

> conectar una afirmación generada con evidencia recuperable que la sustenta.

> También permite construir respuestas con citas o referencias verificables.

- [Volver arriba](#)

---


# 5. Neo4j + Vector Store

Project 001 no utiliza Neo4j para todo.

Utiliza dos mecanismos de retrieval con funciones diferentes.

> Neo4j

Adecuado para:
```
entidades
relaciones
recorridos
multi-hop
```

> Vector store

Adecuado para:
```
texto
embeddings
búsqueda semántica
recuperación de chunks
```
El proyecto propone que ambos compartan los mismos chunk IDs.
```
Conceptualmente:

                  chunk_007
                  /       \
                 ↓         ↓
             pgvector    Neo4j
                 │          │
             contenido    relaciones
                 │          │
                 └────┬─────┘
                      ↓
                misma evidencia
```

El chunk_id funciona como un puente entre ambos mundos.
- [Volver arriba](#)

---

# 16. Router: elegir el camino de retrieval

No todas las preguntas necesitan el mismo tipo de retrieval.

Ejemplo:

Pregunta
```
¿Qué es Kubernetes?
```
Naturalmente puede utilizar:
```
Vector Retrieval
```
Pregunta
```
¿Qué producto desarrolló la empresa adquirida por Acme?
```
Naturalmente puede utilizar:
```
Graph Retrieval
```
Por eso el proyecto propone un router:

                  Pregunta
                     ↓
                   Router
                  /      \
                 ↓        ↓
             Vector      Graph
             path         path

Conceptualmente:
```
Vector

→ definiciones
→ hechos simples
→ búsquedas semánticas

Graph

→ conexiones
→ multi-hop
→ comparaciones entre entidades
→ relaciones
```

Existe además la posibilidad de un fallback que utilice ambos caminos cuando el router tenga baja confianza.
- [Volver arriba](#)

---
# 17. El sistema que estamos construyendo

Hasta este punto, podemos representar Project 001 así:

```

DOCUMENTOS
     ↓
   CHUNKS
     ↓
 ┌───┴───────────────┐
 ↓                   ↓
LLM              EMBEDDINGS
 ↓                   ↓
ENTIDADES +       VECTOR
RELACIONES        STORE
 ↓                   │
ENTITY              │
RESOLUTION           │
 ↓                   │
NEO4J                │
 └──────────┬────────┘
            ↓
          ROUTER
        ↙         ↘
     GRAPH       VECTOR
        ↘         ↙
          ANSWER
             ↓
         CITATIONS

```

La arquitectura busca combinar:
```
graph retrieval
+
vector retrieval
```
para responder preguntas que van desde hechos simples hasta preguntas relacionales de varios hops.

- [Volver arriba](#)

---

# 18. Vocabulario nuevo

| Concepto          | Significado                                                                 |
| ----------------- | --------------------------------------------------------------------------- |
| RAG               | Retrieval-Augmented Generation                                              |
| Vector RAG        | RAG basado en búsqueda semántica mediante vectores                          |
| Embedding         | Representación numérica/semántica de información                            |
| Chunk             | Fragmento de un documento                                                   |
| Knowledge Graph   | Grafo que representa entidades y relaciones                                 |
| Node              | Entidad representada en el grafo                                            |
| Relationship      | Conexión entre entidades                                                    |
| Property          | Dato asociado a un nodo o relación                                          |
| Hop               | Un salto a través de una relación                                           |
| Graph traversal   | Recorrido de un grafo                                                       |
| Graph Database    | Base de datos orientada a grafos                                            |
| Neo4j             | Graph database utilizada en el proyecto                                     |
| Cypher            | Lenguaje de consultas de Neo4j                                              |
| MATCH             | Busca un patrón en el grafo                                                 |
| RETURN            | Indica qué devolver                                                         |
| Entity Resolution | Proceso para determinar si menciones distintas representan la misma entidad |
| Canonical Name    | Representación canónica de una entidad                                      |
| Grounding         | Vincular una afirmación con evidencia                                       |
| Router            | Componente que decide qué camino de retrieval utilizar                      |


- [Volver arriba](#)

---

# 19. Conexión con nuestros estudios previos

Este proyecto está conectando conocimientos que hasta ahora habían sido estudiados en contextos separados.

### CS50 AI

Los conceptos de representación de conocimiento, relaciones, inferencia y búsqueda ayudan a comprender la idea de trabajar con conocimiento estructurado y recorrer estructuras.

### AI Engineer — Chip Huyen

Conceptos como:

embeddings;
retrieval;
context construction;
evaluación;
operaciones de aplicaciones de IA;

empiezan a aparecer ahora dentro de un sistema concreto.

### Project 001 — Baswe

El proyecto funciona como un espacio práctico donde esas ideas dejan de ser conceptos aislados y comienzan a formar una arquitectura.

La conexión general es:
```
Conceptos teóricos
       ↓
comprensión conceptual
       ↓
sistema concreto
       ↓
decisiones de ingeniería
       ↓
implementación
```

- [Volver arriba](#)

---


# 20. Repaso Vector vs Graph

### Vector

Un **vector** es una representación numérica de información.

En AI, un texto puede transformarse mediante un modelo de embeddings en un vector de números que representa aspectos de su significado semántico.

```text
"Acme acquired Beta"
        ↓
    embedding
        ↓
[0.12, -0.37, 0.81, ...]
```
Vector retrieval utiliza estos vectores para encontrar información semánticamente similar.

> Vector → representa significado semántico mediante números.

### Graph

Un graph (grafo) representa conocimiento mediante entidades y las relaciones entre ellas.

```
Acme ──ACQUIRED──> Beta
                     │
                  DEVELOPED
                     ↓
                  Product X
```

Sus elementos principales son:
```
Nodes → entidades
Relationships → conexiones entre entidades
Properties → información asociada a nodos o relaciones
```

>Graph retrieval permite recorrer esas relaciones para responder preguntas de conexión o multi-hop.

> Graph → representa estructura y relaciones entre entidades.

#### Diferencia fundamental
```
VECTOR
  ↓
Números
  ↓
Similitud semántica
  ↓
"¿Qué información se parece a esta pregunta?"


GRAPH
  ↓
Entidades + relaciones
  ↓
Recorrido de conexiones
  ↓
"¿Cómo están relacionadas estas entidades?"
```

#### En Project 001

Vector y Graph no son sinónimos ni competidores directos.

Son mecanismos complementarios:

                 PREGUNTA
                    ↓
                  ROUTER
                 /      \
                ↓        ↓
             VECTOR     GRAPH
                ↓         ↓
           similitud   relaciones
                \         /
                 ↓       ↓
                  RESPUESTA

> Vector → encuentra contenido relevante.

> Graph → permite recorrer relaciones entre entidades.

> El objetivo de Knowledge Graph RAG es combinar ambos cuando sea útil.

- [Volver arriba](#)


---

# 21. Objetivo fase 02

En Phase 1 construimos el grafo:

```text
DOCUMENTOS
    ↓
  CHUNKS
    ↓
ENTIDADES + RELACIONES
    ↓
  NEO4J

```

En Phase 2 tomamos esos mismos chunks y construimos el camino vectorial:
```
DOCUMENTOS
    ↓
  CHUNKS
    ↓
 EMBEDDINGS
    ↓
 PGVECTOR
    ↓
 BÚSQUEDA VECTORIAL
```

El objetivo es tener ambos mecanismos de retrieval disponibles:

                    CHUNKS
                   /      \
                  ↓        ↓
             GRAPH PATH  VECTOR PATH
                ↓            ↓
              Neo4j       pgvector


La arquitectura utiliza ambos porque representan y recuperan información de maneras diferentes.

- [Volver arriba](#)
---

# 22. Chunks y embeddings

Un chunk es un fragmento de un documento.

Ejemplo:
```
Chunk 007:

"Acme Corporation acquired Beta Inc. in 2020."
```

Ese texto puede transformarse mediante un modelo de embeddings en una representación numérica:
```

"Acme Corporation acquired Beta Inc. in 2020."
                    ↓
             Embedding Model
                    ↓
       [0.12, -0.37, 0.81, ...]
```
El resultado es un embedding.

## Definiciones

### Chunk

Fragmento de texto utilizado como unidad de procesamiento y recuperación.

### Embedding

Representación numérica de un contenido, generada por un modelo de embeddings.

### Vector

La representación matemática formada por una secuencia de números.
```
Embedding → representación vectorial
Vector    → números que forman esa representación
```

### Vector ≠ Graph

Un vector y un graph son dos formas diferentes de representar información.
```
VECTOR
  ↓
Números
  ↓
Similitud semántica
  ↓
"¿Qué contenido se parece a esta pregunta?"

GRAPH
  ↓
Entidades + relaciones
  ↓
Estructura y conexiones
  ↓
"¿Cómo están relacionadas estas entidades?"

```

- [Volver arriba](#)

---

# 23. ¿Qué significa "embed the same chunks"?

Baswe indica:
```
"Embed the same chunks into pgvector..."
```
"Embed the same chunks" significa:

Tomar los mismos chunks utilizados durante la construcción del grafo y convertirlos en embeddings.

No se crea otro conjunto de chunks.

Los mismos chunks alimentan ambos caminos:

                    DOCUMENTO
                        ↓
                      CHUNK
                    /       \
                   ↓         ↓
             ENTIDADES    EMBEDDING
                   ↓         ↓
                 NEO4J     PGVECTOR

Esto permite que el contenido textual tenga simultáneamente:

- una representación vectorial para búsqueda semántica;
- entidades y relaciones para navegación mediante el grafo.

Ejemplo:
```
chunk_007


"Acme Corporation acquired Beta Inc. in 2020."
```
Se transforma en:
```
chunk_007
    ↓
Embedding Model
    ↓
[0.12, -0.37, 0.81, ...]
```

Ese vector es el embedding de chunk_007

- [Volver arriba](#)

---

# 24. pgvector

pgvector es una extensión de PostgreSQL que permite almacenar y buscar vectores.

La relación conceptual es:
```
PostgreSQL
    │
    ├── SQL
    │
    └── pgvector
          ↓
       vectores
```
PostgreSQL es una base de datos relacional y utiliza SQL como su lenguaje principal de consulta.

Neo4j, en cambio, es una base de datos de grafos y utiliza Cypher para consultar el grafo.
```
PostgreSQL → SQL
Neo4j      → Cypher
```
En nuestro proyecto:
```
Neo4j
  ↓
entidades + relaciones

pgvector
  ↓
embeddings + metadata
```

- [Volver arriba](#)

# 25. Metadata

El embedding no debería almacenarse aislado.

Baswe propone almacenar también metadata:
```
embedding
    +
document_id
section_path
date
entity_ids
```
La metadata conserva información sobre el contexto y procedencia del chunk.

Ejemplo conceptual:
```
chunk_007
│
├── embedding
│      ↓
│   [0.12, -0.37, 0.81, ...]
│
├── document_id
│      ↓
│   document_001
│
├── section_path
│      ↓
│   /Company/History/Acquisitions
│
├── date
│      ↓
│   2020
│
└── entity_ids
       ↓
   [acme_01, beta_02]
```

## Función de cada metadata
```
document_id
    → ¿De qué documento viene?


section_path
    → ¿En qué sección del documento estaba?


date
    → ¿Qué contexto temporal tiene?


entity_ids
    → ¿Qué entidades del grafo menciona?

```
La metadata no crea las entidades ni las relaciones. Esa construcción ocurre en el grafo.

La metadata permite conservar el contexto del chunk y establecer conexiones con el resto del sistema

- [Volver arriba](#)

---

# 26. entity_ids: puente entre vector y graph

entity_ids contiene los identificadores de las entidades que aparecen en un chunk.

Ejemplo:
```
chunk_007

"Acme Corporation acquired Beta Inc. in 2020."
```
Puede tener:
```
entity_ids:
    [Acme_ID, Beta_ID]
```
En Neo4j podemos tener:
```
Acme_ID ──ACQUIRED──> Beta_ID
```
Así, el registro vectorial conoce qué entidades están asociadas al chunk.

Esto crea un puente entre ambos mundos:

                 CHUNK_007
                 /       \
                ↓         ↓
           PGVECTOR      NEO4J
               │            │
           embedding     Acme_ID
               │            │
           búsqueda      ACQUIRED
           semántica         │
               │           Beta_ID
               └─────┬──────┘
                     ↓
                mismo contexto

## Idea clave
```
Vector → encuentra contenido relevante.


Graph → permite recorrer relaciones.


entity_ids → conecta el chunk recuperado
             con las entidades del grafo.
```

- [Volver arriba](#)

---

# 27. chunk_id: la clave compartida

Baswe dice:
```
"Both stores share the same chunk ids. That shared key is the whole trick."
```
El chunk_id funciona como una clave compartida entre los sistemas.

Ejemplo:
```
chunk_id = "chunk_007"
```
En pgvector:
```
chunk_id:    chunk_007
embedding:   [0.12, -0.37, 0.81, ...]
document_id: document_001
entity_ids:  [acme_01, beta_02]
```
En Neo4j:
```
Acme ──ACQUIRED──> Beta
```
```
source_chunk_id:
    chunk_007
```
Ambos sistemas saben que están hablando del mismo fragmento original.

## ¿Por qué es importante?

Permite conectar:
```
Vector → Chunk → Graph
```
y también:
```
Graph → Chunk → Texto original
```
> El chunk_id es, conceptualmente, el identificador común que permite cruzar ambos mundos

- [Volver arriba](#)

---

# 28. Vector → Graph y Graph → Texto

## Vector → Graph

Una pregunta puede ser transformada en embedding y utilizarse para buscar chunks relevantes.
```
Pregunta
    ↓
Embedding
    ↓
Vector Search
    ↓
chunk_007
    ↓
entity_ids
    ↓
Neo4j
    ↓
relaciones
```

## Graph → Texto

Neo4j puede devolver una relación junto con el chunk que proporcionó la evidencia.
```
Acme ──ACQUIRED──> Beta
          │
          ↓
    source_chunk_id
          ↓
       chunk_007
          ↓
"Acme Corporation acquired Beta Inc. in 2020."
```

Esto permite volver desde una relación del grafo hasta la evidencia textual original.

## Idea central

```
Vector → encuentra contenido.


Graph → encuentra conexiones.


chunk_id → permite pasar del contenido
           a las conexiones y volver
           a la evidencia original
```

- [Volver arriba](#)

---


# 29. HNSW

Baswe indica:
```
"Index with HNSW..."
```
HNSW significa:
```
Hierarchical Navigable Small World
```
Es una estructura de índice utilizada para hacer búsquedas vectoriales de manera eficiente.

## El problema

Si tenemos muchos embeddings:
```
chunk_001 → embedding
chunk_002 → embedding
chunk_003 → embedding
...
chunk_10.000.000 → embedding
```

Una solución ingenua sería comparar la pregunta contra todos los vectores.
```
Pregunta
   │
   ├── chunk_001
   ├── chunk_002
   ├── chunk_003
   ├── ...
   └── chunk_10.000.000
```
Esto puede ser muy costoso.

## HNSW

HNSW organiza los vectores de una manera que permite navegar hacia regiones donde probablemente se encuentren los vecinos más cercanos.

Conceptualmente:
```
HNSW
  ↓
estructura de navegación
  ↓
búsqueda eficiente de vecinos
  ↓
vectores similares
```
### Una analogía:

Sin índice:

```
ciudad
  ↓
buscar casa revisando todas las casas
```
Con una estructura de navegación:

```
país
  ↓
región
  ↓
ciudad
  ↓
barrio
  ↓
calle
  ↓
casa
```
La idea es reducir la cantidad de elementos que necesitamos explorar.

- [Volver arriba](#)

---

# 30. ef_search

Baswe indica:
```
"tune ef_search"
```
> ef_search es un parámetro de HNSW que controla cuánto se explora durante una búsqueda.

Una intuición simplificada:
```
ef_search pequeño
    ↓
explora menos
    ↓
más rápido
    ↓
puede perder algunos buenos candidatos
ef_search grande
    ↓
explora más
    ↓
más trabajo
    ↓
puede encontrar mejores candidatos
```
Existe un trade-off:
```
        velocidad
           ↕
      ef_search
           ↕
     calidad de búsqueda
```
Por eso no existe necesariamente un valor universalmente correcto.

Se debe medir y ajustar según el sistema

- [Volver arriba](#)

---

# 31. recall@k

Baswe indica:
```
"measure recall at k on a small labeled set before moving on."
```
Queremos medir si el sistema está encontrando realmente los resultados relevantes.

> k representa la cantidad de resultados que consideramos.

Por ejemplo:
```
recall@5
```
significa evaluar si los resultados relevantes aparecen dentro de los cinco primeros resultados recuperados.

Conceptualmente:
```
Resultados que deberían aparecer
              ↓
¿Cuántos encontramos
dentro de los top-k?
              ↓
          recall@k
```
Esto permite medir la calidad del retrieval antes de construir componentes posteriores sobre él.

- [Volver arriba](#)

---

# 32. Integración de Phase 2

Todo lo aprendido puede representarse así:
```
DOCUMENTOS
    ↓
  CHUNKS
    │
    ├─────────────────────────────┐
    ↓                             ↓
ENTIDADES                     EMBEDDINGS
    ↓                             ↓
ENTITY RESOLUTION            PGVECTOR
    ↓                             ↓
  NEO4J                         HNSW
    │                             ↓
    │                        búsqueda eficiente
    │                             ↓
    │                        ef_search
    │                             ↓
    │                         recall@k
    │                             ↓
    │                    retrieval medido
    │                             │
    └──────────────┬──────────────┘
                   ↓
                 ROUTER
```
El sistema ahora tiene dos caminos de retrieval:

Vector path
```
Pregunta
    ↓
Embedding
    ↓
pgvector
    ↓
HNSW
    ↓
Chunks relevantes
```
Graph path
```
Pregunta
    ↓
Entidades
    ↓
Neo4j
    ↓
Relaciones
    ↓
Paths / conexiones
```
Ambos caminos pueden volver al mismo contenido mediante los identificadores compartidos:

             CHUNK
            /     \
           ↓       ↓
      embedding   entities
          ↓          ↓
      pgvector     Neo4j
          │          │
          └────┬─────┘
               ↓
          mismo contexto


- [Volver arriba](#)

---

# 33. Idea clave

Phase 2 no consiste simplemente en "guardar embeddings".

La idea arquitectónica es construir un camino vectorial complementario al grafo, utilizando los mismos chunks y claves compartidas.
```
CHUNKS
   ↓
   ├──→ EMBEDDINGS → PGVECTOR → HNSW → VECTOR RETRIEVAL
   │
   └──→ ENTITIES → NEO4J → GRAPH RETRIEVAL
```
Las distintas representaciones tienen diferentes funciones:
```
Texto
  → evidencia y contexto

Vector
  → similitud semántica

Graph
  → entidades y relaciones

Metadata
  → contexto y procedencia

chunk_id
  → conexión entre las representaciones

HNSW
  → búsqueda vectorial eficiente

ef_search
  → controla la exploración durante la búsqueda

recall@k
  → mide la calidad del retrieval
```

## Concepto central de Phase 2

Vector y graph representan y recuperan información de maneras diferentes, pero pueden trabajar sobre los mismos chunks y conectarse mediante IDs compartidos.

Esto prepara el sistema para Phase 3:
```
VECTOR RETRIEVAL
       +
GRAPH RETRIEVAL
       ↓
     ROUTER
       ↓
decidir qué camino utilizar
```
- [Volver arriba](#)

---

# Fase 03

## 34.Objetivo de fase 3

En Phase 1 construimos el Graph:

```text
DOCUMENTOS
    ↓
  CHUNKS
    ↓
ENTIDADES + RELACIONES
    ↓
  NEO4J
```
En Phase 2 construimos el camino vectorial:
```
CHUNKS
   ↓
EMBEDDINGS
   ↓
PGVECTOR
   ↓
HNSW
   ↓
VECTOR RETRIEVAL
```

Ahora tenemos dos mecanismos de retrieval.

Phase 3 introduce un componente que decide cuál utilizar:

                    QUESTION
                        ↓
                      ROUTER
                     /      \
                    ↓        ↓
                VECTOR      GRAPH

El objetivo de Phase 3 es:

>Determinar qué camino de retrieval es más apropiado para cada pregunta


- [Volver arriba](#)

---

### 35. El problema del routing

No todas las preguntas requieren el mismo tipo de búsqueda.

Ejemplo:
```
"¿Qué es Kubernetes?"
```
Es principalmente una pregunta de definición o significado.
```
Pregunta
   ↓
VECTOR
   ↓
buscar contenido semánticamente similar
```
En cambio:
```
"¿Qué productos desarrolló la empresa
que Acme adquirió?"
```
requiere recorrer relaciones:
```
Acme
  ↓ ACQUIRED
Empresa X
  ↓ DEVELOPED
Producto Y
```
Por lo tanto:
```
Pregunta
   ↓
GRAPH
   ↓
recorrer relaciones
```

>El problema que resuelve el routing es: 
¿A qué mecanismo de retrieval debemos enviar esta pregunta?

- [Volver arriba](#)

---

###  36. Vector vs Graph

#### VECTOR

El retrieval vectorial es especialmente apropiado para preguntas como:
```
definiciones;
explicaciones;
búsquedas semánticas;
hechos contenidos en un pasaje;
policy lookups;
preguntas sobre información textual.
```
Conceptualmente:
```
Pregunta
   ↓
Embedding
   ↓
Vector Search
   ↓
Chunks similares
```

El vector responde principalmente:

"¿Qué contenido es semánticamente parecido a esta pregunta?"

#### GRAPH

El retrieval mediante graph es especialmente apropiado para:
```
preguntas sobre relaciones;
conexiones entre entidades;
comparaciones entre entidades;
multi-hop questions;
agregaciones sobre relaciones.
```
Conceptualmente:
```
Pregunta
   ↓
Entidades
   ↓
Neo4j
   ↓
Relaciones
   ↓
Graph Path
```
El graph responde principalmente:
```
"¿Cómo están conectadas estas entidades?"
```
Ejemplos:

"¿Qué es pgvector?"
```
→ VECTOR
```
"¿Qué empresa adquirió Acme?"
```
→ GRAPH
````
"¿Qué producto desarrolló la empresa
que Acme adquirió?"
```
→ GRAPH
```
La última requiere dos relaciones:
```
Acme
  ↓ ACQUIRED       ← hop 1
Empresa X
  ↓ DEVELOPED      ← hop 2
Producto Y
```


- [Volver arriba](#)

---

### 37. El Router

El router es el componente que recibe una pregunta y decide qué camino de retrieval utilizar.
```
QUESTION
   ↓
ROUTER
   ↓
┌───────────────┐
│               │
VECTOR        GRAPH
```
El router no necesariamente responde la pregunta.

Su función principal es:
```
Decidir dónde buscar.
```
Conceptualmente:
```
Pregunta
   ↓
Router
   ↓
ruta de retrieval
```
Por ejemplo:
```
"¿Qué es Kubernetes?"
        ↓
      VECTOR
"¿Qué empresa adquirió Acme?"
        ↓
      GRAPH
```
El router puede implementarse mediante una llamada relativamente pequeña y barata a un modelo.

- [Volver arriba](#)

---

### 38. Few-shot y Enum

Baswe propone:

"a small model call with a few shot prompt returning an enum"

#### Small model call

El router no necesita necesariamente un modelo grande.

La tarea es relativamente sencilla:
```
Pregunta
   ↓
clasificar
   ↓
VECTOR o GRAPH
```
Por lo tanto, puede utilizarse un modelo relativamente pequeño y barato.

#### Few-shot prompt

Few-shot significa proporcionar al modelo algunos ejemplos para enseñarle el tipo de clasificación que debe realizar.

Conceptualmente:

Ejemplo:
```
"¿Qué es Kubernetes?"
→ VECTOR
```
Ejemplo:
```
"¿Qué empresa adquirió Acme?"
→ GRAPH
```
Nueva pregunta:
```
"¿Qué productos desarrolló la empresa
que adquirió Acme?"
→ ?
```
Los ejemplos ayudan al modelo a aprender el criterio de routing que queremos.

#### Enum

Un enum representa un conjunto cerrado de opciones posibles.

En nuestro caso:
```
ROUTE = {
    VECTOR,
    GRAPH
}
```
El router no debería devolver libremente cualquier texto.

Idealmente devuelve una opción controlada:
```
VECTOR

o:

GRAPH
```
- [Volver arriba](#)

---

### 39. Confidence

El router puede acompañar su decisión con una estimación de confianza.

Ejemplo:
```
Route:
GRAPH

Confidence:
0.95
```
Interpretación conceptual:
```
El router tiene alta confianza en que GRAPH es el camino apropiado.
```
Otro caso:
```
Route:
GRAPH

Confidence:
0.48
```
Aquí podemos interpretar:
```
El router cree que GRAPH es apropiado, pero no está suficientemente seguro.
```
Importante:

Confidence aquí se refiere a la confianza en la decisión de routing, no necesariamente a la probabilidad de que la respuesta final sea correcta

- [Volver arriba](#)

---

### 40. Fallback: ejecutar ambos caminos

Baswe propone un mecanismo de fallback para los casos de baja confianza:
```
"a low confidence fallback that runs both paths and merges."
```
Conceptualmente:

                 QUESTION
                    ↓
                  ROUTER
                    ↓
             ¿CONFIANZA ALTA?
              /          \
            SÍ            NO
            ↓              ↓
        UN CAMINO       AMBOS
                           ↓
                    VECTOR + GRAPH
                           ↓
                         MERGE

La idea es no obligar al sistema a tomar una decisión dudosa.

Si la confianza es baja:
```
VECTOR
   +
GRAPH
   ↓
MERGE
```
Esto permite que una pregunta ambigua o híbrida sea atendida utilizando ambos mecanismos.

Ejemplo:
```
"¿Qué relación existe entre Acme y Beta
y qué dice el documento sobre esa adquisición?"
```
Podría necesitar:
```
GRAPH
→ relación entre Acme y Beta

VECTOR
→ información textual sobre la adquisición
```
El fallback permite recuperar ambos tipos de información.

#### Idea de ingeniería

El router no tiene que ser perfecto.

Un sistema robusto debe considerar la posibilidad de que el modelo no esté seguro y tener una estrategia alternativa.
```
DECISIÓN
   ↓
¿confianza suficiente?
   │
   ├── Sí → continuar
   │
   └── No → fallback
```

- [Volver arriba](#)

---

### 41. Entity Extraction

Cuando el router decide utilizar GRAPH, todavía necesitamos transformar la pregunta en información que podamos utilizar para consultar Neo4j.

Ejemplo:
```
"¿Qué productos desarrolló la empresa
que Acme adquirió?"
```
Primero identificamos las entidades relevantes:
```
Pregunta
   ↓
Entity Extraction
   ↓
Acme
```
La extracción de entidades identifica elementos del texto que corresponden a entidades presentes en nuestro dominio.

La pregunta pasa conceptualmente de:
```
texto humano
```
a:
```
entidad:
Acme
```
- [Volver arriba](#)

---

### 42. Entity Resolution y Node ID

La entidad extraída puede aparecer de diferentes maneras:
```
Acme
Acme Corp.
Acme Corporation
ACME
```
Estas referencias podrían representar la misma entidad.

> Por eso utilizamos Entity Resolution.
```
Acme
Acme Corp.
Acme Corporation
      ↓
Entity Resolution
      ↓
entidad canónica
```
Por ejemplo:
```
canonical_name:
Acme Corporation
```
Después necesitamos localizar el nodo correspondiente en Neo4j.

Ese nodo puede tener un identificador:
```
node_id:
acme_001
```
Es importante diferenciar:
```
canonical_name
→ forma normalizada de representar la entidad
````
```
node_id
→ identificador del nodo dentro del graph
```

Conceptualmente:
```
"Acme"
   ↓
Entity Resolution
   ↓
"Acme Corporation"
   ↓
acme_001
```
> Por lo tanto, acme_001 representa el ID del nodo de Acme en el grafo, no su embedding ni su chunk_id.

- [Volver arriba](#)

---

### 43. Query Templates

Una vez obtenido el node ID, debemos consultar Neo4j.

Baswe propone utilizar una biblioteca de query templates.

En lugar de permitir que el LLM invente cualquier consulta, definimos previamente consultas permitidas.

Conceptualmente:
```
QUERY TYPE
    ↓
QUERY TEMPLATE
    ↓
Cypher controlado
```
Por ejemplo:
```
query_type:
company_acquisition_products

parameter:
company_id = acme_001
```
La template correspondiente contiene la estructura de la consulta.

¿Por qué?

Porque queremos evitar:
```
LLM
 ↓
Cypher arbitrario
 ↓
Neo4j
```
Y preferimos:
```
LLM
 ↓
determina query type
 ↓
proporciona parámetros
 ↓
template controlada
 ↓
Cypher
 ↓
Neo4j
```
Esto proporciona:
```
SEGURIDAD
+
REPRODUCIBILIDAD
```
Seguridad

Limitamos las operaciones que el modelo puede solicitar.

Reproducibilidad

La misma clase de pregunta utiliza una estructura de consulta conocida y controlada.

- [Volver arriba](#)

---

### 44. Cypher parametrizado

Neo4j utiliza Cypher como lenguaje de consulta.

Conceptualmente podemos querer expresar:
```
Acme
  ↓ ACQUIRED
Empresa X
  ↓ DEVELOPED
Producto Y
```
Una consulta Cypher podría tener una estructura como:
```
MATCH (a)-[:ACQUIRED]->(b)-[:DEVELOPED]->(p)
WHERE a.id = $company_id
RETURN p
```
No es necesario memorizar esta consulta todavía.

Lo importante es entender su lógica:
```
MATCH
→ busca un patrón dentro del graph

WHERE
→ restringe la búsqueda

RETURN
→ devuelve el resultado
```
$company_id es un parámetro.

Por ejemplo:
```
company_id = acme_001
```
La consulta utiliza ese parámetro sin que el modelo tenga que construir libremente todo el Cypher.

Flujo completo
```
Pregunta
   ↓
Entity Extraction
   ↓
Acme
   ↓
Entity Resolution
   ↓
acme_001
   ↓
Query Type
   ↓
Query Template
   ↓
Cypher parametrizado
   ↓
Neo4j
```
- [Volver arriba](#)

---

### 45. Logging y observabilidad

Baswe indica:

"Log every routing decision with the question and the outcome."

Esto significa registrar las decisiones tomadas por el router.

Ejemplo:

```
Question:
"¿Qué empresas adquirió Acme?"

Route:
GRAPH

Confidence:
0.94

Outcome:
3 entidades encontradas
```
Otro ejemplo:
```
Question:
"¿Qué es pgvector?"

Route:
VECTOR

Confidence:
0.98

Outcome:
5 chunks recuperados
```
#### ¿Por qué?

Porque si el sistema falla, necesitamos saber dónde ocurrió el problema.

Sin logs:
```
Respuesta incorrecta
       ↓
      ¿?
```
Con logs:
```
Pregunta
   ↓
Router → GRAPH
   ↓
Confidence = 0.51
   ↓
Neo4j
   ↓
0 resultados
```
Ahora tenemos información para investigar.

Esto es parte de la observabilidad del sistema.

#### Debugging

Un sistema de IA puede entenderse como una cadena:
```
Pregunta
   ↓
Router
   ↓
Retrieval
   ↓
Context
   ↓
LLM
   ↓
Respuesta
```

Si la respuesta es incorrecta, los logs ayudan a determinar en qué etapa pudo aparecer el problema.

#### Relación con Phase 5

Estos logs también pueden utilizarse posteriormente para analizar el comportamiento del sistema.

Por ejemplo:
```
100 preguntas

60 → VECTOR
30 → GRAPH
10 → BOTH
```
Así podemos medir cómo está funcionando realmente el routing.

La filosofía es:

> No basta con decir que el sistema funciona. Debemos poder observar y medir su comportamiento

- [Volver arriba](#)

---

### 46. Integración de Phase 3

El flujo completo de Phase 3 puede representarse así:

                         QUESTION
                             ↓
                           ROUTER
                             ↓
                 ┌───────────┴───────────┐
                 ↓                       ↓
              VECTOR                   GRAPH
                 │                       ↓
                 │                Entity Extraction
                 │                       ↓
                 │                Entity Resolution
                 │                       ↓
                 │                    Node ID
                 │                       ↓
                 │                 Query Type
                 │                       ↓
                 │                Query Template
                 │                       ↓
                 │             Parameterized Cypher
                 │                       ↓
                 │                     Neo4j
                 │                       │
                 └───────────┬───────────┘
                             ↓
                           RESULTS

El router también registra:
```
QUESTION
ROUTE
CONFIDENCE
OUTCOME
```
Y cuando la confianza es baja:

                 QUESTION
                    ↓
                  ROUTER
                    ↓
             confidence baja
                /       \
               ↓         ↓
           VECTOR       GRAPH
                \       /
                 \     /
                  MERGE

- [Volver arriba](#)

---

### 47. Idea clave

Phase 3 introduce la capacidad del sistema de elegir el mecanismo de retrieval adecuado para cada pregunta.
```
Pregunta
   ↓
Router
   ↓
¿qué tipo de información necesito?
   │
   ├── contenido / significado
   │       ↓
   │     VECTOR
   │
   └── relaciones / conexiones
           ↓
         GRAPH
```
El Graph Retrieval sigue este proceso:
```
Pregunta
   ↓
Entity Extraction
   ↓
Entity Resolution
   ↓
Node ID
   ↓
Query Template
   ↓
Parameterized Cypher
   ↓
Neo4j
```
El router debe ser observable:
```
Question
Route
Confidence
Outcome
```
Y debe tener una estrategia para la incertidumbre:
```
confidence alta
    → ruta elegida

confidence baja
    → VECTOR + GRAPH
    → MERGE
```
#### Concepto central de Phase 3

> El router no responde la pregunta: decide cómo buscar la información necesaria para responderla

- [Volver arriba](#)

---

### 48. Objetivo de Phase 4

En las fases anteriores construimos dos formas diferentes de recuperar información:

- **Vector Retrieval** → encuentra información por significado semántico.
- **Graph Retrieval** → encuentra información mediante relaciones entre entidades.

Phase 4 busca combinar ambas fuentes para construir una **única respuesta grounded**.

La idea central:

> Vector aporta significado/contexto; Graph aporta relaciones; juntos permiten construir una respuesta respaldada por evidencias.

[↑ Volver al índice](#)

---

### 49. Vector y Graph aportan evidencias diferentes

#### Vector

El Vector Retrieval recupera fragmentos de texto relevantes:

```text
Pregunta
   ↓
Vector Search
   ↓
Passages
```

Estos passages contienen información textual y contexto.

Ejemplo:
```
"Beta Technologies was acquired by Acme in 2022.
Beta developed the Orion platform."
```
#### El Graph

El Graph Retrieval recupera relaciones entre entidades:
```
Pregunta
   ↓
Graph Search
   ↓
Path
```
Ejemplo:
```
Acme ──ACQUIRED──> Beta ──DEVELOPED──> Orion
```
El Graph permite ver la estructura de la relación.

La diferencia fundamental:
```
VECTOR
→ significado / contenido / contexto

GRAPH
→ relaciones / estructura / conexiones
```
> No son equivalentes ni uno reemplaza necesariamente al otro.

- [Volver arriba](#)

---

### 50. Passage vs Path

Es importante distinguir estos conceptos.

#### Passage

Un passage es un fragmento de texto recuperado.

Responde principalmente:

¿Qué dice la información?

Ejemplo:
```
"Beta developed the Orion platform."
```

#### Path

Un path es un recorrido dentro del Graph que conecta entidades mediante relaciones.

Responde principalmente:

¿Cómo están relacionadas las entidades?

Ejemplo:
```
Acme ──ACQUIRED──> Beta ──DEVELOPED──> Orion
```
- [Volver arriba](#)

---

### 51. Merge: combinar las evidencias

Merge NO significa simplemente:
```
"Respuesta de Vector + respuesta de Graph."
```
El sistema no debería generar dos respuestas independientes.

En cambio, debe combinar las evidencias recuperadas.
```
VECTOR
   ↓
Passages
   │
   ├─────────┐
             │
GRAPH       │
   ↓         │
Paths        │
             ▼
      Evidencias
             ↓
     Selección / filtrado
             ↓
       Evidence Set
             ↓
            LLM
             ↓
      Respuesta única
```
El objetivo es que cada fuente aporte aquello en lo que es más útil.

Por ejemplo:

VECTOR
```
"Beta developed the Orion platform."
```
GRAPH
```
Acme ──ACQUIRED──> Beta ──DEVELOPED──> Orion
```
Juntas permiten construir:

>Acme está relacionado con Orion porque adquirió Beta Technologies, empresa que desarrolló Orion

- [Volver arriba](#)

---
### 52. Grounded Answer

Una respuesta grounded es una respuesta cuyas afirmaciones están respaldadas por la evidencia recuperada.

No basta con que la respuesta sea probablemente correcta.

El sistema debe poder decir:
```
Esta afirmación está respaldada por la evidencia que recuperamos.
```
Por ejemplo:
```
EVIDENCIA
Beta ──DEVELOPED──> Orion

        ↓

RESPUESTA
"Beta desarrolló Orion."
```
La respuesta está grounded porque la afirmación puede relacionarse directamente con la evidencia.

Una distinción fundamental
```
LLM
→ genera el lenguaje

EVIDENCIA
→ proporciona el fundamento
```
> El LLM no debería inventar relaciones que no aparecen respaldadas por la evidencia recuperada
- [Volver arriba](#)

---

### 53. Filtrar la evidencia antes del LLM

El retrieval puede devolver información que no sea útil para la pregunta.

Por eso no necesariamente debemos entregar todo lo recuperado directamente al LLM.

Conceptualmente:
```
VECTOR ──→ passages ──┐
                      │
GRAPH  ──→ paths ─────┤
                      ↓
              FILTRAR / SELECCIONAR
                      ↓
                EVIDENCE SET
                      ↓
                     LLM
```

El objetivo es evitar ruido.

Una evidencia irrelevante podría hacer que el modelo:
```
mezcle información;
se distraiga;
genere una respuesta menos precisa.
```
Importante

Filtrar no significa necesariamente quedarse con una sola evidencia.

Podemos conservar varias evidencias si son complementarias.

Por ejemplo:
```
3 passages relevantes
+
1 path relevante
=
4 evidencias útiles
```
> No existe una regla que diga que siempre debe haber más passages que paths.

- [Volver arriba](#)

---

### 54. Evidence Set

El evidence set es el conjunto de evidencias seleccionadas que finalmente se entrega al LLM.

Podemos imaginarlo como una:

"carpeta temporal de pruebas"

Ejemplo:
```
EVIDENCE SET

[1] Passage
"Beta Technologies was acquired by Acme in 2022."

[2] Passage
"Beta developed the Orion platform."

[3] Graph Path
Acme ──ACQUIRED──> Beta ──DEVELOPED──> Orion
```
El evidence set puede contener diferentes tipos de evidencia:
```
Passages
+
Paths
+
otras evidencias relevantes
```
> Lo importante no es la cantidad sino su relevancia y suficiencia para responder la pregunta

- [Volver arriba](#)

---

### 55. El papel del LLM

Hasta llegar al LLM, el sistema ha realizado principalmente tareas de recuperación y preparación:
```
Pregunta
   ↓
Router
   ↓
Retrieval
   ↓
Evidencias
   ↓
Filtrado / selección
   ↓
Evidence Set
```
Ahora entra el LLM.

Su función es:
```
interpretar las evidencias;
determinar qué información sirve para responder;
combinar evidencias complementarias;
generar una respuesta en lenguaje natural.
```
Por eso podemos resumir:
```
Retrieval encuentra las pruebas; el LLM las interpreta y las convierte en una respuesta
```

- [Volver arriba](#)

---

### 56. Grounding Prompt

El LLM necesita instrucciones claras sobre cómo utilizar la evidencia.

Conceptualmente:
```
PREGUNTA
+
EVIDENCE SET
+
INSTRUCCIONES DE GROUNDING
        ↓
       LLM
        ↓
RESPUESTA
```
Una instrucción conceptual podría ser:
```
Responde utilizando únicamente la evidencia proporcionada.
No inventes información.
Si la evidencia no es suficiente, indícalo.
```
> El objetivo es reducir la posibilidad de que el modelo rellene vacíos con información no respaldad

- [Volver arriba](#)

---

### 57. Evidencia insuficiente

Una regla fundamental:

Si la evidencia recuperada no alcanza para responder, el LLM debe reconocerlo.

Ejemplo:
```
EVIDENCE SET
"No existe información suficiente."
        ↓
       LLM
        ↓
"No hay suficiente evidencia recuperada
para determinar la respuesta."
```
Esto es preferible a que el modelo utilice conocimiento general para completar el vacío.

#### Grounding y alucinaciones

El grounding ayuda a controlar las alucinaciones, pero no significa que el LLM automáticamente deje de alucinar.

La calidad de la respuesta depende también de:
```
calidad del retrieval;
relevancia de las evidencias;
filtrado;
calidad del prompt;
capacidad del modelo.
```
> Por eso el pipeline completo importa.
- [Volver arriba](#)

---


### 58. Ejemplo completo

Pregunta:

¿Qué producto desarrolló Beta?

Tenemos:

VECTOR
```
"Beta Technologies was acquired by Acme in 2022."
```
Este passage es irrelevante para la pregunta.

Pero Graph devuelve:

GRAPH
```
Beta ──DEVELOPED──> Orion
```
Entonces:
```
VECTOR
Acme adquirió Beta
       ↓
   irrelevante ❌


GRAPH
Beta ──DEVELOPED──> Orion
       ↓
   relevante ✅
```
El evidence set puede contener solamente la evidencia relevante:
```
EVIDENCE SET

Beta ──DEVELOPED──> Orion
```
El LLM puede generar:
```
Beta desarrolló Orion.
```
>La respuesta está grounded porque existe evidencia directa que respalda la afirmación

- [Volver arriba](#)

---

### 59. Arquitectura de Phase 4

El flujo conceptual que construimos queda así:

                         PREGUNTA
                            │
                            ▼
                         ROUTER
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
              VECTOR                 GRAPH
                 │                     │
                 ▼                     ▼
             Passages                Paths
                 │                     │
                 └──────────┬──────────┘
                            ▼
                   FILTRAR / SELECCIONAR
                            │
                            ▼
                      EVIDENCE SET
                            │
                            ▼
                    GROUNDING PROMPT
                            │
                            ▼
                           LLM
                            │
                            ▼
                    RESPUESTA FINAL
                            │
                   ┌────────┴────────┐
                   ▼                 ▼
          Evidencia suficiente   Evidencia insuficiente
                   │                 │
                   ▼                 ▼
          Respuesta grounded    Reconocer limitación
- [Volver arriba](#)

---

### 60. Idea central de Phase 4

La idea que debemos llevarnos de esta fase es:
```
Vector aporta significado y contexto; Graph aporta relaciones; el sistema selecciona las evidencias relevantes, las reúne en un Evidence Set y el LLM las utiliza para generar una respuesta grounded.
```
En forma todavía más simple:
```
VECTOR
   ↓
SIGNIFICADO

GRAPH
   ↓
RELACIONES

      ↓

EVIDENCE SET
      ↓
     LLM
      ↓
RESPUESTA GROUNDED
```
Y una última regla fundamental:

> Si la evidencia no alcanza, el sistema debe reconocerlo en lugar de inventar una respuesta

- [Volver arriba](#)

---

## FASE 05: Evaluation

### 61. Objetivo de Phase 5

La pregunta central es:

> ¿Cómo sabemos si nuestro RAG realmente funciona bien?

No basta con mirar una respuesta y decidir:

> "Parece correcta."

Necesitamos poder:

- medir el sistema;
- detectar cuándo falla;
- identificar en qué etapa ocurrió el fallo;
- comparar versiones;
- comprobar si una modificación mejoró o empeoró el sistema.

La evaluación convierte el RAG en un sistema **medible y testeable**.

- [Volver arriba](#)

---

### 62. Por qué evaluar por capas

Un RAG tiene varias etapas.

```text
PREGUNTA
   ↓
ROUTER
   ↓
RETRIEVAL
   ↓
EVIDENCE SET
   ↓
LLM
   ↓
RESPUESTA
````
Una respuesta final puede ser correcta aunque el Retrieval haya sido malo.

Ejemplo:
```
Retrieval → recupera evidencia incorrecta
LLM → responde correctamente por casualidad
```
Si solamente evaluamos la respuesta final, podríamos concluir:
```
"El sistema funciona."
```
Pero internamente existe un fallo.

Por eso debemos evaluar distintas capas:

```
RETRIEVAL
    ↓
¿Encontró la evidencia correcta?

GENERATION
    ↓
¿Construyó una respuesta correcta?

FAITHFULNESS
    ↓
¿La respuesta está respaldada por la evidencia?
```

- [Volver arriba](#)

---

### 63. Evaluación del Retrieval

El Retrieval debe responder:
```
¿Encontramos la evidencia relevante que necesitábamos?
```
Un problema típico es recuperar:
```
mucha paja y poco trigo
```
Es decir, muchos resultados irrelevantes y pocos resultados útiles.

Para medir esto utilizamos, entre otras cosas:
```
Precision
Recall
Recall@k
```

- [Volver arriba](#)

---

### 64. Precision y Recall

#### Precision

Pregunta:
```
De todo lo que recuperé, ¿cuánto era relevante?
```
Conceptualmente:
```
Precision =
resultados relevantes recuperados
---------------------------------
total de resultados recuperados
```
Ejemplo:
```
Recuperamos 10 resultados.
Solo 2 son relevantes.
```
```
Precision = 2 / 10 = 20%
```
Hay mucha paja y poco trigo.

#### Recall

Pregunta:
```
De toda la evidencia relevante que existía, ¿cuánto conseguimos recuperar?
```
Conceptualmente:
```
Recall =
resultados relevantes recuperados
---------------------------------
total de resultados relevantes existentes
```
Ejemplo:
```
Existen 5 evidencias relevantes.
Recuperamos 3.
```
```
Recall = 3 / 5 = 60%
```
Ejemplo importante

Si:
```
20 resultados relevantes existen.
Recuperamos 10.
Los 10 son relevantes.
```
Entonces:
```
Precision = 10 / 10 = 100%

Recall = 10 / 20 = 50%
```
>Por eso Precision y Recall miden cosas diferentes.

- [Volver arriba](#)

---

### 65. Recall@k

k representa cuántos resultados superiores estamos evaluando.

Por ejemplo:
```
recall@5
```

significa que evaluamos los primeros 5 resultados recuperados.

Conceptualmente:

>De la evidencia relevante que necesitábamos, ¿cuánta apareció dentro de los primeros k resultados?

Aumentar k normalmente aumenta las oportunidades de encontrar evidencia relevante.

Pero también puede aumentar el ruido:
```
k pequeño
→ menos ruido
→ mayor posibilidad de perder evidencia

k grande
→ más oportunidades de encontrar evidencia
→ más resultados irrelevantes
```
> Existe, por tanto, un trade-off.

Importante:
```
Recall@k evalúa Retrieval. No evalúa directamente la calidad de la respuesta final del LLM.
```
- [Volver arriba](#)

---

### 66. Evaluation Dataset


Para evaluar sistemáticamente el RAG necesitamos un conjunto de preguntas de prueba.

Por ejemplo:
```
Q1 → ¿Qué empresa adquirió Beta?
Q2 → ¿Qué producto desarrolló Beta?
Q3 → ¿Qué empresa desarrolló Orion?
...
Q100 → ...
```
Esto constituye nuestro:

> Evaluation Dataset

La idea es parecida a una suite de pruebas de QA.

Cada pregunta debe tener información de referencia que nos permita saber si el sistema funcionó correctamente.

- [Volver arriba](#)

---

### 67. Ground Truth

Cada pregunta puede tener una respuesta de referencia:
```
Pregunta:
¿Qué producto desarrolló Beta?

Ground truth:
Orion
```
Ground truth significa la respuesta o referencia considerada correcta para ese caso.

No necesariamente tiene que ser una única frase exacta.

Puede haber varias respuestas lingüísticamente diferentes que sean correctas.

Por ejemplo:
```
Ground truth:
Beta desarrolló Orion.

Respuesta:
El producto desarrollado por Beta fue Orion.
```

Ambas expresan esencialmente la misma información.

- [Volver arriba](#)

---

### 68. Exact Match vs Semantic Similarity


#### Exact Match

Compara literalmente el texto.
```
Ground truth:
Beta desarrolló Orion.

Respuesta:
El producto desarrollado por Beta fue Orion.
```
No son exactamente iguales.

Por lo tanto:
```
Exact Match → ❌
```
Pero la respuesta puede ser semánticamente correcta.

#### Semantic Similarity

Compara el significado de los textos.

Esto conecta con lo estudiado anteriormente sobre embeddings:
```
Texto
  ↓
Embedding
  ↓
Vector
  ↓
Comparación semántica
```
Las dos frases anteriores probablemente serían consideradas muy similares.

Pero semantic similarity tampoco es suficiente por sí sola.

Ejemplo:
```
Ground truth:
Beta desarrolló Orion en 2022.

Respuesta:
Beta desarrolló Orion en 2024.
```
Las frases son semánticamente muy similares.

Pero la respuesta contiene un dato incorrecto.

Por eso necesitamos otras dimensiones de evaluación

- [Volver arriba](#)

---

### 69. Correctness

Correctness pregunta:

¿La respuesta es correcta respecto al ground truth?

Ejemplo:
```
Ground truth:
Acme adquirió Beta en 2022.

Respuesta:
Acme adquirió Beta en 2022.
```
```
Correctness → ✅
```
Pero correctness no nos dice necesariamente si la respuesta está respaldada por la evidencia recuperada.

- [Volver arriba](#)

---

### 70. Faithfulness / Groundedness

Faithfulness o groundedness pregunta:

>¿La respuesta está respaldada por la evidencia que recibió el LLM?

Ejemplo:
```
Evidence:
Beta --DEVELOPED--> Orion

Respuesta:
Beta desarrolló Orion.
```
```
Faithfulness → ✅
```
Pero:
```
Evidence:
Beta --DEVELOPED--> Orion

Respuesta:
Beta desarrolló Orion en 2022.
```
```
Faithfulness → ❌
```
Porque "en 2022" no aparece en la evidencia.

Una idea fundamental:

>No basta con que una afirmación sea verdadera. En un RAG queremos que esté sustentada por la evidencia recuperada.

Faithfulness ayuda a detectar cuando el LLM agrega información no respaldada.

- [Volver arriba](#)

---

### 71. Claims

Una respuesta puede dividirse en afirmaciones individuales llamadas:

>Claims

Ejemplo:

Respuesta:
```
"Beta desarrolló Orion en 2022 y lo lanzó al mercado en 2023."
```
Podemos descomponerla:
```
Claim 1 → Beta desarrolló Orion.
Claim 2 → Beta desarrolló Orion en 2022.
Claim 3 → Beta lanzó Orion al mercado en 2023.
```
Después comparamos cada claim con la evidencia.

Si la evidencia solamente dice:
```
Beta desarrolló Orion.
```
Entonces:
```
Claim 1 → ✅ respaldado
Claim 2 → ❌ no respaldado
Claim 3 → ❌ no respaldado
```
Una forma conceptual de medir faithfulness es:
```
Faithfulness =
claims respaldados
------------------
total de claims
```
Ejemplo:
```
4 claims
3 respaldados

3 / 4 = 75%
```
Esto NO significa:

>"La respuesta es 75% correcta."

Significa:

>"El 75% de sus afirmaciones están respaldadas por la evidencia."

- [Volver arriba](#)

---

### 72. Correctness vs Faithfulness


Son dimensiones diferentes.
```
CORRECTNESS
    ↓
¿La respuesta es correcta respecto al
ground truth?

FAITHFULNESS
    ↓
¿La respuesta está respaldada por
la evidencia?
```
Una respuesta puede ser:
```
Correcta + Faithful
Correcta + No Faithful
Incorrecta + Faithful
Incorrecta + No Faithful
```
Por ejemplo:
```
Ground truth:
Beta desarrolló Orion en 2022.

Evidence:
Beta desarrolló Orion en 2022.

Respuesta:
Beta desarrolló Orion.
```
Resultado:
```
Correctness   → ✅
Faithfulness  → ✅
```
La respuesta es menos completa, pero correcta y respaldada.

- [Volver arriba](#)

---

### 73. Expected Evidence


Además del ground truth podemos definir qué evidencia esperamos que el sistema encuentre.

Ejemplo:
```
Question:
¿Qué empresa adquirió Beta?

Ground truth:
Acme

Expected evidence:
Acme --ACQUIRED--> Beta
```
Así podemos comparar:
```
Expected Evidence
        ↕
Retrieved Evidence
```
Esto permite evaluar específicamente Retrieval.

Un caso de evaluación puede contener:
```
Question
Ground Truth
Expected Evidence
```
Después de ejecutar el RAG obtenemos:
```
Retrieved Evidence
Generated Answer
```

- [Volver arriba](#)

---

### 74. Diagnóstico por capas


Podemos pensar en tres evaluaciones principales:
```
Expected Evidence
        ↕
Retrieved Evidence
        ↓
    RETRIEVAL


Ground Truth
        ↕
Generated Answer
        ↓
   CORRECTNESS


Retrieved Evidence
        ↕
Generated Answer
        ↓
  FAITHFULNESS
```

Ejemplo:
```
Ground truth:
Acme adquirió Beta.

Expected evidence:
Acme --ACQUIRED--> Beta

Retrieved evidence:
Acme --DEVELOPED--> Orion

Generated answer:
Acme adquirió Beta.
```
Diagnóstico:
```
Retrieval    → ❌
Correctness  → ✅
Faithfulness → ❌
```
La respuesta final es correcta, pero el sistema llegó a ella sin recuperar la evidencia adecuada.

Eso es importante porque una respuesta correcta puede ocultar un fallo interno.

- [Volver arriba](#)

---

### 75. Regression Testing


El Evaluation Dataset funciona también como una suite de regresión.

Supongamos:
```
Versión A

Correctness:   86%
Faithfulness:  92%
```
Modificamos el Router.

Volvemos a ejecutar las mismas preguntas:
```
Versión B

Correctness:   79%
Faithfulness:  88%
```
La modificación empeoró el sistema.

El ciclo de ingeniería es:
```
Medir
  ↓
Detectar fallo
  ↓
Identificar dónde ocurre
  ↓
Modificar
  ↓
Volver a medir
  ↓
Comparar
```
Esto convierte la evaluación en parte del desarrollo del sistema.

- [Volver arriba](#)

---

### 76. LLM-as-a-Judge

Algunas evaluaciones son difíciles de resolver mediante reglas simples.

Por ejemplo:
```
Ground truth:
Acme adquirió Beta en 2022.

Respuesta:
La empresa Acme compró Beta durante el año 2022.
```
La respuesta puede ser correcta aunque no coincida literalmente.

Podemos utilizar otro LLM como evaluador:

             RAG
              ↓
          RESPUESTA
              ↓
        ┌─────────────┐
        │ LLM Judge   │
        └─────────────┘
              ↓
        evaluación

El Judge puede recibir:
```
Pregunta
Ground Truth
Evidencia
Respuesta generada
```
Y evaluar aspectos como:
```
Correctness
Faithfulness
Relevance
```
El LLM-as-a-Judge permite automatizar parte de la evaluación.

- [Volver arriba](#)

---

### 77. Human Calibration


No debemos asumir que el LLM Judge es automáticamente correcto.

El Judge también puede equivocarse:
```
RAG
 ↓
LLM Judge
 ↓
¿Podemos confiar?
```
Para comprobarlo podemos comparar sus evaluaciones con evaluaciones humanas.

Ejemplo:
```
100 casos
   │
   ├── LLM Judge → evalúa
   │
   └── Humanos → evalúan una muestra
                  ↓
             comparación
```
Si ambos coinciden frecuentemente:
```
Judge ≈ criterio humano
```
Si existen discrepancias importantes:
```
Judge ≠ criterio humano
```
> Debemos investigar y calibrar el sistema de evaluación.

Por eso:
```
LLM-as-a-Judge no debe considerarse una verdad absoluta.
```
Esto conecta directamente con el proyecto posterior de Baswe:
```
LLM-as-a-Judge with Human Calibration
```
> Ese es el cuarto proyecto de la secuencia original de Baswe; no significa AI Lab Project 004.

- [Volver arriba](#)

---

### 78. Pipeline completo de evaluación


Nuestro RAG:
```
PREGUNTA
   ↓
ROUTER
   ↓
RETRIEVAL
   ↓
EVIDENCE SET
   ↓
LLM
   ↓
RESPUESTA
```
Con evaluación:

```

                         ┌──────────────────────┐
                         │   EVALUATION DATASET │
                         │                      │
                         │ Question             │
                         │ Ground Truth         │
                         │ Expected Evidence    │
                         └──────────┬───────────┘
                                    │
                                    ↓
Pregunta → RAG → Evidence → Respuesta
                    │           │
                    │           │
                    ↓           ↓
              RETRIEVAL    GENERATION
                    │           │
                    └─────┬─────┘
                          ↓
                    EVALUATION
                          ↓
              ┌───────────┼───────────┐
              ↓           ↓           ↓
          Retrieval   Correctness  Faithfulness
```
Podemos agregar:
```
                    EVALUATION
                         ↓
                  LLM-as-a-Judge
                         ↓
                  Human Calibration
```

- [Volver arriba](#)

---


### 79. Idea central de Phase 5


La idea principal de toda la fase:

No evaluamos solamente si el RAG respondió bien. Evaluamos si recuperó bien, si respondió correctamente y si su respuesta está sustentada por la evidencia.

Nuestro sistema deja de ser:
```
"Creo que funciona."
```
y pasa a ser:
```
"Lo probamos con un conjunto de casos
y podemos medir dónde funciona y dónde falla."
```
Esto permite:
```
MEDIR
  ↓
DETECTAR
  ↓
DIAGNOSTICAR
  ↓
MEJORAR
  ↓
VOLVER A MEDIR
```
Eso es ingeniería.

- [Volver arriba](#)

---

### 80. Cierre y paso siguiente

Phase 5 completada conceptualmente

Hemos estudiado:
```
Retrieval
Precision
Recall
Recall@k
Evaluation Dataset
Ground Truth
Expected Evidence
Exact Match
Semantic Similarity
Correctness
Faithfulness / Groundedness
Claims
LLM-as-a-Judge
Human Calibration
Regression Testing
Evaluación por capas
```
Arquitectura conceptual final del Project 001
```
DOCUMENTOS
    ↓
CHUNKS
    ↓
 ┌───────────────┐
 │               │
 ↓               ↓
GRAPH          VECTOR
 │               │
 └───────┬───────┘
         ↓
       ROUTER
         ↓
     RETRIEVAL
         ↓
    EVIDENCE SET
         ↓
       LLM
         ↓
     RESPUESTA
         ↓
    EVALUATION
         ↓
 ┌───────┼──────────┐
 ↓       ↓          ↓
Retrieval Correctness Faithfulness
```

- [Volver arriba](#)

---
- [Volver arriba](#)

---
- [Volver arriba](#)

---

- [Volver arriba](#)

---
- [Volver arriba](#)

---
- [Volver arriba](#)

---
- [Volver arriba](#)

---
# 2X. Estado de Project 001

```
Comprendido conceptualmente
Vector RAG
embeddings
semantic retrieval
chunks
Knowledge Graph
nodes
relationships
properties
hops
graph traversal
Neo4j
conceptos básicos de Cypher
MATCH
RETURN
extracción de entidades y relaciones
entity resolution
canonical entity/name
source chunk ID
grounding
coexistencia de graph + vector retrieval
routing conceptual

Todavía pendiente

chunking strategies
embeddings aplicados al proyecto
pgvector en detalle
metadata
HNSW
recall@k
graph construction real
schema estructurado completo
entity resolution implementado
Cypher real sobre Neo4j
router implementado
hybrid retrieval
answer synthesis
citation validation
benchmark
comparación contra Vector RAG
FastAPI
evaluación
costes y latencia
```

### Punto exacto de reentrada

Phase 2 — Build the vector index alongside the graph

Próximo concepto:
```
CHUNKS
   ↓
EMBEDDINGS
   ↓
PGVECTOR
   ↓
METADATA
```

- [Volver arriba](#)

---


# 21. 🕸️ Grafo compacto de nuestro avance

Este es el que yo guardaría como **snapshot visual**:

```text
PROJECT 001 — BASWE
│
├── Vector RAG
│   ├── Chunks
│   ├── Embeddings
│   └── Semantic Retrieval
│
└── Knowledge Graph RAG
    │
    ├── Entities
    ├── Relationships
    ├── Entity Resolution
    ├── Neo4j
    │   └── Cypher
    │
    ├── Source Chunk ID
    │   └── Grounding
    │
    └── Vector Store
        └── pgvector

              ↓
            Router
          ↙       ↘
       Vector     Graph
        RAG        RAG
          ↘       ↙
           Answer



                            ★ ESTAMOS AQUÍ
                       ↓
                Chunks + Embeddings
                       ↓
                    pgvector

```

- [Volver arriba](#)

---

# 22. 🔄 Texto para retomar trabajo

Te propongo conservar exactamente este texto:

Retomemos Project 001 — Baswe desde Study Notes 01. Ya comprendimos conceptualmente Vector RAG, embeddings, chunks, Knowledge Graph, hops, Neo4j/Cypher, entity resolution, source chunk IDs, grounding y el rol de pgvector. Quedamos entrando en Phase 2: chunks → embeddings → pgvector → metadata. Sigamos secuencialmente con baby steps y checkpoints, sin saltarnos conceptos.


- [Volver arriba](#)

---

- [Volver arriba](#)

---

- [Volver arriba](#)

---

- [Volver arriba](#)

---

- [Volver arriba](#)

---