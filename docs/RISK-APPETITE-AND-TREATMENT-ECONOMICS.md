# Proposal: Risk Appetite and Treatment Economics

## Status

Proposal for review. This document is intentionally non-normative and does not change the current security-audit evidence contract.

## Problem

The security-audit skill answers an important technical question:

> What security boundary failure is established by the available evidence, and how severe is the demonstrated technical impact?

That is necessary, but it is not sufficient for enterprise remediation prioritisation.

A confirmed vulnerability does not automatically imply that the economically rational response is to patch the source immediately. Remediation decisions also depend on:

- the business loss if the vulnerability is exploited;
- the probability or frequency of successful exploitation under the organisation's actual exposure;
- the organisation's risk appetite or tolerance for the affected risk scenario;
- the effectiveness of existing controls;
- the maximum achievable effectiveness of those controls;
- the cost and disruption of each available treatment;
- compensating controls and other treatment options;
- accepted exceptions, their expiry, and the residual risk they leave behind.

Technical severity, business risk, treatment economics, and management priority are related but distinct concepts. They should not be collapsed into one score.

## Design goal

Extend the security-audit workflow so that confirmed findings can feed an optional enterprise risk-decision layer without weakening the audit's source-grounded evidence model.

The key rule is:

> The audit establishes technical truth. The risk layer decides what to do about it.

A management decision must never rewrite or downgrade a technical verdict.

For example:

- a technically `confirmed` vulnerability remains `confirmed` even when its risk is accepted;
- a `high` technical severity remains `high` even when compensating controls reduce residual business risk below appetite;
- a low-cost control improvement may be preferred over a high-cost code change without pretending the underlying vulnerability is fixed.

## Keep the existing evidence contract

Do not overload `findings.json` with organisation-specific financial or risk-management fields.

The current verdicts should remain authoritative for technical evidence:

- `confirmed`
- `needs_validation`
- `rejected`

The current technical severity and confidence fields should also remain unchanged.

Instead, introduce an optional post-audit sidecar artifact keyed by the stable finding fingerprint:

`risk-decisions.json`

This makes the enterprise layer additive and keeps the core skill compatible with upstream changes.

## Proposed flow

```text
repository / pull request
        |
        v
security-audit skill
        |
        v
confirmed finding
        |
        +-----------------------------+
        | technical evidence contract |
        | findings.json               |
        +-----------------------------+
        |
        v
enterprise risk context
        |
        v
risk scenario mapping
        |
        v
current residual risk
        |
        v
compare with risk appetite
        |
        v
enumerate feasible treatments
        |
        +--> fix source
        +--> improve existing control effectiveness
        +--> add or replace a control
        +--> compensating control
        +--> restrict or disable functionality
        +--> architectural change
        +--> accept / exception
        |
        v
estimate residual risk and treatment cost
        |
        v
risk-decisions.json
```

## Trusted enterprise context

The risk layer may consume an optional trusted context document, for example:

`run-context.json`

This data is not inferred by the hunting agent and must have provenance.

Example:

```json
{
  "repository": "PaymentsAPI",
  "application": {
    "business_owner": "Digital Payments",
    "criticality": "critical"
  },
  "data": {
    "classifications": ["Confidential", "Personal Data"]
  },
  "exposure": {
    "internet_exposed": {
      "value": true,
      "source": "Azure Resource Graph",
      "observed_at": "2026-09-20T02:30:00Z"
    }
  },
  "risk": {
    "risk_appetite_id": "RA-CYBER-CUSTOMER-DATA-01"
  }
}
```

The source repository must not be allowed to supply or override trusted enterprise facts about itself.

## Risk scenario mapping

A confirmed technical finding should be mapped to one or more business risk scenarios.

Examples include:

- customer data disclosure;
- fraudulent transaction;
- credential compromise;
- production outage;
- loss of intellectual property;
- unauthorised administrative action;
- regulatory or contractual breach;
- compromise of software supply-chain integrity.

The finding identifies the technical cause and boundary failure. The risk model identifies the affected business scenario.

A useful relationship model is:

```text
Finding
  -> creates or increases -> RiskScenario
  -> affects -> Asset / BusinessProcess
  -> mitigated by -> Control
  -> delivered by -> Capability
  -> implemented by -> Technology
```

## Risk appetite integration

For each risk scenario, compare current residual risk with the applicable appetite.

Conceptually:

```text
current residual risk <= risk appetite
    -> mandatory treatment is not required by appetite
       (subject to legal, contractual, safety, or policy constraints)

current residual risk > risk appetite
    -> treatment is required
```

Risk appetite is therefore a decision constraint, not another severity score.

## Existing-control effectiveness

The treatment engine must explicitly consider whether risk can be brought within appetite by improving the effectiveness of controls that already exist.

This is important because the least-cost treatment may be:

> make an existing control work better

rather than:

> patch the application immediately

Each relevant control may therefore expose:

- current effectiveness;
- maximum achievable effectiveness;
- cost to improve effectiveness;
- dependencies or operational constraints;
- allowed exceptions;
- observed violations;
- confidence/evidence for the effectiveness estimate.

The solver can then determine the risk floor achievable with the current control set.

Example:

```text
Current residual risk:             £420k/year
Risk appetite:                     £100k/year

Improve existing IAM control
  cost:                             £15k
  residual risk:                    £180k/year

Improve IAM + existing WAF control
  cost:                             £31k
  residual risk:                     £85k/year

Patch application
  cost:                            £110k
  residual risk:                     £30k/year
```

In this example, improving existing controls is sufficient to satisfy appetite at lower cost, while the technical vulnerability remains open and must remain recorded as such.

## Maximum achievable control effectiveness and risk floor

For each candidate treatment set, calculate the best risk outcome that can actually be achieved under known constraints.

This produces a useful management concept:

> Given the controls currently available and their maximum achievable effectiveness, this is the lowest residual risk we can reach without changing the system.

Three cases follow:

1. **Already within appetite**  
   No appetite-driven remediation is mandatory.

2. **Above appetite, but existing controls can reach appetite**  
   Improve control effectiveness or apply compensating controls if economically preferable.

3. **Above appetite, and the control-set risk floor is still above appetite**  
   A structural treatment is required: source fix, architectural change, feature restriction/removal, new control, or another material change.

## Treatment economics

For each viable treatment, estimate both risk reduction and cost.

A simple expected-loss representation is:

```text
Expected Loss = probability/frequency of successful exploitation x loss magnitude
```

Then:

```text
Risk Reduction
  = Expected Loss Before
  - Expected Loss After

Net Remediation Value
  = Risk Reduction
  - Treatment Cost
```

A useful efficiency measure is:

```text
Marginal Risk Reduction per Currency Unit
  = Risk Reduction / Treatment Cost
```

This metric is useful for prioritisation under constrained budgets, but it must not override hard constraints such as:

- legal or regulatory obligations;
- contractual requirements;
- safety requirements;
- mandatory internal policy;
- explicit risk-appetite boundaries.

## Treatment cost

Treatment cost should not mean only developer hours.

Where material, include:

- engineering effort;
- testing and assurance;
- deployment and operational effort;
- downtime or service disruption;
- regression probability and expected regression cost;
- architecture rework;
- migration cost;
- customer or business-process impact;
- opportunity cost;
- ongoing operating cost.

Values may be point estimates, ranges, or probability distributions depending on the organisation's risk model.

The LLM must not invent financial values. Missing economic inputs remain explicit unknowns.

## Proposed risk-decisions.json

Example:

```json
[
  {
    "finding_fingerprint": "tenant-document-authz",
    "technical_verdict": "confirmed",
    "risk_scenarios": [
      {
        "scenario_id": "RS-CUSTOMER-DATA-CROSS-TENANT",
        "risk_appetite_id": "RA-CYBER-CUSTOMER-DATA-01",
        "current_risk": {
          "measure": "expected_annual_loss",
          "currency": "GBP",
          "low": 120000,
          "most_likely": 420000,
          "high": 1500000
        },
        "appetite": {
          "measure": "expected_annual_loss",
          "currency": "GBP",
          "value": 100000
        }
      }
    ],
    "treatments": [
      {
        "id": "T1",
        "type": "source_fix",
        "description": "Enforce tenant ownership at the final resource authorisation decision.",
        "estimated_cost": {
          "currency": "GBP",
          "most_likely": 110000
        },
        "residual_risk": {
          "currency": "GBP",
          "most_likely": 30000
        },
        "satisfies_appetite": true
      },
      {
        "id": "T2",
        "type": "improve_existing_controls",
        "description": "Increase IAM control effectiveness and apply existing WAF policy.",
        "estimated_cost": {
          "currency": "GBP",
          "most_likely": 31000
        },
        "residual_risk": {
          "currency": "GBP",
          "most_likely": 85000
        },
        "satisfies_appetite": true,
        "underlying_vulnerability_fixed": false
      }
    ],
    "decision": {
      "state": "treatment_selected",
      "selected_treatment_ids": ["T2"],
      "underlying_vulnerability_state": "open",
      "residual_risk_state": "within_appetite"
    }
  }
]
```

This schema is illustrative. A production schema should support ranges/distributions, confidence, provenance, multiple scenarios, control dependencies, and explicit unknowns.

## Finding lifecycle is separate from verdict

The technical verdict answers:

> What has the audit established?

The enterprise lifecycle answers:

> What is the organisation doing about it?

Suggested lifecycle states include:

- `new`
- `open`
- `treatment_selected`
- `fix_proposed`
- `remediation_in_progress`
- `mitigated`
- `fixed`
- `accepted_risk`
- `exception`
- `regressed`

Do not map accepted risk or an exception to `rejected`.

Do not map a compensating control to `fixed` unless the underlying vulnerability itself has been removed.

A useful distinction is:

```text
technical vulnerability: OPEN
residual risk: WITHIN APPETITE
treatment: COMPENSATED
```

## Exceptions and violations

A treatment may rely on an approved exception or a control with allowed exceptions.

Record at least:

- exception identifier;
- owner/approver;
- scope;
- justification;
- start date;
- expiry/review date;
- compensating controls;
- residual risk;
- evidence source.

Control violations should also be explicit inputs because actual control performance matters more than nominal control design.

An expired exception or newly observed control violation should trigger risk recalculation.

## Recalculation triggers

Risk decisions should be recomputed when relevant facts change, including:

- source changes;
- finding state changes;
- application exposure changes;
- business criticality changes;
- threat intelligence changes;
- control effectiveness changes;
- control violations;
- exception expiry;
- architecture changes;
- asset/data classification changes;
- remediation deployment or rollback.

This allows the same technically open vulnerability to move above or below appetite as its real operating context changes.

## Relationship with remediation

The audit should continue to recommend the smallest effective source fix.

A separate remediation workflow may then:

1. consume a confirmed finding;
2. generate a candidate patch;
3. generate or update a regression test for the violated invariant;
4. run normal tests;
5. rerun the security invariant;
6. submit a remediation pull request.

Risk treatment selection may choose a non-source treatment instead. That management decision must not prevent the source remediation from remaining available as an option.

## Relationship with conventional scanners

SAST, SCA, secret scanning, IaC scanners, SBOM/CVE feeds, and similar tools may provide leads or evidence.

Their output should not automatically become a confirmed finding.

The security-audit candidate gate and independent verification remain authoritative for the technical verdict.

Likewise, a risk engine must not manufacture technical findings from financial impact alone.

## Suggested implementation boundaries

To preserve upstream compatibility, implement this concept initially as an optional enterprise extension around the existing skill:

```text
security-audit skill
    |
    +-- findings.json
    +-- coverage-ledger.json
    +-- REPORT.md
    |
    v
enterprise risk adapter
    |
    +-- trusted run-context.json
    +-- control/capability context
    +-- risk appetite
    +-- treatment costs
    |
    v
risk-decisions.json
```

Only after experience with this interface should changes to the core skill or `report-schema.json` be considered.

## Review questions

Before implementation, review the following:

1. Should `risk-decisions.json` be part of this skill or a separate enterprise companion skill?
2. What minimum risk-context schema is generic enough to remain target-neutral?
3. How should risk scenarios be identified without allowing the LLM to invent enterprise facts?
4. How should uncertainty and distributions be represented?
5. What is the cleanest interface for external risk engines such as FAIR-based models?
6. How should control effectiveness, maximum achievable effectiveness, exceptions, and violations be represented?
7. Which changes should trigger automatic risk recalculation?
8. Should risk treatment economics ever influence hunter prioritisation, or only post-verdict remediation priority?
9. How should a finding be represented when residual risk is within appetite but the underlying vulnerability remains technically open?
10. How can this be implemented without making the core audit methodology dependent on any particular enterprise GRC platform?

## Non-goals

This proposal does not:

- weaken the evidence bar for a confirmed vulnerability;
- replace technical severity with financial risk;
- allow risk acceptance to change a technical verdict;
- require organisations to use monetary risk models;
- require FAIR specifically;
- allow the LLM to invent business impact, appetite, control effectiveness, or cost data;
- make the auditor itself a system of record for enterprise risk decisions.

The intended result is a clean separation:

```text
technical evidence
    -> business risk
    -> treatment alternatives
    -> treatment economics
    -> risk appetite decision
```

while preserving the original security-audit skill as the evidence-generating foundation.
