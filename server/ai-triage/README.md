# ai-triage

Supporting files for [AI Alert Triage](../SRV-27-AI-Alert-Triage.md), the daily Suricata-plus-local-LLM report. The doc explains what each one does and how the pipeline fits together; this folder only holds the files themselves.

| File | Deployed to | Purpose |
|---|---|---|
| `aggregate.py` | server, `~/ai/aggregate.py` | Reads `eve.json*`, labels known noise, groups the rest, prints one JSON summary |
| `analyze.py` | laptop, `~/ids/analyze.py` | Fetches that summary over SSH, queries Ollama, validates, writes the report |
| `devices.example.json` | — | Template for the server's `devices.json` (IP or subnet → name, optional `"role": "public"`) |
| `ids-report.service` | laptop, `/etc/systemd/system/` | Runs the analyzer once |
| `ids-report.timer` | laptop, `/etc/systemd/system/` | Triggers it daily at 08:30, catching up after a missed run |

Placeholders (`<USERNAME>`, `<SERVER_IP>`, `<GPU_IP>`, `<PUBLIC_HOST_IP>`) follow the repository convention — see `.env.example`. Substitute real values on the machine, never in these files.

The real `devices.json` and the generated `ids-reports/` contain internal addresses and device names and are gitignored.
