# Agentic Business Decision Support System

A multi-agent AI workflow for commercial scenario planning, validation, and decision support.

This project explores how Large Language Models (LLMs) and deterministic tools can be combined to support business decision-making while improving the **reliability, traceability, and controllability** of AI-generated outputs.

The system is built around a commercial terms-planning scenario for a cosmetics retailer, using sales and shipment data to generate, validate, compare, and present alternative business proposals.

---

## 1. Project Overview

The project implements a four-agent workflow for a commercial scenario-planning task based on a public GDPval business case.

The workflow:

1. ingests quarterly sales and shipment data;
2. generates three commercial terms scenarios;
3. validates AI-generated outputs using deterministic calculations and structured rules;
4. evaluates output quality;
5. produces an Excel-based decision-support deliverable;
6. records validation and retry information for traceability.

The central design idea is:

> **Use LLMs for ambiguous reasoning and business interpretation; use deterministic tools for calculation, validation, and file generation.**

---

## 2. Business Problem

Commercial decision-making often combines qualitative reasoning with deterministic calculations.

In this project, the business needs to compare alternative commercial terms across several dimensions:

- retailer margin;
- payment terms;
- marketing allowance;
- wholesale revenue;
- net wholesale revenue;
- cash-flow implications;
- partnership and growth considerations.

A single LLM can produce plausible recommendations, but it may also introduce:

- arithmetic errors;
- inconsistent field formats;
- unsupported assumptions;
- schema drift;
- incorrect structured outputs;
- recommendations that are difficult to verify.

The objective is therefore **not simply to ask an LLM for a recommendation**.

Instead, the project builds a workflow in which AI-generated reasoning is independently checked before being used in a business decision.

---

## 3. Solution Overview

The workflow separates the task into four major components:

1. **Data ingestion and validation**
2. **AI-driven scenario generation**
3. **Independent quality verification**
4. **Deterministic report generation**

This separation allows different tools to handle the tasks they are best suited for.

| Task | Approach |
|---|---|
| Business context interpretation | LLM |
| Scenario reasoning | LLM |
| Recommendation generation | LLM |
| Structured output validation | Pydantic |
| Numerical calculation | Python |
| Business-rule verification | Python |
| Excel generation | openpyxl |
| Final decision | Human review |

---

## 4. System Architecture

```text
Sales & Shipment Data
        │
        ▼
Agent 1: Data Ingester
Python + Pydantic
        │
        │  Human Check #1
        │  Validate source data
        ▼
Agent 2: Scenario Builder
LLM + Structured Prompting
        │
        │  Inner Feedback Loop
        │  Schema Validation & Retry
        ▼
Agent 3: Quality Verifier
Independent Python Recalculation
+ Quality Evaluation
        │
        ├──── Fail ───────► Corrections returned to Agent 2
        │                        ↑
        │                        │
        │                   Outer Feedback Loop
        │
        ▼ Pass
Agent 4: Excel Compiler
Python + openpyxl
        │
        ▼
Business Analysis Workbook
+ Executive Recommendation
+ Audit Log
        │
        ▼
Human Review & Final Sign-off
```

The architecture combines automated verification with **Human-in-the-Loop (HITL)** checkpoints so that AI outputs are not automatically treated as trustworthy.

---

# 5. Agent Design

## 5.1 Agent 1 — Data Ingester

**Role: data extraction and structural validation**

The Data Ingester reads quarterly sales and shipment information from the source workbook and converts it into a structured `QuarterlyData` object.

### Key responsibilities

- identify the relevant sales and shipment rows;
- extract Q1–Q4 and annual values;
- validate non-negative numerical values;
- check whether quarterly values reconcile with reported annual totals;
- generate warnings when inconsistencies are detected;
- produce typed outputs using Pydantic.

The ingestion process searches for relevant row labels instead of relying entirely on fixed cell positions, making it more robust to minor layout changes.

### Why this matters

Downstream AI reasoning is only useful if the source data is valid.

This agent therefore acts as the first reliability layer before any LLM reasoning occurs.

---

## 5.2 Agent 2 — Scenario Builder

**Role: business reasoning and structured scenario generation**

The Scenario Builder uses an LLM to generate three commercial terms scenarios based on predefined business constraints.

Each scenario considers:

- retailer margin;
- payment terms;
- marketing allowance;
- wholesale revenue;
- net wholesale revenue;
- quarterly and annual results.

The model receives business context explaining the retailer's expansion, cash-flow considerations, and marketing capabilities.

### Structured output

The LLM is required to return JSON that conforms to predefined Pydantic models.

The schema constrains fields such as:

- scenario names;
- margin ranges;
- payment-term values;
- marketing allowance;
- quarterly labels;
- numerical outputs.

If the model returns malformed output, the Pydantic validation errors are captured and returned to the model for correction.

### Inner feedback loop

Typical schema failures during development included:

- incorrect field names;
- percentages returned as strings;
- inconsistent quarter labels;
- missing fields;
- invalid output structures.

Instead of allowing these failures to propagate downstream, the system automatically creates a correction prompt and requests a revised response.

This creates an **inner self-correction loop**:

```text
LLM Output
    │
    ▼
Pydantic Validation
    │
    ├── Valid ─────► Continue
    │
    └── Invalid
          │
          ▼
   Validation Errors
          │
          ▼
   Correction Prompt
          │
          └────────► LLM Retry
```

---

## 5.3 Agent 3 — Quality Verifier

**Role: independent validation and quality control**

The Quality Verifier operates independently from the Scenario Builder.

A major design principle is:

> The LLM should not be responsible for deciding whether its own calculations are correct.

Instead, Python independently recalculates expected numerical values.

### Numerical verification

The verifier checks calculations such as:

```text
Wholesale Revenue
= Shipments × (1 − Retailer Margin)
```

```text
Marketing Allowance
= Shipments × Marketing Allowance %
```

```text
Net Wholesale Revenue
= Wholesale Revenue − Marketing Allowance
```

If the AI-generated values differ from the independently calculated values beyond the accepted tolerance, the output is rejected.

### Quality evaluation

The final deliverable is evaluated across five dimensions:

- **Correctness**
- **Completeness**
- **Clarity**
- **Format**
- **Usefulness**

The verifier also checks whether the output:

- contains all required scenarios;
- includes quarterly and annual views;
- identifies an explicit recommendation;
- provides financial justification;
- discusses business trade-offs;
- addresses cash-flow implications;
- includes appropriate business context.

### Outer feedback loop

If arithmetic problems are detected, the verifier returns structured error messages upstream.

For example:

```text
Scenario A / Q1:
Wholesale Revenue received = $99,999
Expected = $42,000
```

These corrections can then be inserted into the Scenario Builder's next prompt.

The resulting process is:

```text
Scenario Builder
      │
      ▼
Quality Verifier
      │
      ├── Approved ───► Continue
      │
      └── Rejected
             │
             ▼
      Error Diagnosis
             │
             ▼
      Correction Prompt
             │
             └────────► Scenario Builder
```

This forms the workflow's **outer self-correction loop**.

---

## 5.4 Agent 4 — Excel Compiler

**Role: deterministic report generation**

The final business deliverable is generated using Python and `openpyxl`.

The LLM does **not** directly generate the Excel workbook.

This design was chosen because deterministic software is more appropriate for:

- spreadsheet creation;
- formulas;
- formatting;
- charts;
- conditional formatting;
- data types;
- cell references.

### Workbook outputs

The generated workbook contains four main sheets:

1. **Deliverable**
2. **Comparison & Chart**
3. **Executive Summary**
4. **Assumptions**

The workbook includes:

- native Excel formulas;
- scenario comparison tables;
- quarterly and annual calculations;
- cash-flow timing logic;
- conditional formatting;
- favourability heat maps;
- grouped bar charts;
- executive recommendation.

Using native Excel formulas rather than hardcoded values also makes the resulting workbook easier to inspect and audit.

---

# 6. Reliability Design

Reliability is a core focus of the project.

The system does not assume that a stronger language model automatically produces trustworthy business outputs.

Instead, multiple control mechanisms are built into the workflow.

---

## 6.1 Typed Data Contracts

Pydantic models define the data structures exchanged between agents.

These models constrain:

- field names;
- data types;
- accepted values;
- numerical ranges;
- list lengths;
- scenario names;
- quarter labels.

Malformed LLM responses therefore cannot silently propagate through the workflow.

---

## 6.2 Schema Validation

Every structured LLM output is validated before being accepted.

If validation fails:

1. the exact errors are recorded;
2. the errors are converted into correction instructions;
3. the instructions are appended to the next prompt;
4. the model attempts to regenerate the output.

This reduces the risk of schema drift across repeated LLM calls.

---

## 6.3 Independent Numerical Verification

Critical calculations are independently recomputed using Python.

This is intentionally separated from the LLM reasoning process.

The architecture therefore distinguishes between:

- **generating an answer**, and
- **verifying whether the answer is correct**.

---

## 6.4 Automated Feedback Loops

The workflow includes two different feedback mechanisms.

### Inner loop

Handles structured-output failures.

```text
LLM
 ↓
Pydantic
 ↓
Schema Error
 ↓
Correction Prompt
 ↓
LLM Retry
```

### Outer loop

Handles logically valid but numerically incorrect outputs.

```text
Scenario Builder
 ↓
Quality Verifier
 ↓
Arithmetic Error
 ↓
Structured Correction
 ↓
Scenario Builder
 ↓
Re-verification
```

This distinction is important because an output can be **valid JSON but still contain incorrect business calculations**.

---

## 6.5 Human-in-the-Loop

Automation does not eliminate the need for human accountability.

The workflow therefore incorporates human review at decision-relevant stages.

### Human Check 1 — Source data

Confirm that:

- the correct workbook has been used;
- source values are reasonable;
- quarterly totals reconcile.

### Human Check 2 — Recommendation

Review:

- AI-generated business reasoning;
- commercial assumptions;
- recommendation language;
- strategic interpretation.

### Human Check 3 — Final sign-off

A human decision-maker remains responsible for approving the final commercial recommendation.

The system is therefore designed to **support decision-making rather than replace decision ownership**.

---

# 7. LLM vs. Deterministic Tool Boundary

A key product-design question in the project was:

> **Which parts of the workflow genuinely benefit from an LLM?**

The answer was not "everything."

### LLM responsibilities

The LLM is used where ambiguity and contextual reasoning are valuable:

- interpreting business context;
- reasoning about alternative commercial terms;
- generating scenario descriptions;
- producing an executive recommendation;
- explaining business trade-offs.

### Deterministic responsibilities

Traditional software is used where correctness and reproducibility are more important:

- arithmetic;
- data validation;
- schema validation;
- business-rule checking;
- spreadsheet formulas;
- workbook generation.

### Human responsibilities

Humans retain responsibility for:

- validating assumptions;
- assessing high-risk outputs;
- interpreting strategic implications;
- final approval.

This produces a three-layer decision structure:

```text
LLM
Reasoning & Interpretation
        │
        ▼
Deterministic Tools
Calculation & Validation
        │
        ▼
Human
Judgement & Accountability
```

---

# 8. Failure Testing

The project explicitly tests failure cases rather than evaluating only successful outputs.

A dedicated stress test deliberately corrupts generated scenario values.

Three example errors are injected:

- incorrect Wholesale Revenue;
- incorrect Marketing Allowance;
- incorrect Net Wholesale Revenue.

The Quality Verifier then:

1. independently recalculates the expected values;
2. identifies each inconsistency;
3. rejects the corrupted output;
4. produces structured correction messages;
5. passes those corrections back to the Scenario Builder;
6. re-evaluates the corrected output.

### Example stress-test flow

```text
Valid Scenario
     │
     ▼
Deliberate Corruption
     │
     ▼
Quality Verifier
     │
     ▼
3 Arithmetic Errors Detected
     │
     ▼
Output Rejected
     │
     ▼
Corrections Returned
     │
     ▼
Scenario Regenerated / Restored
     │
     ▼
Re-verification
     │
     ▼
Approved
```

The test demonstrates that the validation architecture can identify numerical failures even when the structured output itself is syntactically valid.

---

# 9. Key Learnings

## 9.1 AI capability is different from AI reliability

A capable language model can still produce:

- schema errors;
- incorrect formatting;
- inconsistent labels;
- numerical mistakes.

Model capability therefore does not eliminate the need for validation.

---

## 9.2 Validation should be part of the architecture

Reliability mechanisms are more effective when they are designed into the workflow from the beginning.

Instead of:

```text
Generate → Hope it is correct
```

the system follows:

```text
Generate
   ↓
Validate
   ↓
Diagnose
   ↓
Correct
   ↓
Re-validate
```

---

## 9.3 Not every task should use an LLM

Using an LLM for deterministic calculations or binary file generation introduces unnecessary uncertainty.

Python is more appropriate for tasks where the correct result is explicitly computable.

This project therefore treats **tool selection itself as a product-design decision**.

---

## 9.4 Structured outputs still require validation

Requiring JSON does not guarantee that the output is correct.

An LLM can return:

- valid JSON with incorrect values;
- incorrect field names;
- wrong data types;
- values outside allowed ranges.

Structured generation therefore needs both **schema validation** and **business validation**.

---

## 9.5 Business recommendations require trade-offs

The most profitable option is not automatically the most appropriate business option.

The workflow considers factors such as:

- profitability;
- retailer cash flow;
- marketing support;
- partnership value;
- working-capital exposure.

The goal is therefore not simply to maximise one metric, but to make the underlying trade-offs explicit.

---

## 9.6 Business AI systems need traceability

For decision-support systems, it should be possible to understand:

- what data entered the system;
- what the model generated;
- what failed validation;
- what corrections were made;
- why the final output was accepted.

Typed models, validation logs, retry records, and human review improve this traceability.

---

# 10. Technology Stack

### AI & LLM

- Anthropic Claude
- Structured Prompting
- Multi-Agent Workflow
- Automated Feedback Loops
- Human-in-the-Loop

### Data & Validation

- Python
- Pandas
- Pydantic

### Business Output

- openpyxl
- Excel formulas
- Conditional formatting
- Charts and business reporting

### Reliability

- Schema validation
- Independent arithmetic verification
- Retry mechanisms
- Audit logging
- Failure testing

---

# 11. Repository Structure

```text
agentic-business-decision-system/
│
├── README.md
│
├── agentic_workflow_demo.ipynb
│
├── data/
│   └── source_data.xlsx
│
├── outputs/
│   └── CosmoGenics_Scenario_Analysis.xlsx
│
├── notes/
│   └── development_log.md
│
└── .gitignore
```

> The repository structure may evolve as the project is further reorganised for portfolio presentation.

---

# 12. How to Run

## 12.1 Install dependencies

```bash
pip install pandas openpyxl pydantic anthropic
```

---

## 12.2 Configure the API key

Store the Anthropic API key as an environment variable rather than hardcoding credentials in the notebook.

### macOS / Linux

```bash
export ANTHROPIC_API_KEY="YOUR_API_KEY"
```

### Windows PowerShell

```powershell
$env:ANTHROPIC_API_KEY="YOUR_API_KEY"
```

The notebook can then access it using:

```python
import os

ANTHROPIC_API_KEY = os.getenv("ANTHROPIC_API_KEY")
```

---

## 12.3 Prepare the input data

Place the required sales and shipment workbook in the project directory or update the input path in the notebook.

Example:

```python
INPUT_FILE = "Sales_and_Shipment_CosmoGenics.xlsx"
```

---

## 12.4 Run the notebook

Open:

```text
agentic_workflow_demo.ipynb
```

and execute the cells in order.

The workflow will:

1. load and validate source data;
2. construct commercial scenarios;
3. validate structured outputs;
4. independently verify calculations;
5. evaluate the scenarios;
6. generate the business workbook;
7. produce an audit / execution log.

---

# 13. Outputs

The workflow generates two main outputs.

## Excel Business Analysis

```text
CosmoGenics_Scenario_Analysis.xlsx
```

The workbook contains:

- scenario-level calculations;
- commercial assumptions;
- quarterly results;
- annual totals;
- cash-flow timing;
- comparison tables;
- visual charts;
- executive recommendation.

## Audit Log

```text
notes_log.md
```

The log records information related to:

- execution;
- validation;
- retries;
- detected issues;
- workflow decisions.

---

# 14. Current Limitations

The project remains a prototype and has several limitations.

### Scenario scope

The current workflow evaluates a predefined commercial scenario rather than a wide variety of business problems.

### Limited external knowledge

The system primarily reasons over a specific input workbook and business context.

It does not currently require a full Retrieval-Augmented Generation (RAG) architecture.

### Evaluation design

The quality rubric is tailored to the current business task.

A production system would require broader evaluation datasets and repeated testing across more diverse scenarios.

### Human judgement

Some strategic considerations remain difficult to evaluate automatically.

Human review is still required for high-impact commercial decisions.

### Model dependency

LLM behaviour can vary across:

- models;
- prompt versions;
- API versions;
- sampling settings.

The workflow therefore focuses on reducing this uncertainty rather than assuming perfectly reproducible LLM behaviour.

---

# 15. Future Improvements

Potential next steps include:

- building a systematic bad-case evaluation dataset;
- tracking schema failure rates across repeated runs;
- measuring retry frequency;
- monitoring token usage and inference cost;
- comparing multiple LLMs under the same workflow;
- testing alternative prompt strategies;
- adding sensitivity analysis for commercial assumptions;
- expanding scenario configuration;
- adding more granular validation rules;
- introducing RAG when external market or policy information is required;
- integrating live business data sources;
- developing a lightweight user interface for business users;
- adding automated regression tests for agent behaviour;
- creating an evaluation dashboard for model and workflow performance.

---

# 16. Product Perspective

This project is designed not only as an AI implementation exercise, but also as an exploration of **AI product design decisions**.

Key product questions include:

### Where should AI be used?

AI is most valuable for tasks involving:

- ambiguity;
- contextual reasoning;
- natural-language interpretation;
- scenario generation.

### Where should AI not be used?

Traditional software is preferable when:

- a deterministic formula exists;
- outputs must be fully reproducible;
- numerical correctness is critical;
- structured file generation is required.

### How should AI failures be handled?

Instead of assuming failures can be eliminated entirely, the workflow is designed to:

1. detect failures;
2. classify them;
3. provide targeted feedback;
4. retry when appropriate;
5. escalate uncertain outputs to humans.

This reflects a broader principle:

> **A reliable AI product is not one in which the model never makes mistakes, but one in which mistakes can be detected, contained, corrected, and traced.**

---

# 17. Project Context

This project was originally developed as an academic group project and has been reorganised as a portfolio demonstration of:

- Agentic AI workflow design;
- business problem decomposition;
- LLM and tool boundary design;
- structured output validation;
- AI reliability mechanisms;
- Human-in-the-Loop governance;
- business decision support.

The portfolio version focuses on the **system architecture, product-design logic, reliability mechanisms, and business implications** of the workflow.
