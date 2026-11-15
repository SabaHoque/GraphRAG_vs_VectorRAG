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
| `notebooks/01_data-preparation-and-graphrag-neo4.ipynb` | Data card, EDA, extraction, graph load into Aura, GraphRAG pipeline |
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

Main generator `qwen2.5:7b-instruct`, 34 questions (27 answerable + 7 unanswerable), temperature 0. Source: `results/table_overall.csv`.

| Metric | GraphRAG | VectorRAG-dense | VectorRAG-hybrid | VectorRAG-fact |
|---|---|---|---|---|
| **Accuracy, answerable (n = 27)** | **0.81** (22/27) | 0.30 (8/27) | 0.26 (7/27) | 0.44 (12/27) |
| Accuracy, all 34 | 0.85 [0.71, 0.97] | 0.44 [0.26, 0.62] | 0.41 [0.24, 0.59] | 0.56 [0.38, 0.71] |
| Refusal accuracy (n = 7) | 1.00 | 1.00 | 1.00 | 1.00 |
| Hit@5 | 1.00 | 0.93 | 0.93 | 0.81 |
| Recall@5 / Recall@10 | 0.94 / 1.00 | 0.81 / 0.83 | 0.78 / 0.82 | 0.70 / 0.79 |
| Precision@5 | 0.76 | 0.56 | 0.52 | 0.35 |
| MRR / nDCG@5 | 0.87 / 0.84 | 0.81 / 0.77 | 0.80 / 0.78 | 0.65 / 0.62 |
| Token F1 | 0.47 | 0.27 | 0.32 | 0.27 |
| BERTScore F1 | 0.84 | 0.76 | 0.79 | 0.77 |
| LLM-judge correctness | 0.66 | 0.57 | 0.52 | 0.53 |
| Faithfulness (supported claims) | 0.93 | 0.94 | 0.91 | 0.93 |
| Citation validity | 1.00 | 1.00 | 1.00 | 1.00 |
| Entity linking P / R / F1 | 1.00 / 1.00 / 1.00 | n/a | n/a | n/a |
| Median retrieval latency | 488 ms | 15 ms | 17 ms | 15 ms |
| Median generation latency | 1.45 s | 2.86 s | 2.77 s | 1.76 s |
| Median prompt tokens | 382 | 2,144 | 2,148 | 808 |

Accuracy intervals are bootstrap 95% CIs over questions. With 34 questions, one question is about 3 accuracy points.

**Key observations**

* **Answer accuracy:** GraphRAG answered 22 of 27 answerable questions with all required items, against 8, 7 and 12 for the vector variants.
* **Retrieval is not GraphRAG's bottleneck.** Its Recall@10 is 1.00, yet 5 answerable questions still failed, so its remaining errors come from generation or answer format, not from missing evidence.
* **Vector RAG finds something but not everything.** Hit@5 is high (0.81-0.93), but Recall@10 is only 0.79-0.83 and accuracy is low: list, count and ranking questions need every matching record, and top-k returns only a few.
* **Smaller chunks helped the generator.** Fact-level chunks beat record-level chunks on accuracy (12 vs 8 questions) despite lower Precision@5 and Recall: retrieval metrics and answer accuracy disagree.
* **Hybrid BM25 + dense did not help** over dense (7 vs 8 questions; within noise).
* **Cost trade-off:** GraphRAG retrieval is about 30x slower (488 ms vs 15 ms: linking, a Cypher round trip to Aura), but its prompts are 5.6x shorter than dense (382 vs 2,144 tokens), so generation is about twice as fast.
* **Grounding is good everywhere:** faithfulness is 0.91-0.94 for all systems, so the main failure mode is *incomplete* answers, not invented ones.

**By question type** (answer accuracy and evidence Recall@10; source `results/table_by_type.csv`; scoring v1, see the re-scoring note below)

| Type (n) | GraphRAG | Vector dense | Vector hybrid | Vector fact | Recall@10 (Graph / dense / hybrid / fact) |
|---|---|---|---|---|---|
| single-hop (7) | 1.00 (7/7) | 1.00 | 1.00 | 1.00 | 1.00 / 1.00 / 1.00 / 1.00 |
| multi-hop (7) | **0.71** (5/7) | 0.14 (1/7) | 0.00 (0/7) | 0.14 (1/7) | 1.00 / 0.59 / 0.62 / 0.61 |
| aggregation (7) | **1.00** (7/7) | 0.00 | 0.00 | 0.29 (2/7) | 1.00 / 0.76 / 0.76 / 0.57 |
| semantic / descriptive (6) | 0.50 (3/6) | 0.00 | 0.00 | 0.33 (2/6) | 1.00 / 1.00 / 0.92 / 1.00 |
| unanswerable (7) | 1.00 | 1.00 | 1.00 | 1.00 | n/a |

* **Single-hop is a tie** (7/7 everywhere): one fact about one named drug is easy for both approaches.
* **The whole gap comes from multi-hop and aggregation.** GraphRAG scores 12/14 on these two types; the vector variants score 1-3/14. These questions need *all* matching records or a join across records, which a top-k similarity search does not return.
* **Semantic questions are the hardest for every system, and the vector variants fail even though the evidence is there:** Recall@10 is about 1.00 for all four, yet dense and hybrid score 0/6. Retrieval is not the reason; the answers do not state the required items, or are scored as refusals (see the note below). GraphRAG's own 3/6 on this type is also below what perfect retrieval would suggest.
* **The expected advantage for vector RAG on descriptive questions did not appear** in this benchmark.

**Re-scoring note (to be completed).** Two aspects of the answer-accuracy metric are being re-checked in notebook 3 (`rescoring_v2.csv`): (a) whether answers were wrongly counted as refusals because
their *text* matched a refusal pattern even though the model had set `answerable=true`, and (b) whether answers that list extra, wrong names were scored as correct. Report v1 and v2 side by side, and state which is used for the conclusions.

**Generator ablation (7B vs 3B)** *(add `results/table_ablation.csv` here)*

**Judge validation against hand labels** *(add agreement and kappa from notebook 3)*

**Error analysis** *(summarise at least five cases from `results/error_cases.csv`)*

**Main finding:** *(write after the by-type table: where GraphRAG wins, where vector RAG ties or wins, and the recommendation)*

**Caveats on these numbers**

* All four systems score 1.00 on refusal and on citation validity, so these two metrics do not separate the systems here. The unanswerable questions name fields the schema note already says are absent, and citation validity only checks that a cited id exists in the prompt.
* Entity-linking F1 of 1.00 is measured on questions built from the data's own names (exact spellings); typos and aliases are not in the benchmark.
* The key-item accuracy checks that required items are present, not that no extra wrong items are listed, so it can flatter systems that over-list. The LLM judge is stricter for GraphRAG (0.66 vs 0.85 on all 34) and ranks the vector variants differently from key-item accuracy, which needs to be explained (see the judge validation above).
* Templates were written knowing the question *types*; GraphRAG's perfect Hit@5 is partly by design.
* 34 questions on 112 drugs: differences of one or two questions are noise, and the GraphRAG and fact-level intervals just touch ([0.71, 0.97] vs [0.38, 0.71]).

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
