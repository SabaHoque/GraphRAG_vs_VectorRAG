# AI-Use Statement


**Project:** PharmaGraph-OTC: GraphRAG vs Vector RAG on over-the-counter drug data

## Tools used
- **Claude** (Anthropic), used through the Claude chat interface
- **ChatGPT** (OpenAI)

## What each tool was used for

**ChatGPT**
- Brainstorming project topics and finding candidate datasets at the start of the project. 

**Claude**
- Comparing candidate datasets from Mendeley Data and checking them against the assignment requirements (size, relationships, licence).
- Designing the graph schema and the rule-based extraction of side effects, contraindications and interactions from free text.
- Writing the first versions of the three Kaggle notebooks: data preparation and GraphRAG pipeline, Vector RAG pipeline, and the benchmark and evaluation code. The retrieval, linking, metric and benchmark-generation code was drafted with Claude.
- Debugging errors I hit while running the notebooks on Kaggle, such as dataset column mapping, Neo4j database name handling and plotting.
- Drafting the README and the structure of this statement.

## What I did myself
- Chose the project topic and the final dataset, and checked its licence and DOI on Mendeley Data.
- Created the Neo4j Aura instance and Kaggle Secrets, and ran every notebook on Kaggle.
- Reviewed the generated code and the Cypher queries, and checked outputs against the data.
- Hand-labelled the sample used to validate the LLM judge, and hand-checked the extracted side-effect and contraindication terms.
- Wrote the analysis, the failure-case discussion and the recommendation in the report in my own words.

## Verification and responsibility
I read, ran and tested the code before submitting it, and I can explain the design decisions and every notebook cell. All reported numbers come from my own runs of the submitted notebooks. I am responsible for every line of code and every statement in this submission.

## Models used inside the project
These are part of the system being studied, not writing aids: Qwen2.5 (7B, 3B and 14B) served through Ollama, as generator, ablation model and LLM judge, and the `BAAI/bge-small-en-v1.5` embedding model.
