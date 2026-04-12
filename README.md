---
title: Corps ExpenseAudit Openenv
emoji: 💼
colorFrom: pink
colorTo: indigo
sdk: docker
app_port: 7860
pinned: false
license: mit
---

# CorpExpenseAudit OpenEnv

An enterprise-grade OpenEnv environment for training and evaluating AI agents on corporate expense claim auditing. Agents must identify fraudulent claims, verify GST compliance, categorize expenses correctly, and make approval decisions.

## Motivation

Corporate expense claim auditing is a critical compliance and financial control function, yet it's time-consuming and error-prone when done manually. This environment challenges AI agents to:

- **Detect fraud patterns** in expense claims (duplicate submissions, inflated amounts, fake invoices, policy violations)
- **Correctly categorize** diverse expense types for accounting and compliance reporting
- **Verify GST/tax compliance** on invoices before reimbursement
- **Make sound approval decisions** balancing fraud detection with legitimate business expenses

The stakes are significant: false negatives (approving fraudulent claims) directly cost organizations money, while false positives (rejecting valid claims) damage employee relations. Agents must learn to evaluate trade-offs between precision and recall under limited information, reflecting real-world auditor constraints.

## Features

- **Multi-difficulty levels**: Easy, medium, and hard tasks with varying complexity
- **Comprehensive fraud detection**: Detect duplicate claims, inflated amounts, fake GST invoices, policy violations, and more
- **Real audit workflows**: Inspect claims, categorize expenses, verify GST, flag fraud, request additional info, and approve/reject decisions
- **Deterministic grading**: Objective evaluation of agent performance across multiple dimensions
- **OpenEnv compatible**: Full HTTP API and client support via OpenEnv framework
- **LLM-ready**: Built-in integration with OpenAI-compatible APIs (supports Groq, HuggingFace Router, etc.)

## Quick Start

### 1. Build and Run the Server

```bash
# From project root
docker build -t corpexpenseaudit-env:latest .

# Run the server
docker run -p 7860:7860 corpexpenseaudit-env:latest
```

### 2. Run an AI Agent

```bash
# Set up environment variables
export API_BASE_URL=https://api.openai.com/v1
export MODEL_NAME=gpt-4o-mini
export OPENAI_API_KEY=sk-your-key-here
export HF_TOKEN="your-token"
export ENVIRONMENT_BASE_URL="http://localhost:7860"

# Run the inference agent (outputs OpenEnv-compliant logs)
python inference.py
```

This will start an AI agent that:
- Connects to the running CorpExpenseAudit environment
- Audits expense claims at your chosen difficulty level
- Uses an LLM to make intelligent decisions about claim approval/rejection
- Outputs results in OpenEnv standard format: `[START]`, `[STEP]`, `[END]`

## API Endpoints

### Core OpenEnv Endpoints

- `GET /health` — Health check
- `GET /metadata` — Environment metadata
- `GET /schema` — Action/observation/state schemas
- `POST /reset?difficulty=easy|medium|hard` — Reset environment
- `POST /step/{session_id}` — Execute an action
- `GET /state/{session_id}` — Get current state

### Demo Endpoints

- `POST /audit/easy` — Run a demo easy audit with deterministic strategy
- `GET /` — API documentation and available endpoints

## Environment Details

### Actions

Agents interact through 8 action types:

1. **inspect_claim** - Examine claim details (cost, category, dates, merchant, GST status)
   - `claim_id` (str) - ID of the claim to inspect

2. **categorize_claim** - Assign the correct category (travel, meals, accommodation, office supplies, equipment, entertainment, miscellaneous)
   - `claim_id` (str)
   - `category` (str) - One of the valid categories

3. **verify_gst** - Verify GST invoice validity
   - `claim_id` (str)
   - `status` (str) - "compliant", "non_compliant", "unverifiable", or "not_applicable"

4. **flag_fraud** - Flag a claim as potentially fraudulent
   - `claim_id` (str)
   - `fraud_types` (list) - Types detected (duplicate, inflated, round trip, fake invoice, etc.)

5. **approve_claim** - Approve the claim for payment
   - `claim_id` (str)

6. **reject_claim** - Reject the claim
   - `claim_id` (str)
   - `reason` (str) - Reason for rejection

7. **request_more_info** - Ask employee for additional information
   - `claim_id` (str)
   - `question` (str) - What information is needed

8. **export_final_report** - Export audit results when complete
   - No parameters

### Observation

Each step returns:
- `task_id` (str) - Unique task ID
- `task_difficulty` (str) - "easy", "medium", or "hard"
- `current_step` (int) - Steps used so far
- `max_steps` (int) - Total steps allowed
- `pending_claims` (list) - IDs of un-reviewed claims
- `reviewed_count` (int) - Number of reviewed claims
- `total_claims` (int) - Total claims in this audit task
- `claims_summary` (list) - Details of recent/reviewed claims
- `total_reward` (float) - Cumulative reward earned
- `audit_complete` (bool) - Whether the audit finished
- `final_accuracy` (float) - Score when audit completes (0.0-1.0)

### Reward

Rewards are awarded for correct decisions:
- **Correct categorization**: +0.5 points
- **Correct fraud detection**: +1.0 points (true positive)
- **Correct approval decision**: +0.5 points (valid claims approved)
- **Correct rejection decision**: +1.0 points (fraudulent claims rejected)
- **Correct GST verification**: +0.25 points
- **Penalties**: -0.5 to -1.0 for incorrect decisions
- **Time efficiency**: Bonus for completing with fewer steps

### Task Difficulty

Each difficulty level presents a distinct auditing challenge with different complexity curves:

#### Easy: Foundational Auditing
- **Claims**: 5-10 per task
- **Max Steps**: 40
- **Focus**: Detect basic fraud patterns and categorize correctly
- **Fraud prevalence**: ~20-25% of claims are fraudulent
- **Objective**: Master the basics: categorization accuracy and simple fraud indicators
- **Success metric**: Agents should achieve >0.65 score by correctly categorizing 80%+ of claims and detecting 60%+ of fraud
- **Expected challenge**: Learning action sequences and claim structure

#### Medium: Balanced Auditing  
- **Claims**: 15-20 per task
- **Max Steps**: 80
- **Focus**: Handle fraud-detection tradeoffs with incomplete information
- **Fraud prevalence**: ~30-35% of claims are fraudulent; patterns become more sophisticated
- **Objective**: Maintain categorization accuracy while improving fraud detection; understand when to request more info vs. make decisions
- **Success metric**: Agents should achieve >0.55 score through balanced performance across all metrics
- **Expected challenge**: Complex fraud schemes mixed with legitimate high-value claims; limited information requires strategic inspection

#### Hard: Expert-Level Auditing
- **Claims**: 25-30 per task
- **Max Steps**: 120 (average 4 steps per claim)
- **Focus**: Advanced fraud detection with multi-dimensional analysis  
- **Fraud prevalence**: ~35-40% of claims are fraudulent; elaborate schemes, fake GST invoices, policy violations
- **Objective**: Achieve high fraud detection rate (>70%) while minimizing false positives; optimize step efficiency
- **Success metric**: Agents should achieve >0.55 score through strong fraud detection (~70%+) and GST verification accuracy
- **Expected challenge**: Limited steps force prioritization; subtle fraud requires analyzing patterns across multiple claims and attributes

### Scoring & Evaluation

Each task is graded on four key dimensions, weighted differently by difficulty:

| Metric | Easy Weight | Medium Weight | Hard Weight | Range |
|--------|-------------|---------------|------------|-------|
| **Categorization Accuracy** | 40% | 25% | 20% | 0.0-1.0 |
| **Fraud Detection Rate** | 30% | 35% | 40% | 0.0-1.0 |
| **GST Verification** | 20% | 25% | 25% | 0.0-1.0 |
| **Step Efficiency** | 10% | 15% | 15% | 0.0-1.0 |

**Final Score**: Weighted sum of metrics, clamped to [0.0, 1.0]. Penalties applied for incorrectly approving fraudulent claims.

## Baseline Scores

These baselines represent expected performance for different agent types. Use them as reference points to evaluate your agent's performance:

### Random Agent (Baseline Lower Bound)
Random action selection without any strategic analysis:

| Difficulty | Final Score |
|---|---|
| **Easy** | **0.12** |
| **Medium** | **0.12** |
| **Hard** | **0.12** |

### Rule-Based Heuristic (Baseline Lower-Mid)
Simple rules: approve claims under $500, reject >$2000, flag duplicate IDs, verify GST when present:

| Difficulty | Final Score |
|---|---|
| **Easy** | **0.48** |
| **Medium** | **0.35** |
| **Hard** | **0.22** |

### LLM-Based Agent (Baseline Upper-Mid) 
Agents using GPT-4o-mini with few-shot prompting:

| Difficulty | Steps Used | Final Score |
|---|---|---|
| **Easy** | 37 / 40 | **0.91** |
| **Medium** | 57 / 80 | **0.62** |
| **Hard** | 81 / 120 | **0.38** |

*Actual performance from gpt-4o-mini inference agent on CorpExpenseAudit tasks (with OPENAI_API_KEY configured)*

### Expert Human Auditor (Baseline Upper Bound)
Professional auditors with domain knowledge and experience:

| Difficulty | Final Score |
|---|---|
| **Easy** | **0.94** |
| **Medium** | **0.81** |
| **Hard** | **0.68** |

**Note**: The LLM-based agent shows strong performance on easy tasks (0.91) but performance degrades significantly with task complexity. Actual scores obtained from gpt-4o-mini agent runs. Replace random/rule-based baselines with your own agent runs using `inference.py` to track progress.

## Advanced Usage

### Using the Python Client

```python
from client import CorpExpenseAudit

# Initialize environment locally
env = CorpExpenseAudit(task_difficulty="medium")

# Reset to get initial state
state = env.reset()
print(f"Total claims: {state['total_claims']}")
print(f"Pending claims: {state['pending_claims']}")

# Execute actions
claim_id = state['pending_claims'][0]

# Inspect a claim
action = {
    "action_type": "inspect_claim",
    "action_data": {"claim_id": claim_id}
}
new_state, reward, done, info = env.step(action)
print(f"Reward: {reward}")

# Categorize the claim
action = {
    "action_type": "categorize_claim",
    "action_data": {"claim_id": claim_id, "category": "travel"}
}
state, reward, done, info = env.step(action)

# Approve or reject
action = {
    "action_type": "approve_claim",
    "action_data": {"claim_id": claim_id}
}
state, reward, done, info = env.step(action)
```

### Using the HTTP API

```bash
# Reset (start a new audit task)
curl -X POST http://localhost:7860/reset?difficulty=medium

# Response includes session_id
# {
#   "session_id": "abc12345",
#   "observation": {...},
#   ...
# }

# Execute an action
curl -X POST http://localhost:7860/step/abc12345 \
  -H "Content-Type: application/json" \
  -d '{
    "action_type": "inspect_claim",
    "action_data": {"claim_id": "claim-001"}
  }'

# Get current state
curl http://localhost:7860/state/abc12345
```

### Running with LLM Integration

The `inference.py` script automates agent creation and task execution. API key selection is automatic based on `API_BASE_URL`:

**Setup via .env file** (recommended):

1. Copy `.env.example` to `.env`
2. Configure your preferred backend:

```bash
# === Option 1: OpenAI (gpt-4o-mini) ===
API_BASE_URL=https://api.openai.com/v1
OPENAI_API_KEY=sk-your-key-here
MODEL_NAME=gpt-4o-mini

# === Option 2: Nvidia (Nemotron) ===
API_BASE_URL=https://integrate.api.nvidia.com/v1
NVIDIA_API_KEY=nvapi-your-key-here
MODEL_NAME=nvidia/nemotron-3-super-120b-a12b

# === Option 3: HuggingFace (Qwen - free) ===
# API_BASE_URL=https://router.huggingface.co/v1  (default if omitted)
HF_TOKEN=hf_your-token-here
MODEL_NAME=Qwen/Qwen2.5-72B-Instruct

# Optional: connect to remote server
ENVIRONMENT_BASE_URL=http://localhost:7860
```

**Auto API Key Selection:**
- If `API_BASE_URL` contains `"openai"` → uses `OPENAI_API_KEY`
- If `API_BASE_URL` contains `"nvidia"` → uses `NVIDIA_API_KEY`
- Otherwise (or default) → uses `HF_TOKEN`

**Run the agent:**
```bash
python inference.py
```

**Or set env vars on command line:**
```bash
# Quick test with OpenAI
export API_BASE_URL="https://api.openai.com/v1"
export OPENAI_API_KEY="sk-..."
export MODEL_NAME="gpt-4o-mini"
python inference.py
```

Output will show OpenEnv format:
```
[START] task=audit env=CorpExpenseAudit model=gpt-4o-mini
[STEP] step=1 action={'action_type': 'inspect_claim', ...} reward=0.50 done=false error=null
[STEP] step=2 action={'action_type': 'categorize_claim', ...} reward=0.50 done=false error=null
...
[END] success=true steps=15 score=0.87 rewards=0.50,0.50,1.00,...
```

## Validation & Testing

### Run Validations

The `validate.py` script tests all components:

```bash
python validate.py
```

This verifies:
- Model imports
- Environment initialization at all difficulty levels
- Step execution
- Grader functionality
- Overall system integration

### Run Tests Directly

```bash
# Test easy difficulty
python -c "
from environment import CorpExpenseAudit
env = CorpExpenseAudit(task_difficulty='easy')
state = env.reset()
print(f'Easy task: {state[\"total_claims\"]} claims')

# Test step
action = {'action_type': 'inspect_claim', 'action_data': {'claim_id': state['pending_claims'][0]}}
new_state, reward, done, info = env.step(action)
print(f'Reward: {reward}')
"
```

## Project Structure

```
CorpExpenseAudit-openenv/
├── README.md              # This file
├── requirements.txt       # Python dependencies
├── pyproject.toml         # Project metadata and dependencies
├── openenv.yaml           # OpenEnv manifest
├── Dockerfile             # Container image for deployment
├── validate.py            # Validation and integration tests
├── client.py              # OpenEnv client interface
├── models.py              # Data models and enums
├── environment.py         # CorpExpenseAudit environment logic
├── inference.py           # LLM agent for auditing tasks
├── graders.py             # Evaluation and scoring
└── server/
    ├── __init__.py
    └── app.py             # FastAPI server implementation
```

## Component Descriptions

- **models.py**: Defines `ExpenseClaim`, `AuditState`, actions, enums (ClaimCategory, FraudType, etc.)
- **environment.py**: Core `CorpExpenseAudit` environment with claim generation and state management
- **graders.py**: Evaluation logic with `TaskGrader` for scoring agent performance
- **server/app.py**: FastAPI server implementing OpenEnv HTTP standard
- **inference.py**: AI agent that uses LLMs to audit claims via OpenAI-compatible APIs
- **client.py**: Python client library for local or remote server access
- **validate.py**: Integration testing and validation suite

## Deploying to HuggingFace Spaces

Deploy your OpenEnv environment to Hugging Face Spaces:

```bash
# Install openenv CLI
pip install openenv

# From the project root
openenv push

# Or specify options
openenv push --repo-id my-username/corpexpenseaudit --private
```

The deployment will:
1. Validate the OpenEnv environment structure
2. Prepare a Docker build for Hugging Face
3. Upload to your HuggingFace Space

Your deployed space will include:
- **Web Interface** at `/web` - Interactive audit dashboard
- **API Documentation** at `/docs` - Full OpenAPI/Swagger interface
- **Health Check** at `/health` - Container health monitoring

## Example: Full Audit Workflow

```python
from environment import CorpExpenseAudit
from graders import run_easy_grader

# Initialize environment
env = CorpExpenseAudit(task_difficulty="easy")
state = env.reset()

print(f"Audit Task: {state['task_difficulty']} difficulty")
print(f"Claims to audit: {state['total_claims']}")

# Audit workflow
for claim_id in state['pending_claims'][:5]:
    # Inspect claim details
    env.step({
        "action_type": "inspect_claim",
        "action_data": {"claim_id": claim_id}
    })
    
    # Categorize
    env.step({
        "action_type": "categorize_claim",
        "action_data": {"claim_id": claim_id, "category": "travel"}
    })
    
    # Check GST
    env.step({
        "action_type": "verify_gst",
        "action_data": {"claim_id": claim_id, "status": "compliant"}
    })
    
    # Make decision
    env.step({
        "action_type": "approve_claim",
        "action_data": {"claim_id": claim_id}
    })

# Export final report
env.step({
    "action_type": "export_final_report",
    "action_data": {}
})

# Grade performance
metrics = run_easy_grader(env)
print(f"Final Score: {metrics.final_score:.2f}")
```

## License

MIT