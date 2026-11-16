# ideation

A config store for Agentic AI workflow processing.

## Layout

```
.
├── README.md
├── LICENSE
├── workflows/
│   └── rote-demo.toml          # one TOML file per runnable workflow
├── schemas/
│   └── workflow.schema.json    # JSON Schema every workflow must satisfy
└── .github/workflows/
    └── validate.yml            # CI: validate every workflows/*.toml against the schema
```

## What a workflow config contains

- **`[workflow]`** — `name` (required); `description`; `max_replans` 0–5 (default 2).
- **`[budget]`** — `run_usd` and `episode_usd` hard caps in USD (both required, > 0); `max_output_tokens` (default 256).
- **`[pricing]`** — `input_per_mtok` and `output_per_mtok`, nominal USD per million tokens for cost accounting (required).
- **`[memory]`** — `skip_critic_confidence` 0–1 (default 0.75).
- **`[policy]`** — settings for the reuse-vs-plan router: `ucb_c` (≥ 0, default 0.3), `cost_weight` (≥ 0, default 0.5), `reference_cost_usd` (> 0, default 0.004).
- **`[[tasks]]`** — each entry has a `text` string.

Required tables: `workflow`, `budget`, `pricing`.

## Secrets

API keys and other credentials come only from environment variables of the runner. Never write them into a config file. Configs in this repo hold no secrets. CI makes no model calls and uses no secrets.

## Validating locally

Python 3.11+:

```bash
pip install jsonschema
```

```python
import glob, json, tomllib
from jsonschema import Draft202012Validator

schema = json.load(open("schemas/workflow.schema.json"))
Draft202012Validator.check_schema(schema)
validator = Draft202012Validator(schema)

for path in sorted(glob.glob("workflows/*.toml")):
    with open(path, "rb") as f:
        data = tomllib.load(f)
    errors = sorted(validator.iter_errors(data), key=lambda e: [str(p) for p in e.absolute_path])
    if errors:
        for e in errors:
            loc = e.json_path if hasattr(e, "json_path") else "/" + "/".join(str(p) for p in e.absolute_path)
            print(f"{path}: {loc}: {e.message}")
        raise SystemExit(1)
    print(f"ok: {path}")
```

## Ideas

| Idea | Status | Summary |
|------|--------|---------|
| Rote | Offline prototype | An agent that learns its own routines: critic-approved traces become reusable templated skills, a UCB1 router learns when to reuse a skill versus plan from scratch, and a Beta-posterior confidence score decides when a routine can run without the critic, so model spend falls as experience grows. |

### Rote

- Planner / executor / critic loop: the critic catches a seeded unit mistake and forces a replan.
- Corrected plans are stored as templates (e.g. `convert {n0} {u0} to {u1} then add {n1}`) and reused on structurally identical tasks.
- Critic is skipped once skill confidence (mean of Beta(1+s, 1+f)) reaches 0.75.
- Router reward = success − cost_weight × (episode cost / reference_cost_usd); if reuse and fresh planning both fail, the task is judged infeasible and nothing is blamed.
- Per-run and per-episode dollar caps are checked before every call.
- Offline demo: 10 of 12 tasks succeed (2 failures expected: one infeasible, one out of scope); 49% lower model spend than the same run with memory disabled (18 vs 34 calls). The planner and critic in the demo are rule-based mocks; a real-provider adapter exists but is off by default. Config: `workflows/rote-demo.toml`.

## License

MIT — see [LICENSE](LICENSE).
