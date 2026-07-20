# followme

A small set of standalone CLI scripts that discover GitHub repositories,
grade them with a local **Ollama** model, then follow promising authors
and star strong projects. State lives in a single SQLite file. No
database server, no extra services.

## Pipeline

Four independent step scripts live in `scripts/`, each runnable on its
own. `main.py` at the project root chains them together.

| Script                  | What it does                                                                 |
| ----------------------- | ---------------------------------------------------------------------------- |
| `scripts/fetch.py`      | Pull `N` new repositories from GitHub Search and insert them into the DB.    |
| `scripts/evaluate.py`   | Clone unrated repos, ask Ollama for `idea`, `skill`, `description` + a malware verdict, store. |
| `scripts/subscribe.py`  | Follow profiles whose repos updated in the last `W` hours scored above `X`.  |
| `scripts/star.py`       | Star repos updated in the last `W` hours that scored above `Y`.              |
| `main.py`               | Runs `fetch -> evaluate -> subscribe -> star` once, or in an infinite loop.  |

## Schema

One SQLite table — `entries` — one row per repository:

| Column        | Meaning                                              |
| ------------- | ---------------------------------------------------- |
| `repo`        | `owner/name` (primary key)                           |
| `profile`     | repository owner login                               |
| `clone_url`   | git URL used for the shallow clone                   |
| `html_url`    | browser URL                                          |
| `created_at`  | when the row was inserted (UTC ISO-8601)             |
| `updated_at`  | last time we touched the row (UTC ISO-8601)          |
| `followed`    | 1 if we follow the `profile`, mirrored across rows   |
| `starred`     | 1 if we star the `repo`                              |
| `idea`        | 1.0–10.0 novelty grade from Ollama                   |
| `skill`       | 1.0–10.0 engineering grade from Ollama               |
| `description` | one-sentence English summary                         |
| `security_flag`   | 1 when Ollama judged the repo malicious          |
| `security_reason` | one sentence naming the malicious behaviour      |

Scoring uses `idea + skill` so thresholds live in `[2.0, 20.0]`.

`evaluate.py` runs a malicious-behaviour screen (credential/secret harvesting
sent off-host, exfiltration, obfuscated remote-code execution, hardcoded C2,
typosquatting, tools that secretly phone home). A flagged repo gets `idea` and
`skill` set to 1.0, a `⚠ SECURITY:` description, and is **excluded from
`subscribe` and `star`** — the malicious author is never promoted.

## Requirements

- Python 3.11+ (standard library only)
- Git CLI
- Network access to GitHub
- A running Ollama with the configured model installed
  (e.g. `ollama pull qwen2.5-coder:7b`)
- A GitHub personal access token with `user:follow` and `public_repo` scopes

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
cp env.example .env
# edit .env: at minimum set GITHUB_TOKEN and OLLAMA_URL / OLLAMA_MODEL
```

`requirements.txt` exists but is empty — no third-party packages are needed.

## Usage

Every script reads `.env` from the project root. Flags override env values.

### Fetch new repositories

```bash
python3 scripts/fetch.py -n 5
```

### Evaluate everything not yet rated

```bash
python3 scripts/evaluate.py            # rate all pending
python3 scripts/evaluate.py -l 10      # rate up to 10
```

### Follow high-scoring authors (recent window)

```bash
python3 scripts/subscribe.py -s 14 -w 24
python3 scripts/subscribe.py --dry-run
```

### Star high-scoring repos (recent window)

```bash
python3 scripts/star.py -s 16 -w 24
python3 scripts/star.py --dry-run
```

### Run the full cycle

```bash
python3 main.py                            # one cycle, defaults from .env
python3 main.py -n 5 --subscribe-threshold 14 --star-threshold 16 -w 24
python3 main.py --dry-run                  # safe rehearsal
python3 main.py -i --sleep 600             # loop forever
```

The default cycle is exactly what the project was built around:

1. **fetch 5 new repos**
2. **evaluate them**
3. **follow profiles** updated in the last 24h with `idea + skill > 14`
4. **star repos**     updated in the last 24h with `idea + skill > 16`

## Files written

- `data/followme.sqlite` — single source of truth
- `data/repo/` — scratch clone directory, wiped between repos

## Grade scale

The Ollama prompt asks for strict anchors:

- `1.0` — trivial / junior
- `5.0` — ordinary / middle
- `9.0` — strong / senior

Both `idea` and `skill` are clamped into `[1.0, 10.0]`.
