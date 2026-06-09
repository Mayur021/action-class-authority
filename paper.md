# Action-Class Authority for AI Agents

## A Verification-Side Reference

**Version 1.0** · 2026-06-09

**Author:** Mayur Agnihotri
*Senior Information Security Specialist, StraightArc Technologies Pvt. Ltd.*
*Head of Threat Research, SecSphere SOC / SkyVirtRange*
*OWASP AISVS Contributor · CSA IAM Working Group Reviewer*

**License:** Creative Commons Attribution 4.0 International (CC BY 4.0)

---

## Abstract

When an AI agent takes an action, the question that determines whether the system stays safe is not whether the agent intended well, but whether the agent was *allowed* to take that action without a human in the loop. Existing controls answer this question by asking the agent itself — a self-classification pattern that prompt injection moves directly through. This paper proposes a different axis: classify each action by **reversibility** (the mechanism by which the action can be undone), declare the classification in the tool or action manifest at design time, and bind the enforcement gate to that declaration rather than to anything the agent produces at runtime. Four classes — read-only, reversible, external-reversible, and irreversible — cover the operational surface. A worst-case chain rule extends the model to multi-step composed sequences, where the chain's worst-case class governs the gate from the start. The architecture is anchored in OWASP AISVS C9.2.6 and C9.2.7 (proposed for 1.01, merged into main on 2026-05-27) and is compatible with the policy-engine enforcement requirement codified in AISVS C9.6.4. This reference is a verification-side companion to those standards.

---

## Executive Summary

**The problem.** Agentic AI systems plan, decide, and execute actions across systems faster than human oversight can respond. The control we have reached for — Human-in-the-Loop (HITL) — fails under two conditions that production deployments routinely produce: notification fatigue (the user clicks through approvals because they arrive too often) and self-classification (the agent decides which actions need approval, which makes the approval gate as trustworthy as the agent's classification claim).

**The axis.** Reversibility, not risk, is the cleaner gating axis. Risk drifts under load — the same action gets labeled "high-impact" on Tuesday and "moderate" on Friday. Reversibility does not drift. An action is either rollback-capable by the engine, rollback-capable by another system, rollback-capable by a person, or not rollback-capable at all.

**The taxonomy.** Four classes:

1. **Read-only.** The agent observes. Nothing in the environment changes. Allow.
2. **Reversible.** The engine itself can undo the action. Allow with log.
3. **External-reversible.** The action can be undone, but the undo routes through another system or another person. Approval gate.
4. **Irreversible.** No clean rollback. Approval required, with an evidence threshold agreed in advance.

**The chain rule.** When an agent composes actions into a multi-step sequence, the gate evaluates against the *worst-case* class the chain can reach. A read-only step followed by an irreversible step does not get to argue the gate down to "average reversibility." The irreversible step is what the gate sees.

**The architectural floor.** The classification must live in the tool or action manifest — declared at design time, not derived from the agent's runtime output. The gate consults the manifest. The agent can argue, the agent can reason, the agent can produce a confidence score. The gate does not care. It evaluates against the declared class. This is what AISVS C9.2.6 says, and it is what makes the model resistant to prompt injection.

**The standards-track anchor.** OWASP AISVS C9.2.6 (manifest-declared reversibility classification, evaluated by gate from declaration not from agent output) and C9.2.7 (blast radius as independent axis, raise-only, worst-case-governs across chains) were merged into AISVS main on 2026-05-27 and are proposed for 1.01. The verification-side architecture described in this paper is what those two requirements verify.

**The DFIR pairing.** Chain of custody is exactly this kind of declaration: a fixed, externally-verifiable record that the system can check against, that the actors in the system cannot rewrite. Action-class authority for AI agents is the same idea, applied one layer up.

---

## Table of Contents

| Part | Chapter | Page |
|---|---|---|
| **Part I: The Problem** | 1. The Decision-Rights Vacuum in Agentic AI | 4 |
|  | 2. Why "Risk Level" Fails as the Gate | 6 |
| **Part II: The Architecture** | 3. The Four-Class Reversibility Taxonomy | 8 |
|  | 4. Manifest-Declared Action Class | 11 |
|  | 5. The Worst-Case Chain Rule | 13 |
|  | 6. Gate Decisions per Class | 15 |
| **Part III: The Standards Anchor** | 7. OWASP AISVS C9.2.6 and C9.2.7 | 17 |
|  | 8. Adjacent Standards: CSA NHI, PieterKas, SANS AISMM | 19 |
|  | 9. Cross-Substrate Convergence | 21 |
| **Part IV: Applied Patterns** | 10. Malware Triage Workflow | 23 |
|  | 11. Forensic Readiness and Chain of Custody | 25 |
|  | 12. DFIR Use Cases | 26 |
| **Part V: Implementation** | 13. Manifest Schema | 27 |
|  | 14. Gate Implementation Notes | 28 |
|  | 15. Anti-Patterns to Avoid | 29 |
| **Closing** | 16. Future Work | 30 |
|  | 17. Citations and References | 31 |
|  | 18. About the Author | 33 |

---

# Part I: The Problem

## Chapter 1. The Decision-Rights Vacuum in Agentic AI

Agentic AI systems differ from chat-style AI systems in one operationally consequential way: they take actions. A chat-style system suggests text. An agentic system reaches across an API, executes a command, opens a ticket, pushes a configuration, deploys a signature, sends an email, or invokes a function. The system's output stops being a recommendation the human acts on; it becomes the action itself.

When the surface of action is this broad, the question of *what an agent is allowed to do without a human in the loop* becomes the load-bearing question of the security model. It is not the question most security frameworks were designed to answer. Identity and Access Management answers who an actor is. Network segmentation answers what an actor can reach. Endpoint detection answers what happened. Least privilege answers what an identity should be able to do. None of these answer the action-time question: *should this autonomous system, at this moment, be allowed to take this specific action without human review?*

We have reached for two controls. The first is Human-in-the-Loop (HITL), where an action that triggers a threshold pauses for a person to approve or deny. The second is Least Privilege, where the agent's identity is scoped tightly enough that broad swathes of dangerous actions are simply unavailable. Both are necessary. Both are insufficient.

HITL is insufficient because production deployments produce notification fatigue. When a SOC analyst sees twelve approval prompts an hour, the twelfth gets the same scrutiny as the first did for thirty seconds. The Lapsus0 breaches of Uber, Microsoft, and Cisco in 2022 demonstrated the operational pattern at scale, in a slightly different context: when humans are bombarded with approval requests, they approve. The same dynamic applies inside an AI agent's harness — perhaps more strongly, because the approval requests come faster and the actions they cover are more abstract.

Least Privilege is insufficient because real-world workflows require the agent to be able to do useful things. An incident-response agent needs to be able to read alerts, query systems, run sandboxes, and stage signatures. Scoping the agent down to read-only is a way of saying you do not have an agent; you have a viewer. The interesting and risky deployments are precisely the ones where the agent has been granted enough authority to do useful work, and where the authority therefore needs to be governed at finer granularity than identity-level scoping provides.

What is missing is a third primitive that sits between identity and action: a structural answer to *which action, at this moment in time, is this agent allowed to take without a human, and which classes of action require an approval gate.* That primitive is what this paper calls **action-class authority**.

The decision-rights vocabulary — borrowing the term from corporate governance, where decision rights describe which actor is empowered to decide what — fits this third primitive cleanly. Decision rights are not access rights. They are not identity claims. They are the structural declaration of which classes of action the agent is empowered to execute autonomously, and which classes the agent must route through a different control.

The argument of this paper is that **the right axis for decision-rights is reversibility**, the manifestation of that axis is a **manifest-declared action class**, and the enforcement mechanism is a **deterministic gate** that evaluates against the manifest declaration rather than against anything the agent produces at runtime. These three properties together form the verification-side architecture that OWASP AISVS C9.2.6 codifies.

The remainder of Part I makes the case for the axis. The architecture is developed in Part II.

## Chapter 2. Why "Risk Level" Fails as the Gate

Most workflows that gate forensic or operational AI actions use a risk-level taxonomy: low, medium, high, critical. The taxonomy looks intuitive. It fails operationally for four reasons.

**Reason 1: Risk drifts under load.** What feels like a "high-impact" call on Tuesday at 10am gets reclassified by Friday at 5pm. The SOC queue spikes, the team is tired, and a category that was carefully maintained when the system was first deployed becomes a category that means "whatever the on-call analyst feels right now." This is not a failure of the people; it is a failure of the definition. Risk is contextual. Contextual definitions cannot serve as the floor of a security control.

**Reason 2: Risk collapses two axes into one.** "High-risk" can mean an action with broad impact (blast radius) or an action that cannot be undone (irreversibility), or both. When the two are collapsed into a single score, the system loses the ability to treat them differently. A read-only action with broad scope is not the same problem as an irreversible action with narrow scope; the controls that mitigate each are different. AISVS C9.2.7 keeps blast radius as an axis *independent* of reversibility, raise-only, and worst-case-governing for chains. The structural separation matters. Collapsing the two axes is the architectural error that lets a "low blast-radius" estimate relax the gate on an irreversible action.

**Reason 3: The agent participates in its own risk classification.** If the gate evaluates the action's risk against what the agent claims the action's risk is, the gate is only as trustworthy as the agent's claim. An agent that has been prompt-injected, or that is operating on poisoned memory, or that is simply confused, can mislabel an irreversible action as low-risk and clear its own gate. The 2024-2025 incident catalog now includes multiple examples of this exact pattern: the AWS Kiro "delete and recreate" case from December 2025 and the PocketOS/Railway "staging task escalates to production deletion" case from April 2026, among others. The 2026 NIST IR 8596 draft on AI cybersecurity profiles names this category of failure as a recurring pattern, not a one-off bug.

**Reason 4: Risk does not naturally compose across chains.** When an agent plans a sequence of actions — a multi-step workflow that includes a query, a transformation, a write, and a push — the question of "what is the risk of the whole chain" has no clean answer if the only data available is a per-step risk score. Averaging them is wrong. Taking the maximum is closer but still does not capture the *kind* of failure mode that the chain can produce. Reversibility composes naturally: the worst-case class the chain can reach governs the gate, because that is the worst-case rollback story the chain can produce.

Reversibility, by contrast, has none of these problems:

- It is a property of the *mechanism* by which an action could be undone, not a property of the operator's judgment in the moment. It does not drift.
- It is structurally orthogonal to blast radius. A small-scope irreversible action and a large-scope reversible action are different problems; the taxonomy keeps them separate.
- It can be declared once at design time, by the team that wrote the tool, and stored outside the agent's runtime path. The agent cannot relabel its own action's reversibility class through injected content or a crafted tool response.
- It composes across chains: the worst-case class reachable governs, deterministically, without requiring the system to compute a synthetic per-chain risk score.

A cleaner way to put the failure: risk classification answers the question *what is the cost if this goes wrong*. Reversibility classification answers the question *can this be undone, and by what mechanism*. The first is a judgment. The second is a property. Judgments drift. Properties do not.

The next chapter introduces the four-class taxonomy that this paper proposes for the reversibility axis.

---

# Part II: The Architecture

## Chapter 3. The Four-Class Reversibility Taxonomy

The reversibility axis admits four operational classes. The classes are exhaustive for most practical deployments and are deliberately stable across time and context.

### Class 1: Read-only

**Definition.** The agent observes. Nothing in the environment changes as a result of the action. Examples: static analysis, signature lookup, hash check, metadata extraction, querying a database, reading a log, fetching a file's contents, computing a checksum, parsing a configuration.

**Gate decision.** Allow. The agent may execute these actions autonomously without per-action approval. Logging is recommended for audit completeness but is not load-bearing for safety.

**Failure mode that defines the class.** A read-only action cannot, by construction, change the environment. The worst case is that the agent gets bad data back. The data may then drive the agent to propose a more dangerous action — but the dangerous action belongs to its own class and is gated separately. Read-only actions feed downstream decisions; they do not commit them.

**Boundary cases.** Some "read" actions in practice do produce a side effect — for example, a query that emits a webhook, increments a counter, or charges per call. If the side effect is observable and consequential, the action is no longer read-only; classify it on the basis of what the side effect produces.

### Class 2: Reversible

**Definition.** The action changes the environment, but the engine that hosts the agent can undo the action. Examples: a sandbox spin-up that gets torn down, a signature added to a local detection corpus that can be pulled, an indicator pushed to a staging feed before it replicates, a transient configuration applied to a local test environment, a file written to a scratch directory that can be deleted.

**Gate decision.** Allow with log. The agent may execute these actions autonomously. Logging is required so the engine can roll back if needed.

**Failure mode that defines the class.** If the action was wrong, the engine can undo it from the same console. The undo is in the engine's authority; no external system, no other person, no approval chain. Rollback is bounded in time and effort.

**Boundary cases.** "Reversible" requires that the rollback is genuinely available to the engine, not just theoretically possible. A signature push that the engine could pull but only after twelve hours of replication delay is not reversible; it is external-reversible. The mechanism of reversal — and its cost — determines the class.

### Class 3: External-reversible

**Definition.** The action can be undone, but the undo routes through another system or another person. Examples: a signature that has already replicated to production endpoints, an indicator-of-compromise that went to a threat-intel partner, a case ticket in the SOC queue with another person's name on it, a deployment to a shared infrastructure tier, an email sent to a customer, a public artifact published to a partner portal.

**Gate decision.** Approval gate. The action requires explicit approval before execution. The approver does not have to be the same person who oversees the agent; it can be the owner of the downstream system. What matters is that the undo cost is non-trivial, and that the approval makes the cost visible to a human before the action commits.

**Failure mode that defines the class.** The undo is possible but not free. A signature can be pulled from production, but pulling it requires coordination with the deployment pipeline, may affect detections in flight, and may surface as a ticket on someone else's queue. The undo has externalities. The approval gate exists to capture those externalities before they propagate.

**Boundary cases.** External-reversible is the largest of the four classes in practice and the most prone to drift. An action that is genuinely reversible-by-engine in one deployment may be external-reversible in another, depending on how the rollback story works. The classification must be made by the team that owns the tool, in the deployment context where the tool runs.

### Class 4: Irreversible

**Definition.** No clean rollback. Examples: a public threat-intel publication, a containment action that interrupts a production service, a counter-intrusion swing at an attacker IP, a deletion of a database table, an outbound API call that triggers an irreversible business event (a charge, a notification, a regulatory filing), a destruction of a forensic artifact.

**Gate decision.** Human-in-the-loop with evidence threshold. Approval required, with the evidence the approver needs agreed in advance — not at 3am when the page goes off, but in the runbook when the action class was declared. The evidence threshold is part of the manifest, not part of the moment-of-decision.

**Failure mode that defines the class.** The action commits state that cannot be unwound. If it was wrong, the cost is permanent. The control that fits is one that pauses the agent and requires explicit, evidence-backed human judgment before the action proceeds.

**Boundary cases.** Some actions that look irreversible have partial rollback paths — a public publication can be retracted, a regulatory filing can be amended, a counter-intrusion can be followed by a clarifying statement. These do not change the class. The action's *first commit* is irreversible; the subsequent action (the retraction, the amendment, the clarification) is a separate action in its own class. The taxonomy describes individual actions, not multi-action recovery stories.

### Same action, different classes

A single tool can sit in different classes depending on how it is invoked. The clearest example is a signature push:

- A signature added to a local detection corpus that the engine can pull is **reversible**.
- The same signature pushed to a partner's feed that has replicated to other consumers is **external-reversible**.
- The same signature pushed to a public threat-intel publication that is now indexed by aggregators is **irreversible**.

The classification follows the *mechanism of reversal*, not the label on the action. A taxonomy that classifies by name (e.g., "signature push → external-reversible") will misclassify the local case and the public case. A taxonomy that classifies by mechanism (e.g., "engine-rollback possible → reversible; partner-coordinated rollback → external-reversible; no rollback → irreversible") gets the same action correctly placed in all three contexts.

This is the discipline AISVS C9.2.6 codifies: classification is per-tool, declared in the manifest at deployment, and follows the mechanism of reversal, not the surface label.

## Chapter 4. Manifest-Declared Action Class

The most important architectural property of action-class authority is that **the classification lives outside the agent's runtime output path**. The gate consults a static declaration, not a runtime claim.

### Why declaration matters

If the agent decides which class an action belongs to, the agent is participating in the decision that is supposed to constrain it. This is not theoretical. Prompt injection moves directly through this surface. An attacker (or a poisoned memory state, or a confused agent) can steer the model into mislabeling an irreversible action as reversible. The gate, looking at the agent's classification, accepts the label and proceeds. The agent did not bypass the gate. It walked through the front door.

The fix is structural: bind the gate to a classification the agent did not solely produce. In practice, this means the classification belongs in the **tool or action manifest** — a static declaration, authored by the team that owns the tool, stored outside the agent's runtime output path (a tool registry, an MCP server manifest, a policy store, a configuration file).

The gate consults the manifest. The agent can argue, the agent can reason, the agent can produce a confidence score in any direction. The gate does not care. It evaluates against the declared class.

### Where the manifest lives

A non-exhaustive list of valid storage locations:

- A **tool registry** maintained by the deployment team, where each callable tool has an entry that declares its reversibility class, blast radius range, intended use, and decommissioning criteria.
- An **MCP server manifest** that publishes available tools with their declared properties, including the reversibility classification.
- A **policy engine store** (such as an OPA bundle) where each tool's class is encoded as a fact that policy rules can read.
- A **service mesh annotation** that tags each callable endpoint with its class, enforceable at the proxy layer.
- A **deployment configuration** in version control, where each agent's allowable tools are listed alongside their declared classes.

What matters is not the storage technology. What matters is that the storage location is **(a) authored at design time, (b) outside the agent's runtime output path, (c) read by the gate at the moment of evaluation, and (d) versioned so that changes are auditable.**

### Fail closed

A tool with no declared class should fail closed to the most restrictive gate. The architectural reason is straightforward: if the manifest does not declare the class, the gate cannot know whether the action is read-only or irreversible. The safe default is irreversible. Anything else lets a missing classification be a path to silent privilege escalation.

This requirement is in AISVS C9.2.6's verification text: *"Verify that a new or modified tool with no declared class fails closed to the most restrictive gate."* The implementation work is small. The architectural integrity it provides is large.

### The agent can still reason; it just cannot rewrite the class

This is the most-frequently-misread part of the architecture. The manifest-declared classification does not mean the agent is forbidden from reasoning about an action's risk. It means the agent's reasoning is *input to the agent's planning*, not *input to the gate's evaluation*. The agent can produce a confidence score, a risk estimate, an explanation of why it thinks the action is safe — and that reasoning can usefully inform the human approver during an approval-gate moment. What the reasoning cannot do is *change the class*. The class is the manifest's. The gate reads the manifest.

The agent reasons. The manifest declares. The gate enforces. Three roles, three artifacts, no overlap.

## Chapter 5. The Worst-Case Chain Rule

Single-action gating is the start. Multi-step chains are where most production deployments live.

### What a chain is

A chain is any sequence of actions an agent composes to accomplish a goal. The agent does not decide each action in isolation; it plans a sequence in which each action's output becomes the next action's input. A malware-triage chain might run:

1. Collect sample (read-only)
2. Run sandbox (reversible)
3. Generate signature (reversible)
4. Push to local detection (reversible)
5. Push to production (external-reversible)

Each step in isolation is gateable. The chain as a whole reaches an external-reversible end state.

### Why per-step gating is insufficient

A per-step gate evaluates each action when the agent is about to execute it. The pattern looks safe — every action that crosses the gate gets the right authority level applied. But it has a structural failure mode: **the gate has no visibility into what the chain will reach**. The agent can string together three innocuous-looking read-only steps and arrive at a fourth step that is irreversible. The per-step gate sees the first read-only action, sees the second, sees the third, and only at step four discovers that an approval is needed. By that time, the agent has demonstrated intent, has set up dependencies, and the approval moment lands in a context where saying "no" is operationally expensive.

This pattern is not hypothetical. The AWS Kiro "delete and recreate" case from December 2025 reached an irreversible end state by composing individually low-gate steps. The PocketOS/Railway staging-to-production deletion case from April 2026 followed the same pattern. Neither was a single-step gate failure. Both were chain-composition failures.

### The chain rule

The chain rule is structural: **before a multi-step agent chain begins execution, the gate is set by the least-reversible (and, per AISVS C9.2.7, the highest-blast-radius) action the chain can reach.** Worst-case governs.

This means:

- A chain that contains any irreversible action gates at irreversible from the start.
- A chain that contains any external-reversible action (and no irreversible) gates at external-reversible from the start.
- A chain of reversible and read-only actions only gates at reversible.
- A chain of purely read-only actions gates at read-only.

The agent does not get to argue the gate down to "average reversibility." Average is meaningless. The worst case is what the chain can produce, and the worst case is what the gate evaluates against.

### Computing the worst case

For declarative tool graphs — workflows where the agent's plan is enumerable in advance — computing the worst-case class is a graph traversal: identify all tools the chain can reach, look up their declared classes in the manifest, take the worst.

For open-ended agents whose tool graph is not enumerable in advance — agents that can call any tool granted to them, in any order — the safe default is the worst-case class reachable by the agent's *granted tool set*. If the agent has been granted access to an irreversible tool, then any plan the agent generates gates at irreversible, even if the agent claims it will not use the irreversible tool. The agent's promise is not the gate's evidence.

This is the structural cost of granting an agent access to an irreversible tool: every plan gates at irreversible. The structural benefit is that the cost is visible. A deployment that wants chains to gate at lower classes must scope the agent's tool set accordingly.

### Why the chain rule is in C9.2.7

AISVS C9.2.7's verification text says: *"Verify that … before a multi-step agent chain begins execution, the gate is set by the least-reversible, highest-blast-radius action the chain can reach (worst-case governs)."* The architectural reason is the failure mode described above. The verification reason is that an organization that does not enforce the chain rule has a per-step gating story that looks complete on paper but is bypassable by composition.

## Chapter 6. Gate Decisions per Class

A gate that knows the class can act on it. This chapter spells out the operational decision per class.

### Read-only: Allow

The gate emits a decision of "allow" without further evaluation. Logging is recommended for audit completeness. The action proceeds immediately.

The architectural rationale: read-only actions cannot, by construction, change the environment. The cost of gating them with an approval requirement is operational friction without safety benefit. The cost of allowing them is zero or near-zero. Allow.

### Reversible: Allow with log

The gate emits a decision of "allow" with a logging requirement. The action proceeds immediately. The log records the action, its parameters, and a reference sufficient to reconstruct the action for rollback if needed.

The architectural rationale: reversible actions can be undone by the engine, but only if the engine knows the action happened and what its parameters were. The log is what makes the reversibility operational. Without the log, the action is in practice irreversible — not because the mechanism is gone, but because the engine no longer knows what to undo.

### External-reversible: Approval gate

The gate pauses the action and emits an approval request to the appropriate human or downstream system. The action proceeds only when the approval returns affirmative. The approval payload includes the action's canonicalized parameters; AISVS C9.2.2 requires that the parameters be cryptographically bound so a poisoned classification cannot route a different action through the same approval.

The approver does not have to be the agent's overseer. The approver is the owner of the downstream system that will receive the action. A signature push to a partner gates to the partner's intake team. A ticket created in another team's queue gates to that team's lead.

### Irreversible: Human-in-the-loop with evidence threshold

The gate pauses the action and emits an approval request that explicitly requires an evidence threshold. The evidence threshold is declared in the manifest alongside the class, not negotiated at the moment of decision.

The architectural rationale: irreversible actions cannot be undone, so the cost of being wrong is the cost of the action itself. The control must therefore make wrong actions less likely, not just trace them. Evidence threshold is the structural mechanism — the manifest declares what evidence the approver needs to see, and the gate does not permit the action to proceed without that evidence being present in the approval payload.

A simple example: a containment action that interrupts a production service might declare an evidence threshold of "(a) corroborating detection from a second source, (b) on-call lead approval, (c) timestamped business-impact assessment." All three must be in the payload. The on-call lead clicking "approve" is not sufficient if the corroborating detection is missing.

The evidence threshold is the difference between approval-as-rubber-stamp and approval-as-control. The threshold is the control. The click is the artifact.

### Summary table

| Class | Gate decision | Logging | Approver | Evidence threshold |
|---|---|---|---|---|
| Read-only | Allow | Recommended | None | None |
| Reversible | Allow with log | Required | None | None |
| External-reversible | Approval gate | Required | Downstream system owner | None or low (system-specific) |
| Irreversible | HITL | Required | Designated human | Declared in manifest |

---

# Part III: The Standards Anchor

## Chapter 7. OWASP AISVS C9.2.6 and C9.2.7

The verification-side standards artifact for the architecture described in Parts I and II is OWASP AISVS C9.2.6 and C9.2.7. Both requirements were merged into AISVS main on 2026-05-27 and are proposed for the 1.01 release. They live in the C9 chapter — Orchestration and Agents — under the C9.2 section on high-impact action approval.

### C9.2.6 (verbatim, Level 2)

> Verify that every action an agent can invoke carries a reversibility classification (read-only, reversible, external-reversible, or irreversible) declared in its tool/action manifest, and that the execution gate in 9.2.1 evaluates this declared classification rather than one derived solely from the agent's output at execution time.

The requirement names the four classes. It places the classification in the tool or action manifest. It binds the gate to the manifest rather than to the agent's runtime output. It is the verification-side codification of the architectural floor that Parts I and II of this paper describe.

The threat C9.2.6 addresses is named in the requirement's commentary: *agent-manipulated action classification* (MITRE ATLAS AML.T0099 AI Agent Tool Data Poisoning; OWASP LLM06:2025 Excessive Agency). When the same agent both proposes and classifies an action, a prompt-injected or compromised agent can mislabel an irreversible action as low-risk to clear its own gate. C9.2.6's verification text closes the gap by construction.

### C9.2.7 (verbatim, Level 3)

> Verify that blast radius is evaluated as an axis independent of reversibility; that it can only raise, never lower, the authority required for an action within its reversibility class; and that before a multi-step agent chain begins execution, the gate is set by the least-reversible, highest-blast-radius action the chain can reach (worst-case governs).

The requirement has three components. The first keeps blast radius as an axis distinct from reversibility — the structural separation discussed in Chapter 2. The second makes blast radius raise-only — a lower blast-radius estimate cannot reduce the gate on an action whose reversibility class is already irreversible. The third is the chain rule: before a multi-step chain begins, the gate is set by the worst-case reachable action.

The threat C9.2.7 addresses is named in its commentary: *gate evasion by composition or dimension-collapsing* (OWASP LLM06:2025 Excessive Agency; related to AML.T0099). The named cases include the AWS Kiro "delete and recreate" event from December 2025 and the PocketOS/Railway staging-task-to-production-deletion event from April 2026.

### Why two requirements

The split is structural. C9.2.6 names the manifest-declared class as the gate's input — a Level 2 requirement, expected of any organization with a non-trivial agentic deployment. C9.2.7 names the orthogonal blast-radius axis and the chain rule — a Level 3 requirement, expected of organizations with multi-tool agents or composed-action workflows.

C9.2.7 cannot be implemented without C9.2.6 (chain rule requires per-action class declarations to compose). C9.2.6 can be implemented without C9.2.7 (single-action manifest classification is meaningful even without chain composition). The dependency direction is one-way.

### Companion control: C9.6.4 (Level 2)

A closely related Level 2 requirement codified earlier in AISVS C9.6 specifies the enforcement boundary:

> Access control decisions are enforced by a policy engine, never by the AI model itself, and model-generated output cannot override or bypass access control checks.

C9.6.4 names the enforcement side: a deterministic policy engine, outside the model's reasoning loop, that the model cannot persuade. C9.2.6 names the verification side: the input the policy engine evaluates is the manifest, not the agent's runtime output. Together they form a closed loop. C9.6.4 says the gate cannot be persuaded. C9.2.6 says the gate's input cannot be supplied by the persuader. C9.2.7 says the gate sees the worst-case composition before the agent starts the chain.

## Chapter 8. Adjacent Standards: CSA NHI, PieterKas, SANS AISMM

Action-class authority sits in a standards-track territory that other working groups are addressing from adjacent angles. The architecture is compatible with — and complementary to — the work in those groups.

### CSA IAM Working Group: Defining Non-Human Identity

The CSA Identity and Access Management Working Group is under peer review for *Defining Non-Human Identity*, a paper that codifies the architectural properties of NHIs — the credentials, tokens, service accounts, and cryptographic identities that AI agents present when they act independently of human users.

A chain-level audit schema developed in joint peer-review work with Mallikarjunarao Sunke during the paper's review cycle proposes six properties that, taken together, make the chain attributable across composed agent actions. The six properties (joint contribution; Mallikarjunarao Sunke and the author of this paper) are:

1. **Chain id** — A stable identifier issued at chain start that propagates across every action in the chain.
2. **Declared worst-case action class at chain start** — The manifest-declared class for the worst-case action the chain can reach, bound at chain-id issuance, immutable through the chain.
3. **Gate decision and justification per step** — The gate's decision and the evidence it relied on, recorded for every step the chain executed.
4. **Per-step API record** — The action invoked, its parameters, the response, and the timestamp, for each step.
5. **Originating principal** — The identity that initiated the chain, recorded once at chain start.
6. **Immutability of the originating principal** — A cryptographic property that ensures the originating principal cannot be rewritten downstream of the chain's first commit.

Property 2 is the action-class manifest, expressed at the chain level. Property 6 is what makes "the agent acting outside its scope" attributable across composed steps rather than stopping at one credential. The six together pair with AISVS C9.2.6 + C9.2.7 to form an end-to-end audit story: AISVS verifies that the gate read the declared class; the chain-level audit schema verifies that the declaration, the gate decision, and the originating principal survived every step the chain executed.

### IETF-adjacent: PieterKas/agent2agent-auth-framework

The PieterKas/agent2agent-auth-framework draft (an IETF-adjacent specification under development) addresses agent-to-agent authorization protocols, including a CIBA (OpenID Connect Client-Initiated Backchannel Authentication) step-up mechanism for high-impact agent actions. Issue #114 on that repository (filed 2026-05-23) proposes that the CIBA step-up trigger be bound to the reversibility-graded action class rather than to a runtime risk estimate. The proposal frames CIBA as the *transport* for the human-in-the-loop step; the reversibility class is the *trigger* that determines when the transport fires.

This positioning matters because it keeps the protocol layer (CIBA, RFC 9396 Rich Authorization Requests, transaction tokens) decoupled from the gating-axis decision. The protocol can transport approvals for any axis the deployment chooses. The action-class taxonomy provides a structural axis that does not drift under load.

### SANS AI Security Maturity Model

The SANS AI Security Maturity Model (Chris Cochran, SANS Institute, May 2026) codifies a five-stage maturity progression for AI security programs (Unaware → Reactive → Defined → Managed → Optimizing). Stages 3 (Defined) and 4 (Managed) explicitly require:

- *"Human-in-the-loop thresholds and automated policy enforcement (policy decision points, intent gates, and consequence-based checks)"* — Stage 3, Govern pillar
- *"Just-in-Time (JIT) access controls (intent validation, consequence-based authorization gates, and real-time scope evaluation) for routine agent actions, reserving manual human-in-the-loop review for high-impact, irreversible operations where the latency cost is justified by the risk"* — Stage 4 commentary

The Stage 3–4 operational language describes the same architecture this paper develops, expressed in maturity-model vocabulary. The action-class taxonomy is the structural primitive that maps to *"consequence-based authorization gates"* and the chain rule is what makes *"intent validation"* operational beyond single-step evaluation.

### Cross-references

| Architecture primitive | OWASP AISVS | CSA IAM WG | PieterKas | SANS AISMM |
|---|---|---|---|---|
| Manifest-declared action class | C9.2.6 (verbatim) | NHI audit Property 2 | #114 (trigger axis) | Stage 3–4 (consequence-based gates) |
| Worst-case chain rule | C9.2.7 (verbatim) | NHI audit chain-id binding | — | Stage 4 (intent validation) |
| Originating principal immutability | C9.6.4 (companion) | NHI audit Property 6 (joint) | Chain-id propagation | Stage 3 (trace IDs across agent steps) |
| Deterministic policy engine | C9.6.4 (verbatim) | — | — | Stage 4 (JIT access controls) |

The cross-references are not coincidence. The same architectural floor is visible from each substrate. Each names it in its own vocabulary. Each requires the others to operationalize.

## Chapter 9. Cross-Substrate Convergence

The pattern that the action-class authority architecture appears in multiple standards-track substrates simultaneously is not unusual when an idea is structurally correct. It is the operational signal that the architecture is being independently rediscovered.

Beyond the four substrates named in Chapter 8, the same architectural primitives appear in:

- **Identient/AuthR (v0.1)** — Steve Tout's Author/Actor split and monotonic scope attenuation pattern. The Author/Actor split is the manifest-declaration property in identity vocabulary; monotonic scope attenuation is the worst-case-governs principle expressed at the scope layer.
- **Digital Identity Forum (Uday Bhaskar Hari's seven-episode Trust Frameworks series)** — A seven-pillar framework that includes pre-declared authority bounds, runtime gate enforcement, and immutable provenance of the originating principal. Pillars 4 (Authority Bounds) and 6 (Provenance Continuity) map directly to AISVS C9.2.6 and the chain-level audit schema's property 6, respectively.
- **CSA NHI WG (Ken Huang and colleagues)** — Beyond the Defining NHI paper, the working group's broader output names runtime gate enforcement, identity-bound action class, and chain-level audit as the three operational primitives for NHI governance at scale.
- **James A. Bex's AI Engineering Handbook (Decoupling Governance Logic from Application Logic, 2026)** — Three Architectural Laws and a Five-Layer Governance Fabric in which Layer 3 (Governance Enforcement Fabric) is the deterministic policy engine; Layer 5 (Decision Observability) is the immutable ledger of gate decisions; the agent is what Bex describes as *"governance-blind"* — the policy is enforced around it, not within it.
- **Riddhi Mohan Sharma's Ethical Hyper-Velocity (EHV) arXiv paper** — A Governance-Aware JIT Compiler that relocates the Policy Enforcement Point into the inference pipeline by construction, making non-compliant actions computationally unreachable. The EHV framing is the same architectural floor expressed in hardware-runtime terms; non-compliant actions cannot reach the gate because the gate is upstream of code generation.
- **CSA AARM Working Group (Autonomous Action Runtime Management)** — A standards-track group that codifies runtime enforcement of action authority, with a v1.0 specification published and v2.0 in active drafting. AARM and AISVS C9.2.6+C9.2.7 are companions: AARM names the runtime enforcement; AISVS names the verification of the runtime enforcement.

The convergence is the evidence. When OWASP, CSA, IETF-adjacent, academic, production-engineering, identity-protocol, and training-institution substrates each name the same architectural primitive from their own vocabulary, the primitive is no longer one practitioner's framing. It is the architectural floor of the domain.

What this paper provides is the verification-side reference: a single document that maps the primitives across substrates, anchored in OWASP AISVS C9.2.6 and C9.2.7, with operational guidance for DFIR and SOC deployments where the primitives become production controls.

---

# Part IV: Applied Patterns

## Chapter 10. Malware Triage Workflow

The clearest place to see action-class authority in production is the malware-triage chain that DFIR teams now increasingly automate with AI agents. The chain is short, the actions are concrete, and the reversibility classes vary across the chain.

### The chain

A standard automated malware-triage chain runs:

1. **Collect sample.** The agent retrieves a candidate malware artifact from the sandbox queue, the email gateway, the EDR's quarantine, or wherever the candidate originated. Reading and computing a hash. **Read-only.**

2. **Run sandbox.** The agent submits the sample to a dynamic-analysis sandbox, captures the behavioral indicators, and tears down the sandbox instance when complete. The sandbox is intentionally ephemeral; the engine fully controls its lifecycle. **Reversible.**

3. **Generate signature.** The agent derives detection logic (a YARA rule, a Suricata signature, an indicator-of-compromise structure) from the behavioral analysis. The signature is data; the engine can delete it without external side effect. **Reversible.**

4. **Push to local detection.** The agent stages the signature in the local detection corpus — the SOC's own SIEM or EDR. The engine controls the corpus; the signature can be retired without coordination. **Reversible.**

5. **Push to production.** The agent pushes the signature to the production detection tier, where it replicates to consumers — endpoint sensors, partner feeds, cloud-edge filters. Once replicated, the signature can be retired but only through the deployment pipeline. **External-reversible.** (Or **irreversible**, depending on the rollback story of the production pipeline.)

### What the chain rule produces

Under the worst-case chain rule, the gate evaluates the chain at the worst-case class reachable. For this chain, that is external-reversible (or irreversible if the rollback story is weak). The gate's decision applies *from the start of the chain*, not at step 5.

Concretely, this means:

- The agent's plan to execute the chain is approved at chain start by whoever owns external-reversible authority for production detection pushes — typically the detection-engineering lead or the SOC manager.
- The approval payload includes the canonicalized chain (all five steps, their parameters, their expected reversibility classes), so the approver sees the worst-case before approving the read-only first step.
- If the approval is granted, the chain executes through to step 5 without further per-step gating, because the per-chain gate already covered the worst case.
- If the approval is denied or revoked, the chain halts before step 1; the agent does not start collecting samples it cannot finish.

### What goes wrong without the chain rule

Without the chain rule, the gate evaluates each step in isolation. Step 1 reads as read-only; allow. Step 2 reads as reversible; allow with log. Step 3 reads as reversible; allow with log. Step 4 reads as reversible; allow with log. Only at step 5 does the gate detect external-reversible and request approval.

Two failure modes:

- **Operational waste.** The agent has spent compute on steps 1–4, holding sandbox runs and stored signatures, before learning at step 5 that the chain will not be approved. The first four steps are sunk cost.
- **Approval pressure.** The approver at step 5 sees a chain that has *already executed* through step 4. Denying step 5 means abandoning work the agent has already performed. The approval moment lands in a context where the human's incentive is to approve the action even if doing so feels wrong, because the alternative is wasted work. This is the same notification-fatigue pattern that defeats HITL more generally.

The worst-case chain rule eliminates both failure modes by setting the gate's decision *before* the chain begins, when the operational cost of denial is zero.

### Same chain, different deployment, different class

The same five-step chain can sit in different worst-case classes depending on the production push's rollback story:

- A deployment with a fully automated rollback pipeline that can retire a production signature within minutes: chain gates at **external-reversible**.
- A deployment where production signatures replicate to dozens of partner consumers with their own update cadences and no central rollback: chain gates at **irreversible**.
- A deployment where the production signature triggers an automatic block on customer endpoints with no retraction path: chain gates at **irreversible** with declared evidence threshold (e.g., second-source corroboration of the IOC before push approval).

The classification follows the mechanism of reversal in the specific deployment, not the surface label "push to production." This is the discipline AISVS C9.2.6 specifies: the team that owns the tool, in the deployment context where the tool runs, declares the class.

## Chapter 11. Forensic Readiness and Chain of Custody

DFIR teams have practiced a structurally similar discipline for decades, in a different context: chain of custody.

### What chain of custody does

A piece of evidence — a disk image, a memory capture, a network packet capture — has a chain of custody declaration that records who acquired it, when, under what authority, who accessed it after acquisition, and what transformations were applied. The chain of custody is **declared at acquisition**, **propagated through every subsequent handling**, and **immutable**: the chain cannot be rewritten by anyone in the chain. If a step is missing or contradictory, the evidence loses its admissibility.

The architecture is:

- **Declared at acquisition** — like the manifest declaration at design time.
- **Propagated through every handling** — like the chain-id binding across composed actions.
- **Immutable** — like originating principal immutability.
- **Externally verifiable** — like the gate evaluating against the manifest.

The actors in the chain (the responders, the analysts, the courts) cannot rewrite the chain of custody. They can add to it. They can reference it. They cannot edit it. This is the property that makes evidence admissible — the property of the system being structurally incapable of being persuaded to lie about its own provenance.

### Action-class authority is chain of custody, applied one layer up

The mapping is exact:

| Chain of custody | Action-class authority |
|---|---|
| Evidence is declared at acquisition | Action class is declared at design time (manifest) |
| Chain propagates through every handling | Chain-id propagates through every composed step |
| Chain is immutable; actors cannot rewrite | Originating principal is immutable; agents cannot rewrite |
| External verification (court, peer audit) | External verification (deterministic gate, audit log) |
| Failure to maintain chain → evidence inadmissible | Failure to maintain class → action gates at most restrictive |

The DFIR discipline already knows this pattern. The cybersecurity world already solved the chain of custody problem for human-handled evidence in the 1990s. Agentic AI is the same problem applied one layer up, where the actors are software, the cadence is machine-speed, and the chain is the composed-action sequence the agent executes.

The implication is operationally useful: DFIR teams already have intuition for what makes a chain robust. The architectural patterns that produced admissible evidence in 1995 also produce attributable agent action in 2026. The vocabulary is different. The structure is the same.

### Forensic readiness as pre-bound authority

Forensic readiness is the discipline of declaring, *in advance*, what evidence will be collected, in what format, with what acquisition procedures, before any incident requires the evidence. The readiness work is done before the page goes off, not during it.

Action-class authority is pre-bound authority: declared in the manifest, in advance, before any action requires the authority. The declaration work is done before the agent runs, not at the moment the action is about to execute.

The two disciplines share the architectural commitment that the *moment of stress is not the moment for decisions about authority*. Decisions made under pressure drift. Decisions made in advance hold.

## Chapter 12. DFIR Use Cases

A non-exhaustive set of DFIR workflows where action-class authority is operationally load-bearing:

### IOC enrichment and pivoting

An AI agent enriches an indicator-of-compromise by querying threat-intel platforms, sandboxes, and historical detection data. The enrichment chain is mostly read-only. If the chain includes a write to a shared IOC platform (TIP, MISP, OpenCTI), that step is external-reversible. The chain rule applies; the gate evaluates at external-reversible from start.

### Containment

An AI agent recommends or executes a containment action on a host or network segment: isolating the host from the network, blocking outbound DNS, killing a process, terminating a session. These actions are typically external-reversible (engineering can release the isolation; the firewall rule can be removed) but in environments where the isolation breaks a customer-facing service, the action is functionally irreversible during the outage window. Class determination follows the operational impact, not the surface label.

### Threat-hunting query authoring

An AI agent generates a hunting query and runs it against the SIEM. The query itself is read-only on the data. If the hunting workflow includes a step that writes the query results to a shared hunt-tracking system, the chain reaches external-reversible. If it includes a step that opens a case ticket on another team's queue, external-reversible. Read-only by default; the chain rule catches the write step.

### Memory and disk acquisition

An AI agent collects volatile memory or disk images from a candidate-compromised host. The acquisition is read-only on the host's state (assuming proper acquisition methods). The chain becomes external-reversible if the acquired artifacts are pushed to a shared evidence vault; irreversible if the acquisition triggers a notification to a third party (regulator, customer, partner) with a published cadence.

### Counter-intrusion

An AI agent considers an active response: scanning the adversary's infrastructure, deploying a beacon, executing a retaliation. Counter-intrusion is irreversible in nearly all operational deployments. The gate's decision is HITL with a declared evidence threshold; the threshold typically includes legal review and on-call lead approval. The agent does not get to execute counter-intrusion under any chain that does not satisfy the threshold.

### Report generation

An AI agent generates a draft incident report. The report itself is read-only or reversible. If the chain includes a step that publishes the report (to a customer portal, a regulatory filing, a public threat-intel feed), the chain reaches irreversible. The agent can draft autonomously. The publication step requires the irreversible-class gate.

The pattern holds across the DFIR portfolio. Read-only and reversible actions are the bulk of the work and are where AI agency genuinely accelerates the team. External-reversible and irreversible actions are the small fraction of actions that require an explicit human checkpoint — and the chain rule ensures that an agent's plan that *reaches* an external-reversible or irreversible action does not get to slip past the checkpoint by composition.

---

# Part V: Implementation

## Chapter 13. Manifest Schema

A minimal manifest schema for a tool or action declaration:

```yaml
tool:
  id: signature_push_to_production
  description: "Push detection signature to production replication pipeline"
  owner: detection-engineering@example.com
  reversibility_class: external-reversible
  blast_radius_range: [3, 5]   # see C9.2.7; 1-5 scale
  rollback_mechanism: "Engineering pipeline retracts within ~15 min coordinated window"
  evidence_threshold:           # required when class is irreversible
    required: false
  approval_routing:
    approver: detection-engineering-lead
    sla_target_minutes: 30
  manifest_signed_by: detection-engineering-lead@example.com
  manifest_signature: "..."     # cryptographic signature; see C9.2.3
  declared_at: 2026-04-12T14:22:00Z
  declared_by: detection-engineering-lead
  last_reviewed: 2026-06-01T10:00:00Z
```

The schema is illustrative; the production schema for a given deployment will include additional fields (logging, audit-trail integration, deprecation criteria, idempotency markers, parameter constraints). What matters is the load-bearing properties:

- `reversibility_class` — one of the four classes, declared explicitly.
- `blast_radius_range` — orthogonal axis, per C9.2.7; can raise the gate within the class.
- `rollback_mechanism` — the mechanism that justifies the class; auditable when class is questioned.
- `evidence_threshold` — required when class is irreversible; declared in advance, not at action time.
- `manifest_signature` — cryptographic binding so a poisoned manifest cannot be substituted at runtime.

The manifest is the source of truth. The gate reads it. The agent does not write to it.

## Chapter 14. Gate Implementation Notes

The gate's implementation has three responsibilities:

1. **Read the manifest** for every action the agent proposes. If the manifest entry is missing, the action is treated as irreversible (fail closed).

2. **Compute the chain's worst case** when the agent proposes a multi-step plan. For declarative tool graphs, this is a graph traversal; for open-ended agents, the worst case is the worst class in the agent's granted tool set.

3. **Emit the gate decision** before the chain begins. The decision binds for the duration of the chain — once approved at chain start, the chain executes through without per-step gating (assuming the chain's tool set is closed; if the agent adds tools mid-chain, the gate re-evaluates).

Common implementation patterns:

- **Sidecar policy engine** (Open Policy Agent or similar) consulting a manifest store, with the agent's tool calls intercepted at the proxy layer.
- **MCP server with policy hooks** that consult the manifest before forwarding the agent's call to the underlying tool.
- **Kubernetes admission controller** that gates the agent's outbound calls at the network layer, with the policy bundle keyed off manifest declarations.
- **Service-mesh annotation enforcement** with the manifest declarations encoded as service annotations and the mesh's authorization policy reading them.

The technology choice is operational. The architectural commitment is that the gate is *deterministic* (per C9.6.4), *outside the model's reasoning loop* (per C9.6.4), and *evaluating against the manifest* (per C9.2.6), with the *chain's worst case computed before chain start* (per C9.2.7).

### Logging

The audit log must capture, for each gate decision:

- Chain id
- Action proposed
- Manifest entry read (and its hash)
- Class determined
- Gate decision
- Approver (if approval was required)
- Evidence presented (if evidence threshold applies)
- Timestamp

The log is the input to the chain-level audit schema described in Chapter 8. Without the log, the schema cannot reconstruct the chain. With the log, the chain becomes attributable end to end.

## Chapter 15. Anti-Patterns to Avoid

A non-exhaustive catalog of architectural mistakes that defeat action-class authority:

### Anti-pattern 1: Agent classifies its own action

The most direct failure. The agent's classification is sent to the gate, the gate accepts it, and an injected or confused agent walks an irreversible action through a reversible-class gate. The fix is C9.2.6: classify in the manifest, evaluate the manifest, ignore the agent's runtime claim.

### Anti-pattern 2: Per-step gating without chain awareness

The chain composes around the gate. Each step looks safe; the composition does not. Fix: C9.2.7's chain rule. Compute worst-case at chain start.

### Anti-pattern 3: Risk-level taxonomy instead of reversibility

Risk drifts. Definitions of "high" change under load. Fix: use reversibility, which is a property of the mechanism, not a judgment about consequence.

### Anti-pattern 4: Collapsing blast radius and reversibility into a single score

A low blast-radius estimate relaxes the gate on an irreversible action. Per C9.2.7, the two axes must be independent, and blast radius can only *raise* authority within a class, not lower it.

### Anti-pattern 5: Approval as click-through, not as evidence-bound

The approver clicks "approve" without seeing the canonicalized action parameters or the required evidence. The action proceeds, the audit log shows an approval, and the system is structurally indistinguishable from no approval at all. Fix: AISVS C9.2.2 (canonical parameter rendering at approval time), C9.2.3 (cryptographically bound approval), evidence threshold declared in the manifest for irreversible actions.

### Anti-pattern 6: Manifest authored by the agent (or by the agent's prompt context)

If the manifest is generated by the same model that the gate is supposed to constrain, the manifest is not outside the agent's runtime output path; it is *inside* it. Fix: manifest authored by humans, signed at declaration time, stored in a registry the agent cannot write to.

### Anti-pattern 7: HITL as the gate for everything

HITL applied uniformly produces notification fatigue. The approver burns out, approvals get rubber-stamped, and the control degrades to no control. Fix: HITL only for irreversible actions, with declared evidence threshold. Read-only and reversible actions allow autonomously. The reservation of HITL for the small fraction of actions where it is structurally necessary is what keeps the control sharp.

### Anti-pattern 8: No fail-closed default

A new or modified tool with no declared class is treated as low-class by default. The result is silent privilege escalation as new tools land. Fix: missing declarations fail to the most restrictive class. The work to *add* a tool to the registry is the same work that determines its class; there is no scenario in which a tool is in the system but has not been classified.

---

# Closing

## Chapter 16. Future Work

This paper is version 1.0 of a reference. Several extensions are open:

**Joint chain-level audit schema specification.** The six-property schema referenced in Chapter 8 is joint peer-review work with Mallikarjunarao Sunke under CSA IAM Working Group review. A formal specification of the schema, including data types, audit-log integration, and cross-reference to AISVS C9.2.6+C9.2.7, is in development as a separate companion document.

**AARM integration.** The CSA AARM Working Group's v2.0 specification (in active drafting as of June 2026) is the runtime-enforcement complement to AISVS verification. A cross-walk between AARM's runtime semantics and AISVS C9.2.6/C9.2.7 verification semantics would close the verification-enforcement loop at the standards-track level.

**Open research questions on calibration.** AISVS C9.2.6 names the *manipulation* half of the verification problem (bind the gate to the manifest, not to the agent). The *calibration* half — what counts as low-risk within a class — remains organization-specific. A reference calibration profile for common deployment contexts (DFIR, SOC, IR, threat intel) would reduce per-deployment work.

**Multi-axis composition beyond reversibility and blast radius.** Some deployments may benefit from additional axes (e.g., reputational impact, regulatory scope, financial materiality). The architecture is extensible: any axis that is (a) raise-only and (b) worst-case-governs across chains can be added as a parallel axis without changing the reversibility model. Specifying when additional axes are warranted, and how they interact with the four-class reversibility taxonomy, is open work.

**PieterKas #114 disposition.** The reversibility-graded CIBA step-up trigger proposed in PieterKas/agent2agent-auth-framework Issue #114 is under chair disposition. If accepted, the protocol-layer integration with this paper's architecture would be tightened. If declined, the gap between AISVS verification and the protocol layer is named explicitly, motivating further standards work.

**Practitioner playbooks.** This paper provides architectural reference. Deployment-specific playbooks — for SOC teams, for DFIR, for AppSec, for cloud-native operations — would translate the architecture into the operational rituals that teams execute weekly. The author is preparing a companion *Decision-Rights Reference Architecture for AI SOC and Agentic SOC* as the SOC/SIEM/Agentic-SOC playbook.

## Chapter 17. Citations and References

### Standards

- **OWASP AISVS** — github.com/OWASP/AISVS — C9.2.6 + C9.2.7 (proposed for 1.01, merged 2026-05-27 into main, PR #822); C9.6.4 (Level 2, deterministic policy engine enforcement)
- **OWASP LLM Top 10** — LLM06:2025 Excessive Agency
- **MITRE ATLAS** — AML.T0099 AI Agent Tool Data Poisoning
- **CSA Identity and Access Management Working Group** — *Defining Non-Human Identity* (under peer review)
- **CSA Autonomous Action Runtime Management (AARM)** — aarm.dev — v1.0 published, v2.0 in active drafting
- **SANS Institute** — *AI Security Maturity Model™* (Chris Cochran, May 2026)
- **NIST IR 8596** — *Cybersecurity Framework Profile for AI* (preliminary draft Dec 2025)
- **NIST AI RMF** — Risk Management Framework for AI Systems
- **NSA AISC CSI** — *MCP Security Design Considerations* (2026-05-20)

### Reports and incident catalog

- AWS Kiro "delete and recreate" incident — December 2025
- PocketOS/Railway "staging-task-to-production-deletion" incident — April 2026
- Lapsus0 breaches of Uber, Microsoft, Cisco — 2022 (notification-fatigue precedent)

### Adjacent work

- **PieterKas/agent2agent-auth-framework** — github.com/PieterKas/agent2agent-auth-framework — Issue #114 (reversibility-graded CIBA step-up)
- **Identient/AuthR (v0.1)** — Steve Tout — Author/Actor split and monotonic scope attenuation
- **CoSAI WS4 (Secure Design Agentic Systems)** — github.com/cosai-oasis/ws4-secure-design-agentic-systems — PR #108 MCP security guidance v3.5
- **James A. Bex** — *AI Engineering Handbook: Decoupling Governance Logic from Application Logic* (2026)
- **Riddhi Mohan Sharma** — *Ethical Hyper-Velocity (EHV): A Hardware-Rooted Zero-Trust Runtime Enforcement Architecture for Agentic AI Systems* (arXiv)
- **Digital Identity Forum** — Uday Bhaskar Hari — seven-episode Trust Frameworks series

### Joint contributions

- **CSA NHI chain-level audit schema (six properties)** — joint peer-review contribution with Mallikarjunarao Sunke, under CSA IAM Working Group review

### Related figures

- **Figure 1** — *Action-Class Authority Reference Model* — two-panel diagram showing decision flow (Panel 1) and chain-rule worst-case example via malware-triage chain (Panel 2). Source: this paper, derived from companion article in eForensics Magazine (June 2026).

## Chapter 18. About the Author

**Mayur Agnihotri** leads threat research at the intersection of AI and security operations, focusing on the security of agentic AI systems and the decision-rights architecture that governs them.

**Current roles:**

- **Senior Information Security Specialist** at *StraightArc Technologies Pvt. Ltd.* (since 2020) — primary employer; AppSec, threat research, and security architecture across the StraightArc product portfolio.
- **Head of Threat Research** at *SecSphere SOC* — leading the threat-research function for the AI SOC platform, including detection engineering, response automation, and adversarial-AI workstreams.
- **Head of Threat Research** at *SkyVirtRange* — cyber-range exercise design, red-team scenario authoring, and DFIR-training workflows.
- **Board Member** at *SkyVirt* (since 2017).

**Standards-track work:**

- **OWASP AISVS Contributor** — authored the merged C9.2.6 and C9.2.7 requirements (proposed for 1.01) on manifest-declared reversibility classification and the worst-case chain rule for composed agent actions.
- **CSA IAM Working Group Reviewer** — review contributions on the *Defining Non-Human Identity* paper, including a joint chain-level audit schema with Mallikarjunarao Sunke.
- **PieterKas/agent2agent-auth-framework Issue #114** — filed proposal for reversibility-graded CIBA step-up triggers.
- **CoSAI WS4 collaborator** — Secure Design Agentic Systems Working Group, with substantive contributions to MCP security guidance.
- **Former OWASP AppSec India Co-Leader** (2016–2020), bringing AppSec discipline into agentic-AI standards work.

**Operational background:** Twelve-plus years across SOC engineering, OT/ICS security, DFIR, cyber-range exercise design, and threat research. Certifications include EC-Council C|EH, ICS/OT and operational-security training, and incident-analysis and risk credentials. His work centers on bringing the discipline that DFIR and forensic-readiness teams have practiced for decades into the agentic-AI security domain.

**Contact:** linkedin.com/in/mayur-agnihotri

*Note: This whitepaper is the author's independent work and does not represent the official position of StraightArc Technologies, SecSphere SOC, SkyVirt, or any standards body referenced herein. All standards-track citations are to the publicly merged or proposed text as of the publication date.*

---

*This document is licensed under Creative Commons Attribution 4.0 International (CC BY 4.0). Cite as: Agnihotri, Mayur. (2026). Action-Class Authority for AI Agents: A Verification-Side Reference. Version 1.0.*

*The Figure 1 reference model image referenced throughout this paper is available in the supplementary materials at* `figures/fig1-action-class-authority-reference-model.png`.

---

**End of Whitepaper v1.0**

*Approximate word count: 14,500. Approximate page count rendered: 28-30 pages.*
