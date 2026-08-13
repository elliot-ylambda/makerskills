---
name: domain
description: "Brainstorm and rank domain name candidates by budget and quality; prepare user-run availability, trademark, social-handle, and aftermarket checks."
metadata:
  version: 0.2.0
---

# /domain — Brainstorm and validate domain candidates

Use Laura Roeder's rule: work backwards from names that are both suitable and
actually obtainable; do not fall in love with a name before checking it.

## Hosted Agent execution status

The Hosted Agent may brainstorm, rank, and prepare links. Live registrar,
RDAP/WHOIS, trademark, social-handle, aftermarket, and purchase operations are
`execution_unavailable` until a reviewed typed domain action or an explicitly
advertised attended-browser tool exists.

Never run a registrar CLI, WHOIS client, direct HTTP request, headless browser,
or provider SDK from shell. Never ask for registrar/API credentials or place
them in commands, files, or chat. Do not claim that a domain or handle is
available from memory or from an unverified HTTP status.

Domain purchase is a spend action and is never performed by this skill. The
user must review price, renewal terms, registrant details, and complete checkout
directly with their registrar.

## Step 1 — Establish constraints

Ask for:

- what the product does and the feeling/category it should communicate;
- `.com` only or acceptable alternatives such as `.dev`, `.co`, `.io`, `.ai`;
- maximum first-year purchase price and acceptable renewal price;
- literal/descriptive versus brandable preference;
- words, competitors, or trademark-sensitive terms to avoid.

Useful budget buckets:

| Budget | Candidate policy |
|---|---|
| Registration fee only | Unregistered names only |
| $250–$1,000 | Modest aftermarket names may qualify |
| $1,000–$2,000 | Broader brandable aftermarket set |
| Above $2,000 | Premium candidates, but still filter before preference |

## Step 2 — Generate candidates

Create 20–40 candidates, favoring:

1. Two recognizable words combined naturally.
2. A useful prefix (`try`, `use`, `with`, `join`, `get`, `go`, `hey`, `the`) or
   suffix (`labs`, `hq`, `app`) around a strong category word.
3. A lightly modified real word that remains easy to hear and spell.
4. Invented names only when the sound, spelling, and category signal are clear.

Avoid hard-to-spell strings, generic single-word objects, accidental negative
meanings, and names that depend on another company's mark. Say each candidate
aloud and apply the phone-call test: could someone spell it correctly after
hearing it once?

## Step 3 — Rank before checking

Score each candidate on:

- category/benefit signal;
- memorability;
- spelling and pronunciation;
- differentiation in search;
- likely trademark collision;
- length;
- fit with the stated budget.

Return a short list of 10–15 names. Availability remains `unverified` until the
user supplies a result from a trusted live source.

## Step 4 — Prepare user-run validation

For each shortlisted domain, provide direct, non-signed links the user can open:

- RDAP lookup: `https://rdap.org/domain/<domain>`
- ICANN lookup: `https://lookup.icann.org/en/lookup`
- USPTO search: `https://tmsearch.uspto.gov/`
- HugeDomains: `https://www.hugedomains.com/domain-profile.cfm?d=<label>&e=<tld>`
- Afternic: `https://www.afternic.com/domain/<domain>`
- Sedo search: `https://sedo.com/search/`
- Social profiles: the normal public profile URL for each requested network.

Ask the user to paste or summarize the registrar/RDAP result and the actual
first-year and renewal prices. Treat availability, trademark, and handle checks
as separate facts; one never proves another.

Interpret registration evidence conservatively:

- an explicit live registrar offer or authoritative RDAP "not found" result is
  evidence of availability at that moment, not a reservation;
- a registrar/registration record means taken;
- a parked or marketed name may be purchasable aftermarket;
- disagreement between sources remains `unclear` and requires the registrar's
  checkout page or an authoritative registry lookup.

## Step 5 — Trademark and social screening

For the top 3–5 survivors, ask the user to search the exact phrase and close
variants in the USPTO UI, filter to live marks, and note the goods/services
classes. This is screening, not legal advice; recommend professional clearance
before committing to a material brand investment.

Ask the user to open the relevant social profile URLs while logged in. A public
page status alone can be misleading because several networks return generic app
shells for missing handles.

## Step 6 — Bucket and recommend

Group results into:

- available at normal registration cost;
- aftermarket-listed within budget;
- taken or over budget;
- unclear and needing another authoritative check.

Recommend at most three finalists with concise tradeoffs. Do not recommend a
winner until availability, actual price, spelling, trademark screen, and social
fit have all been reviewed.

## Step 7 — Purchase handoff

When the user chooses a winner, provide a checklist rather than executing:

1. Recheck availability and both first-year and renewal price.
2. Confirm exact spelling and registrant contact details.
3. Review auto-renew, privacy, transfer lock, and aftermarket escrow terms.
4. Complete checkout directly in the registrar/marketplace.
5. Enable MFA and auto-renew, then configure DNS deliberately.

Never turn a request to "buy it" into an automated charge. Explain that the
current Hosted Agent lacks a reviewed domain-purchase action.

## Composes with

- `business-brainstorm` — test the idea before optimizing its name.
- `decide` — compare the verified finalists.
- `skillify` — capture a niche-specific naming rubric, not provider secrets.

## Reference

Laura Roeder's domain-acquisition essay:
https://lauraroeder.com/how-i-nabbed-the-com-for-my-bootstrapped-startup-without-spending-a-million-bucks-6dc35c4606e9
