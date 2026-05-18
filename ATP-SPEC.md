# Agent Telephony Protocol (ATP) — Specification

**Status:** Draft v0.1
**Editor:** AgentLyne
**Date:** May 2026
**Target submission:** Linux Foundation AAIF (Agentic AI Industry Foundation), Q4 2026

> ATP is an open protocol for AI agents to discover, place, receive, observe, transfer, and audit telephony — voice calls, SMS, MMS, RCS, iMessage Business, WhatsApp Business, Google Business Messages, and the identity, attestation, and compliance metadata that ride alongside them. ATP is a sibling protocol to the Model Context Protocol (MCP); ATP normatively references MCP for its base session and tool-invocation semantics, and extends them with the resources, events, schemas, and conformance criteria that telephony specifically requires.

---

## Table of Contents

0. [Introduction](#0-introduction)
1. [Relationship to MCP and design philosophy](#1-relationship-to-mcp-and-design-philosophy)
2. [Architecture overview](#2-architecture-overview)
3. [Canonical RFC list](#3-canonical-rfc-list)
4. [RFC details](#4-rfc-details)
   - [Foundation: ATP-1, ATP-2, ATP-3](#foundation)
   - [Voice: ATP-4 to ATP-9](#voice)
   - [Messaging: ATP-10 to ATP-14](#messaging)
   - [Cross-channel: ATP-15 to ATP-17](#cross-channel-conversation)
   - [Identity & provisioning: ATP-18 to ATP-22](#identity--provisioning)
   - [Compliance & evidence: ATP-23 to ATP-27](#compliance--evidence)
   - [Multi-agent: ATP-28, ATP-29](#multi-agent)
   - [Operational: ATP-30 to ATP-32](#operational)
   - [Conformance: ATP-33 to ATP-35](#conformance)
5. [Phased shipping roadmap](#5-phased-shipping-roadmap)
6. [Strategic logic](#6-strategic-logic)
7. [Open questions and known unknowns](#7-open-questions-and-known-unknowns)
8. [Glossary](#8-glossary)
9. [Contribution and governance model](#9-contribution-and-governance-model)

---

## 0. Introduction

The Model Context Protocol (MCP, Anthropic, November 2024) became the universal standard for AI agent tool invocation: ~17,000 servers and ~97M monthly SDK downloads as of May 2026, adopted by Anthropic, OpenAI, Google, Cursor, Zed, OpenHands, and the broader ecosystem. MCP defines how an agent discovers and invokes tools over stdio, HTTP, and HTTP+SSE transports, including content blobs for text, image, and audio file payloads.

MCP does **not** define how an agent participates in telephony: real-time bidirectional voice, SMS/MMS/RCS messaging, iMessage Business, WhatsApp Business, Google Business Messages, voicemail, number provisioning, caller identity, attestation, or compliance evidence. Today, every voice agent platform reinvents this layer behind a proprietary surface (Vapi, Retell, Bland, ElevenLabs Conversational AI), and the dozen community "voice MCP" repositories on GitHub each define their own incompatible tool shapes.

The Agent Telephony Protocol (ATP) closes that gap. It is a protocol for the entire telephony surface — voice, every message channel, identity, attestation, and compliance — designed from first principles for AI agents and the gateways that broker between agents and the public switched telephone network (PSTN). ATP is layered on MCP (it normatively references MCP 2025-03 or later for sessions and tool invocation) and adds the resources, events, schema, and operational primitives that MCP intentionally omits.

ATP is structured as a series of **35 numbered RFCs**, each narrow and independently authorable, grouped into eight functional blocks (Foundation, Voice, Messaging, Cross-channel, Identity, Compliance, Multi-agent, Operational) and a Conformance block. The series is modeled on IETF practice; the structure is intentional, so that working-group co-authors can edit individual RFCs in parallel, and so that adopters can implement subsets at well-defined conformance tiers without ambiguity.

This document is the canonical reference for ATP. It is a living draft. Each section below contains the abstract and scope of one RFC; finalized RFCs are published as separate documents under the ATP repository.

---

## 1. Relationship to MCP and design philosophy

ATP is a **sibling protocol** to MCP, not a replacement. The design rule of thumb:

| Use MCP for... | Use ATP for... |
|---|---|
| Discovering tools an agent can call | Discovering telephony capabilities a gateway exposes |
| Invoking a tool synchronously | Placing a call (long-lived, event-driven) |
| Reading a resource (file, blob, URL) | Reading a `Number`, `Call`, `Message`, `Conversation`, `EvidencePack` resource |
| Returning text/image/audio-file blobs | Streaming call events, message receipts, attestation envelopes |
| Sampling from a language model | Routing an LLM into a voice call as the responding speaker |

Concretely, ATP relies on MCP for:

- **Session initialization and capability negotiation** (MCP `initialize` extended with ATP `capabilities` block)
- **Tool discovery and tool calling** (MCP `tools/list`, `tools/call` for ATP's standardized tool catalog)
- **Resource discovery and reading** (MCP `resources/list`, `resources/read` for ATP resources)
- **Notifications and SSE streaming** (MCP `notifications/*` for ATP event streams)

And adds:

- A standardized tool catalog with telephony semantics (ATP-3)
- Resource types: `Number`, `Call`, `Message`, `Conversation`, `EvidencePack`, `Brand`, `Agent`, `KYACert`
- A call-events vocabulary that streams over MCP notifications (ATP-5)
- Identity envelopes that ride inside MCP tool calls and events (ATP-21, ATP-22)
- A compliance evidence schema that maps to NIST AI RMF, EU AI Act, HIPAA, ISO 42001, SOC2-AI (ATP-26)
- A transfer envelope for cross-runtime handoff of live calls (ATP-28)
- A binary audio streaming extension for true bidirectional voice (ATP-9, future)

**Design principles**

1. **Agent as customer.** Every ATP capability is discoverable and usable by an agent without human intervention. Human consoles, where present, are projections of the agent-readable view, not the reverse.
2. **Channel-neutral where possible.** Messaging primitives operate on channel-abstract resources; channel-specific richness is a layered capability.
3. **Conformance tiers, not all-or-nothing.** Gateways advertise the ATP capabilities they implement. A small voicemail-bot server and a multi-CPaaS regulated gateway can both be "ATP-compatible" at different tiers.
4. **Audit-grade by default.** Every call and message produces a hash-chained, tenant-signed audit record. Evidence packs are first-class resources, not after-the-fact exports.
5. **Identity layered, not monolithic.** Carrier attestation (STIR/SHAKEN), display identity (RCD, branded sender), and agent operator identity (KYA) are separate envelopes that ride alongside calls and messages.
6. **Standards before features.** Every ATP feature has a written RFC. The implementation follows the spec; the spec is not retrofitted to whatever shipped first.

---

## 2. Architecture overview

```
┌──────────────────────────────────────────────────────────────────┐
│  AGENT (Anthropic / OpenAI / Cursor / Claude Code / custom)      │
│  Speaks MCP. Speaks ATP as a layered extension on MCP.           │
└─────────────────────────┬────────────────────────────────────────┘
                          │ MCP session + ATP extensions
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  ATP GATEWAY (reference implementation: AgentLyne)               │
│  - Implements ATP tool catalog, events, resources, schemas       │
│  - Brokers between agent and PSTN via one or more CPaaS          │
│  - Owns: call sessions, message inboxes, audit log, evidence,    │
│    knowledge graph, KYA cert issuance, brand registry mapping    │
└────┬─────────────────┬─────────────────┬─────────────────────────┘
     │                 │                 │
     ▼                 ▼                 ▼
┌────────────┐   ┌────────────┐   ┌────────────────────────────────┐
│ Carrier A  │   │ Carrier B  │   │ Rich messaging API providers   │
│ (Telnyx)   │   │ (Twilio)   │   │ (Meta WABA, Google RBM,        │
│ SIP / SMS  │   │ SIP / SMS  │   │  Apple Messages for Business)  │
└────────────┘   └────────────┘   └────────────────────────────────┘
     │                 │                 │
     ▼                 ▼                 ▼
                  PUBLIC SWITCHED TELEPHONE NETWORK
                  + RCS / iMessage / WhatsApp / GBM
```

**Roles**

- **Agent.** Any MCP-speaking program. Plays the role of LLM brain for calls and messages. Discovers ATP capabilities at session init.
- **ATP Gateway.** The persistent broker. Implements ATP. Owns call state, message inboxes, audit logs, and credentials with CPaaS providers and rich-messaging vendors.
- **Carrier / CPaaS.** Telephony providers (Twilio, Telnyx, Bandwidth, Plivo, Vonage, Sinch). Provide SIP termination, SMS, number rental.
- **Rich messaging vendors.** Meta (WhatsApp Business), Google (RCS/RBM, GBM), Apple (Messages for Business). Each requires brand approval and per-channel registration.

**The two-layer agent loop (architectural pattern, not a normative requirement of ATP)**

For inbound calls, ATP supports — but does not require — a two-layer model:

- **Fast Driver:** a hosted LLM running inside the gateway that handles the live conversation in real time with low latency.
- **Smart Observer:** the customer's own agent, connected via MCP, subscribed to call events; can `observe`, `inject` text, or `takeover` the conversation.

This pattern is documented in ATP-4 (Call lifecycle), ATP-5 (Call events), and ATP-6 (Voice tool semantics). It is the reference architecture; gateways MAY implement single-layer or multi-layer arrangements as long as they expose the ATP-required tool surface.

---

## 3. Canonical RFC list

| # | Title | Phase |
|---|---|---|
| **Foundation** | | |
| ATP-1 | Architecture & Terminology | 0 |
| ATP-2 | Sessions, Capability Negotiation, Transports | 0 |
| ATP-3 | Tool Catalog Conventions | 0 |
| **Voice** | | |
| ATP-4 | Call Resource & Lifecycle States | 1 |
| ATP-5 | Call Events Vocabulary | 1 |
| ATP-6 | Voice Tool Semantics | 1 |
| ATP-7 | Voice Tier Abstraction | 1 |
| ATP-8 | Voicemail | 1 |
| ATP-9 | Audio Streaming Frames | 3 |
| **Messaging** | | |
| ATP-10 | Message Resource & Channel Abstraction | 1 |
| ATP-11 | Send & Receive Primitives + Delivery Receipts | 1 |
| ATP-12 | Rich Content Schema | 2 |
| ATP-13 | Channel Fallback Rules | 1 |
| ATP-14 | Threading & Conversation IDs | 1 |
| **Cross-channel Conversation** | | |
| ATP-15 | Conversation Resource | 2 |
| ATP-16 | Channel Switching Mid-conversation | 2 |
| ATP-17 | Multi-modal Handoff | 2 |
| **Identity & Provisioning** | | |
| ATP-18 | Number Resources & Capability Discovery | 1 |
| ATP-19 | Provisioning, Porting, Release | 1 |
| ATP-20 | Brand Verification & RCD | 2 |
| ATP-21 | STIR/SHAKEN Attestation Envelope | 2 |
| **ATP-22** | **Know Your Agent (KYA) Attestation** | **2** |
| **Compliance & Evidence** | | |
| ATP-23 | Consent Receipt Envelope | 1 |
| ATP-24 | Recording Disclosure Metadata | 1 |
| ATP-25 | Audit Log Format | 1 |
| ATP-26 | Evidence Pack Schema | 2 |
| ATP-27 | Jurisdiction Tagging & Quiet Hours | 2 |
| **Multi-agent** | | |
| ATP-28 | Transfer Envelope | 2 |
| ATP-29 | Agent-to-Agent Handoff Semantics | 2 |
| **Operational** | | |
| ATP-30 | Pricing & Quote Discovery | 3 |
| ATP-31 | Quotas, Budgets, Rate Limiting | 3 |
| ATP-32 | Health & Observability Resources | 3 |
| **Conformance** | | |
| ATP-33 | Required vs Optional Capabilities | 3 |
| ATP-34 | Conformance Test Suite | 3 |
| ATP-35 | Compliance Tiers | 3 |

---

## 4. RFC details

### Foundation

#### ATP-1 — Architecture & Terminology

**Status:** Phase 0. **Depends on:** MCP base spec.

**Abstract.** Defines the canonical roles (Agent, Gateway, Carrier, Rich Messaging Vendor), the data model (Tenant, Agent, Number, Call, Message, Conversation, EvidencePack, Brand, KYACert), and the terminology used by every subsequent RFC. Establishes naming conventions (snake_case for tools and events; PascalCase for resource type names; ISO 8601 for timestamps; E.164 for phone numbers; ISO 3166 for countries; BCP-47 for locales).

**Scope.** Glossary, role definitions, the resource-type taxonomy, identifier formats, version numbering, and conformance language conventions (MUST / SHOULD / MAY per RFC 2119).

**Non-goals.** Wire formats (deferred to ATP-2), tool semantics (ATP-3), or any specific telephony behavior.

**Key concepts.** *Tenant* (the legally accountable customer of a Gateway), *Agent* (a distinct AI persona configured under a Tenant), *Number* (an E.164 phone number with capabilities), *Call* (a voice session), *Message* (a single message on any channel), *Conversation* (a logical thread across channels), *EvidencePack* (signed audit bundle).

**Notes.** This is intentionally short — 6-10 pages of definitions. The terminology lock-in is more important than the depth; every other RFC cites it.

---

#### ATP-2 — Sessions, Capability Negotiation, Transports

**Status:** Phase 0. **Depends on:** ATP-1, MCP base.

**Abstract.** Specifies how an ATP-speaking agent and an ATP gateway negotiate which ATP capabilities the gateway supports at session initialization. Extends MCP's `initialize` request with an `atp_capabilities` block and standardizes the transport modes ATP uses (stdio, HTTP, HTTP+SSE, WebSocket).

**Scope.** Capability advertisement (which channels, tiers, RFCs the gateway implements), version negotiation (`atp_version: "0.1"`), authentication conventions (bearer tokens, tenant keys), transport selection (when SSE vs WebSocket is required by which capability), session lifecycle (init, ready, drain, terminate).

**Key schema sketch:**

```json
{
  "atp_capabilities": {
    "atp_version": "0.1",
    "channels": ["voice", "sms", "mms", "rcs", "imessage_business", "whatsapp", "gbm"],
    "voice_tiers": ["economy", "standard", "premium"],
    "rfcs_implemented": [1, 2, 3, 4, 5, 6, 7, 10, 11, 18, 19, 23, 24, 25],
    "conformance_tier": "basic",
    "evidence_pack_schemas": ["nist_ai_rmf_v1", "eu_ai_act_art12_v1"],
    "max_concurrent_calls": 1000
  }
}
```

**Non-goals.** Wire encryption (use TLS; out of scope), HTTP-level auth flows beyond bearer tokens (refer to upstream auth specs).

**Notes.** Capability advertisement is what makes ATP work across heterogeneous gateways. A community-built stdio voicemail server with capabilities `["sms"]` and RFCs `[1, 2, 3, 10, 11]` is "ATP-compatible at basic tier." A regulated multi-CPaaS gateway with the full feature set is "ATP-compatible at advanced tier." Same protocol, different conformance.

---

#### ATP-3 — Tool Catalog Conventions

**Status:** Phase 0. **Depends on:** ATP-1, ATP-2.

**Abstract.** The standardized tool catalog: tool names, argument shapes (JSON Schema), return types, error semantics. This is the single most-cited RFC because it is the surface every agent code-paths against. ATP-3 specifies the *interface*; individual tool *semantics* are detailed in their channel-specific RFCs (ATP-6 for voice tools, ATP-11 for messaging tools, ATP-19 for number tools, etc.).

**Scope.** Tool naming (`phone.*`, `sms.*`, `messages.*`, `calls.*`, `numbers.*`, `conversations.*`, `brands.*`, `evidence.*`, `kya.*`, `pricing.*`), argument JSON Schemas, return value contracts, error code enumeration.

**The canonical tool set:**

```
phone.dial             { to, intent, voice_tier?, mandate_id?, max_duration_sec? }
phone.transfer         { call_id, to, mode: warm|cold|blind }
phone.hangup           { call_id, reason }
phone.hold             { call_id, hold: bool }
phone.dtmf             { call_id, digits }

calls.list             { since?, status_filter?, limit? }
calls.get              { call_id }
calls.tail             { call_id, follow: bool, since_seq? }
calls.send_message     { call_id, text, mode: tts|whisper }
calls.takeover         { call_id, handoff_envelope_id? }

messages.send          { from, to, body, channel?, media_url[]?, rich_content? }
messages.list          { number, since?, contains_pattern?, channel? }
messages.wait_for      { number, pattern, timeout_sec }
sms.send               { from, to, body }    # convenience wrapper
otp.wait_for           { number, pattern, source_pattern?, timeout_sec }

conversations.list     { tenant?, participant? }
conversations.get      { conversation_id }
conversations.tail     { conversation_id, follow: bool }

numbers.provision      { country, area_code?, capabilities[], use_case, tier? }
numbers.list           { tenant_id? }
numbers.release        { number_id }
numbers.port_in        { number, current_carrier_credentials }

brands.register        { legal_name, ein, address, vertical, contact }
brands.get             { brand_id }
kya.issue              { agent_id, persona, tenant_kyc_level, expires_at }
kya.verify             { kya_cert }
kya.revoke             { kya_cert_id, reason }

evidence.export        { call_id|conversation_id|date_range, schema }

pricing.quote          { spec }
account.get            { }
account.health         { }
usage.get              { period }
```

**Notes.** Every gateway implements the tools whose RFCs it conforms to. Discovery happens via standard MCP `tools/list`. Tools not implemented MUST NOT appear in the tool list (rather than throwing on call).

---

### Voice

#### ATP-4 — Call Resource & Lifecycle States

**Status:** Phase 1. **Depends on:** ATP-1, ATP-3.

**Abstract.** Defines the `Call` resource — the canonical representation of one voice session from setup through teardown — and the state machine that governs its progression. Specifies the required fields, the legal state transitions, and the timestamps and provenance metadata.

**State machine:**

```
            ┌─────────┐
            │ queued  │
            └────┬────┘
                 ▼
            ┌─────────┐
            │ dialing │
            └────┬────┘
        ┌───────┼─────────┐
        ▼       ▼         ▼
   ┌────────┐ ┌──────┐ ┌────────┐
   │ringing │ │ busy │ │noanswer│  (terminal)
   └───┬────┘ └──────┘ └────────┘
       ▼
   ┌─────────────┐
   │ in_progress │
   └──┬────────┬─┘
      ▼        ▼
   ┌──────┐ ┌───────────┐
   │ ended│ │transferred│  (terminal)
   └──────┘ └───────────┘
   ┌────────┐
   │rejected│ (terminal, set at gateway pre-dial for compliance failures)
   └────────┘
```

**Call resource (sketch):**

```json
{
  "call_id": "call_abc123",
  "tenant_id": "tnt_acme",
  "agent_id": "agt_intake",
  "direction": "inbound|outbound",
  "from": "+14155550987",
  "to": "+14155550123",
  "status": "in_progress",
  "started_at": "2026-05-14T22:31:09Z",
  "answered_at": "2026-05-14T22:31:14Z",
  "ended_at": null,
  "duration_sec": null,
  "voice_tier": "standard",
  "recording_state": "recording_with_disclosure|recording_silent|not_recording",
  "stir_shaken_attestation": "A|B|C|gateway-only",
  "kya_cert_id": "kya_xyz",
  "rejection_reason": null,
  "transferred_to": null,
  "metadata": {}
}
```

**Notes.** `rejected` is set pre-dial when compliance checks fail (TCPA opt-out, invalid consent, quiet hours, etc.). This is intentional so that a single `Call` resource exists for every dispatch attempt, including the ones that never reached the carrier.

---

#### ATP-5 — Call Events Vocabulary

**Status:** Phase 1. **Depends on:** ATP-2, ATP-4.

**Abstract.** Defines the event stream that ATP gateways emit per `Call` resource, consumed by agents via `calls.tail` or `notifications/calls/*`. Standardizes event names, payload shapes, sequence numbering, and delivery semantics (at-least-once with monotonic sequence numbers).

**Event vocabulary (canonical):**

| Event name | Payload (key fields) |
|---|---|
| `call.queued` | `call_id`, `dispatched_at` |
| `call.ringing` | `call_id`, `attestation?`, `from`, `to` |
| `call.answered` | `call_id`, `answered_at` |
| `call.user_speaking` | `call_id`, `started_at` |
| `call.user_partial` | `call_id`, `text`, `confidence` |
| `call.user_final` | `call_id`, `text`, `confidence`, `language` |
| `call.agent_speaking` | `call_id`, `text`, `started_at`, `tts_voice_id` |
| `call.tool_call` | `call_id`, `tool`, `args` |
| `call.tool_result` | `call_id`, `tool`, `result`, `duration_ms` |
| `call.dtmf` | `call_id`, `digits` |
| `call.hold` | `call_id`, `held: bool` |
| `call.transferred` | `call_id`, `to`, `mode` |
| `call.takeover_requested` | `call_id`, `by_agent_id` |
| `call.takeover_complete` | `call_id`, `by_agent_id`, `handoff_envelope_id` |
| `call.ended` | `call_id`, `reason`, `duration_sec` |
| `call.evidence_ready` | `call_id`, `evidence_pack_url`, `signature` |

**Notes.** Every event carries a strictly increasing `seq` number scoped to the `call_id` so consumers can detect gaps and resume after disconnect. Partial transcripts (`call.user_partial`) are best-effort; finals (`call.user_final`) MUST be reliably delivered.

---

#### ATP-6 — Voice Tool Semantics

**Status:** Phase 1. **Depends on:** ATP-3, ATP-4, ATP-5.

**Abstract.** Detailed behavioral semantics for the `phone.*` and `calls.*` tools defined in ATP-3. Covers preconditions, postconditions, error cases, and interaction with the call state machine. In particular, specifies the three Smart Observer intervention modes: `observe`, `inject`, `takeover`.

**Scope.** Behavior of `phone.dial`, `phone.transfer` (warm vs cold vs blind semantics), `phone.hangup`, `phone.hold`, `phone.dtmf`, `calls.tail` (with `mode`), `calls.send_message`, `calls.takeover`.

**Intervention modes (the model that distinguishes ATP from prior voice frameworks):**

- **`observe`** — read-only event subscription. The Smart Observer agent receives events but cannot modify the call.
- **`inject`** — the Smart Observer can call `calls.send_message({ call_id, text })`. The gateway's Fast Driver (or the in-call LLM, however the gateway is architected) TTSes the injected text as the agent's next utterance and incorporates it into context for subsequent turns.
- **`takeover`** — the Smart Observer becomes the LLM-of-record for the remainder of the call. Requires ATP-9 (audio streaming) for true bidirectional audio, or in v1 of ATP, falls back to text-mode takeover where the Smart Observer sends text turns and the gateway TTSes them.

**Notes.** Takeover in pure-text mode is shippable in Phase 1; full audio takeover is Phase 3 (ATP-9). The mode is identical from the protocol viewpoint; only the transport changes.

---

#### ATP-7 — Voice Tier Abstraction

**Status:** Phase 1. **Depends on:** ATP-2, ATP-6.

**Abstract.** Defines the named voice tiers (`economy`, `standard`, `premium`) and the contractual quality, latency, and cost envelopes a gateway commits to per tier. Hides specific provider names (Deepgram, Cartesia, ElevenLabs, etc.) behind tier names so that gateways can swap providers without breaking agent contracts.

**Tier contract sketch:**

| Tier | TTFT (TTS) | STT latency | Voice quality (MOS) | Indicative cost to gateway |
|---|---|---|---|---|
| `economy` | ≤ 250 ms | ≤ 150 ms | ≥ 3.8 | ~$0.015/min |
| `standard` | ≤ 150 ms | ≤ 100 ms | ≥ 4.3 | ~$0.065/min |
| `premium` | ≤ 100 ms | ≤ 80 ms | ≥ 4.6 | ~$0.12/min |

**Scope.** Tier naming, quality envelopes, voice id namespace (so a cloned voice survives provider swaps), language coverage requirements per tier, fallback behavior on tier-provider failure.

**Notes.** Gateways MAY add tiers (e.g., `enterprise`) but MUST implement at least `standard`. The economy/standard/premium naming is normative for cross-gateway compatibility.

---

#### ATP-8 — Voicemail

**Status:** Phase 1. **Depends on:** ATP-3, ATP-4, ATP-10.

**Abstract.** Specifies voicemail behavior: greeting management, inbound voicemail deposit, transcription, retrieval as an agent-readable resource, and the relationship between voicemails and the `Message` resource (voicemails are a special channel).

**Scope.** Greeting registration (TTS-rendered or human-recorded uploaded), inbound voicemail capture, transcription via the gateway's STT stack, voicemail resource representation, retention.

**Notes.** Voicemail is modeled as a `Message` on a `voicemail` channel rather than as a separate resource type — preserves cross-channel conversation semantics in ATP-15.

---

#### ATP-9 — Audio Streaming Frames

**Status:** Phase 3 (2027). **Depends on:** ATP-2 (transports), ATP-5, ATP-6.

**Abstract.** Bidirectional binary audio frames over WebSocket (or future MCP binary-frame extension). Required for true takeover mode where the Smart Observer becomes the in-the-loop LLM with sub-300ms turn latency. Specifies codec negotiation (opus, g711µ, g711a, PCM), framing, jitter buffering, sequence numbering, and recovery semantics.

**Scope.** Codec list, frame header format, sequence and timestamp fields, packet loss recovery, jitter buffer guidance, integration with MCP transports.

**Non-goals.** Echo cancellation, noise suppression, voice-activity detection — these are gateway-side concerns, not protocol concerns.

**Notes.** This is the hardest RFC in the series and the reason it's deferred to Phase 3. Doing it well requires WebRTC-grade attention to packet loss and jitter, and ideally coordination with the MCP base spec to add a `transport/binary-streaming` mode. Doing it badly creates the next decade's incompatibility mess. Better to wait.

---

### Messaging

#### ATP-10 — Message Resource & Channel Abstraction

**Status:** Phase 1. **Depends on:** ATP-1, ATP-3.

**Abstract.** Defines the channel-abstract `Message` resource and the `Channel` enumeration. Lets an agent send and receive messages across SMS, MMS, RCS, iMessage Business, WhatsApp Business, and Google Business Messages with a single resource shape, while still surfacing channel-specific capability flags for rich content.

**Channel enum:** `sms`, `mms`, `rcs`, `imessage_business`, `whatsapp`, `gbm`, `voicemail`, `email_fallback`.

**Message resource:**

```json
{
  "message_id": "msg_xyz",
  "conversation_id": "conv_abc",
  "tenant_id": "tnt_acme",
  "agent_id": "agt_intake",
  "number_id": "num_415",
  "direction": "inbound|outbound",
  "channel": "rcs",
  "channel_capabilities_used": ["rich_card", "quick_reply"],
  "from": "+14155550987",
  "to": "+14155550123",
  "body": "Your appointment is confirmed.",
  "rich_content": { /* per ATP-12 */ },
  "media": [],
  "status": "queued|sending|sent|delivered|read|failed",
  "delivery_attempts": [ /* per ATP-13 fallback rules */ ],
  "received_at": "2026-05-14T22:32:00Z",
  "sent_at": "...",
  "delivered_at": "...",
  "read_at": "...",
  "metadata": {}
}
```

**Notes.** Channel-specific capabilities (e.g., "can this number receive iMessages?") are advertised via `Number.capabilities` (ATP-18). The Message resource itself is uniform.

---

#### ATP-11 — Send & Receive Primitives + Delivery Receipts

**Status:** Phase 1. **Depends on:** ATP-10.

**Abstract.** Defines `messages.send`, `messages.list`, `messages.wait_for`, and the receipt event stream. Specifies how delivery receipts (sent / delivered / read / failed) flow back as events on a subscribed conversation or message.

**Scope.** Send arguments (including channel preference, fallback policy), inbound receive semantics (event `message.received` on the conversation stream), receipt events (`message.delivered`, `message.read`, `message.failed`), long-polling semantics for `messages.wait_for` (used by `otp.wait_for`).

**Notes.** `otp.wait_for` is a thin convenience wrapper over `messages.wait_for` with a pre-set regex hint. Both are defined here; `otp.*` lives in this RFC, not a separate one.

---

#### ATP-12 — Rich Content Schema

**Status:** Phase 2. **Depends on:** ATP-10.

**Abstract.** Channel-portable schema for rich messaging content — cards, list pickers, quick replies, buttons, media attachments, suggested actions. Maps each abstract content type onto each channel's native rich format (RCS rich card → Google's RBM schema; iMessage list picker → Apple Messages for Business schema; WhatsApp interactive message → Meta's WABA schema). Specifies what gracefully degrades on channels that don't support a given content type.

**Scope.** Content type taxonomy, per-channel mapping table, fallback strategies (e.g., RCS card → SMS plaintext summary).

**Notes.** This is where ATP starts to actually compete with proprietary CPaaS rich-messaging APIs. Authoring the channel-portable abstraction is itself novel and is one of the AAIF submission's value props.

---

#### ATP-13 — Channel Fallback Rules

**Status:** Phase 1. **Depends on:** ATP-10, ATP-11.

**Abstract.** Declarative rules for cross-channel fallback (RCS → SMS if not delivered within N seconds; iMessage → SMS if recipient is not Apple). Lets the agent specify intent ("reach this number with this content") and the gateway picks the channel and handles fallback.

**Fallback policy sketch:**

```json
{
  "preferred_channels": ["rcs", "imessage_business", "sms"],
  "fallback_timeout_sec": 30,
  "fallback_on": ["not_delivered", "not_supported"]
}
```

**Notes.** Channel negotiation is the messaging equivalent of voice-tier abstraction — hides provider/channel complexity behind a declarative intent.

---

#### ATP-14 — Threading & Conversation IDs

**Status:** Phase 1. **Depends on:** ATP-10.

**Abstract.** Specifies how individual messages group into threaded conversations. Each `Message` carries a `conversation_id`; messages with the same `(tenant_id, number_id, peer_number)` tuple default to the same conversation unless explicitly broken.

**Scope.** Conversation ID assignment rules, thread-break semantics ("start a new conversation for this campaign"), conversation lookup by participant.

**Notes.** Pure threading lives here; the richer cross-channel `Conversation` resource that spans voice + every message channel is ATP-15.

---

### Cross-channel Conversation

#### ATP-15 — Conversation Resource

**Status:** Phase 2. **Depends on:** ATP-4, ATP-10, ATP-14.

**Abstract.** Defines the `Conversation` resource — a logical thread that spans one or more `Call`s and zero or more `Message`s across any subset of channels. Lets an agent reason about "the ongoing relationship with this person" rather than channel-isolated events.

**Conversation resource:**

```json
{
  "conversation_id": "conv_abc",
  "tenant_id": "tnt_acme",
  "agent_id": "agt_intake",
  "participants": [
    { "number": "+14155550123", "role": "agent" },
    { "number": "+14155550987", "role": "human" }
  ],
  "first_event_at": "2026-05-12T10:00:00Z",
  "last_event_at": "2026-05-14T22:32:00Z",
  "event_count": 47,
  "channels_used": ["voice", "sms", "rcs"],
  "summary": "Appointment scheduling for May 15 dental cleaning",
  "open": true
}
```

**Notes.** This is one of the three headline AAIF submissions. CPaaS vendors think in per-channel inboxes; agents need to think in cross-channel relationships. Standardizing the Conversation resource is the protocol-level shift that makes ATP genuinely agent-native.

---

#### ATP-16 — Channel Switching Mid-conversation

**Status:** Phase 2. **Depends on:** ATP-15.

**Abstract.** Specifies the semantics of switching channels within a conversation. "Text me about that instead" or "call me back about this" become first-class operations the agent can invoke without losing thread state.

**Scope.** Channel switch tools (`conversations.switch_channel`), continuity requirements (the new channel inherits transcript context), capability checks (is the target channel reachable for this peer?).

**Notes.** Particularly important for the AI persona / creator GTM, where a fan starts on SMS but wants to escalate to a voice call ("can you actually call me?"). ATP-16 makes that one tool call.

---

#### ATP-17 — Multi-modal Handoff

**Status:** Phase 2. **Depends on:** ATP-15, ATP-16.

**Abstract.** Specifies automatic cross-modal follow-up: a call ends → an SMS confirmation is auto-generated and sent; an inbound SMS triggers a callback; a missed call generates a voicemail-to-SMS notification. Declarative rules at the gateway, agent-controllable.

**Scope.** Handoff rule format, trigger events, content templating, opt-out behavior.

**Notes.** This is mostly orchestration — but standardizing it means an agent moving between gateways gets the same handoff behavior, instead of every gateway reinventing post-call SMS flows differently.

---

### Identity & Provisioning

#### ATP-18 — Number Resources & Capability Discovery

**Status:** Phase 1. **Depends on:** ATP-1.

**Abstract.** Defines the `Number` resource — its capabilities (voice, SMS, MMS, RCS, iMessage Business reachability, WhatsApp connection status), jurisdiction, ownership chain, and the metadata needed for agents to choose the right number for a task.

**Number resource:**

```json
{
  "number_id": "num_415",
  "e164": "+14155550123",
  "tenant_id": "tnt_acme",
  "country": "US",
  "region": "CA",
  "type": "local|toll_free|mobile|short_code",
  "capabilities": {
    "voice_inbound": true,
    "voice_outbound": true,
    "sms_inbound": true,
    "sms_outbound": true,
    "mms": true,
    "rcs_brand": "brand_xyz",
    "imessage_business": false,
    "whatsapp_waba": null,
    "stir_shaken_level": "A"
  },
  "campaign_id": "tcr_campaign_qwe",
  "monthly_cost_usd": 1.50,
  "provisioned_at": "2026-05-14T22:31:09Z"
}
```

**Notes.** `Number` is a long-lived resource; calls and messages reference numbers by `number_id`. Numbers can be released (returned to pool) or transferred between tenants.

---

#### ATP-19 — Provisioning, Porting, Release

**Status:** Phase 1. **Depends on:** ATP-18.

**Abstract.** Specifies the operations for acquiring numbers programmatically (`numbers.provision`), porting in numbers the tenant already owns (`numbers.port_in`), and releasing numbers (`numbers.release`). Defines the linkage between provisioning and 10DLC/RCS/WABA brand registration.

**Scope.** Provisioning request schema, pool semantics, port-in workflow (LOA upload, port date scheduling), release semantics, hold-or-release timing.

**Notes.** The standardized interface lets an agent provision a number without knowing whether the underlying carrier is Twilio, Telnyx, Bandwidth, or other. The gateway is responsible for routing the request to the cheapest qualified CPaaS for the requested capabilities.

---

#### ATP-20 — Brand Verification & RCD

**Status:** Phase 2. **Depends on:** ATP-18.

**Abstract.** Specifies Brand resource (the verified business identity used for display) and Rich Call Data (RCD) — the display name + logo + verification chain that appears on a recipient's iPhone or Android during an inbound call. Defines how the gateway registers and renews brand verifications with Numeracle, Bandwidth, Apple, Google, Meta, and other identity authorities.

**Scope.** Brand resource schema, RCD verification chain, branded caller registration flow, branded sender registration flow (RBM, iMessage Business).

**Notes.** RCD is what makes "Aria from Acme Health" show up instead of "+14155550123" on the recipient's screen. It is the display-layer identity that complements the carrier-level identity (ATP-21) and the agent-level identity (ATP-22).

---

#### ATP-21 — STIR/SHAKEN Attestation Envelope

**Status:** Phase 2. **Depends on:** ATP-4, ATP-18.

**Abstract.** Specifies how STIR/SHAKEN attestation tokens (the IETF-defined PASSporT identity headers carried in SIP INVITE) are surfaced in ATP — both on inbound calls (so agents can see the attestation level of the caller) and on outbound calls (so the gateway's signing chain is transparent). Standardizes the attestation envelope format that rides inside `call.ringing` events and is exposed as a field on the `Call` resource.

**Scope.** Envelope schema (attestation level A/B/C, signing authority, orig-id), inbound attestation propagation, outbound attestation request flow, integration with carrier-level STI-AS (Service Authorization Service).

**Notes.** ATP doesn't try to be a STIR/SHAKEN replacement; STIR/SHAKEN is the carrier-level standard. ATP-21 standardizes how the agent layer can read and reason about STIR/SHAKEN attestations. Pairs with ATP-22 (KYA) which adds the agent-operator layer of identity.

---

#### ATP-22 — Know Your Agent (KYA) Attestation

**Status:** Phase 2. **Depends on:** ATP-1, ATP-21, ATP-25.

**Abstract.** Defines the Know Your Agent (KYA) attestation envelope — a signed certificate that binds an AI agent's runtime identity to an accountable real-world entity, an operating history, and a reputation score. Where STIR/SHAKEN (ATP-21) attests to *carrier-level* caller identity, KYA attests to *agent-level* operator identity. KYA certs are issued by KYA issuers (initially ATP gateways themselves; future: independent KYA authorities), travel alongside calls and messages, and are verifiable by recipients, recipient carriers, and regulators.

**Motivation.** As FCC AI-call rules, EU AI Act enforcement, and state-level AI disclosure regulations come into force through 2026-2027, recipients and recipient carriers will need a verifiable answer to "who is the AI agent that's calling me, and who's accountable for it?" No standard exists for this as of May 2026. ATP-22 is the first credible candidate.

**KYA cert (sketch):**

```json
{
  "kya_attestation_v1": {
    "agent_id": "agt_aria",
    "agent_display_name": "Aria",
    "agent_persona_hash": "sha256:1b3c...",
    "tenant_legal_entity": "Acme Health LLC",
    "tenant_jurisdiction": "US-DE",
    "tenant_kyc_level": "verified_business",
    "tenant_kyc_method": "stripe_identity_v2",
    "agent_first_active": "2026-05-14",
    "agent_reputation_score": 0.94,
    "agent_reputation_sample_size": 1287,
    "issuer": "agentlyne.ai",
    "issuer_root_cert": "did:web:agentlyne.ai#kya-root",
    "issued_at": 1747234567,
    "expires_at": 1747320967,
    "signature": "..."
  }
}
```

**Scope.** Cert format and signing algorithm (Ed25519 with a published issuer DID), issuer root-of-trust model, reputation score schema and computation guidance (non-normative), revocation list endpoint, integration points (call setup, SMS headers, RCS branded sender metadata).

**Verification flow.**

1. Recipient gateway receives a call or message.
2. Extracts the KYA envelope from the protocol metadata.
3. Resolves `issuer_root_cert` via DID or HTTPS to the issuer's published public key set.
4. Verifies the signature.
5. Checks expiry and revocation list.
6. Surfaces the verified attestation to the recipient agent / carrier / regulator.

**Non-goals.** Reputation algorithm details (issuers compete on this), federation of KYA issuers (Phase 3 deepening), monetization (out of scope).

**Notes.** This is one of the three headline AAIF submissions, alongside ATP-15 (Conversation Resource) and ATP-28 (Transfer Envelope). It is also the RFC most likely to be cited normatively by regulators because it solves a problem they are already asking about.

**Open questions.**

- Should KYA issuers be required to publish revocation feeds in a specific format (e.g., CRL-style)?
- Should ATP-22 define a "trusted issuers" registry, or defer that to a future RFC / external policy body?
- How does KYA interact with the EU AI Act's "high-risk AI system" registration — can a KYA cert assert an EU registration ID?

---

### Compliance & Evidence

#### ATP-23 — Consent Receipt Envelope

**Status:** Phase 1. **Depends on:** ATP-1.

**Abstract.** Specifies the consent receipt — cryptographic proof that the principal (the human on whose behalf the agent is calling or messaging) authorized this specific action. Defines the HMAC-signed token format, the bound fields (recipient, intent hash, principal id, TTL), and the verification flow.

**Scope.** Token format (header.payload.signature, base64url-encoded), bound fields, TTL semantics, replay-protection considerations, integration with `phone.dial` and `messages.send`.

**Notes.** This is the same shape as what was prototyped in the v0.1 scaffold under `src/compliance/consent.ts`. ATP-23 standardizes it as a protocol artifact.

---

#### ATP-24 — Recording Disclosure Metadata

**Status:** Phase 1. **Depends on:** ATP-4, ATP-18.

**Abstract.** Specifies how state-by-state and country-by-country recording disclosure requirements are surfaced in ATP. Lets the agent (and the gateway) reason about whether disclosure is required for a given call, what the disclosure clause is, and whether disclosure has been made.

**Scope.** Per-jurisdiction recording-consent classification (all-party vs one-party), disclosure clause templates, recording state field on `Call`, opt-out semantics if the recipient objects mid-call.

**Notes.** Pairs with ATP-27 (jurisdiction tagging).

---

#### ATP-25 — Audit Log Format

**Status:** Phase 1. **Depends on:** ATP-5, ATP-11.

**Abstract.** Specifies the hash-chained, tenant-signed audit log format that every ATP gateway maintains. Every call event, message event, tool call, and configuration change appends to the log with a Merkle-style hash chain. The final log root is signed with the tenant's key and exposed as an `EvidencePack` resource.

**Scope.** Log entry schema, hash chain construction, Merkle root computation, signing algorithm (Ed25519), retention requirements (gateway-configurable, with minimum recommendations per compliance tier).

**Notes.** This is the foundation that everything else in the compliance block builds on. Get the log format right and the rest is layering.

---

#### ATP-26 — Evidence Pack Schema

**Status:** Phase 2. **Depends on:** ATP-25.

**Abstract.** Specifies the `EvidencePack` resource — a bundle of audit log entries, transcripts, recording references, attestation envelopes, and structured metadata, formatted for direct submission to compliance auditors. Defines profile-specific schemas mapped to NIST AI RMF, EU AI Act Article 12, HIPAA Security Rule (AI updates), ISO 42001, and SOC2-AI.

**Scope.** Base `EvidencePack` schema, profile-specific schemas as JSON Schema documents, signing requirements, time-bound retrieval, redaction rules (PHI / PII / payment data).

**Notes.** This is where AgentLyne's compliance moat lives. Other gateways can implement ATP-25 (the log) but the auditor-ready profiles in ATP-26 are months of compliance-officer + legal work. Sponsored RFC, AgentLyne edits.

---

#### ATP-27 — Jurisdiction Tagging & Quiet Hours

**Status:** Phase 2. **Depends on:** ATP-18, ATP-24.

**Abstract.** Specifies how jurisdiction is inferred and carried for each call and message — for TCPA (US) quiet-hours enforcement, recording disclosure (per ATP-24), state-level AI disclosure laws (California, Texas as of 2026), and EU country-specific AI Act enforcement variations. Standardizes the opt-out registry interface for cross-tenant interoperability.

**Scope.** Jurisdiction inference rules (E.164 area code → US state, country code → country), explicit override mechanism, quiet-hours window definition, opt-out registry schema and query interface.

---

### Multi-agent

#### ATP-28 — Transfer Envelope

**Status:** Phase 2. **Depends on:** ATP-4, ATP-5, ATP-25.

**Abstract.** Defines the cross-runtime transfer envelope — the schema for "everything Agent A knows about this call so far" that Agent B (potentially on a different runtime, model lab, or gateway) can pick up cold. This is the spec that makes cross-runtime agent-to-agent handoff possible.

**Transfer envelope (sketch):**

```json
{
  "transfer_envelope_v1": {
    "call_id": "call_abc123",
    "transferred_at": "2026-10-15T14:23:09Z",
    "from_agent": {
      "agent_id": "agt_intake",
      "runtime": "anthropic",
      "model_hint": "claude-sonnet-4-6"
    },
    "to_agent": {
      "agent_id": "agt_billing_specialist",
      "runtime": "openai",
      "model_hint": "gpt-5-realtime"
    },
    "transcript": [ /* full ATP-5 event sequence so far */ ],
    "structured_state": { /* customer-defined: cart, account_id, etc. */ },
    "active_tools": ["billing.get_invoice", "billing.refund"],
    "call_metadata": { /* from, to, attestation, recording_state, jurisdiction */ },
    "kya_chain": [ /* KYA certs of both agents */ ],
    "audio_handoff_token": "opq:...",
    "signature": "..."
  }
}
```

**Scope.** Envelope schema, signing (tenant key + handoff timestamp), audio handoff token semantics (Phase 3 unlocks audio resumption), redaction (sensitive structured state can be marked encrypted-to-target).

**Notes.** Headline RFC for the AAIF submission. Co-author candidates: Anthropic agent team, OpenAI Agents team, Cursor.

---

#### ATP-29 — Agent-to-Agent Handoff Semantics

**Status:** Phase 2. **Depends on:** ATP-28.

**Abstract.** Specifies the semantics of warm, cold, and blind agent-to-agent transfers, both within the same call (ATP-28) and across separate calls (e.g., "Agent A schedules a follow-up call that Agent B will handle"). Defines how transcript context, structured state, KYA certs, and audit chains carry across the transfer.

**Scope.** Warm transfer (both agents online during handoff), cold transfer (Agent A leaves, Agent B picks up), blind transfer (Agent A drops without context handoff), forwarding across channels (the conversation continues but the agent of record changes).

---

### Operational

#### ATP-30 — Pricing & Quote Discovery

**Status:** Phase 3. **Depends on:** ATP-3, ATP-18.

**Abstract.** Specifies `pricing.quote` — a tool that returns a machine-readable cost estimate for a proposed call or message before it is dispatched. Lets agents budget, compare gateways, and respect tenant cost ceilings.

**Scope.** Quote request shape (call spec or message spec), quote response shape (cost breakdown by component: minutes, segments, attestation premiums, recording surcharges), validity TTL, currency.

---

#### ATP-31 — Quotas, Budgets, Rate Limiting

**Status:** Phase 3. **Depends on:** ATP-30.

**Abstract.** Standardizes per-agent and per-tenant quotas — concurrent calls, daily minute caps, monthly spend ceilings, rate-limit policies. Specifies the events agents subscribe to when a quota threshold is approached (e.g., 80% of monthly spend) and when a hard cap blocks an action.

**Scope.** Quota types, threshold events, hard-cap enforcement, override semantics (tenant-admin escalation).

---

#### ATP-32 — Health & Observability Resources

**Status:** Phase 3. **Depends on:** ATP-2.

**Abstract.** Defines `account.health` and the standardized observability resources — error rates, latency distributions per voice tier, message delivery success rates per channel, CPaaS provider health. Lets agents (and ops dashboards) reason about gateway operational state.

**Scope.** Health resource schema, metric naming conventions, status-page schema.

---

### Conformance

#### ATP-33 — Required vs Optional Capabilities

**Status:** Phase 3. **Depends on:** All earlier RFCs.

**Abstract.** Defines the MUST / SHOULD / MAY level for every ATP capability per conformance tier. Specifies what a gateway must implement to claim conformance at each tier.

**Tiers (defined in ATP-35):**

- **basic** — ATP-1, ATP-2, ATP-3, ATP-4..7, ATP-10..14, ATP-18..19, ATP-23..25
- **+identity** — basic plus ATP-20, ATP-21, ATP-22
- **+evidence-pack** — +identity plus ATP-26, ATP-27
- **+cross-channel** — +evidence-pack plus ATP-12, ATP-15, ATP-16, ATP-17, ATP-28, ATP-29
- **+realtime-audio** — +cross-channel plus ATP-9
- **+operational** — +cross-channel plus ATP-30, ATP-31, ATP-32

---

#### ATP-34 — Conformance Test Suite

**Status:** Phase 3. **Depends on:** ATP-33.

**Abstract.** Defines the executable test suite that an implementation runs against itself (or that a third-party verifier runs against an implementation) to demonstrate conformance at a given tier. Open-source reference test runner; vendor-neutral.

---

#### ATP-35 — Compliance Tiers

**Status:** Phase 3. **Depends on:** ATP-33, ATP-26.

**Abstract.** Defines the named compliance tiers that gateways can certify against, the audit-evidence pack profiles they require, and the recertification cadence. Provides regulator-facing badging.

---

## 5. Phased shipping roadmap

| Phase | Timing | RFCs |
|---|---|---|
| **Phase 0** | Months 1-2 (with AgentLyne v1) | ATP-1, 2, 3 |
| **Phase 1** | Q3 2026 | ATP-4, 5, 6, 7, 8, 10, 11, 13, 14, 18, 19, 23, 24, 25 |
| **Phase 2** | Q4 2026 — **AAIF submission window** | ATP-12, 15, 16, 17, 20, 21, **22**, 26, 27, 28, 29 |
| **Phase 3** | 2027 | ATP-9, 30, 31, 32, 33, 34, 35 |

**Submission bundle (Phase 2 highlight RFCs):**

- **ATP-15** Conversation Resource (cross-channel unification)
- **ATP-22** KYA Attestation (agent-operator identity)
- **ATP-28** Transfer Envelope (cross-runtime handoff)

These three together are the AAIF working-group proposal. Each solves a distinct "cross-" problem (channel, accountability, runtime) that no single vendor today addresses.

---

## 6. Strategic logic

**Why this structure works.**

The 35-RFC split is what makes ATP authorable by a small team. Each RFC is 5-15 pages, narrow scope, and largely independent. AgentLyne (or any editor) can ship Phase 0 in a month, Phase 1 alongside product implementation, and the AAIF-submission bundle in Phase 2 without holding the entire spec hostage to any one document's review cycle.

**Why MCP-as-base is the right choice.**

MCP has the adoption (97M monthly SDK downloads, May 2026), the framework (sessions, tool discovery, capability advertisement), and the political legitimacy (Linux Foundation AAIF, model labs co-signed). Layering on MCP gives ATP all of that for free. Forking would invite an unwinnable second-system fight against Anthropic and OpenAI; layering invites them as allies.

**Why the AAIF submission targets cross-runtime + cross-channel + cross-accountability.**

These three are the topics:

- Uniquely AgentLyne's to author (no incumbent has all three pieces),
- Beneficial to multiple downstream constituents (model labs, agent platforms, carriers, regulators) so co-signing is rational,
- Hard to walk back once standardized (the schemas compound — Transfer Envelope cites KYA cites Audit Log cites Call Events).

Three RFCs is also small enough to land cleanly with an AAIF working group, rather than presenting a 35-RFC firehose.

**Why streaming audio is Phase 3.**

ATP-9 is the most technically demanding RFC and the one where bad design choices have the highest cost. Deferring to 2027 buys time for:

- MCP base spec to potentially add a binary streaming transport mode (which would make ATP-9 lean rather than fat).
- Real-world implementation experience with ATP-5 events + ATP-6 takeover mode (text-streaming variant) to inform what audio frames actually need.
- WebRTC-grade engineers to join the AgentLyne team to do this correctly.

Shipping ATP-9 in Phase 1 would either lock in a poor design or stall the rest of the series. Shipping it in Phase 3 lets the cheap RFCs land first and lets the political capital from those landings carry the hard one.

---

## 7. Open questions and known unknowns

- **MCP binary transport.** If the MCP base spec ratifies a binary-streaming transport in 2026, ATP-9 becomes easier; if it does not, ATP-9 will need to spec a sibling WebSocket transport.
- **KYA issuer federation.** ATP-22 v1 assumes single-issuer trust. A v2 will need a federation model. Open question: who runs the root of trust?
- **EU AI Act mapping precision.** ATP-26 profiles depend on EU AI Act enforcement guidance, which is still being clarified. Schema may need point releases through 2026-2027.
- **STIR/SHAKEN authority delegation.** ATP-21 assumes gateways have an STI-AS relationship; in practice, gateways will partner with carriers (Bandwidth, Inteliquent). The protocol should specify how the partner relationship surfaces but not require gateways to be STI-AS themselves.
- **Voice-tier objective measurement.** ATP-7's tier contracts use MOS (Mean Opinion Score) — but MOS for AI-generated voice is contested. A future RFC may need to define an "AVQS" (Agent Voice Quality Score) more applicable to TTS systems.
- **Rich-content channel fragmentation.** ATP-12 has to map onto WhatsApp's WABA schema, Apple's Messages for Business schema, Google's RBM schema. These evolve. The protocol-portable abstraction may need versioning by channel.

---

## 8. Glossary

- **A2P 10DLC** — Application-to-Person messaging on US 10-Digit Long Codes. Requires TCR brand and campaign registration.
- **AAIF** — Agentic AI Industry Foundation, Linux Foundation, December 2025.
- **ATP** — Agent Telephony Protocol.
- **Brand** — A verified business identity registered with rich-messaging vendors (Google RBM, Apple, Meta) and/or RCD providers.
- **CPaaS** — Communications Platform-as-a-Service. Twilio, Telnyx, Bandwidth, Plivo, Vonage, Sinch.
- **Fast Driver** — In AgentLyne's reference architecture, the hosted LLM that runs the live conversation. Not normative for ATP.
- **Gateway** — A program that implements ATP and brokers between agents and carriers.
- **KYA** — Know Your Agent attestation, ATP-22.
- **MCP** — Model Context Protocol, Anthropic, November 2024.
- **MOS** — Mean Opinion Score, a 1-5 subjective audio quality scale.
- **RBM** — Rich Business Messaging, Google's RCS Business platform.
- **RCD** — Rich Call Data, the display-name and logo metadata carried in STIR/SHAKEN.
- **RCS** — Rich Communication Services, the modern SMS replacement.
- **Smart Observer** — In AgentLyne's reference architecture, the customer's own agent connected via MCP to observe / inject / takeover a call. Not normative for ATP.
- **STIR/SHAKEN** — Secure Telephony Identity Revisited / Signature-based Handling of Asserted information using toKENs. Carrier-level caller authentication.
- **STI-AS** — Service Authorization Service, the carrier-side signing endpoint in STIR/SHAKEN.
- **TCR** — The Campaign Registry, the US gatekeeper for A2P 10DLC.
- **Tenant** — The customer of an ATP gateway, holding the legal accountability for the agents under it.
- **WABA** — WhatsApp Business API, Meta's enterprise messaging product.

---

## 9. Contribution and governance model

ATP follows the IETF/AAIF model adapted for AI-era pacing:

- **Editor:** AgentLyne (until AAIF working-group formation).
- **Repository:** `github.com/agentlyne/atp-spec` (public, MIT license for the spec; reference code MIT-licensed).
- **RFC numbering:** stable from first published draft; cited as "ATP-NN".
- **Drafts:** every RFC has a draft status (`draft-01`, `draft-02`, etc.) until working-group adoption.
- **Working-group submission target:** Linux Foundation AAIF, Q4 2026, with ATP-15, ATP-22, ATP-28 as the inaugural submission package.
- **Co-author recruitment priorities:** Anthropic agent team, OpenAI Agents team, Cursor, OpenHands, Telnyx, Bandwidth, Pipecat (Daily.co), LiveKit Agents team.
- **Comment period:** every RFC has a 30-day public comment window before being marked stable.
- **Versioning:** the protocol carries a top-level `atp_version` advertised at session init (ATP-2). Capability flags within `atp_capabilities` let gateways advertise partial implementations without forking the version.

---

*End of ATP specification v0.1 draft.*
