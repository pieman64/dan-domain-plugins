---
name: comp-domain-research
description: Use when a user reports a domain sale ("X sold on Y marketplace for $Z") and wants 10 similar unregistered names that follow the same syntax. Takes the sale as given, sweeps 120 to 160 candidates for availability, conflict-checks the top 10, builds a market-comps table from live asking prices, suggests pricing, and writes a report with no em dashes. Requires the Unstoppable Domains MCP server for availability checks.
version: "1.2"
license: MIT
---

# Comp-Driven Domain Research

Turn one reported domain sale into a ranked shortlist of 10 unregistered names that follow the same syntax, backed by live availability checks, conflict checks, market comps, and pricing guidance.

## When to use

Trigger on any message shaped like:

- "X.com recently sold on Y for $Z. Find 10 similar names that are not registered."
- "Research names like X that follow the same pattern."
- "What else could I hand-reg in the X pattern?"

The user supplies the comp (domain, marketplace, price). Everything else is derived.

## Requirement: Unstoppable Domains MCP server

This skill does not run without a live availability source. Availability is the entire product. A name that is already registered is worthless to the user, so a guess is worse than no answer.

### Setup

The skill needs the Unstoppable Domains MCP server connected and authenticated, exposing the `ud_domains_search` tool. Depending on the host, the tool is named `ud_domains_search` or is namespaced, for example `mcp__<server>__ud_domains_search`. Match on the `ud_domains_search` suffix.

To connect it in Claude Code:

```bash
claude mcp add --transport http unstoppabledomains https://api.unstoppabledomains.com/mcp/v1
```

Then run `/mcp` in an interactive session and complete the browser sign-in. Authentication is interactive and cannot be completed in a non-interactive session.

Verify with a single known-registered domain before starting a sweep:

```
ud_domains_search(query: "google.com", tlds: ["com"], limit: 1)
```

A result with `available: false` and `status: "registered"` confirms the server is live and authenticated. A 401 or an authentication error means it is not connected.

### If the server is unavailable

Stop and tell the user the server is not connected, with the setup command above. Do not substitute a guess, a recollection, or a bare web search.

Only these sources count as availability evidence, in this order:

1. **Authenticated `ud_domains_search`.** Preferred. The only source that also returns live marketplace asking prices, which Step 5 needs.
2. **The Unstoppable marketplace UI** at `https://domains.unstoppable.ai/marketplace?view=search`. Counts only when the page shows an enabled "Add <domain> to cart" control and a registration price. A page that merely fails to find the name is not evidence.
3. **RDAP**, a 404 for unregistered and a 200 for registered.

```bash
curl -s -o /dev/null -w "%{http_code}" https://rdap.org/domain/ainorsemen.com
```

Opening `https://api.unstoppabledomains.com/mcp/v1` directly in a browser returns an authentication error. That is not an availability check.

If a run falls back to source 2 or 3, say so in the report Method line and in the chat reply, and note that asking-price comps are unavailable from those sources.

## Inputs to extract

| Field | Example | Notes |
|-------|---------|-------|
| Comp domain | AiVikings.com | Keep the casing the user wrote |
| Marketplace | Atom | Where the sale happened |
| Price | $3,700 | Sale price, not asking |
| Sale link | https://www.atom.com/sold/... | Optional. Include it in the report if the user provides one. Do not ask for it |
| TLD | .com | Lock all searches to this TLD |
| Syntax | Ai + plural noun | Derived in Step 2 |

## Workflow

Run the steps in order. Batch every independent call in one turn.

### Step 1. Record the comp

Take the domain, marketplace, and price exactly as the user stated them. Do not spend any tool calls verifying the sale. It is the user's data point and the research does not depend on its provenance.

If the user included a sale link, carry it into the trigger line of the report. If not, omit the link and move on. Never write "user-reported", "unverified", or similar hedging anywhere in the report or the chat reply.

### Step 2. Decompose the syntax

Split the comp into prefix, noun, and TLD. Identify the noun family so candidates stay semantically close. Keep the grammatical number of the comp noun: a plural comp yields plural candidates, a singular comp yields singular candidates.

| Comp noun type | Candidate families to generate |
|----------------|-------------------------------|
| Warrior or historical people (Vikings, Spartans) | Warrior cultures, historical peoples, military ranks and units, mythic beings, explorers and seafarers, sports-team plurals |
| Animal (Wolves, Falcons) | Predators, birds of prey, pack animals, mythical beasts, sports-team plurals |
| Role or profession (Wizards, Pilots) | Sibling roles, ranks, guild names, fantasy classes |
| Object or tech noun (Rockets, Engines) | Sibling objects in the same domain, tools, vehicles |
| Place or nature noun (Summits, Rivers) | Landforms, weather, celestial bodies |
| Service or trade verb (Fix, Wash) | Trade verbs, agent nouns, tools of the trade, service categories, the systems being serviced |

Keep the prefix and the noun form fixed. Do not switch TLD.

### Step 3. Generate 120 to 160 candidates

Write 12 to 16 themed batches of 10 nouns each. Favour nouns that:

- A broad audience recognises without explanation.
- Are 5 to 11 letters so the full name stays under 14 characters.
- Spell one obvious way.
- Carry the same energy as the comp (Vikings implies bold, collective, adventurous).

Include a few reaches (longer or more obscure nouns) so the bench has depth.

Generic, widely developed patterns yield far fewer unregistered names than distinctive ones. If two waves return a thin pool, run a third wave weighted toward more specific nouns rather than shipping a short list.

### Step 4. Check availability

Confirm `ud_domains_search` is reachable before the first wave, per the requirement section above.

- Up to 10 terms per call, `tlds` locked to the comp TLD.
- Run 8 calls in parallel per wave. Two waves cover 160 names.
- Unregistered names return `status: available` with `marketplace.status: available` and a hand-reg price (about $10.67 for .com).

Read the response fields carefully. `available: true` alone does not mean unregistered: it is also true for names listed for sale on Afternic, Sedo, or Unstoppable. The distinguishing field is `marketplace.status`.

| `marketplace.status` | Meaning | Bucket |
|----------------------|---------|--------|
| `available` | Unregistered, hand-reg at the quoted price | Candidate pool |
| `registered-listed-for-sale` | Registered, has an asking price | Comps table |
| `registered-listed-for-offers` | Registered, make-offer only | Comps, no price |
| `registered-not-for-sale` | Registered and held | Reference list |

Never place a name in the top 10 on anything other than `marketplace.status: available`.

### Step 5. Classify every result

Sort all checked names into three buckets:

1. **Unregistered.** The candidate pool.
2. **Listed for sale.** Record asking price and venue. These become the market-comps table.
3. **Registered, not for sale.** List them for reference so the user does not re-check.

Count each bucket. The three counts must sum to the number of candidates checked.

### Step 6. Conflict-check the top 10

For each finalist, run one bounded web search using the spaced phrase and the joined phrase together, for example `"AI Templars" OR "AiTemplars" company OR startup OR trademark`. Allowed domains: linkedin.com, crunchbase.com, x.com, github.com, producthunt.com, trademarkia.com, uspto.gov, justia.com.

Grade each result:

| Grade | Meaning |
|-------|---------|
| Clean | No company or mark using the noun in a related sector |
| Low | Unrelated marks only, or a singular-form company in a different sector |
| Medium | A singular-form company in the same sector as the likely buyer (for example an AI firm named after the noun) |
| High | Exact or near-exact mark in a related class. Drop the name |

Run all 10 searches in one parallel turn.

### Step 7. Rank

Order finalists by, in priority:

1. Conflict grade (Clean beats Low beats Medium).
2. Total characters (shorter wins).
3. Noun familiarity (would a general audience know it without a lookup).
4. Semantic closeness to the comp.
5. Spelling and pronunciation risk.

Apply the skip rules below before ranking.

### Step 8. Write the report and show it

Save the report to the project folder root as `<prefix>-<pattern>-hand-reg-report.md` (for example `ai-nouns-hand-reg-report.md`). Open it in the file panel. If the panel shows stale content after later edits, rename the file to force a fresh tab rather than reopening in raw view.

## Skip rules

Exclude a name even if it is unregistered when:

- The noun names a living ethnic, national, or religious group rather than a historical archetype (Mongols, Persians, Cossacks).
- The noun is a registered game, product, or sports-team mark with no generic meaning (Paladins, Steelers, Outriders).
- The noun carries a negative everyday meaning (Vandals).
- The noun is a strong mark in computing or AI hardware and software (Corsairs, given Corsair Gaming).
- The noun is commonly misspelled (flag rather than drop, and say so in the table).
- The noun is truncated or ambiguous standing alone (Odd from "odd job"), or its everyday sense overrides the intended one (Polish, Spruce).

## Report template

Use these headings and columns exactly. The user has calibrated on this format.

```markdown
# <Prefix>(noun).<tld> Hand-Reg Candidates Modeled on the <Comp> Sale

**Date:** YYYY-MM-DD
**Trigger:** <Comp> sold on <Marketplace> for $<Price>. Add the date and "Sale record: <link>" only if the user supplied them.
**Method:** <N> <prefix> + noun .<tld> candidates checked for availability via <tool> on YYYY-MM-DD. <M> were unregistered. Each of the top 10 was also checked for an existing company or trademark using the exact "<Prefix> <noun>" phrase.

## Top 10 (unregistered, hand-reg at $<price> each on <registrar>)

| # | Domain | Chars | Why it fits the <Comp> pattern | Conflict check |

Total to register all 10: about $<sum>.

## Bench: <K> more unregistered names, ranked

**Strong alternates:** ...
**Playable but longer or more obscure:** ...
**Unregistered but skip:** bullet list with the reason for each.

## Market comps (asking prices, not sales)

One sentence saying these bracket the sale. Then a table: Domain | Asking | Venue, sorted ascending. Then a line for make-offer-only names. Then one sentence naming the realistic mid-band and where the sale sits in it.

## Suggested pricing if registered

- Tier 1 (top 5): BIN, minimum offer, LTO term.
- Tier 2 (next 5): BIN, minimum offer, LTO term.
- Where to list.
- Time horizon.

## Registered and not for sale (<count> names, for reference)

Comma-separated nouns only.

## Caveats

- Availability was checked once on <date>. Re-check before buying.
- Conflict checks were web-level only. A formal trademark search is advisable before listing at four figures.
```

## Pricing guidance

- Bracket the sale with the live asking prices from Step 5. The mid-band is where most well-known nouns in the pattern are listed.
- Tier 1 BIN sits just above the sale price. Tier 2 BIN sits just below it.
- Minimum offer is about half of BIN.
- Lease-to-own 12 months for anything under five figures.
- List on the marketplace where the comp sold, plus Afternic and Unstoppable Domains. If one venue holds most of the live inventory in the pattern, name it.
- If the user has a stated price-ending convention, follow it. Otherwise end prices in 65 (for example $3,965) to stay under the next round number.
- Hand-reg cost is trivial. The real cost is renewal float over a 12 to 24 month horizon. Say so.

## Writing rules

These are non-negotiable for both the report and the chat reply.

- No em dashes anywhere, including the title. Use a comma, a period, or parentheses instead.
- No hedging about the sale figure. Take it as given and never write "reported" or "unverified".
- One idea per sentence, about 20 words, with a verb.
- Tables for parallel data. Bullets for parallel items. Prose for argument.
- No emoji. No motivational filler. No closing offer.

## Chat reply structure

The final chat message stands alone. Lead with the outcome and the file name, then:

1. The top 10 as a numbered list, one line each: domain, why it fits, conflict grade.
2. A short "What the research covered" block: candidates checked, split across the three buckets, what the bench and comps show, the pricing suggestion.
3. One or two caveats.
4. A "Sources" line with markdown links to the pages used for conflict checks.

## Quality bar

A run meets the bar when all of the following are true:

- `ud_domains_search` was confirmed reachable before the sweep, or the fallback source used is named in the report and the chat reply.
- At least 120 candidates were checked and the count is stated.
- Every one of the 10 finalists returned `marketplace.status: available` on a recorded check in this run.
- No availability claim rests on recollection, inference, or a name's obscurity.
- Every finalist has a conflict grade with a one-line reason.
- The comps table has at least 12 rows or every listed name found, whichever is smaller.
- The bench, skip list, and registered list are all present, and the three bucket counts sum to the number checked.
- Any count stated in a heading matches the items actually listed under it.
- Zero em dashes in the file. Verify with a search for the character before showing the file.

## Worked examples (calibration)

### Plural-noun pattern

Comp: AiVikings.com sold on Atom for $3,700 on September 16, 2026. Record: https://www.atom.com/sold/26948-f027528b07efef8d

Run: 160 Ai + plural-noun .com names checked. 52 unregistered, 37 listed for sale, 71 registered and held.

Top 10 delivered: AiRomans, AiNorsemen, AiShoguns, AiTemplars, AiCenturions, AiWarlords, AiSultans, AiPharaohs, AiValkyries, AiBarbarians. All hand-reg at $10.67.

Comps found: asking prices for well-known warrior and people nouns clustered between $2,700 and $5,600 (AiRangers $2,699, AiPirates $2,988, AiGladiators $4,299, AiWizards $4,588, AiCommanders $4,888, AiScouts $5,399, AiConquerors $5,599). The $3,700 sale sat in the middle of that band.

Pricing suggested: Tier 1 BIN $3,965, Tier 2 BIN $2,965, minimum offers $1,965 and $1,465, LTO 12 months, listed on Atom, Afternic, and Unstoppable Domains.

Skips applied: AiCorsairs (Corsair Gaming), AiPaladins (Hi-Rez game), AiMongols, AiPersians, AiCossacks (living groups), AiVandals (negative), AiBroncos, AiSteelers, AiBengals (NFL marks), AiOutriders (Square Enix title).

### Singular service-noun pattern

Comp: LocalFix.com sold on GoDaddy for $3,226.

Run: 200 Local + service-noun .com names checked across three waves. 36 unregistered, 72 listed for sale, 92 registered and held. The pattern is heavily developed, so two waves returned a thin pool and a third wave weighted toward trade-specific nouns was needed.

Top 10 delivered: LocalWeld, LocalGrout, LocalRenew, LocalRefit, LocalCaulk, LocalSewer, LocalMender, LocalSolder, LocalBreaker, LocalMaintain. All hand-reg at $10.67, all Clean.

Comps found: single trade nouns clustered between $2,450 and $4,995 (LocalDrywall $2,450, LocalChore $2,688, LocalTune $2,788, LocalBodyshop $2,999, LocalTile $3,195, LocalDrain $3,695, LocalHandy $4,395, LocalCarpenter $4,500, LocalFloor $4,995). The $3,226 sale sat in the middle. Most live inventory in this pattern sits on Sedo rather than the venue where the comp sold.

Skips applied: LocalSpruce (tree reading), LocalOdd (truncated, adjective), LocalPolish (nationality collision).

## Tooling notes

- Web search in some environments requires an `allowed_domains` list or it errors.
- Run availability batches and conflict searches in parallel. A full run fits in four tool turns: wave one, wave two plus conflict checks, write plus show, memory save.
- Save two workspace memory notes at the end: the comp (with its link if given) and the comps band, and the report file location.
