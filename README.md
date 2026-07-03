# 📡 Veille IA V1 — Automated Daily Intelligence Digest

> An end-to-end n8n workflow that collects articles from RSS feeds, enriches them with Gemini AI, and delivers a structured daily briefing by email — fully automated, no manual intervention.

---

## What it does

Every morning at 7:00 AM, this workflow:

1. Pulls articles from 3 RSS sources (Gaming, Horror/Cinema, Tech)
2. Deduplicates and filters to the last 24 hours only
3. Sends each article to Gemini AI for analysis: summary, key points, category, relevance score, importance level, and estimated reading time
4. Ranks the top 10 articles by relevance score
5. Generates an editorial synthesis of the day (120–180 words) via a second Gemini call
6. Assembles a formatted HTML report
7. Sends it by Gmail

The result is a newsletter-style email you can read in under 5 minutes.

---

## Example output

```
📡 Veille IA & Tech — Thursday, July 2, 2026

🧠 Synthesis of the day
The tech industry is accelerating its shift away from physical media.
Sony's announcement ending disc production by 2028 is echoed by Xbox
planning a similar move. Meanwhile, AI continues to attract massive
investment while raising environmental concerns...

📊 10 articles analysed · Average score: 7.9/10
   Gaming: 5 | Business: 2 | Cybersecurity: 2 | AI: 1

🔥 Top 10

🎮 Gaming · ⭐ High · ⏱️ 2 min · 9/10
Sony to End Physical Game Discs by 2028
Sony announced the end of physical disc production for PlayStation...
```

---

## Architecture

```
Schedule Trigger (07:00)
        │
        ├── RSS Gaming ──┐
        ├── RSS Horror ──┼──► Merge (Append) ──► Remove Duplicates
        └── RSS Tech   ──┘         │
                                   │
                          [Set nodes: normalize fields]
                          [Sort by date → Filter IF < 24h]
                                   │
                          Basic LLM Chain (Gemini Flash)
                          [Per article: summary, score, category]
                                   │
                          Code node (JSON parsing + cleanup)
                                   │
                          Sort by pertinence_score (desc)
                                   │
                          Limit → Top 10
                          │              │
                          │         Code node (aggregate)
                          │              │
                          │         Basic LLM Chain (day synthesis)
                          │              │
                          └────► Merge ◄─┘
                                   │
                          Code node (HTML report assembly)
                                   │
                                Gmail (send)
```

---

## Stack

| Tool | Role |
|------|------|
| n8n Cloud | Workflow orchestration |
| Google Gemini Flash | Article analysis + editorial synthesis |
| Gmail API | Email delivery |
| RSS feeds | Content sources (Eurogamer, Dread Central, TechCrunch) |
| JavaScript (n8n Code nodes) | JSON parsing, data transformation, HTML generation |

---

## Key technical decisions — and why

**Gemini Flash over GPT-4o mini:** tested both; Flash produced more consistent JSON output on French-language synthesis tasks and costs less at this usage scale (~€0.01/day).

**Two-stage AI processing:** one Gemini call per article (scoring + summary), then one global call for the editorial synthesis. This avoids sending a 10,000-token context to the model for each article, reducing cost and improving synthesis quality.

**Custom JSON parsing node:** Gemini occasionally returns trailing commas or markdown fences (` ```json `) despite explicit instructions. A Code node strips these before parsing, so one malformed response never breaks the entire workflow.

**24h filter before AI processing:** articles older than 24 hours are filtered out before any Gemini call, not after. This avoids paying for AI analysis of irrelevant content.

**Relevance scoring over chronological ranking:** articles are ultimately ranked by an AI-attributed relevance score (0–10), not just publication date. The scoring prompt prioritises official announcements, industry impact, and novelty over promotional content.

---

## What I learned building this

**The Fixed vs Expression trap in n8n.** Fields that look like expressions (`{{ $json.title }}`) but are set to "Fixed" mode pass the literal string through instead of evaluating it. This caused silent data corruption that only surfaced three nodes downstream, at the Remove Duplicates step. Systematic node-by-node testing was the only reliable diagnostic method.

**RSS date formats are not standardised.** Three different sources produced three different date string formats (`pubDate`, `isoDate`, RFC 2822 with timezone offsets). Sorting on raw strings produced incorrect chronological ordering. The fix: `new Date($json.pubDate).toISOString()` in the Set node to normalise before any sort operation.

**Gemini free tier quotas are per model, per day.** Switching models mid-debug consumes quota on the new model immediately. Under free tier constraints, retries triggered by n8n's built-in retry mechanism can exhaust daily quotas within minutes. Disabling automatic retry during testing is essential.

**Loop Over Items doesn't behave like a standard for-loop in n8n.** Manually connecting a return edge from inside the loop to the Loop node itself creates an infinite loop, not a controlled iteration. The correct pattern: Basic LLM Chain processes items natively as a batch — no Loop node required for this use case.

---

## Customisation

The workflow is designed to be adapted to any domain. To repurpose it:

1. Replace the 3 RSS feed URLs with sources relevant to your sector
2. Update the Gemini system prompt categories to match your domain vocabulary
3. Adjust the relevance scoring criteria in the prompt to reflect what matters to your specific audience
4. Change the Gmail recipient

Typical setup time for a new domain: 30–45 minutes.

---

## Known limitations

- **Duplicate topics, not duplicate URLs.** If 3 sources cover the same story on the same day, all 3 may appear in the Top 10. URL-based deduplication works; semantic deduplication does not (planned for V2).
- **Gemini output consistency.** Despite strict prompting, category assignment can vary between runs for articles that span multiple domains (e.g. a game adaptation becoming a Netflix series). The parsing node handles malformed JSON but cannot fix classification drift.
- **No error alerting.** If Gemini returns an error at 7:00 AM, the workflow fails silently. No email means no report that day. Error notification via Gmail or Slack is planned for V2.
- **n8n Cloud dependency.** Currently running on n8n Cloud trial. Production deployment requires self-hosted n8n on a VPS (Docker, ~€5–10/month). Migration is non-destructive: export workflow JSON, import to self-hosted instance.

---

## Roadmap

**V1 (current)** — MVP running in production, daily email delivery ✅

**V2 (planned)**
- Semantic deduplication (cluster similar articles before AI processing)
- Error notification system (Gmail/Slack alert on workflow failure)
- Execution confidence score ("9/10 articles processed successfully")
- Self-hosted deployment on VPS via Docker

**V3 (exploratory)**
- Multi-client support (different RSS sources per client, single infrastructure)
- Notion integration for searchable article archive
- Weekly digest mode for low-volume categories

---

## About this project

This is the first project in a portfolio built around the intersection of **AI automation** and **creative/editorial domains**.

The workflow was built from scratch over several sessions, including debugging real production issues (quota exhaustion, JSON parsing failures, n8n Loop node behaviour, RSS date normalisation). It runs daily in production on a real email address, not as a demo.

**Author:** Quentin Pequiot
**GitHub:** github.com/quentinpequiot
**Formation:** Digi Atlas — Formation Digitale Appliquée IA, Automatisation & No-Code (RS7311, 2026)

---

## FR — Version française

### Ce que ça fait

Un workflow n8n qui tourne automatiquement chaque matin à 7h : collecte d'articles RSS, analyse par Gemini IA (résumé, score de pertinence, catégorie), génération d'une synthèse éditoriale, et envoi d'un rapport HTML par email. Sans intervention manuelle.

### Décisions techniques clés

- **Deux appels Gemini distincts** : un par article (analyse individuelle) + un global (synthèse de la journée). Plus économique et plus précis qu'un seul appel massif.
- **Filtre 24h avant l'IA** : on ne paie que pour analyser les articles pertinents.
- **Node de parsing JSON** : Gemini peut retourner du JSON mal formé malgré les instructions. Un node de nettoyage évite qu'un seul article défaillant plante tout le workflow.
- **Tri par score de pertinence** : pas par date. Le Top 10 reflète l'importance réelle de l'information, pas sa fraîcheur.

### Limites connues

- Les doublons sémantiques (même sujet, URLs différentes) ne sont pas filtrés en V1.
- Pas d'alerte en cas d'échec : si le workflow plante à 7h, aucun email n'arrive.
- Déploiement actuel sur n8n Cloud (essai). Production = VPS auto-hébergé via Docker.

### Auteur

Quentin Pequiot — [github.com/quentinpequiot](https://github.com/quentinpequiot)
