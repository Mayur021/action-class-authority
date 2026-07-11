# Action-Class Authority

**Why the Impossibility of Complete Guardrails Forces a Reversibility-Graded Verification Layer for Agentic AI**

Mayur Agnihotri — StraightArc Technologies Pvt. Ltd., India
ORCID: [0009-0007-0137-3780](https://orcid.org/0009-0007-0137-3780)

> A version of the core pattern documentation in this work is under review at *IEEE Security & Privacy* magazine.

## Abstract

Production agentic AI systems fail at one architectural seam: a prompt-injected, memory-poisoned, or supply-chain-compromised agent retains valid credentials and reasons over its own action authority. This paper argues that the response to that failure is not a stronger guardrail but a differently-placed one. A formal impossibility result establishes that no finite guardrail can be universally robust against adversarial input, and four independent empirical studies from May 2026 measure agent-reasoning compromise at rates high enough that any reasoning-based defense is statistically dominated. Read together, the formal and empirical results force a specific architecture. Because prevention cannot be completed and detection carries latency, the actions safe to leave under detection alone are exactly the reversible ones, while irreversible actions require authority established before execution by a mechanism the agent cannot rewrite. We call this pattern action-class authority: a verification layer that decouples a trusted, design-time reversibility classification from agent runtime reasoning, enforces it at a deterministic gate, and governs multi-step chains by the worst-case reversibility class present anywhere in the chain. We document the OWASP AISVS v1.0 verification requirements that operationalize this pattern (C9.2.3, C9.2.4, C9.2.10), analyze three canonical 2026 incidents, position the pattern against the 2026 literature on agent runtime enforcement and containment, and close with open research questions.

## The paper

- [`action-class-authority.pdf`](action-class-authority.pdf) — 10-page PDF
- [`action-class-authority.tex`](action-class-authority.tex) — LaTeX source

## What it argues, in one line

When complete guardrails are provably impossible and detection carries latency, irreversible actions cannot be made safe by detection alone; they require authority established before execution, on a trusted classification the agent does not supply and a gate the agent does not control. Reversible actions can rest on detection; irreversible ones cannot.

## Keywords

agentic AI security · action-class authority · reversibility-graded authority · guardrail impossibility · deterministic gate · worst-case chain rule · OWASP AISVS · agent verification · capability-based security

## How to cite

```
Mayur Agnihotri, "Action-Class Authority: Why the Impossibility of Complete Guardrails
Forces a Reversibility-Graded Verification Layer for Agentic AI," preprint, 2026.
```

A DOI will be added here once the repository is archived via Zenodo. See `CITATION.cff`.

## License

Text and figures: Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0). See [`LICENSE`](LICENSE).
