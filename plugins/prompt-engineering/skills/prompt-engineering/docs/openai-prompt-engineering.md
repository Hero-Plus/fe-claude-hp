# Prompt Engineering — OpenAI

Source: https://developers.openai.com/api/docs/guides/prompt-engineering
Fetched: 2026-03-03

## Choosing a Model

- **Reasoning models** generate internal chain of thought, excel at complex tasks and multi-step planning. Generally slower and more expensive than GPT models.
- **GPT models** are fast, cost-efficient, and highly intelligent, but benefit from more explicit instructions.
- **Large vs small models**: Large models are more effective at understanding prompts and solving problems across domains; small models are faster and cheaper.

## Prompt Engineering

Prompt engineering is the process of writing effective instructions for a model. Different model types (reasoning vs GPT) may need different prompting. Best practices:

- Pin production applications to specific model snapshots for consistent behavior.
- Build evals that measure prompt performance as you iterate or change model versions.

## Message Roles and Instruction Following

Messages have roles with different priority levels:

| Role        | Purpose                                                      |
| ----------- | ------------------------------------------------------------ |
| `developer` | Application instructions, prioritized ahead of user messages |
| `user`      | End user input, prioritized behind developer messages        |
| `assistant` | Model-generated responses                                    |

Think of developer and user messages like a function and its arguments:

- **developer messages** = function definition (rules and business logic)
- **user messages** = function arguments (inputs and configuration)

## Message Formatting with Markdown and XML

Use Markdown and XML tags to communicate structure:

- **Markdown**: Headers and lists mark distinct sections and hierarchy.
- **XML tags**: Delineate where content begins/ends. Attributes define metadata.

Recommended developer message structure (order may vary by model):

1. **Identity**: Purpose, communication style, high-level goals
2. **Instructions**: Rules, guidance, what to do and not do
3. **Examples**: Possible inputs with desired outputs
4. **Context**: Additional data (proprietary data, RAG results) — best positioned near the end

### Prompt Caching

Keep content you reuse across requests at the **beginning** of your prompt and among the **first API parameters**. This maximizes cost and latency savings from prompt caching.

## Few-Shot Learning

Steer a model toward a new task by including input/output examples in the prompt. Show a diverse range of possible inputs with desired outputs.

## Include Relevant Context

Add additional context the model can use:

- Give access to proprietary data outside training data.
- Constrain responses to a specific set of resources.

### Context Window

Models have finite context windows (100K to 1M+ tokens). Refer to model docs for specific sizes.

## Prompting GPT-5 Models

GPT models benefit from **precise, explicit instructions**. Best practices:

### Coding

- Define the agent's role with well-defined responsibilities.
- Enforce structured tool use with examples.
- Require thorough testing for correctness.
- Set Markdown standards for clean output.

### Front-End Engineering

Recommended libraries: Tailwind CSS, shadcn/ui, Radix Themes, Lucide icons, Motion.

For large codebases, include in prompts:

- **Principles**: Visual quality standards, modular/reusable components, consistent design.
- **UI/UX**: Typography, colors, spacing, interaction states, accessibility.
- **Structure**: File/folder layout for integration.
- **Components**: Reusable wrapper examples and backend-call separation.
- **Pages**: Templates for common layouts.

### Agentic Tasks

Three core practices for agentic rollouts:

1. **Planning and persistence**: Resolve the full query before yielding. Decompose into sub-tasks, reflect after each tool call.
2. **Preambles for transparency**: Explain why a tool is being called at notable steps.
3. **Progress tracking**: Use TODO lists or rubrics for structured planning.

Key prompt pattern for agent persistence:

> "Keep going until the user's query is completely resolved. Decompose into all required sub-requests and confirm each is completed. Only terminate when sure the problem is solved."

## Prompting Reasoning Models

Key differences from GPT models:

- **Reasoning models** = senior co-worker. Give high-level goals, trust them on details.
- **GPT models** = junior coworker. Provide explicit instructions for specific output.
