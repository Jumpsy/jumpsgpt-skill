---
name: jumpsgpt
description: Omni skill for JumpsGPT / JumpStudy / JumpCode development. Load when working on the Jumpsy/jumpgpt repo. Covers stack, design system, Convex rules, Cloudflare deployment, CodeRabbit pipeline, subscription tiers, Leap mascot, and all agent rules.
---

# JumpsGPT Omni Dev Skill

You are working on **JumpsGPT** — a Next.js 16 + Convex + Clerk + Stripe SaaS AI platform with two main products:
- **JumpsGPT** (`/app`) — multi-model AI chat workspace with Auto Router
- **JumpStudy** (`/jumpstudy`) — AI-powered study pod system (flashcards, quiz, matching, AI tutor, Mermaid whiteboard)

**Repo:** `Jumpsy/jumpgpt` · **Live:** `jumpsgpt.com` · **Deployed on:** Cloudflare Workers via OpenNext

---

## Stack

| Layer | Tech |
|-------|------|
| Framework | Next.js 16 App Router + React 19 |
| Database | Convex (real-time, `exuberant-rabbit-720.convex.cloud`) |
| Auth | Clerk v7 (`clerk.jumpsgpt.com`) |
| Payments | Stripe |
| AI | OpenAI via `AI_API_KEY` + `AI_API_BASE_URL` |
| Email | Resend (`support@jumpsgpt.com`) |
| Analytics | PostHog |
| Errors | Sentry |
| Deploy | Cloudflare Workers (`wrangler deploy`, CI via `.github/workflows/deploy-cloudflare.yml`) |
| Runtime | `nodejs_compat` + `global_fetch_strictly_public` |

---

## Non-Negotiable Rules

### Git
- **Never force-push** to main
- **Never amend a published commit** — create a new one
- **Never skip hooks** (`--no-verify`)
- **Never commit secrets** — they live in `.env.local` (gitignored) or Cloudflare secrets
- Commit format: `<type>(<scope>): <description>` — types: `feat fix refactor style docs test chore ci`

### Security
- Never introduce XSS, SQLi, command injection, SSRF
- All user input must be validated before hitting Convex
- Stripe webhooks must verify `STRIPE_WEBHOOK_SECRET` via `stripe.webhooks.constructEvent`
- Never bypass Clerk auth on protected routes
- Rate-limit every public API route

### Code Style
- TypeScript strict — never use `any` unless forced
- No `console.log` in committed code — use Sentry
- No placeholder / "not implemented" functions — finish or don't add
- No comments explaining WHAT — only WHY (non-obvious invariants, workarounds)

### Convex
- Never edit `convex/_generated/` by hand (except adding module imports)
- Every mutation touching sensitive data must call `ctx.auth.getUserIdentity()`
- Never query entire table without an index
- Run `npx convex dev --once` after schema changes

### AI / Models
- Default: `process.env.AI_MODEL` (currently `gpt-4o-mini`)
- Never hardcode model strings
- `AI_API_BASE_URL` = `https://api.openai.com/v1`; use plain OpenAI slugs, not OpenRouter format

### Deployment
- **Zero Vercel** — Cloudflare Workers only
- Use `cf-ipcountry` not `x-vercel-ip-country`
- `export const runtime = "edge"` for most routes; `"nodejs"` only when Node APIs needed
- Build command: `npm run build:cloudflare` (OpenNext)
- Deploy: `npx wrangler deploy`
- Secrets go in Cloudflare via `wrangler secret put` — never in `wrangler.jsonc`

---

## Design System

- **Brand color:** `--pink` (#d61a6f light / #ff3d92 dark) — accessible 4.5:1
- Full pink palette: `--pink-50` through `--pink-900` (use `bg-pink-100`, `text-pink-600`, etc.)
- Full gray palette: `--gray-50` through `--gray-900`
- **No gradients** unless opacity-only overlays
- **Icons:** bare SVG, NO background box/badge/container unless explicitly requested
- **Skeletons:** always use `.skeleton` utility class from `globals.css`
- **Fonts:** Space Grotesk (display) + Geist (body) + handwritten font switchable via `data-font="handwritten"` on `<html>`
- Mobile-first: test at 375px, 768px, 1280px
- JumpStudy font toggle: `JumpStudyFontToggle` + `JumpStudyFontApplier` from `components/ui/jumpstudy-font-applier.tsx`

---

## Leap Mascot

**Leap** is a cute pink frog — JumpStudy's mascot. She appears on:
- Empty states in JumpStudy
- Floating helper when users highlight text in Notes mode
- PWA install prompt
- Loading/success states for pod creation
- AI Tutor replies (signed "🐸 Leap")

**Never replace Leap with a generic emoji or icon.** Defined in `components/ui/leap-mascot.tsx`.

---

## Subscription Tiers (do not change without explicit instruction)

| Tier | Price | Pods | Cards/pod | AI tutor |
|------|-------|------|-----------|---------|
| Free | $0 | 3 | 50 | no |
| Starter | $20/mo | 20 | 200 | yes |
| Pro | $50/mo | ∞ | ∞ | yes |
| Advanced | $120/mo | ∞ | ∞ | priority models |
| Plus | $200/mo | ∞ | ∞ | team features |
| Max | $300/mo | ∞ | ∞ | everything |

Stripe Price IDs live in `.env.local` and `wrangler.jsonc` vars — never change without creating new products.

---

## JumpStudy — Key Architecture

**Pod creation flow:** `/jumpstudy` → user drops notes + subject → `POST /api/study/generate` → Convex `studyPods` + `studyCards` created → redirect to `/jumpstudy/pod/[id]`

**Pod page layout:** 3-column `[200px nav | flex-1 content | 200px stats]` — both sides symmetric, `border-r` / `border-l` dividers

**6 study modes (all science-backed):**
| Mode | Science | Component |
|------|---------|-----------|
| Flashcards | SM-2 Spaced Repetition | `Flashcard` in pod/[id]/page.tsx |
| Quiz | Testing Effect (Roediger 2006) | `QuizMode` |
| Matching | Interleaving (Bjork 1994) | `MatchingGame` |
| Notes | Generative Learning (Wittrock 1992) | `NotesViewer` + `AudioRecorder` |
| AI Tutor | Socratic Method (King 1992) | `AITutor` with 7 Dunlosky method orchestration |
| Whiteboard | Dual Coding (Paivio 1986) | `MermaidWhiteboard` |

**AI Tutor orchestrator:** `ORCHESTRATOR_METHODS` in pod page — picks best Dunlosky method from `api.learningProfiles.getBestMethod`, injects directive into system prompt.

**Marketplace:** `/jumpstudy/marketplace` — public pods, search, subject filters, likes, CAC economics panel ($15 target CAC, $160 LTV, 10.7x ratio).

---

## CodeRabbit Auto-Fix Pipeline

**Flow:** CodeRabbit PR review → email to `support@jumpsgpt.com` → Resend forwards to `POST /api/email/inbound` with `x-inbound-secret` header → `isCodeRabbitEmail()` detects it → `triggerCodeRabbitFix()` dispatches `coderabbit-fix` to GitHub API → `.github/workflows/coderabbit-autofix.yml` runs → `scripts/coderabbit-fixer.mjs` calls GPT-4o to fix → commits to PR branch → comments on PR.

**Required secrets:**
- Cloudflare Worker: `GITHUB_PAT`, `INBOUND_EMAIL_SECRET`, `AI_API_KEY`, `RESEND_API_KEY`
- GitHub Actions: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `AI_API_KEY`
- `GITHUB_REPO` var in `wrangler.jsonc`: `"Jumpsy/jumpgpt"`

---

## Key Files

```
jumpgpt/
├── app/
│   ├── api/
│   │   ├── chat/route.ts          Main AI chat endpoint
│   │   ├── study/generate/        Pod card generation
│   │   ├── email/inbound/         Resend inbound + CodeRabbit detection
│   │   └── billing/               Stripe webhook + health
│   ├── jumpstudy/
│   │   ├── page.tsx               Pod creation chatbot UI
│   │   ├── pod/[id]/page.tsx      3-column study pod (all 6 modes)
│   │   └── marketplace/page.tsx   Public pod grid + CAC panel
│   └── settings/personalization/  Tone, style, language, memory (514 lines)
├── components/
│   ├── landing/
│   │   ├── remotion-player.tsx    22s product tour (JumpsGPT + JumpStudy)
│   │   ├── jumpstudy-section.tsx  JumpStudy landing section with live demo
│   │   └── demo-video.tsx         Interactive chat demo
│   └── ui/
│       ├── leap-mascot.tsx        Leap the pink frog mascot
│       └── jumpstudy-font-applier.tsx  Handwriting font toggle
├── convex/
│   ├── schema.ts                  Source of truth for all DB types
│   └── _generated/                Never edit by hand
├── .github/workflows/
│   ├── deploy-cloudflare.yml      Auto-deploy on push to main
│   └── coderabbit-autofix.yml     CodeRabbit AI fix pipeline
├── scripts/
│   ├── cloudflare-secrets.sh      Interactive Cloudflare secrets wizard
│   ├── setup-github-secrets.sh    GitHub Actions secrets setup
│   └── coderabbit-fixer.mjs       GPT-4o fixer called by autofix workflow
├── wrangler.jsonc                 Cloudflare Worker config (no secrets here)
├── CREDENTIALS.md                 All production credentials (gitignored)
├── LAUNCH.md                      7-step go-live checklist
└── AGENTS.md                      This file (agent rules)
```

---

## Convex Schema (key collections)

- `studyPods` — `title, subject, notes, coverEmoji, isPublic, userId, likesCount`
- `studyCards` — `podId, front, back, hint, tags, dueAt, repetitions, easeFactor`
- `studySessions` — `userId, podId, reviewed, correct, streak`
- `learningProfiles` — `userId, methods (ranked)` — powers AI tutor orchestration
- `userSettings` — `userId, aboutMe, responseInstructions, tone, creativity, language`

Always check `convex/schema.ts` before adding fields. Never remove indexes.

---

## Session Workflow

1. Read `CLAUDE.md` (auto-loaded) and this skill
2. Run `npm run build` to verify baseline
3. Make the smallest change that satisfies the request
4. Run `npm run build` again before finishing
5. Commit with `<type>(<scope>): <description>` format
6. After each session, append one line to `CHANGELOG.md`:
   `## YYYY-MM-DD — <brief summary>`
