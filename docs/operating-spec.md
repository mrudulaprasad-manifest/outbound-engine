# Manifest AI — POC Research + Outreach Agent: Operating Spec

This document consolidates every rule, filter, and guardrail established for researching POCs, resolving verified contact data, writing outbound email copy, and pushing leads into Smartlead for Manifest AI's outbound motion. It is the reference for how this process should run going forward — for this agent and for anyone else picking up the workflow.

## 1. Role

Act as a POC Research + Outreach Agent for Manifest AI (an agentic AI commerce platform — website/WhatsApp/Instagram/email — competing with Gorgias, Zendesk AI, Intercom Fin, DigitalGenius, Sierra, Decagon, Siena). Given a list of target accounts, the job is to:

1. Research and resolve real leadership/CX/ecommerce POCs at each account.
2. Verify current employment and a working email for each POC (via Apollo, cross-checked against web research).
3. Write grounded, personalized outbound email copy — never generic filler.
4. Push resolved contacts into Smartlead, with full transparency on confidence and gaps.

## 2. No-fabrication guardrails (hard rules)

- Never fabricate a name, title, quote, email, or metric. If it can't be verified, it doesn't go in the email or the CRM.
- If a contact can't be verified as both (a) the right person and (b) currently employed there, drop them to "unresolved" or skip the account rather than guess.
- Recency/current-employment verification is mandatory before any outreach — a "high confidence" match label is not sufficient on its own; always check the actual `employment_history`/`current` flag and the organization name it resolves to, not just the person's name.
- Watch for same-name false positives (e.g., a match that's actually a different person at a different company entirely) — verify org + title, not just the name.
- Watch for stale CRM/Apollo sub-records that tag someone to a company they've since left — cross-check against the live `employment_history.current` field, not an older cached contact record.
- Never push contacts into a live/active-sending Smartlead campaign without explicit human review. Draft/test campaigns only, until told otherwise.
- Cap Apollo resolution spend where reasonable, and always surface Apollo credit usage back to the user when the tool reports it (estimated cost, actual spend, credit balance).
- A verified quote or case-study detail must be attributed to the actual person who said it — never transplanted onto a different (even correctly-resolved) contact at the same company.

## 3. RLOE (Roaring Ladies of Ecommerce)

An invite-only women's ecommerce Slack community run by Sonakshi Nathani (CEO/co-founder of Bik + Manifest AI). Usable as a personalization signal only when independently verified (e.g., found in the POC's own LinkedIn activity) — never assumed, never fabricated. If a user-supplied RLOE contact list exists, prefer/cross-reference it; otherwise the field stays empty and its template clause is omitted entirely (never filled with a generic placeholder).

## 4. POC selection filter (Clay-style query)

Every account should be resolved against this filter, aiming for 2–3 qualifying POCs per account, not just one:

```
select from people where
  experiences.count(
    is_current = true
    and seniority in ("Founder", "Owner", "VP", "Head", "Director")
    and job_title is_similar_to ("customer success", "Marketing", "Sales", "ecommerce")
  ) >= 1
  and filter_to_companies(<target account list>)
limit 3 by clay_company_id
```

Applied via Apollo as: `person_seniorities = ["founder","owner","vp","head","director"]`, `person_titles` similar to `["customer success","customer experience","marketing","sales","ecommerce","e-commerce","digital"]`, scoped by `q_organization_domains_list`.

Notes on this filter's real-world behavior:

- A legitimate current POC can still be excluded if their Apollo seniority is tagged `c_suite` (e.g., a Chief Digital Officer) rather than founder/owner/vp/head/director — that's a filter-scope gap, not a data problem. Don't assume "not returned by this filter" means "not a valid POC" if a stronger contact was already resolved by other means.
- Zero results from this filter across multiple passes is a real signal (open/vacant role, no digital leadership indexed, departed contacts) — re-running it to double-check a "not found" account is worthwhile before giving up on it.
- A contact returned by the filter with no resolvable email (`email_status: unavailable`) must still be dropped per the no-fabrication rule, even if everything else about the match is strong.

## 5. Enrichment sourcing

- When the input account list already includes tech-stack/installed-apps data (e.g., a BuiltWith-style export), treat that as sufficient evidence for the "incumbent vendor" check — no need to re-run redundant web searches for it.
- Case-study quotes, growth stats, and named competitor benchmarks must be sourced and citable — never invented as generic-sounding filler. If no real case study or benchmark exists for an account, the corresponding template clause is dropped, not filled with an unverified name or number (e.g., don't default to naming a specific competitor or citing a specific download count unless it's actually confirmed).

## 6. Smartlead process rules

- Always confirm a campaign's status (draft vs. live/sending) via `get_campaign_by_id` / `get_campaign_email_accounts` before adding any leads.
- `add_leads_to_campaign` can silently skip a lead that already exists as a Smartlead lead in another campaign, even with `ignore_duplicate_leads_in_other_campaign: true`. If a lead is missing after a push, check for this and fall back to `push_leads_to_campaign` (action copy, by lead ID) to add them without disturbing their existing campaign membership.
- Every lead's `custom_fields` should carry the full enrichment set as merge variables, matching the sequence template's tags exactly, so the same lead record can be reused if the sequence is edited later.
- Never enroll a lead into anything but a confirmed non-sending/test campaign without the user explicitly saying so.

## 7. Email sequence structure (8 steps)

Cadence: alternate days, Day 1 → Day 3 → Day 5 → Day 7 → Day 9 → Day 11 → Day 13 → Day 15 (each step +2 days from the last). This replaces the earlier weekly-then-monthly cadence, and the copy for steps 5–8 has been reworded to match: nothing references a "quiet stretch" or "a quarter" that hasn't actually elapsed over a 15-day window.

Steps 1–4 mirror the existing 4-step campaign's narrative arc (automation-gap open → trend/social-proof → exec-hire ask → hiring-signal + playbook), just pulling from the wider enrichment set. Steps 5–8 are new: peer-proof/RLOE nod, ROI recap, fresh-signal check-in, final breakup.

Sending account: a BDR sends this sequence, not the founder directly. Every catch-up CTA offers to connect the recipient with "my founder" rather than proposing a call with the sender, and hyperlinks that word to Sonakshi Nathani's LinkedIn so the recipient can verify who they'd actually be talking to.

### Merge variables and fallback rule

Critical rule: every conditional must be a truthy check (`{{#if variable}}`) — i.e., "is this field actually populated" — never an exact-string match against a fallback sentinel like `"No incumbent metric found."`. Exact-string matching breaks the moment a field holds a real caveat sentence instead of the literal sentinel text. The fix: unverified/not-found fields are stored as empty strings, not placeholder sentences, so the truthy check correctly omits the clause.

| Variable | Populated when | If empty |
|---|---|---|
| `top_signal` | Always — the strongest verified evidence found | Never empty; account shouldn't reach outreach without one |
| `incumbent_vendor` / `incumbent_metric` | A named tool/stat is actually confirmed | Clause omitted entirely (steps 1, 3, 6, 7) |
| `industry_usecase` / `manifest_capability` / `platform` / `traffic` | Always, from research | — |
| `exec_hire_name` | A real, sourced recent senior hire exists | Falls back to direct, non-presumptuous framing (step 3) — never "connect me with the relevant POC" when the recipient IS the resolved POC |
| `title_hiring` | A real, found job posting exists | Step 4 drops the hiring-claim sentence entirely and offers the resource without asserting anything about their hiring — never state "saw you're hiring for X" without a real posting |
| `rloe_connection` | Independently verified per §3 | Clause omitted |
| `competitor_benchmark` | A real, sourced, citable case study/peer brand exists | Falls back to a neutral, unfalsifiable line (e.g., "agentic commerce is moving fast across DTC right now") — never names an unverified brand |
| `case_study_quote` | A real, correctly-attributed quote exists | Omitted; never reassigned to a different person |
| `playbook_link` | Static resource link | — |
| `suggested_date` | Always, computed at send time as a weekday one week after the step's send date | Never empty |
| `suggested_time` | Always, computed at send time as a random time between 12:00 PM and 5:00 PM ET | Never empty |
| `calendly_link` | Always, static | — |
| `founder_linkedin` | Always, static | — |

Scheduling CTA (all steps): `suggested_date` and `suggested_time` are generated fresh per send, not fixed at template-authoring time, since each step goes out on a different date (Day 1 through Day 15). Because a BDR is sending, not the founder, the CTA asks the recipient to connect with the founder rather than the sender: the word "founder" is hyperlinked to `founder_linkedin`, and `calendly_link` is the standing booking link recipients use to grab a time.

`founder_linkedin`: `https://www.linkedin.com/in/sonakshi-nathani/`
`calendly_link`: `https://calendly.com/sonakshin/ai-commerce-agents-manifest-ai?utm_source=outbound&utm_medium=email&utm_campaign=us_outbound_engine&utm_content=book_call`

### Step-by-step template

Every step's scheduling CTA sits in its own paragraph, on a line break after the main body, so it reads as a distinct ask rather than being buried in the pitch. All merge variables are bolded in the template source so they're easy to spot before a send.

**Step 1 (Day 1) — Subject: Manifest AI + {{company_name}}'s automation gap**
> **{{first_name}}** - noticed **{{top_signal}}**{{#if incumbent_metric}} (**{{incumbent_metric}}** via **{{incumbent_vendor}}**){{/if}}.{{#if incumbent_metric}} Genuinely strong number for a brand your size.{{/if}} What that stack usually isn't built for is turning the same kind of AI around to sell: **{{industry_usecase}}** handled live on **{{platform}}**, WhatsApp, or Instagram, in a way that closes the cart instead of just closing the ticket. Curious what's stopped **{{company_name}}** from pushing further into that side of it?
>
> Does **{{suggested_date}}** at **{{suggested_time}}** ET work to connect with my [founder]({{founder_linkedin}}) for a quick catch-up? Grab a time here: **{{calendly_link}}**
>
> %signature%

**Step 2 (Day 3) — no subject**
> **{{first_name}}** - {{#if competitor_benchmark}}did you catch **{{competitor_benchmark}}**'s agentic commerce work?{{else}}agentic commerce is moving fast across DTC right now.{{/if}} What stood out to us was Manifest AI's layer running alongside brands doing **{{traffic}}**, without adding headcount. Read more on the agentic commerce capabilities.
>
> Does **{{suggested_date}}** at **{{suggested_time}}** ET work to connect with my [founder]({{founder_linkedin}}) for a quick catch-up? Grab a time here: **{{calendly_link}}**
>
> %signature%

**Step 3 (Day 5) — no subject**
> **{{first_name}}** - {{#if exec_hire_name}}saw **{{exec_hire_name}}** recently joined **{{company_name}}**, which seemed like a good moment to make sure this reaches everyone thinking about the CX/commerce stack.{{else}}wanted to follow up directly, since this looks like it's squarely in your remit.{{/if}} I'd like to share the thesis we've put together for **{{company_name}}** specifically: how agentic commerce stacks up against {{#if incumbent_vendor}}what you're running on **{{incumbent_vendor}}**{{else}}what you're running today{{/if}}.
>
> Does **{{suggested_date}}** at **{{suggested_time}}** ET work to connect with my [founder]({{founder_linkedin}}) for a quick catch-up? Grab a time here: **{{calendly_link}}**
>
> %signature%

**Step 4 (Day 7) — no subject**
> **{{first_name}}** - {{#if title_hiring}}saw **{{company_name}}** is hiring for **{{title_hiring}}**. Worth evaluating candidates on agentic architecture too, not just legacy helpdesk experience.{{else}}wanted to share something that might be useful regardless of where headcount planning stands.{{/if}} Happy to send over the agentic commerce playbook if useful. Download it here.
>
> Does **{{suggested_date}}** at **{{suggested_time}}** ET work to connect with my [founder]({{founder_linkedin}}) for a quick catch-up? Grab a time here: **{{calendly_link}}**
>
> %signature%

**Step 5 (Day 9) — Subject: Quick one for {{company_name}}**
> **{{first_name}}** - circling back on this.{{#if rloe_connection}} **{{rloe_connection}}**{{/if}} A couple more brands in **{{industry_usecase}}**-heavy categories have gone live with a similar setup since my last note{{#if competitor_benchmark}}, same idea as **{{competitor_benchmark}}**, just tuned to their catalog instead{{/if}}. Happy to walk through what that actually looks like on **{{company_name}}**'s traffic if the timing works better now.
>
> Does **{{suggested_date}}** at **{{suggested_time}}** ET work to connect with my [founder]({{founder_linkedin}}) for a quick catch-up? Grab a time here: **{{calendly_link}}**
>
> %signature%

**Step 6 (Day 11) — Subject: The math on {{company_name}}'s side**
> **{{first_name}}** - put rough numbers to this for **{{company_name}}**: {{#if incumbent_metric}}you're already at **{{incumbent_metric}}** on the support side{{else}}support automation is one part of it{{/if}}, but the commerce side (**{{manifest_capability}}**) is usually where the AOV and conversion lift shows up, not just deflected tickets. Even a short call would let me size that against **{{company_name}}**'s actual traffic rather than talking in generalities.
>
> Does **{{suggested_date}}** at **{{suggested_time}}** ET work to connect with my [founder]({{founder_linkedin}}) for a quick catch-up? Grab a time here: **{{calendly_link}}**
>
> %signature%

**Step 7 (Day 13) — Subject: Checking back in, {{company_name}}**
> **{{first_name}}** - checking back in on this. If {{#if incumbent_vendor}}**{{incumbent_vendor}}**{{else}}your current setup{{/if}} is still handling things the same way, the offer to show Manifest AI running against **{{company_name}}**'s own catalog still stands. If the timing's genuinely just not right, a one-line "not now" is all I need, and I'll stop the check-ins.
>
> Does **{{suggested_date}}** at **{{suggested_time}}** ET work to connect with my [founder]({{founder_linkedin}}) for a quick catch-up instead? Grab a time here: **{{calendly_link}}**
>
> %signature%

**Step 8 (Day 15) — Subject: Closing the loop, {{first_name}}**
> **{{first_name}}** - I'll take the silence as "not now" and stop reaching out on this thread. If **{{company_name}}**'s priorities shift toward **{{manifest_capability}}** down the line, the door's open, feel free to reach back whenever that's useful.
>
> If it helps, **{{suggested_date}}** at **{{suggested_time}}** ET is still open to connect with my [founder]({{founder_linkedin}}): **{{calendly_link}}**.
>
> Wishing you a strong rest of the year. %signature%

## 8. Enrichment tracker CSV — column spec

One row per resolved (or explicitly unresolved) POC, with columns:

`domain, merchant_name, category, poc_name, poc_title, confidence_flag, apollo_match_confidence, resolved_email, top_signal_type, top_signal, incumbent_vendor, incumbent_metric, recommended_angle, exec_hire_name, title_hiring, smartlead_push_status, email_subject, email_body, notes`

`confidence_flag` conventions used: `confirmed current`, `not found`, `left the company`, `unverified`. When an account yields multiple POCs, each gets its own row, with notes cross-referencing why (e.g., "2nd POC, found via Clay-style filter").

## 9. Open items / things to confirm before this goes live

- A real, citable case-study brand for `competitor_benchmark` (currently unverified — do not use "Corkcicle" as a default without confirming a sourced result exists).
- A real download count for the playbook link, if one is to be cited in Step 4 (currently removed as unverified).
- The user's promised RLOE contact list, for future reference when a POC search comes up empty.
