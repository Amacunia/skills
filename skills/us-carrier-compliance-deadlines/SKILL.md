---
name: us-carrier-compliance-deadlines
description: Checks a US motor carrier's federal compliance deadlines by USDOT number, from live FMCSA public data - the MCS-150 biennial update deadline (49 CFR 390.19T) and whether it was missed, operating authority per MC docket, insurance and bond filings on file (BMC-91, BMC-34, BMC-84), FMCSA suspension notices and insurance cancellations with effective dates and days left. Calls the paid Amacunia web APIs (x402, USDC on Base) or their MCP servers; a free sample and a small free daily trial exist. Use when a freight broker, shipper, insurer or fleet compliance tool asks whether a trucking company is still authorized, whether its insurance is on file, when its MCS-150 is due, or whether its USDOT number risks deactivation. Do not use for driver records, safety scores or CSA ratings, or to decide on a person's employment, credit or insurance (not a consumer report under the FCRA).
license: MIT
metadata:
  publisher: "Amacunia (amacunia.fr)"
  services: "mcs150.amacunia.fr, authority.amacunia.fr"
---

# US carrier compliance deadlines (MCS-150, operating authority, insurance)

Two web services by Amacunia (amacunia.fr) read public FMCSA data on data.transportation.gov
and return dates, states and days left for 1 to 10 USDOT numbers per call. They return no
personal data: no names of individuals, phone numbers, emails or company officers.

## 1. When to use it, and when not to

Use it when the user asks:
- "Is this carrier's MCS-150 update overdue? When is the next one due?"
- "Is this trucking company's operating authority active, and is its insurance on file?"
- "Has the FMCSA served a suspension notice, or has an insurer canceled a policy, and by when?"
- "Can I tender a load to USDOT 1234567?" (one-call check: `/v1/carrier-compliance`)

Do not use it for:
- driver records, inspections, crash data, safety ratings or CSA scores (not returned);
- intrastate-only registrations outside the federal rule, or carriers outside the US;
- any decision on a person's employment, credit or insurance (not a consumer report under the FCRA);
- a computed revocation date: no time limit between cancellation, suspension and revocation is
  set by law, so the service never computes one.

## 2. Which call to make

All three paid routes take one parameter, `usdot`: 1 to 10 USDOT numbers, comma-separated
(digits only, up to 8 each).

| Question | HTTP GET | MCP tool (endpoint) | Arguments |
|---|---|---|---|
| MCS-150 biennial update only | `https://mcs150.amacunia.fr/v1/mcs150?usdot=3327` | `mcs150_lookup` at `https://mcs150.amacunia.fr/mcp` | `{"usdot": "3327"}` |
| Operating authority and insurance | `https://authority.amacunia.fr/v1/authority?usdot=3327` | `authority_lookup` at `https://authority.amacunia.fr/mcp` | `{"usdot": "3327"}` |
| Everything in one call (authority, insurance, MCS-150, enforcement, sorted deadlines) | `https://authority.amacunia.fr/v1/carrier-compliance?usdot=3327` | `carrier_compliance_lookup` at `https://authority.amacunia.fr/mcp` | `{"usdot": "3327"}` |

The MCP servers use the streamable HTTP transport. Free MCP tools: `mcs150_example`,
`authority_example`, `carrier_compliance_example` (no arguments).

## 3. What it costs

| Route | Price per call |
|---|---|
| `/v1/mcs150` and `mcs150_lookup` | 0.005 USDC |
| `/v1/authority` and `authority_lookup` | 0.01 USDC |
| `/v1/carrier-compliance` and `carrier_compliance_lookup` | 0.012 USDC |

One call covers 1 to 10 USDOT numbers. Prices are also published in `/openapi.json`
(`x-payment-info`) and `/.well-known/x402` on each host.

## 4. How to get a real answer for free

- **Free sample, any time:** `GET https://mcs150.amacunia.fr/v1/exemple` and
  `GET https://authority.amacunia.fr/v1/exemple`. Each returns a real answer for one carrier,
  wrapped as `{"exemple": {...}, "note": "..."}`. The authority sample has the shape of
  `/v1/authority`; the shape of `/v1/carrier-compliance` is available through the MCP tool
  `carrier_compliance_example`.
- **Free trial on the web route, on request only:** add `free_trial=1` to the query string or
  send the header `X-Free-Trial: 1`. Limit: 3 calls per day per caller on each service (each host
  keeps its own count), within a daily cap for that service. The response header
  `X-Free-Trial-Remaining` shows how many are left. Without that signal the route always
  answers 402.
- **Free trial through MCP:** pass `"free_trial": true` to `mcs150_lookup` or `authority_lookup`.
  Same limit, and the same daily quota as the web route (not an extra 3).
- **No free trial** on `/v1/carrier-compliance` or `carrier_compliance_lookup`. To try the
  combined check for free, call `/v1/authority` and `/v1/mcs150` with the trial instead.

## 5. How payment works

The paid routes use x402 version 2 (scheme `exact`, USDC on Base, network `eip155:8453`). An
unpaid call answers HTTP 402 with the payment requirements in the `PAYMENT-REQUIRED` header;
the client retries the same request with a `PAYMENT-SIGNATURE` header. Over MCP, the payment
goes in the request `_meta` under `x402/payment`. Invalid input answers HTTP 400 and is never
charged; errors are never charged.

**Ask the user before spending anything**, and try the free sample or the free trial first.
Do not pay unless the user has a funded x402 wallet set up and has agreed to the price.

## 6. How to read the answer

### `/v1/mcs150`: one entry per USDOT number in `results`

- `found`: false when the number is not in the FMCSA Company Census File.
- `next_deadline`, `days_to_next_deadline`: last day of the filing month, counted in
  `time_zone` (the carrier's state). If `time_zone_known` is false, treat days left as approximate.
- `last_deadline`, `last_mcs150_update`, `overdue`: `true`, `false`, or **`null` when the rule
  or the data does not settle it** (see `overdue_reason`).
- `deactivation_exposure`: **`null` for a missed deadline, not `true`.** For deadlines due on or
  after 2026-06-01 the rule still provides for deactivation but the FMCSA says it is not applying
  it; for earlier ones the FMCSA notice settles neither way. Read `deactivation_exposure_reason`
  and `enforcement_for_last_deadline` (`suspended`, `not_explicitly_covered`, `not_suspended`).
- `usdot_status` (`active`, `inactive`, `pending`), `covered_by_390_19T`, `parity_assumed`.
- Top-level `enforcement`: what the FMCSA says it enforces (`state`, `since`, `until`,
  `source_url`, `read_on` = the date the notice was checked, `quote`).
- `safer_url`: the official SAFER Company Snapshot for that carrier.

### `/v1/authority`: one entry per USDOT number in `results`

- `authorities[]`: one per docket, with `authority_type`, `registry_status`, `last_event`,
  `registry_vs_history` (whether the registry and the history agree) and
  `insurance_requirements` (required vs. on-file amounts, bond and cargo flags).
- `policies_on_file[]`: `form` (BMC-91, BMC-34, BMC-84...), `insurer`, `effective_date`, `in_effect`.
- `suspension_notices[]`: `effective_on`, `days_left`, `state`,
  `insurance_filing_received_since_served`.
- `insurance_cancellations[]`: `reason` (`cancellation` or `replacement`),
  `cancellation_effective_on`, `days_left`, `newer_policy_on_file`.
- `next_deadline`: **`null` when no suspension or cancellation date lies ahead.**
- `official_record_url`, `authority_lookup_url`: where to confirm before acting.

### `/v1/carrier-compliance`

Each entry has `authority` (the block above), `biennial_update` (the MCS-150 block),
`deadlines[]` (`kind`, `date`, `days_left`, `as_of`, `time_zone`, sorted by date) and
`as_of_differs_between_blocks` (the two blocks come from different datasets; a difference is
shown, not settled). The top level carries `enforcement`.

## 7. What the answer does not guarantee

- Information only, not legal advice; it does not replace a qualified attorney.
- Not affiliated with or endorsed by the FMCSA, the U.S. Department of Transportation or any
  U.S. government agency.
- Not a consumer report under the FCRA: do not use it to decide on a person's employment,
  credit or insurance.
- Authoritative records: the FMCSA SAFER Company Snapshot for the MCS-150 status. For operating
  authority, the FMCSA names its Licensing and Insurance (L&I) system as the authoritative
  record; a Federal Register notice of April 29, 2026 announces its retirement in favor of
  Motus, without a date.
- MCS-150: the file does not show the PRISM exemption (49 CFR 390.19T(i)), so an exempt carrier
  may look overdue. A missed deadline does not mean the number is already deactivated: check
  SAFER. The enforcement suspension is temporary, has no published end date, and does not move
  the deadline. Deadlines have no weekend or holiday extension.
- Authority: the suspension notice dataset carries no revocation and no reason, keeps a notice
  even after it was cured, and only goes back to May 18, 2026. Not every suspension is preceded
  by a published notice. No insurance could be filed from May 14 to 19, 2026, and some effective
  dates were backdated when Motus opened.

## Worked example 1: MCS-150 deadline

Free sample (no trial used, no payment):

```bash
curl -s https://mcs150.amacunia.fr/v1/exemple | jq '.exemple.results[0] | {usdot_number, usdot_status, last_mcs150_update, last_deadline, overdue, next_deadline, days_to_next_deadline, time_zone, as_of}'
```

Real call on the free trial, once the user agrees:

```bash
curl -s -D - "https://mcs150.amacunia.fr/v1/mcs150?usdot=3327&free_trial=1"
```

The sample answer (as of 2026-11-10) shows `last_mcs150_update` 2025-11-07,
`last_deadline` 2026-07-31, `overdue` false, `next_deadline` 2028-07-31, 629 days left,
time zone America/Chicago. What to tell the user:

> USDOT 3327 is active and met its last MCS-150 deadline (July 31, 2026; last update on file
> November 7, 2025). The next update is due by July 31, 2028, 629 days from November 10, 2026,
> Central time. Confirm on the SAFER Company Snapshot before relying on it.

## Worked example 2: operating authority and insurance

```bash
curl -s https://authority.amacunia.fr/v1/exemple | jq '.exemple.results[0] | {authorities, policies_on_file, suspension_notices, insurance_cancellations, next_deadline}'
# Real call on the free trial, once the user agrees:
curl -s -D - "https://authority.amacunia.fr/v1/authority?usdot=3327&free_trial=1"
```

The sample answer (as of 2026-11-10) shows docket MC145655, a property broker authority with
`registry_status` active and `registry_vs_history` agree, a BMC-84 bond in effect since
2024-09-10, no suspension notice, no insurance cancellation, and `next_deadline` null. What to
tell the user:

> USDOT 3327 holds an active broker authority (MC145655), with a BMC-84 bond on file and in
> effect since September 10, 2024. No FMCSA suspension notice and no insurance cancellation
> appear, so there is no upcoming deadline. The FMCSA L&I record is authoritative: check it
> before tendering.

More detail on each host: `/llms.txt`, `/rules` (rules quoted, with the dates they were
checked), `/faq`, `/terms`.
