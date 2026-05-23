# Agent Telephony Protocol (ATP)

> *An open protocol for AI agents to use the phone — voice, SMS, MMS, RCS, iMessage Business, WhatsApp Business, Google Business Messages, voicemail, identity, and compliance — layered on the Model Context Protocol (MCP).*

ATP is a sibling protocol to MCP, designed for the full surface of agent telephony. Where MCP defines how AI agents discover and call tools, ATP defines how they place and receive calls, send and receive messages on every modern channel, assert verifiable identity, transfer conversations across runtimes, and produce auditable evidence for compliance regimes (EU AI Act, HIPAA, NIST AI RMF, ISO 42001, SOC2-AI).

The protocol is structured as **35 numbered RFCs** ([ATP-1 through ATP-35](./ATP-SPEC.md#3-canonical-rfc-list)) modeled on IETF practice. Each RFC is narrow, independently authorable, and ships in one of four phases. The full series is being developed openly here, with the goal of submission to the Linux Foundation [AAIF (Agentic AI Industry Foundation)](https://www.linuxfoundation.org/) in Q4 2026.

---

## Status

**Draft v0.1** — May 2026. Pre-working-group. Editor: [AgentLyne](https://agentlyne.ai).

This repository hosts the canonical specification documents and reference materials. Working group adoption at AAIF is targeted for late 2026, with a candidate submission package consisting of ATP-15 (Conversation Resource), ATP-22 (Know Your Agent Attestation), and ATP-28 (Transfer Envelope).

---

## What's in this repo

| Document | Purpose | Phase |
|---|---|---|
| [`ATP-SPEC.md`](./ATP-SPEC.md) | Canonical overview of the full 35-RFC series — abstracts, scope, phase assignments, strategic logic | — |
| [`rfcs/RFC-0001.md`](./rfcs/RFC-0001.md) | **Architecture & Terminology** — roles, resource taxonomy, identifier formats, naming conventions, versioning policy | 0 |
| [`rfcs/RFC-0002.md`](./rfcs/RFC-0002.md) | **Sessions, Capability Negotiation, Transports** — extends MCP `initialize` with `atp_capabilities`; defines transports, auth, tenancy, session durability | 0 |
| [`rfcs/RFC-0003.md`](./rfcs/RFC-0003.md) | **Tool Catalog Conventions** — the standardized tool catalog (35+ tools), the start + subscribe async pattern, error envelope, idempotency, per-tier conformance | 0 |
| [`rfcs/RFC-0022.md`](./rfcs/RFC-0022.md) | **Know Your Agent (KYA) Attestation** — agent-operator identity envelope; CWT/COSE-encoded signed cert binding agent runtime identity to accountable real-world tenant | 2 |

The Phase 0 foundation (ATP-1, ATP-2, ATP-3) plus the Phase 2 headline KYA spec (ATP-22) are publicly published. Additional RFCs are drafted as the spec roadmap advances; see [`ATP-SPEC.md`](./ATP-SPEC.md) for the full list. Reference implementations in TypeScript, Python, and Go will ship under separate companion repositories.

---

## Why ATP

The Model Context Protocol (MCP), released by Anthropic in November 2024, became the universal standard for AI agent tool invocation. By May 2026, MCP carries ~97M monthly SDK downloads and ~17,000 published servers, with adoption by Anthropic, OpenAI, Google, Cursor, Zed, OpenHands, and the broader ecosystem.

MCP intentionally does not cover telephony. It carries no streaming bidirectional audio, no SMS or rich messaging, no number provisioning, no carrier identity (STIR/SHAKEN), no operator-level identity for AI agents, no auditable compliance evidence. Today, every voice agent platform (Vapi, Retell, Bland, ElevenLabs Conversational AI, Synthflow) closes that gap with proprietary, vendor-locked runtimes. The cost is that agents must live inside the vendor's runtime.

ATP fills the gap as a sibling layer to MCP. It is open, vendor-neutral, MCP-compatible, and designed for the full telephony surface from day one.

---

## The phased roadmap

| Phase | Timing | RFCs | Theme |
|---|---|---|---|
| **Phase 0** | 2026 Q2 | ATP-1, 2, 3 | Foundation + tool catalog conventions |
| **Phase 1** | 2026 Q3 | ATP-4..8, 10..14, 18..19, 23..25 | Voice + messaging core + audit log |
| **Phase 2** | 2026 Q4 — **AAIF submission window** | ATP-12, 15..17, 20..22, 26..29 | Rich messaging, cross-channel, identity (including KYA), evidence packs, transfer envelope |
| **Phase 3** | 2027 | ATP-9, 30..35 | Audio streaming, operational, conformance |

See [`ATP-SPEC.md`](./ATP-SPEC.md#5-phased-shipping-roadmap) for details.

---

## How ATP relates to existing standards

- **Layered on MCP.** ATP normatively references MCP 2025-03 (or later) for session initialization, capability negotiation, tool invocation, and resource reading. ATP adds telephony-specific resources (Number, Call, Message, Conversation, EvidencePack, Brand, KYACert), an event vocabulary (`call.*`, `message.*`, `conversation.*`), and the schemas for compliance evidence and identity attestation.
- **Complementary to STIR/SHAKEN.** STIR/SHAKEN (RFC 8224, 8225, 8226, 8588) attests to carrier-level number authorization. ATP-22 (KYA) attests to agent-operator identity. They ride alongside each other in the same SIP setup and answer different questions: SHAKEN says "the number is real and authorized"; KYA says "the AI agent calling from this number is operated by Acme Health LLC, persona Aria, active since May 2026."
- **CWT/COSE encoding.** ATP attestation envelopes are encoded as CBOR Web Tokens (RFC 8392) signed with COSE (RFC 9052), aligning with the IETF STIR working group's [`draft-ietf-stir-passport-cwt`](https://datatracker.ietf.org/wg/stir/) direction.
- **Compliance schemas.** ATP-26 (Evidence Pack Schema) maps to NIST AI Risk Management Framework, EU AI Act Article 12, HIPAA Security Rule (AI updates), ISO 42001, and SOC2-AI.

---

## Reading order for new contributors

1. **[`ATP-SPEC.md`](./ATP-SPEC.md)** — start here. Reads in ~30 minutes; covers the full series, design philosophy, phase plan, and strategic logic.
2. **[`rfcs/RFC-0022.md`](./rfcs/RFC-0022.md)** — the KYA RFC. The first full-detail draft published. Demonstrates the level of detail finished ATP RFCs should reach.
3. **Open issues and questions.** Each RFC ends with an "Open issues for review" section. These are the parts where contributor input is most welcome.

---

## How to contribute

This is an open spec. Contributions are welcomed from:

- AI agent platform vendors (Anthropic, OpenAI, Cursor, OpenHands, Windsurf, others)
- Telephony / CPaaS engineers (Twilio, Telnyx, Bandwidth, Plivo, Vonage, Sinch)
- Standards-track participants (IETF STIR working group, W3C, AAIF members)
- Open-source voice agent framework maintainers (Pipecat, LiveKit Agents, others)
- Compliance, legal, and regulatory voices (especially around EU AI Act Article 12, HIPAA, FCC AI-call rulemaking)
- Independent researchers and security reviewers

To contribute:

- **Open an issue** on GitHub for discussion, design questions, or bug reports against the spec drafts.
- **Open a pull request** for substantive changes. Please include the RFC number you're touching in the PR title (e.g., `[ATP-22] Tighten verification algorithm`).
- **Submit a new RFC.** RFCs ATP-1 through ATP-35 are reserved per [`ATP-SPEC.md`](./ATP-SPEC.md). RFCs ATP-36+ can be proposed for additional capabilities; open an issue first to discuss scope.

The editor and maintainer of this repository is AgentLyne (the company), reachable at the email and contact channels published at [agentlyne.ai](https://agentlyne.ai). All substantive contributions are made under the [LICENSE](./LICENSE).

---

## Reference implementation

The reference implementation of ATP — including the canonical hosted gateway, MCP server, KYA issuer service, and SDKs in TypeScript and Python — is operated by AgentLyne at [agentlyne.ai](https://agentlyne.ai). The protocol itself is open; the hosted operation is commercial. The same model Anthropic uses for MCP-the-spec versus Claude-the-product.

---

## Citation

If you reference ATP in academic, regulatory, or commercial work, please cite as:

> *Agent Telephony Protocol (ATP), Draft v0.1.* Edited by AgentLyne. May 2026. https://github.com/AgentLyne/Agent-telephony-Protocol

Individual RFCs are citable as `ATP-22` (etc.) until permanent IETF or AAIF identifiers are assigned.

---

## License

The specification documents in this repository are licensed under the MIT License — see [LICENSE](./LICENSE). Reference implementations will be released under MIT or Apache 2.0 in separate companion repositories.
