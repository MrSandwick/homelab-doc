---
tags: [homelab, project, security, ids, llm, ollama, troubleshooting]
---

# Troubleshooting — AI Alert Triage

> Companion to [AI Alert Triage](../server/SRV-27-AI-Alert-Triage.md), which records only the working configuration. This file records the problems hit during that work, in the order they occurred.

## Contents

1. [Small models corrupt numbers they are asked to copy](#small-models-corrupt-numbers-they-are-asked-to-copy)
2. [A reasoning model writes its thinking instead of an answer](#a-reasoning-model-writes-its-thinking-instead-of-an-answer)
3. [Rows silently missing from the model's answer](#rows-silently-missing-from-the-models-answer)
4. [An empty report looked like a successful run](#an-empty-report-looked-like-a-successful-run)
5. [The model called attack-family alerts benign](#the-model-called-attack-family-alerts-benign)
6. [Model partly offloaded to the CPU](#model-partly-offloaded-to-the-cpu)

## Small models corrupt numbers they are asked to copy

**Symptom:** the first design handed the model a table of alert counts and asked for a summary. The summaries read well and were wrong — counts were altered, totals invented, and signatures attributed to the wrong device.

**Root cause:** a 3–4B model running locally is not reliable at transcribing structured data. It is doing text continuation, and a plausible-looking number is as good a continuation as the correct one. This is not fixable with prompt wording.

**Fix — a design rule rather than a patch:** code computes every number and the model never emits one. The model receives numbered rows and returns only `id`, `verdict` and a one-sentence `reason`; code re-attaches the facts afterwards. Every count, percentage and response code in the final report comes from `aggregate.py`, never from the model.

**Takeaway:** the useful question was not "which model gets the numbers right" but "how do I design this so the model never touches a number".

## A reasoning model writes its thinking instead of an answer

**Symptom:** `qwen3:4b` returned long chain-of-thought text where a JSON object was expected; the parse step rejected nearly every response.

**Root cause:** a reasoning-style model emits its deliberation as ordinary output. Asking for strict JSON does not suppress that.

**Fix:** excluded the model from the comparison. The remaining candidates were evaluated with a JSON-schema `format` and `temperature 0` so the output shape is enforced rather than requested.

## Rows silently missing from the model's answer

**Symptom:** reports occasionally covered fewer signatures than were sent, with no error — the missing rows simply did not appear.

**Root cause:** with several rows in one request the model sometimes answers for a subset and stops. Nothing fails; the response is valid JSON containing fewer objects than expected.

**Fix:** validate the returned ids against the batch, then retry only the missing rows in progressively smaller batches (8 → 2 → 1), at most 3 passes within a 900 s budget. Anything still unanswered is printed as `unreviewed` rather than quietly dropped or guessed.

## An empty report looked like a successful run

**Symptom:** a run in which the model produced nothing usable still wrote a report file, and the systemd unit reported success.

**Root cause:** the script treated "no verdicts" as an ordinary outcome and exited 0, so the timer had no way to signal a problem.

**Fix:** exit with code 3 when no verdict survives validation. The unit then shows as failed in `systemctl list-timers` and `journalctl`, which is the only signal available on an unattended daily job.

## The model called attack-family alerts benign

**Symptom:** during comparison runs, models occasionally rated reputation-list and exploit signatures as `benign`.

**Root cause:** the verdict is the one thing the model does decide, and a small model's judgement on an unfamiliar signature name is unreliable. Alert text also comes from network traffic, so a crafted domain name could in principle push a verdict.

**Fix:** a guard in code, applied after validation. A row can never end as `benign` if it has severity 1, is flagged priority, or matches an attack/threat-intel family (`ET DROP|CINS|SCAN|EXPLOIT|MALWARE|TROJAN|CNC|ADWARE_PUP|PHISHING|WEB_SERVER|WEB_SPECIFIC_APPS`); it is raised to `worth_a_look` with a `[raised by code: …]` note, so the override is visible in the report.

## Model partly offloaded to the CPU

**Symptom:** a noticeably slower run than the same model had managed before.

**Root cause:** on a 4 GB card a model that only just fits can spill into system memory, and inference then runs partly on the CPU. `ollama ps` showed a `14%/86% CPU/GPU` split instead of `100% GPU`.

**Fix:** check `ollama ps` after a model change and keep the model small enough to fit entirely in VRAM. At one minute per daily run the slowdown is tolerable, so this is recorded as a thing to watch rather than a fault that was chased down.

## Related

- [AI Alert Triage](../server/SRV-27-AI-Alert-Triage.md) — the working configuration
- [SRV-26-TRBL](./SRV-26-TRBL-Suricata-IDS.md) — problems with the sensor that feeds this pipeline
