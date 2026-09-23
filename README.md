# From Prototype to Production: Agentic AI Evaluation and Observability on AWS

Most AI agents die somewhere between the demo and the release decision. This repository is my hands-on build of the full journey across that gap: a product catalog agent for an e-commerce platform, taken from a local prototype with role-based access control through offline evaluation, production deployment on Amazon Bedrock AgentCore, online evaluation on live traffic, and an evidence-based optimization and release decision.

The discipline that ties the five stages together: nothing reaches production, and nothing changes in production, without evidence.




## What is in here

| Stage | Directory | What happens |
|-------|-----------|--------------|
| Module 0 - Setup | `00-prerequisites/` | CDK/DynamoDB provisioning, sample data, environment verification |
| Module 1 - Prototype | `01-single-agent-prototype/` | Strands agent + FastMCP server with 11 tools; RBAC by tool filtering (customer: 6 read tools, admin: all 11) |
| Module 2 - Evaluation baseline | `02-evaluation-baseline/` | 86-case evaluation dataset across 11 categories, 7 custom LLM-as-judge evaluators, deterministic assertions, meta-evaluation against 15 expert-labelled pairs |
| Module 3 - Production deployment | `03-production-deployment/` | Cognito JWT identity, tools as Lambda behind AgentCore Gateway with an RBAC interceptor, agent container on AgentCore Runtime with OTEL, batch release gate |
| Module 4 - Online evidence | `04-online-eval-observability/` | Online evaluation with AgentCore built-in evaluators, trace mining, human review before any trace becomes a test |
| Module 5 - Optimization | `05-agentcore-optimization/` | AgentCore Insights and recommendations, control vs treatment config bundles, A/B test, explicit release decision |

Each module directory contains a runnable notebook, the supporting Python modules, and unit tests.

## Architecture

<img width="2191" height="1767" alt="architecture-production" src="https://github.com/user-attachments/assets/d393a278-5091-45f9-8035-907abfaf763b" />
<img width="2192" height="1937" alt="lifecycle-evidence-loop" src="https://github.com/user-attachments/assets/8272adda-d461-4743-887a-37d33aa9f5b1" />


The prototype and the production system enforce the same access control rule at different layers:

| Prototype (Module 1) | Production (Module 3) |
|---|---|
| `UserSession(role='customer')` | JWT with `cognito:groups: ['customer']` |
| Local MCP server over stdio | Lambda tools behind AgentCore Gateway (MCP over SigV4) |
| Agent-side tool filtering only | Tool filtering + Gateway RBAC interceptor (defense in depth) |
| No observability | OTEL auto-instrumentation, CloudWatch traces and GenAI events |

RBAC works by filtering the tool list before the model ever sees it. A customer's agent receives 6 read-only tools; the write tools do not exist in its world. In production the Gateway interceptor enforces the same rule at the infrastructure layer, so a failure in either layer is caught by the other.

## The evaluation approach

The build follows an evaluation pyramid, cheapest checks first:

1. **Deterministic assertions** - expected tool called, RBAC enforced, required facts present. Milliseconds, zero cost.
2. **LLM-as-judge** - 7 custom evaluators (goal success, helpfulness, RBAC compliance, tool parameter accuracy, policy compliance, response quality, predicted CSAT) run locally to establish the baseline, plus AgentCore built-in evaluators as managed infrastructure for batch gates and live traffic.
3. **Meta-evaluation** - the judges themselves are scored against expert-labelled examples, because an uncalibrated judge is just a confident random number generator.

Two details I now consider non-negotiable for real systems:

- **Production traces are evidence, not truth.** Module 4 mines traces into a review queue; only reviewed examples are promoted into the managed dataset.
- **Recommendations are candidates, not changes.** Module 5 turns AgentCore recommendations into a treatment config bundle, A/B tests it against control, and records an explicit promote/rollback/continue decision. Promotion is never automatic.

## Key technologies

Strands Agents SDK, FastMCP, Amazon Bedrock (Claude Sonnet 4.6 as both agent and judge), Amazon Bedrock AgentCore (Runtime, Gateway, Identity, Evaluations, managed datasets, Optimization), Amazon Cognito, AWS Lambda, Amazon DynamoDB, OpenTelemetry, Amazon CloudWatch, Strands Eval and DeepEval.

## Running it

Prerequisites: an AWS account with Bedrock model access for `global.anthropic.claude-sonnet-4-6`, plus AgentCore, DynamoDB, Lambda, Cognito, CloudWatch, S3, Kinesis Firehose and IAM permissions.

```bash
# Install uv, then:
uv sync

# Run the notebooks in order:
# 00-prerequisites -> 01 -> 02a -> 03a -> 03 -> 04 -> 05
```

Start with `00-prerequisites/0-environment-setup.ipynb` to provision the DynamoDB table and load the sample catalog, then work through the module notebooks in order. Modules 3-5 build on artifacts the earlier notebooks write to disk (baseline metrics, release-gate evidence, deployment manifests), so order matters.

When you are done, run `cleanup/cleanup.py` to remove the AWS resources and avoid ongoing charges.

## Tests

```bash
python3 -m unittest discover -s 01-single-agent-prototype/tests
python3 -m unittest discover -s 02-evaluation-baseline/tests
python3 -m unittest discover -s 05-agentcore-optimization/tests
```

These run standalone. The Module 3 and Module 4 test suites validate the evidence contracts and expect the artifacts produced by running the Module 2 and Module 3 notebooks first.

## What stayed with me

1. A baseline is a contract. The metrics from Module 2 are not a report; they are the numbers every later gate and experiment is judged against.
2. Observability is not a dashboard, it is the evidence plane. Every improvement in Modules 4 and 5 starts from a trace.
3. The strongest access control for an LLM is subtractive: remove the tool from its world instead of moderating its answers.
4. An A/B result without sample size and regression checks is an anecdote.

## Credit

This build follows the AWS workshop [From Prototype to Production with AWS - Agentic AI Evaluation and Observability](https://catalog.us-east-1.prod.workshops.aws/workshops/927fb19e-6733-4986-904c-3e63b28c21e7/en-US). Code is licensed under MIT-0; see [LICENSE](LICENSE).

*Built and measured by Anu Agarwal — [linkedin.com/in/agarwalanu](https://www.linkedin.com/in/agarwalanu)*

<img width="732" height="56" alt="image" src="https://github.com/user-attachments/assets/6d6d2775-4fcf-45af-a872-aa3b19b7db72" />
