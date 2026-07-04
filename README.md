# Citizen Council Watchdog (OParl)

Turn a German city council's public document system into plain-language email
alerts for residents.

German cities publish their council documents through a free, public API called
**OParl**. This project watches that feed for a city, keeps only papers that are
both new and relevant to a resident's chosen topic, summarises them into plain
language with a local LLM, quality-checks each summary, and emails a short digest.

All the reading and summarising runs locally on your machine. The only external
call is to the council's public OParl endpoint — no keys, no private data.

---

## What it does

- **Subscribe** — a small form lets a resident register an email, a keyword
  (a topic or street name), and the city's OParl address.
- **Watch daily** — a scheduled workflow fetches the latest council papers, drops
  anything already sent, and keeps only titles that mention a subscriber's keyword.
- **Summarise** — a local model writes a short, neutral, plain-language summary of
  each matching paper.
- **Quality-check** — a second model pass drops summaries that are unfaithful
  (invented facts / opinion) or where the keyword match was only coincidental.
- **Digest** — surviving items are grouped into one email per subscriber and sent.

---

## Requirements

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- [Ollama](https://ollama.com/) installed and running on your host machine, with `qwen3:14b` pulled
- A city's **OParl address** — search the web for your city name plus `OParl`; you
  want a link ending in something like `/oparl/v1/system`. `Muenster` is a well-known
  public example.
- An email account n8n can send from (any SMTP), for the digest.

---

## Quick start

```bash
docker compose up -d
```

Then open **http://localhost:5678** and follow the first-time setup below.

---

## First-time setup (do this once)

When you import the workflows, the credentials and Data Tables they reference don't
exist on your machine yet — so you create them once here.

**1. Pull the model** (on your host, not inside Docker):

```bash
ollama pull qwen3:14b
```

**2. Create the two Data Tables**
Left sidebar → **Data Tables** → **Create**, for each table below:

`watch_topics`

| Column        | Type   | Holds                                   |
|---------------|--------|-----------------------------------------|
| subscriber    | String | Resident's email                        |
| keyword       | String | Topic or street, e.g. `Radweg`          |
| city_endpoint | String | The OParl papers address to watch       |

`seen_papers`

| Column       | Type   | Holds                          |
|--------------|--------|--------------------------------|
| paper_id     | String | ID of a paper already handled  |
| processed_at | String | When we handled it             |

Column names must match exactly — later nodes look them up by name.

**3. Import both workflows**
Top-right **⋮** menu → **Import from File**, once for each file in `workflows/`:
the **Subscribe** workflow and the **Watcher** workflow.

**4. Create the Ollama credential**
Open the **Ollama Chat Model** node → **Create New Credential** → set Base URL to:

```
http://host.docker.internal:11434
```

**5. Create the email credential**
Open the **Send Email** node → create/select your SMTP credential.

**6. Re-link the Data Tables**
In each imported workflow, open every **Data Table** node (the Insert Row and
Search Rows nodes) and re-select your own `watch_topics` / `seen_papers` table from
the dropdown, so they point at your tables rather than the original author's IDs.

**7. Subscribe yourself, then test**
Run the **Subscribe** workflow once with your email and a common keyword like
`Haushalt` (budget). Then open the **Watcher** workflow and click **Test workflow** —
you should receive a digest. Run it a second time immediately; you should get *no*
email, which proves the "already seen" de-duplication works.

---

## How it works

See [docs/TECHNICAL.md](docs/TECHNICAL.md) for the full architecture, node-by-node
walkthrough, the two-pass summarise/check design, and data-flow notes.

---

## Stack

n8n (Community Edition, via Docker) · Ollama (local, `qwen3:14b`) ·
OParl REST API (free, public) · two Data Tables · SMTP email.

## License

MIT — see [LICENSE](LICENSE).
