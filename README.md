# TeleStar ICP Qualifier Bot

Automated Ideal Customer Profile (ICP) qualification bot for TeleStar's B2B SaaS lead generation. Send a list of company websites via Telegram, get back PASS/FAIL/REVIEW verdicts with structured reasoning.

Built by the Lead Generation team at [TeleStar](https://telestar.vn) to replace manual ICP screening of prospective clients.

## What It Does

1. Receive a list of company websites (text message or spreadsheet upload) via a Telegram bot
2. Check each website against a Google Sheets cache (Qualified, Not Qualified, Do Not Contact)
3. For uncached websites: run a Serper Google search, then send results to Claude AI for ICP evaluation
4. Return verdicts via Telegram (inline message for <=10 results, CSV file for larger batches)
5. Store all results in Google Sheets for future cache hits

**Key outcome:** Reduced per-company ICP screening time from ~7 minutes of manual research to ~15 seconds of automated evaluation.

## Architecture

The system is split into two N8N workflows to isolate input/output handling from the core evaluation loop:

```
                        MAIN WORKFLOW (15 nodes)
                        ========================
Telegram ──> Parse Input ──> Deduplicate ──> Run ICP Core (sub-workflow call)
                                                    │
                                              <=10 results? ──> Send text message
                                              >10 results?  ──> Send CSV file


                        SUB-WORKFLOW: ICP Core (23 nodes)
                        ================================
Receive websites ──> Read cache (Qualified + Not Qualified + DNC tabs)
                         │
                    Pack Cache (match each URL against cached results)
                         │
                    Loop Over Websites (one at a time)
                         │
                    ┌────┴────────────────┐
                    │                     │
               Cache hit?            Not cached?
                    │                     │
              Use cached result     Serper Search
                    │                     │
                    │               Parse Serper results
                    │                     │
                    │               Claude AI Agent
                    │               (ICP evaluation)
                    │                     │
                    └────┬────────────────┘
                         │
                    Save Result ──> Write to Google Sheets
                         │            (Qualified or Not Qualified tab)
                    Rate Limit 2s
                         │
                    Next website...
                         │
                    Return all results to main workflow
```

### Why Two Workflows?

The original v1 was a monolithic 26-node workflow. Splitting into main + sub-workflow solved two problems:

1. **Separation of concerns** - Input parsing, output formatting, and Telegram I/O stay in the main workflow. The core evaluation loop is isolated and testable.
2. **Sub-workflow boundary awareness** - N8N strips internal/metadata fields when items cross the Execute Sub-workflow boundary. By designing around this constraint from the start, we avoid the bugs that plagued v1 (see [Lessons Learned](#lessons-learned)).

## Tech Stack

| Component | Purpose |
|---|---|
| **N8N** (self-hosted) | Workflow orchestration |
| **Telegram Bot API** | Input/output interface |
| **Serper API** | Google search for company information |
| **Claude Sonnet 4.5** (Anthropic API) | ICP evaluation with structured JSON output |
| **Google Sheets** | Result cache + history + DNC list |

## ICP Qualification Framework

The bot evaluates companies against 16 disqualifier codes (NQ1 through NQ16). A company that triggers zero disqualifiers gets a PASS verdict.

| Code | Category | Example Trigger |
|---|---|---|
| NQ1 | Extreme headcount | <=10 or >=1,000 employees |
| NQ2 | Not real B2B buyers | Dating apps, personal finance apps, micro-business POS |
| NQ3 | Non-commercial entity | NGOs, government agencies, research initiatives |
| NQ4 | Website inaccessible | Broken, placeholder, or parked domains |
| NQ5 | Company shut down | No longer operating |
| NQ6 | Fully acquired | Brand merged into parent company |
| NQ7 | Services-first / no IP | IT consulting, staffing, dev agencies, marketing agencies |
| NQ8 | SDR/lead-gen services | Appointment setting agencies, cold outreach agencies |
| NQ9 | Pure BPO | Call center operators, outsourcing operators |
| NQ10 | Marketplace model | Transaction commission, not SaaS subscription |
| NQ11 | India HQ + India workforce | India-majority operations |
| NQ12 | Market mismatch | HQ in blocked regions (Vietnam, Pakistan, mainland China, etc.) |
| NQ13 | Core crypto/Web3 | Crypto exchanges, DeFi, NFT marketplaces, blockchain infra |
| NQ14 | Hardware primary | Revenue from hardware, not software |
| NQ15 | Traditional software | One-time license, no recurring SaaS subscription |
| NQ16 | Hard-to-reach industry | AgriTech, Maritime, Public Sector (RFP-only) |

Notable exceptions built into the framework:

- **EdTech exception** (NQ2): LMS/assessment platforms sold to universities via institutional procurement are valid B2B
- **MSP exception** (NQ7): Companies with their own proprietary platform + managed operations are not services-first
- **Crypto plumbing** (NQ13): Blockchain as underlying tech for supply chain, KYC, etc. is evaluated by actual vertical
- **Virtual address detection**: Companies with Western HQ but all operations in blocked countries get REVIEW, not auto-PASS

The full prompt with all exception logic and counter-examples is in [`prompts/icp-qualifier-v3.md`](prompts/icp-qualifier-v3.md).

## Cache System

The cache prevents re-evaluating websites that have already been scored.

**Read path:** At the start of each batch, the sub-workflow reads ALL rows from Qualified, Not Qualified, and DNC tabs (3 Google Sheets reads, each `executeOnce`). Pack Cache does in-memory URL matching against each input website.

**Write path:** After AI evaluation, fresh results are appended to the appropriate sheet tab. Cached results skip the write entirely.

**Priority order:** DNC > ICP cache. A website in both DNC and Qualified is treated as DNC FAIL, with context like `DNC . Active Account . Jul 15 . John Smith`.

**Cache invalidation:** Manual. Delete the row from the sheet to force re-evaluation.

## Telegram Output Format

For small batches (<=10 websites), results are sent as a formatted text message:

```
ICP Check (5 websites)

✅ acme.io - B2B SaaS, project management platform for agencies, 85 employees US-HQ
✅ datavault.com - Cybersecurity SaaS, data loss prevention, 120 employees UK-HQ

🔵 newcorp.io - Possible virtual address, verify genuine US operations

❌ quickfix.app - NQ2: On-demand home repair marketplace for consumers
❌ blockdex.io - NQ13: Decentralized crypto exchange platform

✅ 2 PASS | 🔵 1 REVIEW | ❌ 2 FAIL
```

For larger batches (>10), results are sent as a CSV file attachment.

## Key Design Decisions

### Sub-workflow boundary handling

N8N's Execute Sub-workflow node strips internal fields from returned items. Fields like `_chat_id` and `_total_unique_count` are available inside the sub-workflow but disappear when items return to the main workflow.

**Rule:** In the main workflow, never reference fields set inside the sub-workflow via `$json.fieldName`. Instead:
- Chat ID: reference `$('Telegram Trigger').item.json.message.chat.id` directly
- Item count: use `$input.all().length` instead of `$json._total_unique_count`

### Rate limiting

A 2-second delay between each website evaluation prevents hitting Serper and Anthropic API rate limits. This makes batch processing sequential but reliable.

### Disqualifier validation

The Save Result node validates AI-returned disqualifier codes against a hardcoded `ALLOWED_CODES` array (NQ1-NQ16). Invalid codes get a warning appended to the reasoning rather than being silently dropped.

### Timestamp handling

Timestamps use manual UTC+7 offset (`new Date(Date.now()+25200000)`) since the N8N instance runs on a VPS without timezone configuration.

## Known Limitations

- **Headquarters confusion:** The bot sometimes misidentifies a company's true HQ location, especially for companies with virtual addresses in Western countries but actual operations elsewhere. The virtual address detection catches some cases but is not 100% reliable.
- **No automation of edge cases:** Companies flagged as REVIEW still require manual verification. The bot is a significant step forward from fully manual screening, but expert judgment is still needed for borderline cases.
- **Cache reads all rows:** Each batch reads the entire Qualified, Not Qualified, and DNC tabs. This works fine at current scale but would need pagination or a database backend at higher volumes.
- **Sequential processing:** The 2-second rate limit means large batches take proportionally longer. A batch of 50 websites takes ~2 minutes.

## File Structure

```
telestar-icp-qualifier/
├── README.md
├── prompts/
│   └── icp-qualifier-v3.md          # Full ICP evaluation prompt with 16 NQ codes
├── n8n/
│   ├── main-workflow.json           # Telegram I/O + parsing + output formatting
│   └── icp-core-sub-workflow.json   # Cache + Serper + AI eval + sheet write
└── assets/                          # Screenshots (coming soon)
```

## Lessons Learned

### Bug: Count Check always sending CSV (fixed)

**Symptom:** All batches went to CSV path regardless of size.

**Root cause:** Count Check used `$json._total_unique_count` which was set inside the sub-workflow. After crossing the Execute Sub-workflow boundary, this field was `undefined`. With `typeValidation: strict`, `undefined <= 10` evaluates to FALSE.

**Fix:** Changed to `$input.all().length` which counts the actual items received by the main workflow.

### Bug: Empty chat ID on Send Text (fixed)

**Symptom:** Telegram Send Text node had empty chatId.

**Root cause:** Same sub-workflow boundary issue. `_chat_id` was set in the parser nodes, passed through the sub-workflow, but stripped on return.

**Fix:** Changed chatId to `$('Telegram Trigger').item.json.message.chat.id`, referencing the trigger node directly in the main workflow.

### General rule

Any field you need AFTER the sub-workflow returns must either be part of the scored output object (explicitly returned by the sub-workflow) or referenced from a node BEFORE the sub-workflow call (using `$('NodeName')` syntax).

## Future Improvements

- **Phase 2 headcount enrichment:** Apify LinkedIn company scraper for structured employee count data (only if "Unknown" headcount rate is too high in practice)
- **Pricing page cache:** Fetch and cache pricing page content for better NQ evaluation
- **LinkedIn employee locations:** For improved virtual address detection
