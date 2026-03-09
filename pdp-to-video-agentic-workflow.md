# PDP-to-Video Agentic AI Workflow

## Goal
Transform a Product Detail Page (PDP) into a publish-ready short product video using a multi-agent workflow that is auditable, repeatable, and human-reviewable.

## Inputs
- PDP URL (or raw PDP JSON/HTML)
- Brand voice guide (tone, banned words, required claims)
- Channel target (TikTok, Reels, YouTube Shorts, Amazon listing)
- Locale and compliance profile (region, legal constraints)
- Optional assets (logo pack, product renders, lifestyle footage, music policy)

## Outputs
- Final video files (e.g., 9:16 MP4, 1:1 MP4)
- Script + shot list + captions/subtitles
- Thumbnail options
- Compliance report + trace log of sources used

## Agent Topology
1. **Orchestrator Agent**
   - Receives request, manages state machine, retries, approvals, and deadlines.
2. **PDP Ingestion Agent**
   - Scrapes/parses title, bullets, specs, reviews, Q&A, media, and pricing context.
3. **Fact & Claim Agent**
   - Converts extracted info into structured claims with confidence and citations.
4. **Audience Strategy Agent**
   - Chooses angle(s): utility, lifestyle, premium, value, comparison.
5. **Creative Brief Agent**
   - Produces hook options, narrative arc, CTA variants, and scene objectives.
6. **Scriptwriter Agent**
   - Generates channel-specific script versions and timing (e.g., 15s/30s/45s).
7. **Storyboard Agent**
   - Maps script lines to shots, transitions, overlays, and B-roll requirements.
8. **Asset Planner Agent**
   - Decides whether to use PDP images, generated visuals, stock, or UGC-style clips.
9. **Video Generation Agent**
   - Produces scenes with text-to-video/image-to-video models + basic motion graphics.
10. **Voiceover Agent**
    - Creates TTS or voice-clone narration with pacing controls.
11. **Audio Agent**
    - Selects music/SFX, ducks audio, and normalizes loudness.
12. **Caption & Localization Agent**
    - Creates burned-in captions and translated subtitle tracks.
13. **Compliance & Brand Guard Agent**
    - Validates claims, prohibited phrasing, trademark usage, and region-specific rules.
14. **QA Scoring Agent**
    - Scores hook strength, visual coherence, readability, and conversion potential.
15. **Human Review Agent (Human-in-the-loop)**
    - Requests approval at policy gates (claims, branding, final export).
16. **Packaging Agent**
    - Exports variants, metadata, thumbnail candidates, and publish-ready bundle.

## End-to-End Workflow (State Machine)
1. **Intake**
   - Validate input completeness.
   - Create run ID, data retention policy, and target SLAs.
2. **Extract**
   - Parse PDP into canonical product schema.
   - Store provenance for every extracted field.
3. **Claim Graph Build**
   - Build fact graph (`claim`, `source`, `confidence`, `risk_level`).
4. **Creative Planning**
   - Select audience angle and generate brief + script options.
5. **Scene Planning**
   - Build timeline with scene durations and required assets.
6. **Asset Generation/Selection**
   - Fetch/generate visuals and voice assets.
7. **Assembly**
   - Render rough-cut with narration, text overlays, music, and captions.
8. **Policy + Quality Gates**
   - Compliance scan and quality scoring.
   - If failed, route back to relevant agent with error context.
9. **Human Approval Gate**
   - Reviewer accepts, requests edits, or rejects.
10. **Final Render + Packaging**
    - Produce final formats and delivery manifest.

## Control Logic (Pseudo-Orchestration)
```python
state = "INTAKE"
while state != "DONE":
    if state == "INTAKE":
        validate_inputs(run)
        state = "EXTRACT"

    elif state == "EXTRACT":
        product_schema = pdp_ingestion_agent.run(run)
        claim_graph = fact_claim_agent.run(product_schema)
        state = "CREATIVE_PLAN"

    elif state == "CREATIVE_PLAN":
        brief = audience_strategy_agent.run(claim_graph, channel=run.channel)
        scripts = scriptwriter_agent.run(brief, lengths=[15, 30])
        storyboard = storyboard_agent.run(scripts.best)
        state = "GENERATE"

    elif state == "GENERATE":
        assets = asset_planner_agent.run(storyboard)
        rough_cut = video_generation_agent.run(storyboard, assets)
        voiced = voiceover_agent.apply(rough_cut)
        mixed = audio_agent.mix(voiced)
        captioned = caption_localization_agent.apply(mixed)
        state = "VALIDATE"

    elif state == "VALIDATE":
        policy = compliance_brand_guard_agent.check(captioned, claim_graph)
        score = qa_scoring_agent.score(captioned)
        if not policy.pass_:
            run.feedback = policy.issues
            state = "CREATIVE_PLAN"
        elif score.total < run.min_quality_score:
            run.feedback = score.weaknesses
            state = "GENERATE"
        else:
            state = "HUMAN_REVIEW"

    elif state == "HUMAN_REVIEW":
        decision = human_review_agent.request_approval(captioned)
        if decision == "approve":
            state = "PACKAGE"
        elif decision == "revise":
            state = "CREATIVE_PLAN"
        else:
            state = "DONE"

    elif state == "PACKAGE":
        packaging_agent.export(captioned, formats=["9:16", "1:1"])
        state = "DONE"
```

## Recommended Data Contracts
- `ProductSchema`: title, features, materials, dimensions, use-cases, warranty, brand, legal disclaimers.
- `ClaimRecord`: claim_text, source_uri, source_excerpt, confidence, jurisdiction_risk.
- `CreativeBrief`: persona, pain-point, hook, promise, CTA, forbidden_terms.
- `ScenePlan`: scene_id, start/end, visual_prompt, overlay_text, vo_style, compliance_tags.
- `QualityReport`: hook_score, readability_score, brand_score, policy_flags, overall.

## Evaluation Rubric
- **Compliance pass rate**: % runs with zero critical policy violations.
- **Edit distance after human review**: lower is better for automation quality.
- **Time-to-first-draft** and **time-to-final**.
- **CTR proxy score** (model-estimated) and retention proxy by scene.
- **Localization quality**: subtitle accuracy + timing drift.

## Minimal Tech Stack (Example)
- Orchestration: Temporal / LangGraph / custom event bus
- Retrieval + provenance: PostgreSQL + object store
- LLMs: one reasoning model + one fast drafting model
- Media: image/video generation API + FFmpeg compositor
- QA: policy rules engine + classifier ensemble
- Observability: run traces, per-agent latency, token/media cost dashboard

## Human-in-the-Loop Checkpoints
- Claim-sensitive categories (health, children, regulated products)
- First creative direction sign-off
- Final legal/brand approval before publication

## Failure Handling
- Missing PDP fields → fallback extractor + uncertainty flag.
- Conflicting claims → escalate to human reviewer with source diffs.
- Low visual quality → regenerate only failing scenes (partial rerender).
- Policy violation → auto-rewrite script segments and revalidate.

## Deployment Pattern
- Start with **shadow mode** (agents generate drafts, humans publish).
- Move to **assisted mode** (agents publish after required approvals).
- Finally **policy-bounded autopublish** for low-risk SKUs.

## Quick Start Runbook
1. Pick one product category (e.g., home fitness accessories).
2. Define compliance rules + brand voice.
3. Run 20 PDPs through the pipeline.
4. Measure QA/compliance metrics and human edit distance.
5. Tighten prompts, policy rules, and scoring thresholds.
6. Expand to additional channels/locales.
