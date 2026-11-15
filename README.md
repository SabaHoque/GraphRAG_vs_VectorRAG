# PharmaGraph-OTC: GraphRAG vs Vector RAG on Over-the-Counter Drug Data

> **AIML506 course assignment**, Department of CSE, East West University, Dhaka (Fall 2026)
> Author: **<Sababa Hoque Saba>** · Kaggle: [sababahoquesaba](https://www.kaggle.com/sababahoquesaba) · Student ID: **<2026-2-74-003>**

Two question-answering systems are built over the **same data** and compared on a 34-question benchmark:

| System | Retrieval |
|---|---|
| **GraphRAG** | Neo4j Aura property graph · entity linking · parameterised Cypher templates (+ optional validated Text2Cypher) |
| **Vector RAG** | ChromaDB · `bge-small-en-v1.5` · dense, hybrid (BM25 + dense) and fact-level chunking |

Both use the **same generator and prompt** (local Qwen2.5 served by Ollama), so **only retrieval differs**.

---

## 1. Dataset

**Pharma-safe Index Dataset**: 112 over-the-counter drugs sold in pharmacies in Yogyakarta, Indonesia, stored as nine linked tables
(drug, composition, side effects, contraindications, interactions, dose, usage rules, usage period, price).

* Authors: Sebastian, Restyandito, Nugroho
* Source: Mendeley Data, DOI **[10.17632/m78j7ctwhn.2](https://doi.org/10.17632/m78j7ctwhn.2)**
* Data article: *Data in Brief* (2024), DOI 10.1016/j.dib.2024.110794
* Licence: CC BY per the data article. **Check the Mendeley record.**
* **The raw data is not included in this repository.** Download it from the record above.

## 2. Graph model

```
(Category)<-[:IN_CATEGORY]-(Drug)-[:CONTAINS {amount, unit}]->(Ingredient)<-[:INVOLVES]-(Interaction)<-[:HAS_INTERACTION]-(Drug)
(Drug)-[:MAY_CAUSE {evidence}]->(SideEffectTerm)        extracted from free text
(Drug)-[:CONTRAINDICATED_IN {evidence}]->(ContraTerm)   extracted from free text
(Drug)-[:SOLD_AS {price}]->(Unit)
(Drug)-[:SHARES_INGREDIENT {n}]-(Drug)                  derived
```

Side effects, contraindications and interactions are free text. A **rule-based vocabulary** (`src/ps_vocab.py`) turns them into shared
concept nodes, so that questions such as *"which drugs in the cough category cause drowsiness?"* become graph traversals.
Every extracted link keeps its **source sentence** as provenance.

## 3. Repository layout

| Path | Content |
|---|---|
| `notebooks/01_data-preparation-and-graphrag-neo4j.ipynb` | Data card, EDA, extraction, graph load into Aura, GraphRAG pipeline |
| `notebooks/02_vector-rag.ipynb` | Serialisation, embeddings, ChromaDB, retrieval, top-k and embedding experiments |
| `notebooks/03_evaluation-and-comparison.ipynb` | Benchmark, metrics, LLM judge, ablation, cost, error analysis |
| `src/` | The same pipeline code as plain modules (`rag_common`, `graph_rag`, `vector_rag`, `ps_prep`, `ps_vocab`, `ps_bench`) |
| `results/` | Result tables exported by notebook 3 |
| `figures/` | Graph schema, EDA and result charts |

## 4. How to run (Kaggle)

1. Upload the dataset files to Kaggle as a **public** dataset (cite the Mendeley DOI) and attach it to notebook 1, so that others can re-run it.
2. Create a free **Neo4j Aura** instance. In Kaggle add the secrets `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD` and enable them for the notebook.
   **Never put credentials in a notebook or in this repository.**
3. Run in order: **notebook 1** → **notebook 2** (add notebook 1's output as an input) → **notebook 3** (add the outputs of 1 and 2).
4. Turn **Internet** and **GPU** on (Ollama downloads the models).
5. Keep the Aura instance running until the viva (Aura Free pauses after inactivity).

## 5. Models and settings

| Role | Model |
|---|---|
| Generator (main) | `qwen2.5:7b-instruct` |
| Generator (ablation) | `qwen2.5:3b-instruct` |
| LLM judge | `qwen2.5:14b-instruct` (different from, and larger than, the generators) |
| Embeddings | `BAAI/bge-small-en-v1.5` (compared with MiniLM and bge-base) |

Generation: temperature 0, seed 42, context window 8192, max 400 output tokens, JSON answer schema
(`answerable`, `answer`, `citations`). Answers must come from the numbered evidence only; the system must decline when the data cannot answer.

## 6. Evaluation

34 questions, gold answers computed from the data, five types (≥ 4 each): **single-hop · multi-hop · aggregation · semantic/descriptive · unanswerable**.

Metrics: Hit@5, Recall@5, Precision@5, MRR, nDCG@5, answer accuracy (key-item match), token F1, LLM-judge correctness
(validated against hand labels), faithfulness, citation validity, refusal accuracy, entity-linking P/R/F1, Text2Cypher execution accuracy,
median latency, context size, build cost.

### Results

*(Fill in after running notebook 3. Paste `results/table_overall.csv` and `results/table_by_type.csv` here, then state the main finding with numbers.)*

| System | Accuracy | Refusal acc. | Recall@5 | Median latency |
|---|---|---|---|---|
| GraphRAG | | | | |
| VectorRAG-dense | | | | |
| VectorRAG-hybrid | | | | |
| VectorRAG-fact | | | | |

**Main finding:** *(one or two sentences)*

## 7. Links

| Notebook | Kaggle |
|---|---|
| 1. Data preparation and GraphRAG (Neo4j) | https://www.kaggle.com/code/sababahoquesaba/notebook-1-data-preparation-and-graphrag-neo4j |
| 2. Vector RAG | https://www.kaggle.com/code/sababahoquesaba/notebook-2-vector-rag |
| 3. Evaluation and comparison | https://www.kaggle.com/code/sababahoquesaba/notebook-3-evaluation-and-comparison |

Notebook 2 takes the output of notebook 1 as an input; notebook 3 takes the outputs of notebooks 1 and 2.
Report (PDF): *(add after submission)*

## 8. Scope and limitations

* A study of retrieval methods on a **small curated corpus** (112 drugs, 34 questions). One question moves accuracy by about 3 points.
* **Not medical advice.**
* The data holds no manufacturer, mechanism of action, storage or regulatory information; such questions are *unanswerable by design*.
* Gold answers for side-effect and contraindication questions are defined by the rule-based vocabulary; it is validated on hand-labelled samples.
* Cypher templates were written knowing the *types* of question, not the questions themselves.

## 9. AI-use statement

I used **Claude** and **ChatGPT** as assistants: ChatGPT for early topic and dataset brainstorming; Claude for dataset comparison, schema and pipeline design,
drafting the notebook code, debugging, and drafting this README. I ran and tested everything on Kaggle, reviewed the code, validated the extraction and the
LLM judge with my own hand labels, and wrote the analysis myself. I am responsible for every line submitted. The full statement is in
[`AI_USE_STATEMENT.md`](AI_USE_STATEMENT.md).

## 10. Citation and licence

Dataset: Sebastian, Restyandito, Nugroho. *Pharma-safe Index Dataset*. Mendeley Data, V2. https://doi.org/10.17632/m78j7ctwhn.2

Code licence: *(choose one, for example MIT, and add a `LICENSE` file)*
