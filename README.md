# GraphRAG Evaluation JAR

A self-contained fat JAR (`graphrag-evaluation-tool-1.0.0.jar`) that evaluates GraphRAG
answer quality using LLM-as-judge (RAGAS metrics).

**Two operating modes:**

| Mode                | How to activate | What it does                                                                                        |
|---------------------|----------------|-----------------------------------------------------------------------------------------------------|
| **Evaluation mode** | Pass `--questions <csv>` (default) | Loads CSV with question and answers pairs and runs GraphRAG for each of them                        |
| **Report mode**     | Pass `--report` | Generates HTML comparison reports from existing JSON test logs (generated from the evaluation mode) |

---

## Prerequisites

Java 21 should be installed.


## LLM Provider Configuration

Supported providers are OpenAI, AWS Bedrock, Azure OpenAI and Gemini

All LLM provider settings are environment variables set **before** the `java` command that runs the jar.
Activate a provider profile with `SPRING_PROFILES_ACTIVE`.

### Base variables — required for all providers

| Variable        | Description                                                                                                                   |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------|
| `N8N_API_KEY`   | N8N API key, needed for extracting n8n execution data. It can be generated in graphrag-workflows UI under Settings -> n8n API |
| `AI_BASE_URL`   | Base URL of the LLM API endpoint                                                                                              |
| `AI_API_KEY`    | LLM provider API key for authentication                                                                                       |
| `AI_CHAT_MODEL` | Chat model identifier, e.g. `gpt-4o-mini`                                                                                     |
| `AI_EMBED_MODEL` | Embedding model identifier, e.g. `text-embedding-3-small`                                                                     |

### Provider-specific variables

**OpenAI** - activate with `SPRING_PROFILES_ACTIVE=openai` or by omitting to set `SPRING_PROFILES_ACTIVE`

No additional variables are needed.

**Azure OpenAI / Foundry** — activate with `SPRING_PROFILES_ACTIVE=azure`

| Variable | Description |
|----------|-------------|
| `AZURE_OPENAI_DEPLOYMENT_NAME` | Azure deployment name for chat |
| `AZURE_EMBED_DEPLOYMENT_NAME` | Azure deployment name for embeddings |

**AWS Bedrock** — activate with `SPRING_PROFILES_ACTIVE=bedrock`

| Variable | Description |
|----------|-------------|
| `BEDROCK_REGION` | AWS region, e.g. `us-east-1` |
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |

**Google Gemini** — activate with `SPRING_PROFILES_ACTIVE=gemini`

| Variable | Description |
|----------|-------------|
| `GCP_PROJECT_ID` | GCP project ID |
| `GCP_LOCATION` | GCP location, e.g. `us-central1` |

---

## Evaluation Mode

Loads questions from a CSV, sends each to GraphRAG, scores the response via the LLM Judge
(RAGAS metrics), and writes per-scenario JSON test logs.

### Quick Start

```bash
# Set LLM provider env vars
export AI_BASE_URL=https://your-azure-endpoint.azure.com
export AI_API_KEY=your-api-key
export AI_CHAT_MODEL=gpt-4o-mini
export AI_EMBED_MODEL=text-embedding-3-small
export AZURE_OPENAI_DEPLOYMENT_NAME=your-deployment
export AZURE_EMBED_DEPLOYMENT_NAME=your-embedding-deployment
export SPRING_PROFILES_ACTIVE=azure

# Run
java -jar graphrag-evaluation-tool-1.0.0.jar \
  --questions ./my-questions.csv \
  --graphrag-url http://localhost:80 \
  --auth-user bob --auth-password bob123
```

> All LLM provider settings are environment variables — not CLI arguments.

### CLI arguments

| Argument | Default                                                                                      | Description                                                                |
|----------|----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| `--questions <path>` | *(required)*                                                                                 | CSV file with questions and expected answers                               |
| `--graphrag-url <url>` | `http://localhost:80`                                                                        | GraphRAG chat API base URL                                                 |
| `--auth-user <user>` | `bob`                                                                                        | Keycloak username                                                          |
| `--auth-password <pass>` | `bob123`                                                                                     | Keycloak password                                                          |
| `--output-dir <path>` | `./graphrag-eval-output-{datetime}` for evaluation mode, and `report` folder for report mode | Directory for all output |
| `--version <string>` | `1.0.0`                                                                                      | Version label in output filenames                                          |
| `--help` | —                                                                                            | Show full help                                                             |

### CSV format

```csv
question,expected_answer,category
```

| Column | Required | Default | Notes |
|--------|----------|---------|-------|
| `question` | **Yes** | — | The question to send to GraphRAG |
| `expected_answer` | **Yes** | — | Golden answer for LLM Judge comparison |
| `category` | No | `general` | Pipeline: `general`, `vector`, `graphdb`, `graphdb+vector`, `multihop`, `comparison`, `unknown` |

Fields with commas or newlines must be RFC 4180-quoted:

```csv
question,expected_answer
"Who ruled Valenmoor?","Queen Seraphina ruled Valenmoor.
She reigned for 47 years."
```

### Evaluation output

```
output-dir/                 
  └── test-record-<scenario>-q<number>.json ← JSON logs

```

Each JSON log record contains the question, GraphRAG's answer, all 6 RAGAS metric scores,
token usage breakdown, and execution time.

---

## Baseline Comparison

Save a baseline run, then compare future runs against it.

```bash
# Save baseline
export AI_BASE_URL=...
export AI_API_KEY=...
export SPRING_PROFILES_ACTIVE=azure

java -jar graphrag-evaluation-tool-1.0.0.jar \
  --questions ./my-questions.csv \
  --graphrag-url http://localhost:80 \
  --auth-user bob --auth-password bob123 \
  --output-dir ./baseline-run

# Compare a new run against the baseline
java -jar graphrag-evaluation-tool-1.0.0.jar \
  --questions ./my-questions.csv \
  --graphrag-url http://localhost:80 \
  --auth-user bob --auth-password bob123 \
  --output-dir ./current-run 
```

---

## Report Mode

Generates HTML comparison reports from existing JSON test logs.
No GraphRAG or LLM services needed — only the JSON log files.

```bash
# Compare two folders
java -jar graphrag-evaluation-tool-1.0.0.jar \
  --report ./baseline-logs ./current-logs --output-dir ./my-report-comparison

# Single folder (no comparison)
java -jar graphrag-evaluation-tool-1.0.0.jar \
  --report ./test-logs --output-dir ./single-report

# Dry-run (verify paths only)
java -jar graphrag-evaluation-tool-1.0.0.jar \
  --report --verify ./baseline-logs ./current-logs

# Explicitly no baseline
java -jar graphrag-evaluation-tool-1.0.0.jar \
  --report ./test-logs
```

The output directory defaults to `./report/` when not specified.

### Report pages

| File | Contents |
|------|----------|
| `index.html` | Summary stats + navigation |
| `comparison-report.html` | Per-query score/precision/recall/F1 deltas. Expand rows to see expected answer, system answer, fact counts, and full claim lists |
| `claim-analysis.html` | Per-query R/G fact counters, question-relevant vs extra facts |
| `baseline-averages.html` | Per-query mean scores — baseline runs only |
| `current-averages.html` | Per-query mean scores — current runs only |
| `baseline-anomalies.html` | Score/precision/recall/F1 variance anomalies |
| `current-anomalies.html` | Score variance across repeated current runs |
| `inter-run-variance.html` | Variance across run folders (multi-run detection) |
| `baseline-tokens-anomalies.html` | Token/exec-time outliers — baseline runs |
| `current-tokens-anomalies.html` | Token/exec-time outliers — current runs |
| `token-comparison.html` | Baseline-vs-current token usage comparison |

### Multi-run detection

When a folder contains multiple run subdirectories, the reporter aggregates per query
(mean ± σ). Expanded detail rows show **all** per-run records side-by-side,
each attributed to its source folder.


---

## Evaluation Metrics

Six RAGAS metrics are computed in parallel for each evaluated scenario.
The primary gate is **Factual Correctness (F1)**.

| Metric | Description                                                                                                                                                          |
|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Factual Correctness (F1)** | Claim-level precision/recall and f1. Decomposes both answers into atomic claims, classifies each as entailed / neutral / contradicted. Default threshold: f1 >= 0.65 |
| **Answer Correctness** | Weighted average of  Factual correctness (75%) + Semantic similarity (25%)                                                                                           |
| **Context Precision** | Retrieved chunks are relevant and in the right order                                                                                                                 |
| **Context Recall** | All facts needed to answer were retrieved                                                                                                                            |
| **Faithfulness** | Answer stays within retrieved context (no hallucination)                                                                                                             |
| **Response Relevancy** | Answer is about what was asked                                                                                                                                       |


---

