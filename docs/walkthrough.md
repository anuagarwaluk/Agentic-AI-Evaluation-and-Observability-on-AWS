# Walkthrough: what each module does and what to expect

This is the run order I followed, with the checks I used to confirm each stage before moving on.

## Module 0 - Environment setup (`00-prerequisites/`)

Run `0-environment-setup.ipynb`. It installs dependencies, provisions the DynamoDB products table via CDK, loads the sample catalog (products, orders, accounts under `sample_data/`), and verifies Bedrock model access.

Check before moving on: `verify_infrastructure.py` passes and the products table has data.

## Module 1 - Single agent prototype with RBAC (`01-single-agent-prototype/`)

Run `01-single-agent-prototype.ipynb`. The agent (Strands SDK, Claude Sonnet 4.6) connects over stdio to a FastMCP server exposing 11 tools: 6 read tools and 5 admin write tools backed by DynamoDB.

RBAC is tool filtering. A `UserSession` carries the role; `get_tools_for_role` hands the customer persona 6 tools and the admin persona 11. The notebook tests both personas and a cross-role boundary: admin creates a product, customer can find it but cannot modify it.

Check: the customer agent reports 6 available tools and refuses create/delete requests; the admin agent completes full CRUD.

## Module 2 - Evaluation and baseline (`02-evaluation-baseline/`)

Run `02a-strands-evaluation.ipynb` (the DeepEval notebook `02b` is an optional alternative). The evaluation dataset (`evaluation_dataset.json`) holds 86 test cases across 11 categories, including RBAC boundary probes, adversarial prompt-injection attempts, and out-of-scope queries.

The notebook runs the pyramid in order: agent responses are generated once and cached, deterministic assertions run first at zero cost, then the 7 custom LLM-as-judge evaluators score the cached responses, then meta-evaluation compares the judges against 15 expert-labelled known-answer pairs. Step 10 cross-checks with AgentCore built-in evaluators via the Evaluate API.

Outputs: `evaluation_results.csv`, `deterministic_results.json`, `baseline_metrics.json`, and release-gate evidence. These files are the quality contract every later module reads.

## Module 3 - Production deployment (`03-production-deployment/`)

Run `03a-ground-truth-dataset.ipynb` first: it converts Module 2 evidence into AgentCore managed dataset examples and simulation scenarios.

Then `03-production-deployment.ipynb`: IAM roles, the product tools Lambda and the RBAC interceptor Lambda, a Cognito user pool with customer/admin groups, an AgentCore Gateway with JWT authorizer, and the agent container (ARM64, OTEL auto-instrumented) deployed to AgentCore Runtime. CloudWatch log delivery routes application logs and traces.

Checks: tool discovery through the Gateway shows 6 tools with a customer token and 11 with an admin token; test invocations appear as `invoke_agent`, `chat` and `execute_tool` spans in CloudWatch; the batch release-candidate evaluation gate passes and writes durable evidence.

`streamlit_app/app.py` gives you a chat UI with Cognito login for manual testing.

## Module 4 - Online evidence and feedback loop (`04-online-eval-observability/`)

Run `04-online-evidence-and-feedback-loop.ipynb`. It attaches built-in evaluators (Helpfulness, GoalSuccessRate, ToolSelectionAccuracy, Coherence) to the deployed runtime as an online evaluation config, generates monitored traffic, and collects spans, runtime logs and evaluation metrics.

The important boundary: mined traces land in a review queue (`production_feedback_candidates.json`), and only reviewed, promoted examples update the managed dataset. Ambiguous or unsafe traces are rejected.

Check: monitored sessions appear in the expected log groups, and the dataset update manifest records a new published dataset version.

## Module 5 - Optimization and release decision (`05-agentcore-optimization/`)

Run `05-agentcore-optimization-experiments.ipynb`. It loads the Module 3 and 4 manifests, runs AgentCore Insights over a bounded trace window, and turns prompt/tool-description recommendations into a treatment config bundle. A config-bundle A/B test routes traffic through the Gateway and polls evaluator metrics for both variants.

The decision states are `continue`, `investigate`, `promote`, `rollback`. The notebook refuses to promote without adequate sample size and operator approval, and it never claims production changed unless the promotion manifest proves it.

## Cleanup

Run `cleanup/cleanup.py` to tear down the AgentCore resources, Lambdas, Cognito pool, ECR repository and DynamoDB table. AgentCore Runtime is pay-per-invocation, but Firehose, ECR storage and CloudWatch ingestion accumulate if left behind.
