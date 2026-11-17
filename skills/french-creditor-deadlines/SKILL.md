---
name: french-creditor-deadlines
description: Finds French creditor deadlines by SIREN from the BODACC, France's official gazette of company notices - the last day to file a claim when a company enters insolvency proceedings (procédures collectives - sauvegarde, redressement judiciaire, liquidation judiciaire; art. R622-24 Code de commerce), and the last day for creditors to object after a business sale (vente de fonds de commerce), merger, demerger or partial asset transfer (art. L141-14, R236-11). Calls the paid Amacunia web APIs (x402, USDC on Base) or their MCP servers; a free sample and a small free daily trial exist. Use when a credit manager, accountant, debt collector, landlord or lawyer asks whether a French customer is insolvent, whether there is still time to file a claim, or whether a debtor is selling its business or merging. Do not use for companies outside France, employee or alimony claims, creditors who must be notified personally, or as legal advice.
license: MIT
metadata:
  publisher: "Amacunia (amacunia.fr)"
  services: "creances.amacunia.fr, oppositions.amacunia.fr"
---

# French creditor deadlines (BODACC: insolvency claims, objections to business sales and mergers)

Two web services by Amacunia (amacunia.fr) read the BODACC (Bulletin officiel des annonces
civiles et commerciales, published by the DILA) live on each call and return, for 1 to 50
companies per call, the exact last day for a creditor to act and the days left. Field names in
the answers are in French; this guide translates the ones that matter.

## 1. When to use it, and when not to

Use it when the user asks:
- "Is this French customer in insolvency proceedings? Is there still time to file my claim?"
  (claim filing deadline: `creances`)
- "Is this debtor selling its business (fonds de commerce) or merging? Until when can I
  object?" (creditor objection deadline: `oppositions`)
- for daily or weekly screening of a portfolio of French customers or suppliers by SIREN.

Do not use it for:
- companies not registered in France (input is a SIREN or SIRET);
- employee or maintenance (alimony) claims, which are outside the claim filing rule;
- creditors with another starting point that the service does not compute: holders of a
  published security interest or contract (notified personally), claims arising after the
  judgment, co-contractors whose contract is terminated, creditors relieved of the time bar;
- reclaiming goods (revendication, art. L624-9), which is not computed;
- legal advice. It gives dates and their sources, not a strategy.

## 2. Which call to make

| Question | HTTP GET | MCP tool (endpoint) | Arguments |
|---|---|---|---|
| Insolvency claim filing deadline | `https://creances.amacunia.fr/v1/creances?sirens=910759158` | `creances_lookup` at `https://creances.amacunia.fr/mcp` | `{"sirens": "910759158", "residence": "metropole"}` |
| Objection deadline after a business sale or merger | `https://oppositions.amacunia.fr/v1/oppositions?sirens=920902343` | `oppositions_lookup` at `https://oppositions.amacunia.fr/mcp` | `{"sirens": "920902343"}` |

- `sirens` (required): 1 to 50 SIREN numbers (9 digits, checksum validated) or SIRET (14 digits,
  reduced to the SIREN), comma-separated.
- `residence` (optional, `creances` only): where the creditor lives. `metropole` (default),
  `etranger` (abroad), or an overseas code: `971` to `978`, `986` to `988`. The window is two
  months longer (four instead of two) when `residence` differs from the court's territory:
  mainland France for a mainland court, or the overseas department or collectivity where the
  court sits.

The MCP servers use the streamable HTTP transport. Free MCP tools: `creances_example`,
`oppositions_example` (no arguments).

## 3. What it costs

0.01 USDC per call on each paid route and paid MCP tool, for 1 to 50 SIREN numbers per call.
Prices are also published in `/openapi.json` (`x-payment-info`) and `/.well-known/x402`.

## 4. How to get a real answer for free

- **Free sample, any time:** `GET https://creances.amacunia.fr/v1/exemple` and
  `GET https://oppositions.amacunia.fr/v1/exemple`. Each returns a real answer, computed on the
  date it shows, wrapped as `{"exemple": {...}, "note": "..."}`.
- **Free trial on the web route, on request only:** add `free_trial=1` to the query string or
  send the header `X-Free-Trial: 1`. Limit: 3 calls per day per caller on each service (each host
  keeps its own count), within a daily cap for that service. The response header
  `X-Free-Trial-Remaining` shows how many are left. Without that signal the route always
  answers 402.
- **Free trial through MCP:** pass `"free_trial": true` to `creances_lookup` or
  `oppositions_lookup`. Same limit, and the same daily quota as the web route (not an extra 3).
- One trial call can cover up to 50 SIREN numbers, so batch the companies the user cares about.

## 5. How payment works

The paid routes use x402 version 2 (scheme `exact`, USDC on Base, network `eip155:8453`). An
unpaid call answers HTTP 402 with the payment requirements in the `PAYMENT-REQUIRED` header;
the client retries the same request with a `PAYMENT-SIGNATURE` header. Over MCP, the payment
goes in the request `_meta` under `x402/payment`. Invalid input (for example a SIREN with a
bad checksum) answers HTTP 400 and is never charged; errors are never charged.

**Ask the user before spending anything**, and try the free sample or the free trial first.
Do not pay unless the user has a funded x402 wallet set up and has agreed to the price.

## 6. How to read the answer

### `creances` (insolvency claim filing deadline)

- `date_calcul`: the date the answer was computed. `entreprises_avec_delai_ouvert`: how many
  companies have at least one open claim window.
- `resultats[]`, one per company: `siren`, `denomination` (company name),
  `en_procedure_collective_publiee` (any insolvency notice published).
- `delais_ouverts[]`: opening judgments whose window is open today. Each has `nature`,
  `date_jugement` (judgment date), `publication_bodacc`, `tribunal`, `annonce` (link to the
  notice), `date_limite_declaration` (**the last day to file**), `jours_restants` (days left),
  `delai_augmente_hors_territoire` (the 4-month window applied), and `texte_du_jugement`, which
  names the court-appointed representative to file with. The judgment text may quote the general
  two-month period even when the extension applies: rely on the computed `date_limite_declaration`.
  When you pass the representative's name on to the user, quote it exactly as the notice
  publishes it, and only so the user can file the claim.
- `report_possible_jour_chome`: the possible extension to the next working day when the deadline
  falls on a weekend or holiday. **`null` when there is no extension.** It is not guaranteed: plan
  on `date_limite_declaration`.
- `annulations_verifiees`: **false when the search for cancellation and correction notices timed
  out**; then check the deadline on the notice itself.
- `texte_du_rectificatif`, `rectifie_par`: a correction notice may name another representative;
  file with the one it names. The deadline still runs from the original, earlier notice.
- `derniere_ouverture`, `dernier_evenement`: latest opening judgment and latest notice of any
  kind, open window or not.
- `marche_a_suivre` (top level of the answer, not inside `resultats[]`): how to file, in French,
  only when a window is open.

### `oppositions` (objection after a business sale or merger)

- `resultats[]`, one per company: `delais_ouverts[]`, `evenements_12_mois[]` (notices of the
  last 12 months), `nombre_avis_12_mois`.
- Each event: `nature` (`vente_fonds` business sale, `apport_fonds` contribution of a
  business, `fusion` merger, `scission` demerger, `apport_partiel` partial asset transfer),
  `role_de_l_entreprise` (only `vendeur`, `apporteur` or `societe_participante` opens a window
  for that company's creditors; as `acquereur`, the buyer, no deadline is given),
  `publication_bodacc`, `annonce`.
- When a window opens: `date_limite_opposition` (**the last day to object**), `delai_jours`
  (10 or 30), `jours_restants`, `date_au_plus_tot` (**true when a later insertion elsewhere may
  push the date back**: the date shown is the earliest possible one), `geste` (what to do),
  `domicile_opposition` (where to serve the objection), `report_possible_jour_chome`.
- `solidarite_ecartee` (demergers and partial transfers only): true when the notice excludes
  joint liability; **`null` when the notice does not say**, in which case the date applies only
  if the project excludes it, and `geste` says so.
- `phrase_de_l_avis`: the notice's own deadline sentence, or `null` when it has none.

## 7. What the answer does not guarantee

- Information only, not legal advice. The notice published in the BODACC prevails. Amacunia is
  an independent publisher; the services only read the BODACC.
- Claim filing: the deadline is the strict one, for an ordinary creditor whose claim arose
  before the judgment (2 months from the BODACC publication, 4 outside the court's territory;
  art. R622-24 Code de commerce, art. 641 Code de procédure civile). The working-day extension
  (art. 642) uses mainland public holidays only and may be wrong before an overseas court.
- Objections: no BODACC notice does not prove there is no merger. A company that publishes its
  merger project on its own website needs no BODACC notice (art. R236-3), and the service does
  not see it. Only notices from the last twelve months are read.
- Personal data: an individual entrepreneur is never named, and natural persons who are parties
  to an operation are not named. The answer quotes the official notice where needed, so the name
  and business address of the court-appointed representative (`creances`) and the address for
  serving an objection, including the name of the lawyer, notary or escrow agent (`oppositions`),
  appear as published.

## Worked example 1: is there still time to file a claim?

```bash
curl -s https://creances.amacunia.fr/v1/exemple | jq '.exemple.resultats[0].delais_ouverts[0] | {nature, date_jugement, publication_bodacc, tribunal, date_limite_declaration, jours_restants, annonce}'
# Real call on the free trial, once the user agrees:
curl -s -D - "https://creances.amacunia.fr/v1/creances?sirens=910759158&residence=metropole&free_trial=1"
```

The sample answer (computed on 2026-11-09) shows a judicial liquidation (liquidation
judiciaire) ordered on 2026-10-29 by the commercial court of Evreux, published in the BODACC
on 2026-11-08, with `date_limite_declaration` 2027-01-08 and 60 days left. What to tell the user:

> SIREN 910759158 was placed in judicial liquidation on October 29, 2026 (BODACC, November 8,
> 2026). If your claim arose before the judgment and you are based in mainland France, the last
> day to file it is January 8, 2027, 60 days from November 9, 2026. File with the
> representative named in the judgment text; the BODACC notice prevails.

## Worked example 2: is the debtor selling its business?

```bash
curl -s https://oppositions.amacunia.fr/v1/exemple | jq '.exemple.resultats[0].delais_ouverts[0] | {nature, role_de_l_entreprise, publication_bodacc, date_limite_opposition, delai_jours, date_au_plus_tot, geste, domicile_opposition}'
# Real call on the free trial, once the user agrees:
curl -s -D - "https://oppositions.amacunia.fr/v1/oppositions?sirens=920902343&free_trial=1"
```

The sample answer (computed on 2026-11-09) shows a business sale (`vente_fonds`) where the
company is the seller, published in the BODACC on 2026-11-06, with a 10-day window and
`date_limite_opposition` 2026-11-16. What to tell the user:

> SIREN 920902343 sold its business; the sale was published in the BODACC on November 6, 2026.
> As a creditor of the seller, you can serve an objection on the sale price until November 16,
> 2026 (10 days, art. L141-14 Code de commerce), at the address given in the notice. After that
> date the price may be paid to the seller.

More detail on each host: `/llms.txt`, `/rules` (rules quoted verbatim), `/faq`, `/terms`.
