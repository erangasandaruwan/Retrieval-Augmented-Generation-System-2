# RAG - Monitoring, Observability & Regression Gating

## Project Overview

> **Application baseline:** This project extends the architecture described in `https://github.com/erangasandaruwan/Retrieval-Augmented-Generation-System` and its `SOLUTION_OVERVIEW.md`.

Most AI projects discontinue once the Retrieval-Augmented Generation (RAG) application can answer questions.

This project goes further.

The objective is to take the production-grade RAG application built in **Project 1** and add a complete **monitoring, observability, quality tracking, and regression-gating layer** around it.

The goal is to demonstrate that the system is not only functional, but also:

- Observable
- Measurable
- Debuggable
- Cost-aware
- Reliable
- Regression-resistant
- Production-ready

In real production environments, building the initial AI system is only part of the work. A large amount of engineering effort goes into understanding whether the system is operating correctly, identifying failures, diagnosing quality degradation, controlling cost, and preventing regressions.

This project demonstrates that system-level thinking.

---

# Problem Statement

A RAG application may appear to work correctly while still suffering from issues such as:

- Slow retrieval or LLM response times
- Poor document retrieval
- Re-ranker degradation
- Hallucinated responses
- Missing or weak citations
- Increased token usage
- Unexpected cost increases
- Prompt regressions
- Model-version regressions
- Intermittent failures
- Unsupported answers
- Changes in configuration that reduce answer quality

Without monitoring and observability, these problems can be difficult to identify and diagnose.

The system therefore needs an operational layer that can answer questions such as:

> What happened when answer quality degraded?

> Which documents and chunks were retrieved?

> How did the re-ranker change the retrieval order?

> Which prompt was sent to the model?

> How many tokens were consumed?

> How much did the request cost?

> Was the final answer supported by retrieved evidence?

> Did a recent code, prompt, model, or configuration change cause a regression?

---

# Project Objectives

The project will extend the existing RAG platform with three major capabilities:

1. **End-to-End AI Tracing**
2. **Production Quality & Reliability Monitoring**
3. **Automated Regression Gating in CI/CD**

---

# High-Level Architecture

<img width="1055" height="1491" alt="image" src="https://github.com/user-attachments/assets/eb68e24a-da3b-4ddc-88b5-7566d74bb0ba" />


---

# Application-Specific Observability Extension

This project extends the RAG architecture described in:

`https://github.com/erangasandaruwan/Retrieval-Augmented-Generation-System`

The existing solution already describes a production-oriented flow with multi-format ingestion, 500–800 token chunking with ~100-token overlap, ChromaDB vector search, BM25, hybrid fusion, cross-encoder re-ranking, evidence-grounded generation, citation/refusal validation, automated evaluation, FastAPI endpoints, OpenTelemetry/Prometheus, Jaeger, Docker, GitHub Actions, and Azure DevOps.

Project 3 therefore focuses on **operating and diagnosing that exact application** rather than rebuilding the RAG pipeline.

The target outcome is that any low-quality response can be traced back to the exact retrieval, ranking, prompt, model, configuration and deployment state that produced it.

---

## Existing RAG Application as the Baseline

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/352f6e80-7fec-4a81-b9ed-88c8dcf7edae" />


The observability layer should wrap every stage of this flow.

---

## Observability Architecture for This Application

Traditional telemetry and AI-specific telemetry should complement each other.

<img width="1448" height="1086" alt="image" src="https://github.com/user-attachments/assets/eb7c57aa-1b82-4370-b9dd-bda1a6ee5234" />


OpenTelemetry answers questions such as *which dependency was slow?* Langfuse answers questions such as *which chunks were retrieved and what prompt produced this answer?*

---

## Trace the Existing Module Boundaries

### `src/ingestion/parser.py`

Create a span such as:

```text
ingestion.parse_document
```

Capture:

```text
document_id
file_type
page_count
section_count
characters_extracted
parse_duration_ms
parser_version
status
error_type
```

Do not attach complete sensitive document text to telemetry by default.

### `src/ingestion/chunker.py`

Create:

```text
ingestion.chunk_document
```

Capture:

```text
document_id
chunk_count
target_chunk_size
overlap_tokens
average_chunk_tokens
minimum_chunk_tokens
maximum_chunk_tokens
chunker_version
```

This is important because chunking changes can directly reduce context recall.

### `src/retrieval/embeddings.py`

Create:

```text
retrieval.embed_query
retrieval.embed_chunks
```

Capture:

```text
provider
embedding_model
model_version
vector_dimension
input_count
token_count
latency_ms
retry_count
```

### `src/retrieval/vector_store.py`

Create:

```text
retrieval.vector_search
```

For every candidate, capture trace-level metadata such as:

```text
chunk_id
document_id
vector_rank
vector_score
source
section
page
```

Example:

```json
{
  "chunk_id": "contract-0082",
  "document_id": "Contract.pdf",
  "vector_rank": 2,
  "vector_score": 0.884,
  "section": "8.2",
  "page": 14
}
```

### `src/retrieval/bm25_search.py`

Create:

```text
retrieval.bm25_search
```

The repository specifically demonstrates why BM25 is useful for exact identifiers such as:

```text
VX-VEH-0012398
```

Capture:

```text
chunk_id
document_id
bm25_rank
bm25_score
search_latency_ms
```

A useful portfolio trace could show:

```text
Query: VX-VEH-0012398

BM25 rank:      1
Vector rank:    9
Hybrid rank:    1
```

That gives a concrete demonstration of the value of hybrid search.

### `src/retrieval/hybrid_retriever.py`

Create:

```text
retrieval.hybrid_fusion
```

Capture:

```text
fusion_strategy
alpha
vector_top_k
bm25_top_k
final_top_k
candidate_count_before_fusion
candidate_count_after_fusion
fusion_latency_ms
```

For each chunk record:

```text
vector_rank
bm25_rank
hybrid_rank
vector_score
bm25_score
hybrid_score
```

This lets you diagnose regressions caused by changing RRF behaviour or the weighted fusion parameter.

### `src/retrieval/reranker.py`

The solution describes the cross-encoder:

```text
cross-encoder/ms-marco-MiniLM-L-6-v2
```

Create:

```text
retrieval.rerank
```

Capture before/after ordering:

```text
Chunk    Hybrid Rank    Rerank Rank    Reranker Score
A             1              3             0.62
B             2              1             0.91
C             3              2             0.80
```

Also record:

```text
reranker_model
reranker_version
input_candidate_count
output_candidate_count
rerank_latency_ms
```

### `src/generation/rag_chain.py`

Use the complete RAG request as the parent trace:

```text
rag.query
│
├── retrieval.embed_query
├── retrieval.vector_search
├── retrieval.bm25_search
├── retrieval.hybrid_fusion
├── retrieval.rerank
├── generation.build_prompt
├── generation.llm_call
├── validation.citation_check
└── response.finalize
```

This creates one end-to-end explanation for every answer.

### `config/prompts/`

Every trace should identify:

```text
prompt_name
prompt_version
prompt_git_commit
prompt_template_hash
context_tokens
system_prompt_tokens
user_query_tokens
```

Prompt changes must be treated like code changes because they can alter RAG behaviour just as significantly.

### `src/generation/llm_client.py`

Create:

```text
generation.llm_call
```

Capture:

```text
provider
model
deployment
model_version
temperature
max_tokens
prompt_tokens
completion_tokens
total_tokens
latency_ms
retry_count
finish_reason
estimated_cost_usd
```

Keep model pricing in configuration so that cost calculation can be updated without code changes.

### `src/generation/citation_validator.py`

Create:

```text
validation.citation_check
```

Capture:

```text
citation_count
valid_citation_count
invalid_citation_count
citation_coverage
grounded
abstained
abstention_reason
validator_version
```

Example refusal reasons:

```text
INSUFFICIENT_CONTEXT
NO_RELEVANT_CHUNK
INVALID_CITATION
LOW_GROUNDING_SCORE
```

A correct refusal should not be treated as the same thing as an application failure.

---

## Instrument the FastAPI Endpoints

The application description includes:

```text
POST /api/v1/query
POST /api/v1/ingest/file
POST /api/v1/ingest/text
POST /api/v1/evaluate
GET  /api/v1/health
GET  /api/v1/metrics
```

### `/api/v1/query`

Track:

```text
request_id
trace_id
total_latency
retrieval_latency
rerank_latency
llm_latency
validation_latency
token_usage
estimated_cost
citation_coverage
abstention_status
status_code
```

Do not use the raw user query as a Prometheus label because it creates high-cardinality metrics.

### `/api/v1/ingest/file`

Track:

```text
file_type
document_count
page_count
chunk_count
embedding_count
parse_duration
chunk_duration
embedding_duration
index_duration
failed_documents
```

### `/api/v1/ingest/text`

Track:

```text
payload_size
chunk_count
embedding_count
processing_duration
index_status
```

### `/api/v1/evaluate`

Track:

```text
evaluation_run_id
dataset_version
question_count
pass_count
fail_count
faithfulness
answer_relevance
context_precision
context_recall
citation_accuracy
evaluation_duration
```

Also correlate every evaluation run with:

```text
application_version
git_commit
prompt_version
retrieval_config_version
embedding_model
reranker_model
llm_model
```

### `/api/v1/health`

A useful production-style health model can report component health rather than only an overall status:

```json
{
  "status": "healthy",
  "components": {
    "vector_store": "healthy",
    "embedding_provider": "healthy",
    "llm_provider": "healthy",
    "reranker": "healthy",
    "telemetry": "healthy"
  }
}
```

A public health endpoint should never expose credentials, secrets or detailed internal exception data.

---

## Recommended Prometheus Metrics

### API and Request Metrics

```text
rag_requests_total
rag_request_failures_total
rag_request_duration_seconds
```

Use only low-cardinality labels such as:

```text
endpoint
method
status_class
environment
```

Do not use `query`, `user_id`, `trace_id`, `document_id` or `chunk_id` as Prometheus labels.

### Retrieval Metrics

```text
rag_vector_search_duration_seconds
rag_bm25_search_duration_seconds
rag_hybrid_fusion_duration_seconds
rag_rerank_duration_seconds
rag_retrieved_chunks_histogram
rag_rerank_candidates_histogram
```

### LLM Metrics

```text
rag_llm_requests_total
rag_llm_failures_total
rag_llm_duration_seconds
rag_prompt_tokens_total
rag_completion_tokens_total
rag_total_tokens_total
rag_estimated_cost_usd_total
```

### Quality Metrics

```text
rag_citation_coverage
rag_invalid_citations_total
rag_abstentions_total
rag_unsupported_answers_total
```

For batch evaluation:

```text
rag_eval_faithfulness
rag_eval_answer_relevance
rag_eval_context_precision
rag_eval_context_recall
rag_eval_citation_accuracy
```

---

## Dashboard Design

### Dashboard 1 — RAG Service Health

Show:

```text
Request Rate
Success Rate
Failure Rate
P50 Latency
P95 Latency
P99 Latency
Embedding Provider Errors
LLM Provider Errors
Vector Store Errors
CPU
Memory
Pod Restarts
```

### Dashboard 2 — Retrieval Quality

Show:

```text
Vector Search Latency
BM25 Search Latency
Hybrid Fusion Latency
Re-ranking Latency
Context Precision
Context Recall
Recall@K
MRR
Exact-ID Retrieval Success
```

A useful comparison is:

```text
Vector-only vs BM25-only vs Hybrid
```

### Dashboard 3 — Generation & Grounding

Show:

```text
Faithfulness
Answer Relevance
Citation Accuracy
Citation Coverage
Invalid Citation Rate
Unsupported Answer Rate
Abstention Rate
```

### Dashboard 4 — LLM Cost

Show:

```text
Requests by Model
Prompt Tokens
Completion Tokens
Tokens per Request
Average Cost per Request
P95 Cost per Request
Daily Cost
Cost by Prompt Version
Cost by Application Version
```

### Dashboard 5 — AI Regression History

Show:

```text
Evaluation Run
Git Commit
Prompt Version
Embedding Model
Reranker Model
Faithfulness
Answer Relevance
Context Precision
Context Recall
Citation Accuracy
Pass / Fail
```

Example:

```text
Commit     Faithfulness    Context Recall    Result
a18f210        0.93             0.89          PASS
92ac44f        0.92             0.88          PASS
cdf919e        0.81             0.73          FAIL
```

---

## Example SLOs and Quality Targets

These are project targets and should be tuned for the deployed model and environment.

```text
Successful Query Rate >= 99.5%
P50 Query Latency <= 2.5s
P95 Query Latency <= 5.0s
```

The repository already uses example evaluation expectations such as:

```text
Faithfulness >= 0.90
Context Precision >= 0.85
Answer Relevance >= 0.85
```

Additional project targets can include:

```text
Citation Coverage >= 0.90
Citation Accuracy >= 0.95
Context Recall >= 0.80
```

For cost, use an environment-specific configured budget rather than hard-coding a universal dollar value.

---

## Application-Specific Regression Scenarios

### Scenario A — Disable BM25

Change:

```text
Hybrid Retrieval → Vector Only
```

Run exact-ID cases such as:

```text
VX-VEH-0012398
```

Expected result:

```text
Exact-ID retrieval success decreases.
Context Recall may decrease.
The quality gate should fail.
```

### Scenario B — Change the Hybrid Weight

Before:

```text
alpha = 0.60
```

After:

```text
alpha = 0.90
```

Semantic queries may remain healthy while keyword-sensitive queries degrade. The hybrid-fusion trace should expose the cause.

### Scenario C — Reduce `top_k`

Before:

```text
top_k = 10
```

After:

```text
top_k = 2
```

Expected effects can include lower context recall, more refusals and weaker citation coverage.

### Scenario D — Remove the Re-ranker

Disable:

```text
cross-encoder/ms-marco-MiniLM-L-6-v2
```

Compare the final ranking and Golden Dataset metrics before and after the change.

### Scenario E — Change Chunking

Before:

```text
500–800 tokens
100-token overlap
```

After:

```text
250 tokens
20-token overlap
```

Potential result:

```text
Relevant concepts become fragmented.
Context Recall decreases.
Multi-sentence contract answers degrade.
```

### Scenario F — Prompt Regression

Weaken the evidence/citation requirements in the prompt.

Expected result:

```text
Citation Accuracy decreases.
Unsupported Answer Rate increases.
```

Because every trace records the prompt version, the problem can be tied directly to the change.

---

## CI/CD Integration for the Existing Solution

The repository describes both GitHub Actions and Azure DevOps pipelines.

Use the following delivery flow:

<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/26112d59-2f94-4c45-9058-7b1763525554" />


The application description already defines a strict quality gate command conceptually as:

```bash
python -m src.evaluation.quality_gate --strict
```

The pipeline should publish both Markdown and JSON reports.

---

## Example AI Quality Gate Report

```text
AI QUALITY GATE
================================================
Application Version:        1.4.2
Git Commit:                 cdf919e
Prompt Version:             1.7.0
Retrieval Config:           2.4.0
Golden Dataset:             v6

Metric                 Current    Threshold    Result
------------------------------------------------------
Faithfulness             0.92        0.90       PASS
Answer Relevance         0.88        0.85       PASS
Context Precision        0.87        0.85       PASS
Context Recall           0.79        0.80       FAIL
Citation Accuracy        0.97        0.95       PASS

Overall Result: FAIL

Likely Regression Area:
Retrieval / Context Selection

Changed Configuration:
top_k: 10 -> 2
```

This makes AI evaluation an engineering control rather than a manual notebook activity.

---

## Extended Local Deployment Topology

The described application already includes a RAG API, Prometheus and Jaeger. Project 3 can extend the local stack to:

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/54b58200-4cb8-4bf2-ba20-6873c9f2c139" />


Data flow:

```text
rag-api
   │
   ├── AI traces ─────────────► Langfuse
   ├── OTEL traces ───────────► OTEL Collector ─► Jaeger
   └── Prometheus metrics ────► Prometheus ─────► Grafana
```

This gives the project both application/SRE observability and AI-specific observability.

---

## Logging, Privacy and Cardinality Controls

Use structured logs with fields such as:

```text
request_id
trace_id
event
duration_ms
retrieved_chunk_count
citation_count
abstained
application_version
```

Do not log:

```text
API keys
access tokens
credentials
full sensitive documents
unredacted personal data
```

Queries and retrieved context may contain sensitive data, so support:

```text
redaction
sampling
hashing
retention controls
environment-specific verbosity
```

Keep high-cardinality details in traces and logs, not Prometheus label values.

---

## Correlation Strategy

Every request should carry:

```text
request_id
trace_id
```

Every evaluation should additionally carry:

```text
evaluation_run_id
golden_dataset_version
```

Every deployment should expose:

```text
application_version
git_commit
build_id
```

Every AI request should identify:

```text
prompt_version
retrieval_config_version
embedding_model_version
reranker_version
llm_model_version
```

Together, these identifiers make an incident reproducible.

---

## Root Cause Analysis Example for This RAG Architecture

### Incident

Queries containing vehicle identifiers stop returning the correct evidence.

Example:

```text
VX-VEH-0012398
```

### Dashboard Observation

```text
API Availability:            Normal
P95 Latency:                 Normal
LLM Error Rate:              Normal
Citation Validator:          Normal
Context Recall:              Degraded
Exact-ID Retrieval Success:  Degraded
```

The service is technically healthy, but RAG quality is not.

### Trace Inspection

```text
Vector Search:
Expected chunk ranked 9

BM25:
Disabled

Hybrid Fusion:
Not executed

Re-ranker:
Expected chunk never entered candidate set
```

### Configuration Diff

```text
retrieval.mode

Before: hybrid
After:  vector
```

### Root Cause

BM25 was unintentionally disabled. Natural-language semantic queries still worked, but exact identifiers could no longer be retrieved reliably.

### Corrective Action

```text
Restore hybrid retrieval.
Add exact-ID Golden Dataset tests.
Add a CI threshold for exact-ID retrieval.
Record retrieval mode in every trace.
```

### Prevention

A future change that disables BM25 should fail the AI quality gate before deployment.

This demonstrates the complete operational lifecycle:

```text
Detection → Diagnosis → Root Cause → Fix → Regression Prevention
```

---

## Golden Dataset Structure for This Application

Instead of one large undifferentiated file, group tests by capability:

```text
golden_dataset/
│
├── exact_ids.json
├── semantic_queries.json
├── contracts.json
├── policy_questions.json
├── multi_document.json
├── citation_tests.json
├── refusal_tests.json
└── regression_cases.json
```

### Exact-ID Tests

Validate BM25 and hybrid retrieval.

### Semantic Tests

Validate embeddings/vector retrieval.

### Contract and Policy Tests

Validate exact evidence, section/page metadata and citation behaviour.

### Multi-Document Tests

Validate evidence combination across more than one source.

### Refusal Tests

Ask questions that are intentionally unsupported by the corpus. The expected behaviour is abstention, not hallucination.

### Regression Tests

Every real defect should eventually become a Golden Dataset case, just as a traditional software defect should become a regression test.

---

## How the RAG system and these new changes for Observability and monitoring fit together ?

### Step 1 - Build the initial RAG System

Demonstrates:

```text
Document ingestion
Chunking
Embeddings
ChromaDB
BM25
Hybrid retrieval
Cross-encoder re-ranking
LLM generation
Citation enforcement
Golden evaluation dataset
CI/CD quality gate
```

### Step 2 — Implement Observability and monitoring on the RAG System

Demonstrates:

```text
End-to-end tracing
Retrieval diagnostics
Prompt observability
Token monitoring
Cost monitoring
Quality dashboards
SLOs and alerts
Incident analysis
Regression detection
Version correlation
Production debugging
```

Together they demonstrate the ability not only to build an AI system, but also to operate it reliably.

---

## Application-Specific Definition of Done

Project 3 is complete when an engineer can take any failed or low-quality request and answer all of these questions:

```text
1. Which application version handled it?
2. Which Git commit was deployed?
3. Which prompt version was active?
4. Which retrieval configuration was active?
5. Which embedding model was used?
6. What did vector search retrieve?
7. What did BM25 retrieve?
8. How were results fused?
9. How did the cross-encoder reorder them?
10. Which chunks reached the LLM prompt?
11. Which LLM/model deployment handled the request?
12. How many tokens were consumed?
13. What was the estimated cost?
14. Were the citations valid?
15. Did the system answer or correctly abstain?
16. What was total and component-level latency?
17. Did the same configuration pass the Golden Dataset?
18. Is there now a regression test preventing recurrence?
```

If those questions can be answered from traces, dashboards, evaluation reports and source-control history, the application has moved beyond a RAG demo toward a **production-operable AI system**.

---

# Phase 1 — Instrument the Entire RAG Pipeline

The first phase introduces distributed tracing across every important step of the RAG request lifecycle.

Each request should generate a complete trace.

## Trace the following stages

### 1. Incoming Request

Capture:

- Request ID
- User query
- Timestamp
- Session ID
- Application version
- Environment
- Model version
- Prompt version
- Configuration version

---

### 2. Query Processing

Capture:

- Original query
- Normalized query
- Query rewriting output
- Intent classification
- Filters applied
- Metadata constraints

---

### 3. Retrieval

Capture:

- Retrieved document IDs
- Retrieved chunk IDs
- Retrieval scores
- BM25 ranking
- Vector similarity score
- Hybrid score
- Number of chunks retrieved
- Retrieval latency

Example:

```json
{
  "query": "How does the refund policy work?",
  "retrieved_chunks": [
    {
      "document": "policy.pdf",
      "chunk_id": "chunk_42",
      "vector_score": 0.91,
      "bm25_score": 8.3
    }
  ]
}
```

---

### 4. Re-ranking

Track how the re-ranker changes retrieval results.

Capture:

- Original ranking
- Re-ranked order
- Re-ranking score
- Re-ranking model
- Re-ranking latency

Example:

```text
Initial Retrieval

1. Chunk A
2. Chunk B
3. Chunk C

After Cross-Encoder Re-ranking

1. Chunk C
2. Chunk A
3. Chunk B
```

This information is extremely useful when investigating retrieval-quality problems.

---

### 5. Prompt Construction

Capture the exact prompt sent to the LLM.

Track:

- System prompt
- User prompt
- Retrieved context
- Prompt template version
- Prompt variables
- Total prompt tokens

Prompt versioning is important because changing a prompt can affect system behaviour as dramatically as changing application code.

---

### 6. LLM Execution

Capture:

- Model name
- Model version
- Temperature
- Max tokens
- Prompt tokens
- Completion tokens
- Total tokens
- LLM latency
- Retry count
- Error information

---

### 7. Final Response

Capture:

- Final answer
- Citations
- Source documents
- Validation results
- Faithfulness score
- Citation coverage
- Total request latency
- Estimated request cost

---

# Observability Tools

Suitable tools include:

- **Langfuse**
- **LangSmith**
- **Braintrust**

For this project, **Langfuse** is a strong option because it is open source and can be self-hosted.

Example deployment options:

<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/f870a136-2875-415d-a310-ad4181c8300f" />


It can also be deployed into Kubernetes.

Example:

<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/51e58a9c-7fc4-4509-b9c6-769bffc2657f" />


---

# Phase 2 — Production Metrics & Quality Monitoring

The second phase introduces operational metrics.

The objective is to measure the system continuously rather than relying on subjective testing.

---

# 1. Latency Monitoring

Do not measure only average response time.

Track latency percentiles.

Important metrics:

```text
P50 Latency
P90 Latency
P95 Latency
P99 Latency
```

Example:

```text
P50 = 1.8 seconds
P95 = 4.6 seconds
P99 = 7.2 seconds
```

P95 and P99 values are important because averages can hide poor worst-case performance.

---

# 2. Cost per Request

Calculate the approximate cost of each RAG query.

Example formula:

```text
Request Cost =
Prompt Token Cost
+
Completion Token Cost
+
Embedding Cost
+
Re-ranking Cost
```

Example:

```text
Average Cost per Request = $0.008

P95 Cost per Request = $0.015
```

Track cost by:

- Model
- Environment
- Feature
- User group
- Prompt version
- Application version

---

# 3. Token Usage

Track:

```text
Prompt Tokens
Completion Tokens
Total Tokens
Tokens per Request
```

A sudden increase in token usage can indicate:

- Excessive retrieval context
- Prompt expansion
- Duplicate chunks
- Poor chunk filtering
- Prompt regression

---

# 4. Citation Coverage

Citation coverage measures how much of the generated response is grounded in retrieved evidence.

Example:

```text
Citation Coverage =
Supported Claims
-----------------
Total Claims
```

Example result:

```text
Citation Coverage = 92%
```

This helps identify unsupported LLM responses.

---

# 5. Faithfulness

Faithfulness measures whether the answer is supported by the retrieved context.

Possible evaluation tools:

- RAGAS
- LLM-as-a-Judge
- Custom validation rules
- Citation validation

Example target:

```text
Faithfulness >= 0.85
```

---

# 6. Failure Rate

Monitor application and AI failures.

Examples:

```text
Retrieval Failure
Embedding Failure
LLM Timeout
Re-ranking Failure
Citation Validation Failure
Unsupported Answer
Parsing Error
Rate Limit Error
```

Example metric:

```text
Failure Rate =
Failed Requests
---------------
Total Requests
```

---

# 7. Retrieval Quality

Track metrics such as:

```text
Context Precision
Context Recall
MRR
Hit Rate
Recall@K
Precision@K
```

These metrics help determine whether the correct documents are being retrieved.

---

# Monitoring Dashboard

The application should provide a dashboard capable of showing:

## System Metrics

```text
Requests per Minute
Error Rate
P50 Latency
P95 Latency
P99 Latency
```

## LLM Metrics

```text
Prompt Tokens
Completion Tokens
Total Tokens
Cost per Request
Model Usage
```

## RAG Quality Metrics

```text
Faithfulness
Answer Relevance
Context Precision
Context Recall
Citation Coverage
Unsupported Answer Rate
```

## Retrieval Metrics

```text
Vector Search Latency
BM25 Search Latency
Hybrid Retrieval Latency
Re-ranking Latency
Top-K Retrieval Accuracy
```

---

# Example Dashboard

```text
------------------------------------------------------
RAG Production Dashboard
------------------------------------------------------

Requests Today                 12,450

Average Latency                 2.1 sec
P50 Latency                     1.6 sec
P95 Latency                     4.7 sec
P99 Latency                     7.9 sec

Failure Rate                    0.8 %

Average Cost / Request          $0.008

Faithfulness                    0.91
Answer Relevance                0.89
Context Precision               0.87
Context Recall                  0.84

Citation Coverage               93 %

Unsupported Answer Rate         2.1 %
------------------------------------------------------
```

---

# Root Cause Analysis Scenario

One of the most important outcomes of this project is the ability to diagnose historical incidents.

Example scenario:

> RAG answer quality degraded last Tuesday.

The engineering team should be able to inspect the monitoring dashboard and discover:

```text
Tuesday 10:00 AM

Faithfulness
0.91 → 0.72

Citation Coverage
94% → 71%

P95 Latency
4.2s → 6.8s
```

Tracing may reveal:

```text
Prompt Version
v12 → v13

Retrieval Top-K
10 → 5

Re-ranking Model
Changed
```

Root cause:

```text
Reduced retrieval context caused relevant evidence
to be removed before the LLM generated its answer.
```

This demonstrates production debugging capability rather than simply application development.

---

# Phase 3 — Automated Regression Gating

The third phase connects evaluation to the software delivery pipeline.

The Golden Evaluation Dataset created in Project 1 is executed automatically during Continuous Integration.

---

# Golden Evaluation Dataset

Example:

```json
[
  {
    "question": "What is the refund period?",
    "expected_answer": "30 days",
    "expected_sources": [
      "refund-policy.pdf"
    ]
  },
  {
    "question": "Can international customers request refunds?",
    "expected_answer": "Yes",
    "expected_sources": [
      "refund-policy.pdf"
    ]
  }
]
```

The dataset should represent:

- Common questions
- Difficult questions
- Edge cases
- Multi-document questions
- Ambiguous questions
- Questions requiring citations
- Known previous failures

---

# Automated Evaluation

The CI pipeline executes the evaluation suite.

Possible metrics:

```text
Faithfulness
Answer Relevance
Context Precision
Context Recall
Citation Accuracy
Latency
Cost
```

Possible tools:

- RAGAS
- DeepEval
- Langfuse Evaluations
- Braintrust
- Custom Python evaluation scripts

---

# Quality Gate

Example quality thresholds:

```yaml
quality_gate:

  faithfulness: 0.85

  answer_relevance: 0.80

  context_precision: 0.80

  context_recall: 0.75

  citation_coverage: 0.90

  p95_latency_seconds: 5

  max_cost_per_request: 0.02
```

If any critical metric falls below the required threshold:

```text
BUILD FAILED
```

The pull request should not be merged until the regression is fixed or intentionally reviewed.

---

# CI/CD Pipeline

Example pipeline:

<img width="1122" height="1402" alt="image" src="https://github.com/user-attachments/assets/acc31014-60d3-4ba7-8647-1d07c670a8ee" />


Possible platforms:

- Azure DevOps
- GitHub Actions
- GitLab CI
- Jenkins

---

# Prompt Versioning

Prompts should be stored and versioned together with application code.

Example structure:

```text
rag-platform/
│
├── src/
│
├── prompts/
│   ├── system_prompt_v1.txt
│   ├── system_prompt_v2.txt
│   └── query_rewrite_prompt.txt
│
├── config/
│   ├── retrieval.yaml
│   ├── models.yaml
│   └── evaluation.yaml
│
├── evaluation/
│   ├── golden_dataset.json
│   └── evaluate.py
│
└── pipelines/
    └── azure-pipelines.yml
```

Every trace should record:

```text
Prompt Version
Model Version
Application Version
Configuration Version
```

This makes regressions easier to diagnose.

---

# Configuration Versioning

Important RAG configuration should also be stored in source control.

Example:

```yaml
retrieval:

  top_k: 10

  hybrid_search: true

  bm25_weight: 0.4

  vector_weight: 0.6

reranker:

  enabled: true

  model: cross-encoder/ms-marco-MiniLM-L-6-v2

llm:

  model: gpt-5

  temperature: 0

  max_tokens: 1500
```

A change to any of these values can significantly affect system behaviour.

---

# OpenTelemetry Integration

For enterprise-level observability, OpenTelemetry can be added to the platform.

Architecture:

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/2ba2e24e-fb0d-4a2f-bfe0-d4e909605a77" />


This allows both traditional application observability and AI-specific tracing.

---

# Recommended Technology Stack

## Application

- Python
- FastAPI
- ASP.NET Core
- LangGraph

## RAG

- Azure AI Search
- Elasticsearch
- ChromaDB
- Weaviate
- PostgreSQL + pgvector

## Embeddings

- Azure OpenAI Embeddings
- OpenAI Embeddings
- Sentence Transformers

## Re-ranking

- Cohere Re-rank
- SentenceTransformers CrossEncoder

## LLM

- Azure OpenAI
- OpenAI
- Ollama
- Local LLM

## AI Observability

- Langfuse
- LangSmith
- Braintrust

## Infrastructure Monitoring

- OpenTelemetry
- Datadog
- Azure Monitor
- Grafana
- Prometheus
- Jaeger

## Evaluation

- RAGAS
- DeepEval
- Custom Evaluation Framework

## CI/CD

- Azure DevOps
- GitHub Actions

## Deployment

- Docker
- Kubernetes
- Azure Kubernetes Service

---

# Suggested Repository Structure

```text
production-rag-observability/
│
├── src/
│   ├── api/
│   ├── retrieval/
│   ├── reranking/
│   ├── generation/
│   └── validation/
│
├── observability/
│   ├── tracing.py
│   ├── metrics.py
│   └── cost_tracker.py
│
├── prompts/
│   ├── system_prompt.txt
│   └── query_rewrite_prompt.txt
│
├── config/
│   ├── retrieval.yaml
│   ├── models.yaml
│   └── monitoring.yaml
│
├── evaluation/
│   ├── golden_dataset.json
│   ├── ragas_evaluation.py
│   └── thresholds.yaml
│
├── dashboards/
│   ├── grafana/
│   └── langfuse/
│
├── tests/
│
├── docker/
│
├── kubernetes/
│
├── pipelines/
│   └── azure-pipelines.yml
│
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

# Key Deliverables

The finished project should contain the following deliverables.

## Deliverable 1 — Production RAG Application

A working RAG application supporting:

- Hybrid retrieval
- Vector search
- BM25
- Re-ranking
- LLM generation
- Citations

---

## Deliverable 2 — End-to-End Traceability

Every request should be traceable from:

```text
User Query
→ Retrieval
→ Re-ranking
→ Prompt
→ LLM
→ Validation
→ Final Response
```

---

## Deliverable 3 — Monitoring Dashboard

Dashboard showing:

- Request volume
- Latency
- P50 / P95 / P99
- Cost
- Token usage
- Failure rate
- Citation coverage
- Faithfulness
- Answer relevance
- Context precision
- Context recall

---

## Deliverable 4 — Root Cause Analysis

Document at least one simulated or real incident.

Example:

```text
Incident:
Faithfulness dropped from 91% to 72%.

Cause:
Retrieval Top-K changed from 10 to 5.

Detection:
Monitoring dashboard.

Diagnosis:
Langfuse trace analysis.

Resolution:
Restore Top-K and add regression test.
```

---

## Deliverable 5 — Golden Evaluation Dataset

Include approximately:

```text
50–200 evaluation questions
```

Cover:

- Normal queries
- Edge cases
- Multi-document questions
- Ambiguous questions
- Citation-sensitive questions

---

## Deliverable 6 — Automated Quality Gate

The CI/CD pipeline should automatically run the evaluation suite.

Example:

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/aeca744b-0e03-4c2e-b2ed-576505868712" />

---

## Deliverable 7 — Prompt & Configuration Versioning

Store prompts and RAG configuration in Git.

Every production trace should identify the active version.

---

# Portfolio Demonstration

The portfolio demonstration should show:

### Demo 1

Send a normal user request and open its trace.

Show:

```text
Query
Retrieved Chunks
Re-ranking
Prompt
LLM Response
Token Usage
Latency
Cost
Citations
```

### Demo 2

Show the monitoring dashboard.

Explain:

```text
P50 Latency
P95 Latency
Failure Rate
Cost
Faithfulness
Citation Coverage
```

### Demo 3

Introduce a deliberate regression.

Example:

```text
top_k = 10
```

change to:

```text
top_k = 2
```

Run the CI pipeline.

Expected result:

```text
Context Recall decreases
Faithfulness decreases

QUALITY GATE FAILED
```

The build should be blocked.

### Demo 4

Restore the correct configuration.

Run the pipeline again.

Expected result:

```text
QUALITY GATE PASSED
```

---

# Engineering Skills Demonstrated

This project demonstrates experience with:

- Production RAG architecture
- LLM observability
- Distributed tracing
- AI monitoring
- Prompt engineering
- Prompt versioning
- Retrieval diagnostics
- Re-ranking analysis
- Cost monitoring
- Token monitoring
- RAG evaluation
- RAGAS
- Golden datasets
- Regression testing
- CI/CD quality gates
- OpenTelemetry
- Kubernetes
- Production incident analysis
- Site Reliability Engineering concepts for AI systems

---


## Key Points on Production RAG Observability & Evaluation Platform

Designed and implemented an observability and regression-testing platform for a production-grade Retrieval-Augmented Generation system.

Instrumented the complete RAG pipeline including query processing, hybrid retrieval, re-ranking, prompt generation, LLM execution, citation validation, token consumption, latency, and request cost.

Implemented AI quality monitoring using metrics including faithfulness, answer relevance, context precision, context recall, citation coverage, P50/P95 latency, cost per request, and failure rate.

Integrated a Golden Evaluation Dataset into the CI/CD pipeline so that RAG quality regressions automatically block deployments when defined quality thresholds are not met.

Technologies:

```text
Python
FastAPI
LangGraph
Azure OpenAI
Azure AI Search
Langfuse
RAGAS
OpenTelemetry
Grafana
Datadog
Docker
Kubernetes
Azure DevOps
GitHub Actions
```

---

# GitHub README Summary

> A production-grade RAG observability platform that traces every stage of the AI pipeline, measures retrieval and answer quality, monitors latency and cost, validates citations, and automatically blocks deployments when evaluation metrics regress.

---

# Final Project Outcome

The final system should allow an engineer to answer:

> Why did RAG quality degrade?

within minutes instead of hours.

The project demonstrates a shift from simply **building AI applications** to **operating reliable AI systems in production**.

That distinction is highly valuable because production AI engineering requires not only model integration, but also observability, evaluation, reliability, cost control, regression prevention, and disciplined software delivery.
