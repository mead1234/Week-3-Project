# Competitive Intelligence Research Agent

## Overview

The Competitive Intelligence Research Agent is an agentic AI workflow built in n8n that automatically researches a target company and its competitive landscape.

A user submits a company name through an n8n form. The workflow verifies the company identity, discovers three relevant competitors, conducts targeted web research for each competitor, synthesizes the findings into a competitive analysis, validates the quality of the research, and generates a user-facing competitive intelligence report.

The system also includes confidence-based routing, human review, low-confidence warnings, and a one-time targeted research retry path.

---

## Problem Statement

Competitive research is often a manual and time-consuming process involving:

- Identifying relevant competitors
- Searching multiple web sources
- Comparing pricing and product capabilities
- Evaluating target customers and market positioning
- Reviewing recent developments
- Distinguishing verified information from uncertain claims
- Synthesizing findings into a decision-ready report

This project automates that process while adding validation and human oversight to reduce unsupported or low-confidence conclusions.

---

## Key Features

- Company identity verification before research begins
- Automated competitor discovery
- Exactly three competitors selected for focused analysis
- Parallel competitor research
- Web and news evidence collection
- Structured AI outputs using JSON schemas
- Competitive analysis and synthesis
- Research quality validation
- Confidence scoring
- Human-in-the-loop review
- Low-confidence warning path
- One-time targeted research retry
- Invalid-company protection
- User-facing final report through an n8n form
- Explicit handling of unverified information

---

## Workflow Architecture

```text
User Form
   ↓
Initialize State
   ↓
Competitor Search
   ↓
Entity Verification Agent
   ↓
Entity Verified?
   ├── FALSE → Entity Verification Failure → User Clarification
   │
   └── TRUE
         ↓
Competitor Discovery Agent
         ↓
Split Competitors
         ↓
Competitor Research Search
         ↓
Competitor Research Agent
         ↓
Aggregate Competitor Research
         ↓
Competitive Analysis Agent
         ↓
Research Validation Agent
         ↓
Confidence Check
         │
         ├── Confidence >= 75
         │       ↓
         │   Final Report Agent
         │       ↓
         │   Prepare Final Response
         │       ↓
         │   User Report
         │
         └── Confidence < 75
                 ↓
             Human Review
                 ↓
          Review Decision Router
           /        |        \
      Approve     Warning     Retry
         |           |          |
         |     Add Warning      |
         |           |     Increment Retry Count
         |           |          ↓
         |           |     Retry Research Search
         |           |          ↓
         |           |     Retry Synthesis Agent
         |           |          ↓
         |           |     Retry Final Report
         |           |          |
         └───────────┴──────────┘
                     ↓
              User-Facing Report
```

---

## Agent Responsibilities

### 1. Entity Verification Agent

Validates that the initial web search results clearly refer to the exact company submitted by the user.

The agent prevents the system from silently substituting a similarly named company.

If the company cannot be confidently verified, research stops and the user is asked to verify the company name or provide a website/domain.

### 2. Competitor Discovery Agent

Uses the initial search evidence to identify exactly three direct competitors.

For each competitor, the agent returns:

- Competitor name
- Reason for selection
- Supporting source URL

### 3. Competitor Research Agent

Researches each competitor independently and creates a structured profile covering areas such as:

- Pricing
- Core features
- Target customers
- Market positioning
- Key differentiators
- Recent news/developments
- Supporting sources
- Research completeness

### 4. Competitive Analysis Agent

Combines the three competitor profiles into a strategic competitive intelligence analysis for the target company.

The analysis separates evidence-backed findings from interpretation and identifies meaningful competitive themes and opportunities.

### 5. Research Validation Agent

Acts as an independent quality-control layer.

It evaluates:

- Competitor coverage
- Data completeness
- Evidence quality
- Recency

It then produces an overall confidence score and routing decision.

### 6. Final Report Agent

Transforms the validated analysis into a concise, user-facing competitive intelligence briefing.

The report preserves research limitations and explicitly identifies information that could not be verified.

### 7. Retry Synthesis Agent

When a human reviewer requests additional research, this agent combines the original analysis with one additional targeted research attempt.

The retry path does not hide unresolved evidence gaps. The final output is explicitly identified as a post-retry report.

---

## Confidence and Human Review

The production confidence threshold is:

```text
75 / 100
```

If:

```text
confidence_score >= 75
```

the workflow proceeds automatically to final report generation.

If:

```text
confidence_score < 75
```

the workflow pauses for human review.

The reviewer can choose:

1. **Approve and generate report**
2. **Continue with warning**
3. **Retry research**

The retry path is intentionally limited to one targeted additional research attempt to prevent uncontrolled retry loops.

---

## Entity Verification Guardrail

Evaluation revealed an important failure mode.

An intentionally invalid company input was initially matched by search results to an unrelated real company. This caused competitor discovery to begin against the wrong entity.

Although the Research Validation Agent later detected the weak evidence and reduced confidence to 45/100, the workflow had already performed unnecessary downstream research.

To address this issue, an Entity Verification Gate was added immediately after the initial search.

The verifier returns:

```json
{
  "submitted_company": "Example Company",
  "matched_company": "Example Company",
  "entity_verified": true,
  "confidence": 95,
  "reason": "Search evidence clearly matches the submitted company."
}
```

If `entity_verified = false`, competitor research does not run.

Instead, the user receives a clarification message asking them to verify the company name or provide the company website/domain.

---

## Evidence and Hallucination Control

The workflow is designed to avoid presenting uncertain information as fact.

When evidence is insufficient, the agents use labels such as:

```text
Not verified
Partial
Insufficient evidence
```

The Research Validation Agent independently evaluates the quality and completeness of the research before the final report is generated.

This is particularly important for pricing information, which may be:

- Negotiated
- Region-specific
- Plan-specific
- Outdated
- Available only from secondary sources

The system therefore avoids manufacturing precise pricing when reliable evidence is unavailable.

---

## Retry and Failure Handling

### Search Query Length Failure

During testing, a retry search generated a query containing 146 words and 1,214 characters.

The search service allowed a maximum of:

```text
50 words / 400 characters
```

The retry search failed.

The issue was corrected by shortening the retry query and using validation issues as synthesis context rather than embedding all validation text directly into the search request.

### Invalid Company Input

Before the Entity Verification Gate was introduced, an invalid company name could be incorrectly resolved to another company.

After mitigation:

```text
Invalid Company
      ↓
Competitor Search
      ↓
Entity Verification
      ↓
entity_verified = false
      ↓
Research stops
      ↓
User asked to verify company
```

This prevents unsupported competitor reports from being generated.

---

## Evaluation

The workflow was evaluated using 15 test cases covering both research quality and agent behavior.

Evaluation included:

- Workday baseline research
- Canva cross-domain research
- Duolingo cross-industry research
- Mercury ambiguous-name handling
- Invalid-company handling
- Exactly-three-competitor enforcement
- Target-company preservation
- Structured-output reliability
- Unverified pricing behavior
- High-confidence routing
- Human approval
- Continue-with-warning routing
- Retry research
- Search API failure/recovery
- User-facing report delivery

### Key Results

| Test | Result |
|---|---|
| Workday baseline | PASS |
| Canva cross-domain | PASS |
| Mercury ambiguity | PASS with limitation |
| Duolingo generalization | PASS |
| High-confidence routing | PASS |
| Human approval | PASS |
| Continue with warning | PASS |
| One-time retry | PASS |
| Unverified pricing handling | PASS |
| Structured output | PASS |
| Exactly 3 competitors | PASS |
| Target preservation | PASS |
| Search query-limit recovery | PASS after fix |
| Invalid-company test | Initial partial failure → mitigated |
| User-facing report | PASS |

A detailed evaluation and failure analysis is provided in:

```text
docs/Competitive_Intelligence_Research_Agent_Evaluation_Report.docx
```

---

## Example Evaluation Findings

### Mercury

The ambiguous input `Mercury` was resolved to the fintech/business-banking company.

Competitors included:

- Brex
- Rho
- Bluevine

Research Validation confidence:

```text
90 / 100
```

Result:

```text
PROCEED
```

### Invalid Company

An invalid company input initially resulted in:

```text
confidence_score = 45
decision = human_review
```

After implementing Entity Verification, the same class of invalid input was stopped before competitor discovery.

### Duolingo Regression Test

After adding Entity Verification:

```text
Entity verification confidence = 95
Entity verified = true
```

Competitors:

- Babbel
- Rosetta Stone
- Busuu

The complete workflow then successfully generated the final user-facing report.

---

## Technology Stack

- **n8n** — workflow orchestration and human-in-the-loop routing
- **OpenAI models** — competitor discovery, research synthesis, analysis, validation, and reporting
- **Structured Output Parsers** — schema-controlled agent outputs
- **Web/Search API** — competitor and market evidence retrieval
- **n8n Forms** — user input, human review, and final report delivery

---

## Project Structure

```text
competitive-intelligence-research-agent/
│
├── README.md
│
├── workflows/
│   └── competitive-intelligence-research-agent.json
│
├── docs/
│   └── Competitive_Intelligence_Research_Agent_Evaluation_Report.docx
│
└── screenshots/
    ├── workflow-overview.png
    ├── entity-verification.png
    ├── human-review.png
    ├── retry-report.png
    └── final-report.png
```

---

## How to Run

1. Import the workflow JSON into n8n.
2. Configure the required OpenAI credential.
3. Configure the required web/search API credential.
4. Verify that credentials are stored in n8n and are **not hard-coded in the workflow or repository**.
5. Execute or publish the workflow.
6. Open the n8n form.
7. Enter a company name.
8. Allow the research workflow to complete.
9. If confidence is below the threshold, complete the Human Review form.
10. View the final Competitive Intelligence Report.

---

## Security

API keys and credentials must never be committed to GitHub.

Credentials should be configured through n8n's credential management system.

The repository should exclude:

```text
.env
.env.*
*.key
credentials.json
secrets.json
```

---

## Known Limitations

- Search quality directly affects downstream research quality.
- Pricing may not be independently verifiable.
- Company-name ambiguity may occasionally require user clarification.
- Confidence scores are model-generated rather than statistically calibrated probabilities.
- The evaluation uses a focused test set rather than a large benchmark.
- The default n8n form interface is functional but not optimized for long-form report presentation.

---

## Future Improvements

Potential improvements include:

- Asking users for a website/domain when company identity is ambiguous
- Prioritizing official vendor sources
- Adding deterministic entity and source-quality checks
- Automated regression testing
- Confidence-score calibration against human evaluation
- Persistent research history
- Scheduled competitor monitoring
- Richer report visualization
- PDF or document export

---

## Conclusion

The Competitive Intelligence Research Agent demonstrates a complete agentic research workflow with automated competitor discovery, multi-step research, evidence-aware synthesis, validation, confidence-based routing, human oversight, targeted retry, and user-facing report generation.

Evaluation identified a real entity-resolution failure, which led to the addition of an Entity Verification Gate. Retesting demonstrated that the new guardrail blocks invalid inputs while preserving successful processing for valid companies.

The final system therefore demonstrates not only successful AI research automation, but also evaluation-driven iteration, failure handling, and responsible use of confidence and human review.
