# Prompt — `rhoai-learning-path.png`

Generated with Gemini (image generation). Use for regenerating or iterating on the learning path diagram.

## Primary prompt

```
Create a wide horizontal learning-path roadmap infographic (16:9, 1920×1080) for "Red Hat OpenShift AI Odyssey".

This is a READABLE CURRICULUM MAP — not decorative space art. Every stop must show:
1) celestial body name (large)
2) a short mission title of 2–4 words only (bold subtitle)
3) step order where applicable

Use EXACT labels below. Text must be large, sharp, legible, correctly spelled.

LAYOUT: Golden curved path left → right. Three zones with subtle background color:

ZONE A (left, warm orange/gold): "INNER SYSTEM ☀️ — Platform crew ⚙️"
ZONE B (center, thin strip): "ASTEROID BELT ☄️" — see belt rules below
ZONE C (right, cool blue/purple): "OUTER SYSTEM 🪐 — App crew 💻"
ZONE D (far right, faint): "DEEP SPACE 🌌 — Bonus"

─── ASTEROID BELT RULES (IMPORTANT) ───
The asteroid belt is a playful transition joke — NOT a real mission step.
- Keep it VERY SMALL: a narrow gap (~5–8% of total width max) between Mars and Jupiter
- Do NOT give it a large banner, checkpoint portal, or dedicated panel
- Just a thin scattered band of tiny asteroids across the path + one small label: "☄️ Crossing Point"
- No step number, no crew tasks, no subtitle longer than those 2 words
- Mars and Jupiter should be visually close — the belt is a subtle divider, not a focal point

─── EXACT STOPS (in order) ───

☀️ SUN
Deploy OpenShift AI

1 · MERCURY
Notebook Playground

2 · VENUS
GPU Power-up

3 · EARTH
Model Serving

🌙 MOON (orbits Earth — visually smaller, on a dashed side orbit attached to Earth)
GPU Model Deploy
Small badge near Moon: "OPTIONAL"

Visual relationship Earth ↔ Moon:
- Earth = first model serving step (lighter / CPU-friendly)
- Moon = optional extra step for a larger GPU model
- Draw Moon clearly orbiting Earth, not in the main planet line

4 · MARS
Distributed Workloads

[thin asteroid belt gap — small label only: ☄️ Crossing Point]

5 · JUPITER
ML Experimentation

6 · SATURN
TrustyAI

7 · URANUS
Models as a Service

8 · NEPTUNE
Agentic RAG & OGX

DEEP SPACE (far right, smaller nodes on dashed branch):
Model Registry · Feast · Catalog Deep Dive

─── STYLE ───
- Education/training roadmap infographic, professional Red Hat enterprise vibe
- Flat illustration + soft sci-fi, clean, not cluttered
- Each planet = stylized icon + number badge + name + 2–4 word subtitle beneath
- Red accent (#EE0000), teal hints, dark navy starfield
- Small red space-suited astronaut at the Sun (start) and near Neptune (goal)
- Tiny optional icons (do NOT replace text): Sun=cluster, Mercury=notebook, Venus=GPU chip, Earth=small model icon, Moon=large GPU/LLM chip, Mars=distributed nodes, Jupiter=pipeline+chart, Saturn=shield/evaluation, Uranus=API gateway, Neptune=agent/docs

─── DO NOT ───
- Make the asteroid belt large, dramatic, or a central focal point
- Give the belt its own big title block or "ORCHESTRATION HUB" style label
- Use long sentences under planets (max 4 words per title)
- Invent wrong titles (no "K8s basics", "Networking", "CI/CD intro")
- Skip Sun, Moon, or the small belt divider
- Put Moon on the main path as a numbered planet — it orbits Earth
- Wrong order or numbering
- Misspell: OpenShift, TrustyAI, EvalHub, OGX
- Tiny unreadable text on planets (belt text can stay small)
- Red Hat logo or watermark

Priority #1: crystal-clear learning path with exact short titles on every planet stop. The belt is a tiny visual joke between Mars and Jupiter.
```

## Regeneration prompt

Use if text is garbled or labels are wrong:

```
Same roadmap. Belt must stay a thin 5% gap with only "☄️ Crossing Point" — not a large section.

Make every planet subtitle 2× larger. Use ONLY these exact pairs:

SUN — Deploy OpenShift AI
MERCURY — Notebook Playground
VENUS — GPU Power-up
EARTH — Model Serving
MOON — GPU Model Deploy (OPTIONAL, orbiting Earth)
MARS — Distributed Workloads
[small belt] ☄️ Crossing Point
JUPITER — ML Experimentation
SATURN — TrustyAI
URANUS — Models as a Service
NEPTUNE — Agentic RAG & OGX
DEEP SPACE — Model Registry · Feast · Catalog Deep Dive

Remove all other text labels.
```

## Title reference

| Stop | Short title |
|------|-------------|
| ☀️ Sun | Deploy OpenShift AI |
| Mercury | Notebook Playground |
| Venus | GPU Power-up |
| Earth | Model Serving |
| 🌙 Moon | GPU Model Deploy *(optional)* |
| Mars | Distributed Workloads |
| ☄️ Belt | Crossing Point |
| Jupiter | ML Experimentation |
| Saturn | TrustyAI |
| Uranus | Models as a Service |
| Neptune | Agentic RAG & OGX |
