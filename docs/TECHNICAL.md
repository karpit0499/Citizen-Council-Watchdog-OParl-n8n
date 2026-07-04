# Technical Documentation — Citizen Council Watchdog (OParl)

This document describes the internal design of the project: the two workflows,
what each node does, how data moves, and why the key decisions were made. It is
written for someone who wants to understand, run, or extend the project.

---

## 1. Goal and constraints

**Goal:** watch a German city council's public document feed and email residents a
plain-language heads-up whenever a new paper touches a topic or street they care
about — without sending them the same alert twice.

**Constraints that shaped the design:**

- **Local-first inference.** All summarising and quality-checking runs on a
  self-hosted Ollama model. The only outbound call is to the council's public
  OParl endpoint, so nothing sensitive leaves the machine.
- **Free, keyless data.** OParl is a shared open format many German councils use to
  publish documents as clean JSON over a normal web address. No API key, and it
  only exposes what is already public.
- **Trust matters.** This is a citizen-facing product, so a wrong or invented
  summary is worse than no summary. A second AI pass exists specifically to catch
  that before anything is emailed.

---

## 2. The two workflows

The project is split into two n8n workflows plus two shared Data Tables:

1. **Subscribe** — a one-node-plus-form workflow that records a resident's
   subscription into `watch_topics`.
2. **Watcher** — the scheduled workflow that does the real work: fetch → filter →
   summarise → check → keep → remember → digest → send.

### Shared Data Tables

`watch_topics` — who is watching what:

| Column        | Type   |
|---------------|--------|
| subscriber    | String |
| keyword       | String |
| city_endpoint | String |

`seen_papers` — papers already handled (the anti-spam memory):

| Column       | Type   |
|--------------|--------|
| paper_id     | String |
| processed_at | String |

---

## 3. Subscribe workflow

```
On form submission (Council Watch)
        │  fields: Your email · Keyword to watch · City OParl address
        ▼
Insert Row → watch_topics
```

- **On form submission** — titled *Council Watch*, with three text fields: `Your
  email`, `Keyword to watch`, and `City OParl address` (the last can be pre-filled
  with your default city).
- **Insert Row** into `watch_topics`, mapping:
  - `subscriber` = `{{ $json["Your email"] }}`
  - `keyword` = `{{ $json["Keyword to watch"] }}`
  - `city_endpoint` = `{{ $json["City OParl address"] }}`

Each submission creates one subscription row.

---

## 4. Watcher workflow

```
Schedule Trigger (daily, e.g. 07:00)
   │
   ├───────────────┐
   ▼               ▼
 Topics         SeenPapers          (Data Table → Search Rows)
 (watch_topics) (seen_papers)
   │
   ▼
HTTP Request  (GET {{ $json.city_endpoint }})   ← OParl /papers
   │
   ▼
Filter (Code)  ── reads HTTP Request + Topics + SeenPapers
   │            keep only new + keyword-matching papers
   ▼
Summarise (AI Agent) ◄──── Ollama Chat Model (qwen3:14b)
   │            plain-language summary, < 60 words
   ▼
Check (AI Agent) ◄──────── Ollama Chat Model (qwen3:14b)
   │            returns JSON { faithful, relevant }
   ▼
Keep Good (Code) ── drop unfaithful or irrelevant summaries
   │
   ▼
Insert Row → seen_papers   (remember paper_id + processed_at)
   │
   ▼
Digest (Code) ── group into one email per subscriber
   │
   ▼
Send Email  (To / Subject / Text)
```

### 4.1 Schedule Trigger
Fires once a day (e.g. 07:00). Both Search Rows nodes connect off it, so each run
starts by loading current state.

### 4.2 Topics / SeenPapers — Data Table Search Rows
- **Topics** reads all rows of `watch_topics` (who is watching what).
- **SeenPapers** reads all rows of `seen_papers` (what's already been sent).

### 4.3 HTTP Request
`GET` with URL set as an expression to `{{ $json.city_endpoint }}`, so it fetches
the council papers feed for the subscriber's city. The OParl response contains a
`data` list; each paper has an `id`, a `name` (title) and a `date`.

### 4.4 Filter — Code node
Drops papers already sent and keeps only titles that mention a subscriber's keyword:

```js
const papers = $('HTTP Request').first().json.data || [];
const topics = $('Topics').all().map(r => r.json);
const seen = new Set($('SeenPapers').all().map(r => r.json.paper_id));
const hits = [];
for (const p of papers) {
  if (seen.has(p.id)) continue;
  const title = (p.name || '').toLowerCase();
  for (const t of topics) {
    if (title.includes((t.keyword || '').toLowerCase())) {
      hits.push({ json: { paper_id: p.id, title: p.name, date: p.date,
        subscriber: t.subscriber, keyword: t.keyword } });
      break;
    }
  }
}
return hits;
```

The `seen` check is the anti-spam guard — without it, residents would get the same
alert every single day.

### 4.5 Ollama Chat Model
Model `qwen3:14b`, Base URL `http://host.docker.internal:11434`. Shared as the
brain for both the Summarise and Check agents.

### 4.6 Summarise — AI Agent
Text = `Paper title: {{ $json.title }}`. System message instructs the model to
explain a council document to ordinary residents in plain, neutral German (or the
reader's language), in under 60 words: one sentence on what it's about, one on why
a resident might care. No opinions, no guessing beyond the title.

### 4.7 Check — AI Agent (the built-in guard)
A second pass that gates quality before anyone is emailed. Text supplies the
`TITLE`, matched `KEYWORD`, and the `SUMMARY` (`{{ $json.output }}` from Summarise).
Its system message starts with `/no_think` and asks for a strict verdict:

```
Return ONLY JSON: { "faithful": true/false, "relevant": true/false }
```

- **faithful** — does the summary only state things supported by the title, with no
  opinion?
- **relevant** — is the keyword a genuine topical match, not a coincidental word?

### 4.8 Keep Good — Code node
Attaches the summary to the record and drops anything that failed either check:

```js
const hit = $('Summarise').first().json; // title, date, subscriber, keyword
const summary = (hit.output || '').trim();
const raw = $input.first().json.output || '';
let ok = false;
try { const c = JSON.parse(raw.slice(raw.indexOf('{'), raw.lastIndexOf('}')+1));
  ok = c.faithful === true && c.relevant === true; } catch(e) {}
if (!ok) return []; // drop it
return [{ json: { ...hit, summary } }];
```

Note the parsing style: it slices from the first `{` to the last `}` and
`JSON.parse`s that, so stray text around the JSON doesn't break it. A parse failure
falls through to `ok = false` and the item is dropped — failing safe toward *not*
emailing a summary it couldn't verify.

### 4.9 Insert Row → seen_papers
Records the paper so it is never re-sent: `paper_id = {{ $json.paper_id }}`,
`processed_at = {{ $now }}`.

### 4.10 Digest — Code node
Groups surviving items into one email per subscriber:

```js
const items = $input.all().map(i => i.json);
const byUser = {};
for (const it of items) (byUser[it.subscriber] ||= []).push(it);
return Object.entries(byUser).map(([subscriber, list]) => ({
  json: { subscriber,
    body: list.map(x => `- ${x.title} (${x.date})\n  ${x.summary}`).join('\n\n') }
}));
```

### 4.11 Send Email
`To = {{ $json.subscriber }}`, `Subject = Council update`, `Text = {{ $json.body }}`.

---

## 5. Design decisions and notes

**Two AI passes, not one.** Summarising and checking are deliberately separated. The
first pass writes; the second judges faithfulness and relevance and can veto. For a
trust product, this catches invented facts and coincidental keyword matches before
they reach a resident's inbox.

**`/no_think` on the Check agent.** `qwen3` is a reasoning model; the `/no_think`
directive at the top of the Check system message suppresses its thinking tokens so
it returns clean JSON that `Keep Good` can parse. The Summarise agent omits the
directive in the base guide — if you ever see reasoning text leaking into a summary,
adding `/no_think` to the Summarise system message is the fix.

**De-duplication is state, not logic.** The `seen_papers` table is what makes a
*daily* schedule safe. The correctness of the whole thing hinges on two links: the
SeenPapers read at the top and the Insert Row near the end. If either is broken,
residents get spammed.

**Single-endpoint assumption.** `Filter` and `Keep Good` read `$('HTTP
Request').first()` and `$('Summarise').first()`. This is fine for the common case of
one city / one feed. Watching several cities at once means looping over endpoints
and handling multiple items — see §7.

**Nothing sensitive leaves the machine.** Summaries and checks are local; the only
external request is a public OParl GET. That's the privacy payoff of the local-first
design.

---

## 6. Troubleshooting

| Symptom | Likely cause |
|---|---|
| HTTP Request returns HTML, not JSON | Wrong OParl address, or the city has no OParl. Re-check the endpoint. |
| Same alert every day | The seen check failed — confirm SeenPapers loads and the seen_papers Insert Row runs. |
| No email | Email node credentials, or `To` doesn't resolve to a real address. |
| Nothing matches | The keyword doesn't appear in recent titles — try a broader one to test. |
| Ollama connection error | Use `http://host.docker.internal:11434`, not `localhost` — n8n is in Docker, Ollama is on the host. |

---

## 7. Extending it

Directions the base guide points to:

- **Full text** — follow each paper's attached PDF and match keywords in the body,
  not just the title.
- **Many cities** — store several endpoints and loop over them (removes the
  single-endpoint assumption in §5).
- **Status tracking** — alert again when a paper moves from proposal to decision.
- **Public sign-up** — expose the Subscribe form as a shared link so anyone can
  register.
- **Accuracy evaluation** — a separate workflow that runs Summarise + Check over a
  fixed, labelled set of papers and records faithfulness and relevance, so you can
  tell whether a model or prompt change actually helped. The metric that matters is
  the hallucination rate (any summary marked not faithful), which you want at zero.
  *Not included in this repo yet.*
