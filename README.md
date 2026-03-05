# STEM Play

**Cambodia's kids' math learning app** — learn math, have fun.
រៀនគណិតវិទ្យា។ សប្បាយ។

STEM Play is a mobile learning app designed for Cambodian children ages 3–11 (PreK through Grade 5). It makes foundational mathematics fun and accessible through interactive, game-based activities with animated Cambodian characters, Khmer voice narration, and a culturally relevant reward system — all designed to work offline on affordable phones.

## Project Files

| File | Description |
|------|-------------|
| [concept-note.md](./concept-note.md) | Product vision, features, target audience, and sustainability model |
| [technical-implementation-guide.md](./technical-implementation-guide.md) | Architecture, animation (Rive/Lottie/Lip-Sync), audio, offline-first design, telco billing, team composition |
| [mockup.html](./mockup.html) | Static visual mockup — 8 app screens side-by-side |
| [prototype.html](./prototype.html) | Functioning interactive prototype — single phone frame, full navigation and gameplay |
| [presentation.html](./presentation.html) | 12-slide partner presentation deck (arrow keys / swipe to navigate) |

## Quick Start

Open any `.html` file directly in a browser — no build step or dependencies required.

```bash
# Interactive prototype
open prototype.html

# Partner presentation (use arrow keys to navigate)
open presentation.html

# Static mockup (all 8 screens)
open mockup.html
```

## Key Features

- **Khmer-first** — full Khmer text, voice-over, and cultural context
- **Game-based learning** — puzzles, quizzes, market challenges with visual math aids
- **Animated characters** — Rive-powered with lip-synced Khmer speech
- **Offline-first** — all core content works without internet
- **Parent dashboard** — progress tracking, accuracy, and multi-child support
- **Telco subscription** — phone-number OTP sign-in, operator-scoped billing
- **Seasonal themes** — Khmer New Year, Water Festival content pushed remotely

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Mobile app | Flutter, Rive, Lottie, Drift (SQLite), Firebase |
| Backend | Django, PostgreSQL, Firebase Firestore |
| Animation | Rive state machines, Rhubarb Lip Sync |
| Audio | Pre-recorded Khmer (AAC 64kbps), just_audio |
| Development | Claude Code (AI-augmented engineering) |

See the [Technical Implementation Guide](./technical-implementation-guide.md) for full details including animation references, team composition, and development workflow.
