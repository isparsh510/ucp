<!-- cSpell:ignore isparsh crosslisting Shopware shopware ucpready RFQ quotable quotability supersession misprice ignorable checkable unexpired unsatisfiable deterministically mislabels -->

# [Proposal]: Quote extension — Business-authoritative quotation binding on Cart and Checkout

> **Status:** filed upstream as [Universal-Commerce-Protocol/ucp#929](https://github.com/Universal-Commerce-Protocol/ucp/issues/929) (Proposal stage; awaiting Technical Council triage), with draft reference PR [#930](https://github.com/Universal-Commerce-Protocol/ucp/pull/930) from branch `feat/quote-extension-pr` (schema + spec only). Working copy on the personal fork (`isparsh510/ucp`, branch `feat/quote-extension`, review copy [isparsh510/ucp#3](https://github.com/isparsh510/ucp/issues/3)). Keep this file and #929 in sync; #929 is authoritative.
> Companion working-draft files in the same commit series: `source/schemas/common/quote.json`,
> `docs/specification/common/extensions/quote.md`, `mkdocs.yml`,
> `source/schemas/common/types/{warning_code,error_code}.json`.
>
> Everything below the rule is the body of #929 (headings match the **Enhancement Proposal** issue template in
> `Universal-Commerce-Protocol/.github`). This file does not ship in the upstream PR — upstream tracks proposals as issues, not files.

---

## Summary

**Decision requested:** whether a Business's counter-quotation should be expressed as a #773 Offer rather than the `status: "quoted"` defined here. This proposal also overlaps #845's placement question (it is one concrete candidate for option B); the Technical Council may prefer to triage them together.

This proposes a small `dev.ucp.common.quote` extension that gives a Business one way to issue a **quotation** — a time-bound, immutable price for the line items of a Cart or Checkout — and gives a Buyer one way to **accept it by id**, whether the quotation was formed in the Cart (the Buyer proposed a price or asked for one) or outside it (the Business's own storefront, a conversation, a vendor extension). The Business reflects an accepted quotation in the authoritative `totals` as a single `quote` entry and projects it onto the resulting Order.

The extension standardizes the **binding contract**: discovery, a durable quotation identity, expiry, acceptance by reference, invalidation when the line items change, and projection to Order. It also carries an optional Buyer `proposal` so that the common consumer and B2B openings ("I'll pay $150 for these three"; "quote me 500 units") travel on existing Create/Update Cart and Checkout operations. It is a one-quotation-at-a-time loop, not a negotiation protocol: no revisions, no multi-party mechanics, no non-price terms, and nothing about how a Business decides. It adds **no new endpoints** and follows the request/response pattern of the Discount extension (Buyer input the Business may reject) and the Payment Terms extension (a Business-authored choice whose acceptance is projected onto the Order).

## Motivation

Commerce today is unidirectional in UCP: the Business lists a price, the Buyer pays it or applies a code. Two kinds of commerce don't fit that model, and both have asked for it in this repository:

* **Consumer offers.** Marketplaces where buyers routinely propose below asking price (resale, collectibles, furniture, high-ticket DTC). Discussion [#502](https://github.com/Universal-Commerce-Protocol/ucp/discussions/502) (*A Buyer's Agent Should Negotiate*) and [#25](https://github.com/Universal-Commerce-Protocol/ucp/discussions/25) (*C2C/Resale marketplaces*) both name the absence of a propose → accept/decline/counter loop as the gap; a crosslisting vendor in #25 notes that for resale "an agent that can't do that loop can't really buy here." A production "make an offer" tool shared in #502 runs this flow today behind a form and a discount code, which is exactly the shape an agent cannot drive.
* **B2B quotes.** Shopware [writes in #502](https://github.com/Universal-Commerce-Protocol/ucp/discussions/502#discussioncomment-18386116) that "a buyer agent needs a discoverable way to know that a Business accepts offers and a structured flow for propose / counter / accept / reject—not just a discount code or an eventual checkout URL," and that because the flow may be asynchronous "a negotiation needs durable identifiers, revisions, expiry, and a safe return path for the agent." They ship `com.shopware.quote` today (live on `sw-ag.dev/.well-known/ucp`) as a standalone quote resource whose acceptance places an order directly, and say in [#812](https://github.com/Universal-Commerce-Protocol/ucp/discussions/812) that "a UCP-native handoff would instead need to bind that accepted commercial state deterministically to the resulting Cart or Checkout." UCPReady describes a `com.ucpready.procurement` RFQ/quote/PO extension in [#93](https://github.com/Universal-Commerce-Protocol/ucp/discussions/93); no schema or manifest for it is published. [#877](https://github.com/Universal-Commerce-Protocol/ucp/issues/877) (*price on request*) documents two sellers with 318 products whose price can only arrive through a quote.

What every one of those threads converges on is not the negotiation loop — which Shopware already runs in its own namespace — but the step after it: a Platform needs **one** way, for any Business, to bind an agreed price to the authoritative Cart or Checkout and read it on the Order. That binding cannot live in a vendor namespace, because the Platform is the party that must implement it once. This proposal is that binding, plus the smallest in-protocol opening that lets a consumer agent make an offer without leaving UCP.

The design keeps the **direction of authority** raised in #812 and [#845](https://github.com/Universal-Commerce-Protocol/ucp/issues/845): a quotation id, however obtained, is an input the Business resolves; the only price that applies is the one the Business returns in `totals`, and the Business can decline to honor an id that is valid, unexpired, and still wrong for the current line items.

Target users: buyer agents and Platforms (a uniform way to ask, to apply a quotation, and to render an answer), Businesses (a uniform way to receive proposals from any agent while keeping full control of the decision — rule, model, or person), and Buyers (a negotiated price they can actually use through an agent).

## Goals

* Let a Buyer accept a Business-issued quotation by id on Create or Update Cart and Checkout, whether it was issued in the Cart or outside it.
* Let a Buyer propose a subtotal, or ask for a quotation, for the current line items through existing operations.
* Let a Business accept, decline, defer (human review), or answer with its own quotation.
* Give every quotation a Business-scoped, immutable identity and an explicit expiry, so asynchronous and multi-day flows are safe.
* Bind a quotation to the line items it priced; make stale ids fail closed with an explicit `quote_changed` warning.
* Reflect an accepted quotation in the authoritative `totals` as exactly one machine-identifiable entry, and carry it onto the Order.
* Keep the Business authoritative at every step and keep `checkout.status` untouched; the Platform never originates a price.
* Reuse existing primitives — `totals`, `messages[]`, `expires_at`, `continue_url`, `request_constraints`, Get Cart/Checkout — rather than add a resource, endpoint, or state-machine machinery to core.

## Non-Goals

* Negotiation strategy, reserve prices, discount logic, customer-specific pricing, or when to escalate to a human.
* Revisions, counters that change line items, auctions, bidding, ranking, supplier selection, or any multi-party mechanism. A Business answer of the form "not at that price, but take two and I will" is a proposed Cart transition and belongs to #738/#773.
* Negotiating anything other than price for the current line items (fulfillment, delivery dates, payment timing, warranty). Payment Terms and Fulfillment continue to govern those and compose with an accepted quotation.
* Per-line quoted prices, an explicit Buyer decline, and cancellation of a pending proposal — deferred to Candidate (see Open questions).
* A portable accepted-terms artifact or binding protocol beyond an opaque Business-issued id (#845). This proposal is one concrete carrier for #845's binding semantics; if the Technical Council adopts a generic artifact, `quote.accepted_id` is the field that would carry its reference.
* Catalog-side signalling that an item has no public price (#877). A Cart cannot carry such an item meaningfully until a catalog marker exists; that work is a dependency of the price-on-request scenario, not part of this proposal.
* Buyer verification, tax identifiers, B2B account onboarding, payment execution, deposits, or holds.

## Detailed Design

### 1. One property on Cart and Checkout: `quote`

```json
{
  "quote": {
    "status": "quoted",
    "id": "qt_8c21",
    "subtotal": 15900,
    "expires_at": "2026-10-10T23:30:00Z",
    "message": "We can do $159 for the set if you order today.",
    "proposal": { "subtotal": 15000 }
  }
}
```

* **Response-only** (`ucp_request: omit`): `status`, `id`, `subtotal`, `expires_at`, `message`.
* **Buyer-written** (create and update; omitted on complete): `proposal` and `accepted_id`. An update carries at most one of them. The Business echoes both on every response while the cycle they belong to is open, so an `accepted` status without `accepted_id` means the Business accepted the Buyer's proposal outright.
* `"quote": {}` on a response means "quotable, no cycle open"; absence means the resource is not quotable in its current state. Profile discovery says the extension is implemented; quotability is a runtime result, like handler eligibility.

### 2. Lifecycle — an open vocabulary with fail-closed Platforms

`status` is an open string with well-known values `pending`, `quoted`, `accepted`, `declined`, `expired`, `withdrawn`, and is absent when no cycle is open. A Platform that receives a value it does not recognize MUST treat no quotation as acceptable and MUST NOT offer to accept one; because `totals` are authoritative in every state, an unknown state cannot misprice a purchase — it only withholds the quote interaction. This gives the safety property a closed enum would, without the one-way door: `allOf` cannot widen an enum, and the only published vendor lifecycle (`com.shopware.quote`: `draft`, `open`, `in_review`, `replied`, `change_requested`, `accepted`, `declined`, `expired`, `withdrawn`, `cancelled`, `reopen`) already has states this draft does not.

| From                                  | Trigger                                              | To                                                             | Side effects                                                 | Messages                                    |
| :------------------------------------ | :--------------------------------------------------- | :------------------------------------------------------------- | :----------------------------------------------------------- | :------------------------------------------ |
| idle · declined · expired · withdrawn | Buyer sends `proposal`                               | pending · quoted (new id) · accepted (new id) · declined       | proposal echoed; prior quotation dropped                     | —                                           |
| quoted                                | Buyer sends `proposal`                               | as above                                                       | unaccepted quotation superseded                              | —                                           |
| pending                               | Buyer sends `proposal`                               | pending                                                        | replaces the pending proposal                                | —                                           |
| accepted                              | Buyer sends `proposal`                               | pending · quoted · accepted (new id) · declined                | accepted quotation invalidated; `quote` totals entry removed | `quote_changed`                             |
| any                                   | Buyer sends a `proposal` the Business won't consider | unchanged                                                      | —                                                            | `quote_unavailable` error                   |
| pending                               | Business answers                                     | quoted · accepted · declined                                   | —                                                            | —                                           |
| any                                   | Buyer sends an `accepted_id` that resolves           | accepted                                                       | `quote` totals entry added; derived state recomputed         | —                                           |
| any                                   | Buyer sends an `accepted_id` that doesn't resolve    | unchanged                                                      | —                                                            | `quote_changed`                             |
| accepted                              | Buyer repeats the current `accepted_id`              | unchanged (no-op)                                              | —                                                            | —                                           |
| quoted · accepted                     | Business withdraws                                   | withdrawn (id kept)                                            | entry removed if accepted                                    | `quote_changed`                             |
| quoted                                | `expires_at` passes                                  | expired (id kept)                                              | —                                                            | `quote_changed` if acceptance was attempted |
| quoted · accepted                     | line items or currency change                        | quoted (new id) · accepted (new id) · idle · `quote` omitted   | entry removed if accepted                                    | `quote_changed`                             |
| pending                               | line items or currency change                        | idle, or pending if the Business will answer for the new lines | —                                                            | `quote_changed`                             |
| declined · expired · withdrawn        | line items or currency change                        | idle · `quote` omitted                                         | —                                                            | —                                           |
| any open cycle                        | Checkout completes                                   | cycle closed                                                   | Order carries `quote` only if accepted                       | —                                           |

A **cycle** opens with a `proposal` or an `accepted_id` and closes when superseded, invalidated, returned to idle, or completed; `proposal` and `accepted_id` are echoed while it is open. `quote_changed` reports changes the Buyer did not ask for; a Buyer's own proposal superseding an unaccepted quotation is silent.

### 3. Proposal

`quote.proposal` carries `subtotal` (the consumer offer or B2B target) and an optional `message`. An empty `{}` asks for a quotation without proposing. A Business MUST NOT require `message` nor personal data in it; identity travels through `buyer` and Identity Linking. Submitting a proposal replaces the previous one and supersedes any unaccepted quotation. A proposal the Business will not consider at all (not quotable, allowance exhausted, malformed update) is answered with a `quote_unavailable` **error** in `messages[]`, leaving `quote` unchanged — distinct from `declined`, which is a decision about the price. There is no cancel in this draft: a Buyer proposes again, accepts a quotation it holds, or lets the resource expire; a Business treats a replacing proposal as supersession in its queue.

### 4. Acceptance and binding

`accepted_id` names either the `id` of the latest `quoted` response or the `id` of a quotation this Business issued to this Buyer outside the Cart or Checkout for the same line items. The Business resolves it, verifies the line items match, and returns the recomputed resource; that response is authoritative for all derived state (`totals`, line prices, discount eligibility, policies, messages, payment handlers and terms). An id that does not resolve — expired, superseded, invalidated, or never issued to this Buyer for these items — is never substituted; the Business returns current state with `quote_changed`. Repeating the current `accepted_id` is a no-op. An update that combines `accepted_id` with line-item changes is evaluated against the changed lines, so it fails closed. `accepted_id` is omitted on complete because a quotation is agreed, if at all, before completion (the `selected_term_id` pattern).

### 5. Reflection in totals

When `accepted`, `totals` carry **exactly one** entry of `type: "quote"` whose signed `amount` is `quote.subtotal` minus the sum of line item subtotals. `quote` is a `totals[].type` value this extension defines; because it is outside the core list, `totals.json` already requires `display_text` on it. Line items keep their listed prices. A Platform identifies the entry by `type` alone and can verify that line subtotals plus the entry equal `quote.subtotal`. The entry is not a discount: it is never `items_discount`, is not allocated, and does not appear in `discounts.applied`, so the Discount extension's invariants hold when both are active.

### 6. Scope, expiry, immutability

A quotation is bound to each `line_items[].id` with its `item.id`, `quantity`, `quantity_unit`, and to the resource `currency`. When an update changes any of these the Business MUST re-quote (new id, `quoted` or `accepted`), return `{}`, or omit `quote` — and in every case report `quote_changed` and remove the totals entry. For a given `id`, `subtotal` and the bound line items never change; `expires_at` may only be extended. A lapsed quotation is `expired`; a revoked one is `withdrawn` with `id` and `subtotal` kept so the Buyer can see what was revoked. An accepted quotation does not lapse. Quotation ids are unique within the Business so the same id names the same quotation on Cart, Checkout, Order, and outside UCP.

### 7. Asynchronous review

`pending` changes nothing else. The Business SHOULD set `message` and extend the resource `expires_at` to cover its review window; the Platform MAY re-read with Get Cart/Checkout under bounded backoff and MUST stop at `expires_at` (the Checkout `complete_in_progress` contract), and SHOULD surface `continue_url`. Silence is never acceptance. The answer is reported exactly as a synchronous one.

### 8. Checkout status and completion

A quote cycle does not change `checkout.status`. A Checkout MAY be completed while `status` is anything other than `accepted`: the Buyer pays `totals` as they stand, completion closes the cycle, and the Order carries `quote` only if a quotation was accepted. A Business whose policy forbids list-price sale while a proposal is pending uses the core status lifecycle (`incomplete` with a message at `$.quote`, or `requires_escalation` with `continue_url`) and MUST NOT hold a Checkout silently. A Business MUST NOT complete a Checkout whose `totals` do not honor an un-withdrawn accepted quotation. Leaving `accepted` on a Checkout is a totals change and carries its consequences: a selected payment term is re-summed with `payment_term_changed`, and an AP2-signed Checkout must be authorized again. A Business MAY pin `quote.accepted_id` to the current quotation in `ucp.request_constraints` so Platforms preflight stale acceptance; it cannot forbid proposals through constraints and declines them with `quote_unavailable` instead.

### 9. Cart → Checkout → Order

A Checkout created from a Cart carries `quote` with the same ids; a Business that extends Cart MUST also extend Checkout when it supports `cart_id`. While linked, the Checkout's `quote` is authoritative. On the Order, `quote` carries `id` and `subtotal` only, mirroring `payment.accepted_term`.

### 10. Schema and discovery

* `source/schemas/common/quote.json` — `name: dev.ucp.common.quote`, homed in `common/` like `dev.ucp.common.loyalty` and the payment extensions because the shape is vertical-neutral (a lodging group booking or a catering order quotes the same way); `$defs`: `quote_proposal`, `quote`, `order_quote`, and `dev.ucp.shopping.cart` / `dev.ucp.shopping.checkout` / `dev.ucp.shopping.order` compositions via `allOf`. Open objects and open vocabularies throughout; conditional `required` via `if`/`then` as `total.json` does.
* Profile declaration: `"dev.ucp.common.quote": [{ "version": …, "extends": ["dev.ucp.shopping.cart", "dev.ucp.shopping.checkout", "dev.ucp.shopping.order"] }]`.
* `warning_code.json` gains `quote_changed`; `error_code.json` gains `quote_unavailable` (example values, as #602 added `payment_term_changed`).
* No OpenAPI/OpenRPC change: no new operations.

### Why this shape

| Alternative                                                           | Why not                                                                                                                                                                                                                                                                                                                                  |
| :-------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A new `Quote` capability/resource (Shopware's current model)          | New endpoints and scaffolds, and a handoff contract into Cart/Checkout that #812/#845 show is the unresolved part. This extension *is* that handoff; a Business-side quote object can back it and `accepted_id` can reference it.                                                                                                        |
| Encode the result as a discount code                                  | Today's workaround. No discovery, no lifecycle, no expiry semantics, collides with `discounts.applied` reconciliation, and the agent is usually out of session when the code arrives.                                                                                                                                                    |
| Reuse #738's `Offer` for the quotation                                | An Offer is a Business-proposed Cart transition surfaced as an Action and applied through a new `apply_offer` operation. A price-only quotation could be expressed that way; this proposal keeps the simpler status field until the Technical Council rules on #773, and should fold into Offer if it lands with price-only transitions. |
| A closed `enum` for `status`                                          | Not provably closed: the only shipped vendor lifecycle has eleven states, and the authoring guide reserves `enum` for sets where unknown values are inherently unsupported. Here unknown values are safely ignorable because `totals` are authoritative.                                                                                 |
| `quotations[]` + `accepted_id` (the `terms[]` pattern)                | Immutability plus supersession make exactly one quotation current; tiers change quantities and are out of scope. A list adds surface for no scenario.                                                                                                                                                                                    |
| Two totals shapes (`discount`/`fee` entry, or rewritten `item.price`) | Two wire shapes for one fact is an interop hazard; `discount` is reconciled against `discounts.applied`, and `fee` mislabels a price. One signed `quote` entry, identified by `type`, is checkable.                                                                                                                                      |
| Per-line proposals and quoted unit prices in v1                       | One implementer precedent, unsatisfiable on create (line ids are Business-assigned), and an unenforceable rounding invariant. Deferred.                                                                                                                                                                                                  |

### Relationship to open proposals

| Thread                                                       | Relationship                                                                                                                                                                                                                                                                                                |
| :----------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| #845 — accepted commercial-term handoff (placement decision) | This proposal is a concrete answer to #845's option B (a domain extension carrying binding semantics on Cart/Checkout) and should be decided together with it. `accepted_id` is the handoff reference; Business authority is the binding rule.                                                              |
| #738 / #773 — state-bound Offer                              | Complementary today: Offer is Business-initiated and may change line items; a quotation is Buyer-initiated and prices the lines as they are. If #773 lands as an Action with `expires_at` and apply-by-reference, the `quoted` leg of this proposal should become an Offer rather than a second vocabulary. |
| #877 — price on request                                      | A catalog marker (`pricing.mode`) is a prerequisite for quoting items without a public price; this proposal does not solve it and should not be cited as if it did.                                                                                                                                         |
| Discount, Payment Terms                                      | Compose: the `quote` entry is not a discount; acceptance on a Checkout with a selected term triggers `payment_term_changed`.                                                                                                                                                                                |

### Backward compatibility

Purely additive. Businesses that do not advertise the extension are unaffected; absence of `quote` on a response is already the "not quotable" signal. No existing field, operation, or required property changes; `totals[].type` is already an open vocabulary. The two registry files gain example values only.

## Risks and Mitigations

* **Security — price-discovery oracle.** A bot could binary-search a Business's floor by proposing repeatedly. *Mitigation:* a Business decides per proposal and may answer `declined` or `quote_unavailable` at any time without explanation; quotations expire; transport `429` and `identity_required` apply; Identity Linking is the sanctioned way to gate quotes to known accounts. Nothing obliges a Business to quote.
* **Security — authority escalation.** A Platform could try to treat a proposal, a lapsed quotation, or a foreign id as a price. *Mitigation:* the Platform never writes a price; only the Business response's `totals` apply; every id the Business cannot resolve for these line items fails closed with `quote_changed`; AP2 mandate verification rejects a Checkout whose signed totals no longer match.
* **Privacy.** Offer tools today collect name/email/phone in the request. *Mitigation:* `proposal.message` is optional, must not be required, and must not be required to carry personal data; Platforms must not add personal data beyond what the Buyer said; identity travels through `buyer` and Identity Linking.
* **Performance.** Synchronous answers add nothing beyond an Update; asynchronous answers are explicitly `pending`, polled under bounded backoff with a hard stop at `expires_at`, or completed via `continue_url`.
* **Backward compatibility.** Additive extension; see above.
* **Complexity.** One object, two Buyer fields, one open status, one totals entry type, two message codes, no new operations.
* **Precedent.** A third id-plus-expiry-plus-accept-by-reference vocabulary alongside Payment Terms and Offer. *Mitigation:* the proposal names #773 as the primitive the `quoted` leg should fold into if adopted, and asks the Technical Council to decide #845 and this together.

## Test Plan

* **Schema:** `ucp-schema lint source/` passes; `ucp-schema resolve` verified for cart create/update and checkout create/update/complete (`quote` omitted on complete), and for response views (`status`/`id` response-only; `proposal`/`accepted_id` echoed).
* **Examples:** 17 CI-validated examples in the spec page (`scripts/validate_examples.py`): idle `{}`, accept, quote → accept, decline, withdraw, proposal → `pending` → `quoted` across a Get, acceptance of an out-of-band quotation on Create Checkout, `request_constraints` pin, Order projection, `quote_changed` and `quote_unavailable` messages. Full-corpus validation, `scripts/test_validate_examples.py`, and `scripts/test_check_links.py` pass.
* **Docs:** strict MkDocs build and internal link check.
* **Conformance (proposed for Candidate):** vectors for each transition row; stale `accepted_id` → `quote_changed`; line item, quantity, `quantity_unit`, and currency changes each invalidate an accepted quotation and remove the `quote` entry; withdrawal of a `quoted` or `accepted` quotation → `withdrawn` with `id` retained; a quotation past `expires_at` reported `expired`, never `withdrawn`; `accepted` never lapses; Cart → Checkout carries ids; Order carries only `{id, subtotal}`; completion while `quoted`/`pending` closes the cycle without an Order `quote`; a combined `proposal` + `accepted_id` update → `quote_unavailable`; an unrecognized `status` is rendered without an accept affordance.

## Graduation Criteria

**Working Draft → Candidate:**

* [ ] Schema merged and documented (with Working Draft disclaimer).
* [ ] Unit and integration tests are passing.
* [ ] Initial documentation is written.
* [ ] One reference Business and one Platform exercising `proposal`, accept, decline, withdraw, `pending`, and out-of-band `accepted_id`.
* [ ] Status vocabulary checked against at least two production vendor lifecycles, with any additional well-known value documented.
* [ ] Open questions below resolved with implementer evidence.
* [ ] Conformance vectors above adopted.
* [ ] TC majority vote to advance.

**Candidate → Stable:**

* [ ] Adoption feedback has been collected and addressed, including at least one B2B and one consumer deployment.
* [ ] Full documentation and migration guides are published.
* [ ] TC majority vote to advance.

## Open questions deferred to Candidate

1. **Explicit Buyer decline.** Should a Buyer be able to decline a quotation in band (e.g. so a human queue stops waiting), or is a new proposal / lapse sufficient? If added, it must not reuse `declined`, which is Business-only here.
2. **Per-line quoted prices.** B2B invoicing may need agreed unit prices on the Order; Shopware quotes per unit. Add `line_prices[]` only if a second implementer needs it and the create-time binding problem is solved.
3. **Cancel of a pending proposal.** Define only if implementers report queues that cannot treat a replacing proposal as supersession.
4. **Id format.** Whether Business-scoped uniqueness should be strengthened to a URI or prefixed form for out-of-band handoff.

## Implementation History

* [YYYY-MM-DD]: Proposal submitted.
* [YYYY-MM-DD]: TC approved "Provisional"; extension enters "Working Draft".
* [YYYY-MM-DD]: TC approved advancement to "Candidate".
* [YYYY-MM-DD]: TC approved "Implemented"; extension enters "Stable".

## Code of Conduct

* [x] I agree to follow this project's Code of Conduct
