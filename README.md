# Amacunia agent skills

Agent Skills that tell a coding or research agent when and how to call the Amacunia web
services. Each service computes or reads a legal deadline from official public records and
returns the date, the days left and the rule it applies, with a link to the official source.
The services return no personal data about individuals, except that the two French services quote,
as published, the name of the professional designated in the official notice (the court-appointed
representative, or the lawyer, notary or escrow agent who receives objections).

Each skill is a folder with a `SKILL.md` file: when to use the service and when not to, the
exact HTTP and MCP calls, the price per call, how to get a real answer for free (a free sample
and a small daily free trial), how to read the answer, and what it does not guarantee.

Published by Amacunia (amacunia.fr). Information only, not legal advice.

## Install

```bash
npx skills add Amacunia/skills
```

## Skills

| Skill | What it covers | Services |
|---|---|---|
| [`us-carrier-compliance-deadlines`](skills/us-carrier-compliance-deadlines/SKILL.md) | US motor carriers by USDOT number: MCS-150 biennial update deadline, operating authority, insurance on file, FMCSA suspension notices and insurance cancellations | mcs150.amacunia.fr, authority.amacunia.fr |
| [`french-creditor-deadlines`](skills/french-creditor-deadlines/SKILL.md) | French companies by SIREN, from the BODACC: insolvency claim filing deadline, creditor objection deadline after a business sale or merger | creances.amacunia.fr, oppositions.amacunia.fr |

## Planned next

- `us-disaster-deadlines`: FEMA individual assistance deadline and IRS tax postponement by county (disasters.amacunia.fr).
- `us-medicare-revalidation`: Medicare enrollment revalidation due date by NPI (revalidation.amacunia.fr).
- `french-property-deadlines`: CatNat home-insurance claim window by commune, and condominium property manager mandate end (catnat.amacunia.fr, syndic.amacunia.fr).
- `french-pesticide-authorization`: whether a plant protection product can still be sold or used, and until when (phyto.amacunia.fr).

## Payment

Paid calls use the x402 protocol (USDC on Base), per call, with no account. Each skill tells
the agent to ask its user before spending and to try the free sample or free trial first.

## License

MIT.
