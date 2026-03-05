# Technical Implementation Guide: Domrey STEM

> **Companion document to [concept-note.md](./concept-note.md)**
> This guide explains the technical approaches behind character animation, audio narration, sound design, offline-first architecture, and telco subscription integration for Domrey STEM. It is written for developers and decision-makers who may be new to building kids' educational apps.

---

## Table of Contents

1. [Character Animation](#1-character-animation)
2. [Lip-Sync for Character Speech](#2-lip-sync-for-character-speech)
3. [Voice-Over & Audio Narration](#3-voice-over--audio-narration)
4. [Sound Design & Audio Layers](#4-sound-design--audio-layers)
5. [Offline-First Architecture](#5-offline-first-architecture)
6. [Low-End Device Considerations](#6-low-end-device-considerations)
7. [Admin Backend & Content Management](#7-admin-backend--content-management)
8. [Telco Integration & Subscription Billing](#8-telco-integration--subscription-billing)
9. [Recommended Technology Stack](#9-recommended-technology-stack)
10. [Animation Implementation References](#10-animation-implementation-references)
11. [Team Composition & Development Workflow](#11-team-composition--development-workflow)

---

## 1. Character Animation

The concept note calls for "original Cambodian characters that guide, encourage, and celebrate with the child" (Section 7). This section explains how to bring those characters to life technically.

### Animation Approaches Compared

| Approach | What It Is | File Size | Interactivity | Best For |
|----------|-----------|-----------|---------------|----------|
| **Skeletal / Bone animation** (Rive, Spine) | A character is built from separate parts (head, arms, legs) connected by an invisible skeleton. You animate the bones, and the parts follow — like a puppet. | Very small (10–100 KB per character) | High — animations respond to user input in real time via state machines | Main characters, interactive mascots, any animation that reacts to the child |
| **Sprite sheets** | A grid of pre-drawn frames played in sequence, like a flip-book. Each frame is a separate image. | Large (hundreds of KB to several MB) | Low — limited to playing predefined sequences | Simple looping animations, retro-style effects |
| **Lottie** | A format that exports After Effects animations as small JSON files. Vectors scale to any screen size. | Small (5–50 KB typical) | Medium — can scrub to specific frames or trigger segments | UI animations: confetti, star bursts, button feedback, loading spinners |

### Recommended Setup

**Rive** for main characters and interactive animations:

- **How it works**: A designer creates a character in the [Rive Editor](https://rive.app) (free tier available). The character's body parts are arranged on a skeleton. The designer then builds a **state machine** — a flowchart of animation states like `idle`, `talking`, `celebrating`, `thinking`. In Flutter code, you trigger transitions between states (e.g., when the child answers correctly, transition from `idle` to `celebrating`).
- **Why Rive**: First-class Flutter support via the `rive` package; `.riv` files are typically 20–80 KB per character; state machines let one file contain dozens of animation states; supports runtime color/skin changes so you can reuse the same skeleton for multiple characters.
- **Shared skeletons and skins**: Design one base skeleton (body proportions, bone structure). Then create **skins** — different hair, clothing, skin tones — on the same skeleton. This means all characters share the same animation set, drastically reducing work and file size. For Domrey STEM's Cambodian characters, you might have 3–4 skins on one skeleton: a boy, a girl, a teacher, and a mascot animal.

**Lottie** for UI micro-animations and rewards:

- **How it works**: A motion designer creates an animation in Adobe After Effects, then exports it as a JSON file using the Bodymovin plugin. In Flutter, the `lottie` package plays these files. The concept note mentions "stickers, badges, and unlockable content" (Section 7) — Lottie is ideal for these reward animations.
- **Use cases**: Star-burst when a child answers correctly, sticker unlock animation, progress bar celebrations, transition effects between activities.

### Asset Size Budgets

| Asset Type | Target Size | Notes |
|-----------|-------------|-------|
| Rive character file | < 100 KB each | Includes all states and skins |
| Lottie animation | < 50 KB each | Keep shapes simple; avoid embedded raster images |
| Total animation assets | < 2 MB | For the starter pack bundled with the APK |

### Optimization Tips

- Use vector shapes in Rive instead of imported bitmap images
- Limit Lottie compositions to under 5 seconds; loop shorter segments
- Test animations on low-end devices early — if frame rate drops below 30 fps, simplify the rig
- Compress Rive files by removing unused artboards before export

---

## 2. Lip-Sync for Character Speech

When characters speak (Khmer narration), their mouths should move believably. There are three approaches, ranging from simple to sophisticated.

### Approach A: Pre-Baked Timeline

The animator manually keyframes mouth positions to match each audio clip. The mouth shapes are baked into the animation file.

- **Pros**: Looks polished; no runtime computation
- **Cons**: Extremely labor-intensive; must redo for every audio clip; does not scale to hundreds of voice lines
- **When to use**: Cinematic intro sequences, app trailer

### Approach B: Amplitude-Based (Volume-Driven Jaw Flap)

At runtime, the app reads the volume (amplitude) of the playing audio and maps it to a single "mouth open" value (0 = closed, 1 = fully open). The character's jaw bone in Rive is driven by this value.

- **How it works in practice**:
  1. Play the audio clip using `just_audio`
  2. Sample the audio amplitude at ~15–20 times per second
  3. Normalize the amplitude to a 0–1 range
  4. Feed that value into a Rive state machine input (a numeric input controlling jaw openness)
- **Pros**: Works automatically with any audio clip; no manual sync; very low CPU cost
- **Cons**: The mouth just opens and closes — it does not form different vowel/consonant shapes
- **When to use**: The majority of voice lines — instructions, encouragement, counting

### Approach C: Phoneme-Based (Rhubarb Lip Sync)

A tool called [Rhubarb Lip Sync](https://github.com/DanielSWolf/rhubarb-lip-sync) analyzes an audio file and outputs a timed sequence of mouth shapes (phonemes). The standard set has 6–9 shapes (e.g., mouth closed, open wide, lips together for "M/B/P", teeth on lip for "F/V", etc.).

- **How it works**:
  1. Run Rhubarb on each audio file at build time (offline, not on device) — it produces a JSON timeline of mouth-shape codes and timestamps
  2. Ship the JSON alongside the audio file
  3. At runtime, as audio plays, read the JSON timeline and set the Rive state machine to the corresponding mouth shape
  4. The Rive file needs 6–9 mouth-shape states that can be triggered by a numeric input
- **Pros**: Realistic lip movement; feels like the character is actually speaking
- **Cons**: Rhubarb's Khmer support is limited (it works best with English); requires processing each audio clip; more states to animate
- **When to use**: Hero moments — character introductions, story narration, important instructions

### Recommended Hybrid Approach

Use **amplitude-based lip-sync as the default** for all voice lines (covers 80–90% of content). Apply **phoneme-based sync for key moments** like character introductions, lesson openers, or story sequences where polish matters most. This balances quality with production effort.

---

## 3. Voice-Over & Audio Narration

The concept note emphasizes "Full Khmer Unicode text and voice-over" and "Audio narration in Khmer for instructions and encouragement" (Section 8). This section covers how to produce and manage that audio.

### Why Pre-Recorded Khmer Voice (Not TTS)

- **No reliable offline Khmer TTS exists** on either iOS or Android. Google's on-device TTS does not support Khmer. Third-party Khmer TTS services require internet and produce robotic-sounding output unsuitable for young children.
- **Kids need warmth**: A real human voice with expressive intonation builds trust and engagement. Monotone TTS can confuse or bore young learners.
- **Offline requirement**: The concept note specifies "offline-first" (Section 9). Pre-recorded audio files work without any network connection.

### Audio Format

| Content Type | Format | Bitrate | Channels | Reasoning |
|-------------|--------|---------|----------|-----------|
| Voice narration | AAC (`.m4a`) | 64 kbps | Mono | Speech clarity at minimal size; ~480 KB per minute |
| Background music | AAC (`.m4a`) | 128 kbps | Stereo | Music benefits from higher quality and stereo imaging |
| Short SFX | AAC (`.m4a`) or Opus | 64 kbps | Mono | Very short clips; keep under 30 KB each |

AAC is chosen over MP3 because it produces better quality at the same bitrate, and both iOS and Android decode it natively with no extra libraries.

### Recording Workflow

1. **Script preparation**: Write every voice line in Khmer script. Organize by category: instructions, number words, encouragement phrases, character dialogue.
2. **Voice talent**: Record with a native Khmer speaker — ideally a warm, friendly voice that children respond well to. For budget-conscious projects, a single versatile voice actor can voice the main character and narration.
3. **Recording environment**: A quiet room with soft furnishings to dampen echo. A USB condenser microphone (e.g., Audio-Technica AT2020) is sufficient. Record at 44.1 kHz / 16-bit WAV.
4. **Loudness normalization**: Normalize all clips to **-16 LUFS** (the broadcast standard for speech). This ensures consistent volume across hundreds of clips so children are never startled by a loud clip or strain to hear a quiet one. Tools: [FFmpeg](https://ffmpeg.org/) (`loudnorm` filter) or [Audacity](https://www.audacityteam.org/) (free).
5. **Export**: Batch-convert WAVs to AAC 64 kbps mono using FFmpeg:
   ```bash
   ffmpeg -i input.wav -c:a aac -b:a 64k -ac 1 output.m4a
   ```
6. **Naming convention**: Use a structured naming scheme:
   ```
   voice/km/numbers/num_01.m4a
   voice/km/numbers/num_02.m4a
   voice/km/instructions/inst_count_objects.m4a
   voice/km/encouragement/enc_great_job.m4a
   voice/en/numbers/num_01.m4a   (English alternative)
   ```

### Bounded Vocabulary

For a math app covering PreK through Grade 5, the voice clip count is manageable:

| Category | Estimated Clips | Examples |
|----------|----------------|---------|
| Number words (0–1000) | ~30–50 | Individual digits, teens, tens, hundreds, "thousand" |
| Math terms | ~30 | "plus", "minus", "equals", "greater than", "multiply", "divide" |
| Instructions | ~50–80 | "Count the mangoes", "Drag the number", "Tap the correct answer" |
| Encouragement / feedback | ~30–40 | "Great job!", "Try again", "Almost!", "You got it!" |
| Character dialogue | ~50–80 | Story intros, lesson openers, celebrations |
| **Total per language** | **~200–300** | **At 64 kbps mono, ~3–5 seconds average = ~100–150 MB of WAV, ~15–25 MB as AAC** |

This is a finite, recordable set — not an open-ended problem.

---

## 4. Sound Design & Audio Layers

The concept note describes "playful sound effects (inspired by Math Kids' squeaky-toy interactions)" (Section 7). Good sound design uses layered audio channels managed independently.

### Three-Layer Audio Model

```
Layer 3 (top):     🎤 Voice Narration    — character speech, instructions
Layer 2 (middle):  🔊 Sound Effects      — taps, correct/wrong, drag, reward
Layer 1 (bottom):  🎵 Background Music   — gentle, looping ambient music
```

Each layer has its own volume control. The parental dashboard (concept note Section 7) can expose these as settings.

### Audio Ducking

When voice narration plays, background music should automatically reduce in volume (called "ducking") so the voice is clearly heard. A typical implementation:

1. Normal music volume: 70%
2. When voice starts: fade music to 20% over 200ms
3. When voice ends: fade music back to 70% over 500ms

This is handled in code — no special audio files needed. The `just_audio` package supports volume control with smooth transitions via `setVolume()`.

### Kid-Friendly Sound Principles

- **No harsh negative sounds**: When a child answers incorrectly, use a gentle "try again" tone (soft xylophone note or a kind voice) — never a buzzer, alarm, or failure horn. Young children can be discouraged by harsh negative feedback.
- **Reward sounds should feel earned**: Bright, ascending chimes or a cheerful character voice. Vary reward sounds slightly to avoid repetition fatigue.
- **Keep sounds short**: SFX should be 0.1–1.0 seconds. Longer sounds risk overlapping with the next interaction.
- **Volume consistency**: All SFX should be normalized to similar loudness so none are startlingly loud.
- **Respect quiet environments**: Default music volume at 50–70%, and remember the user's volume preference.

### Flutter Audio Libraries

| Library | Purpose | Why This One |
|---------|---------|-------------|
| [`just_audio`](https://pub.dev/packages/just_audio) | Voice narration and background music | Robust playback, streaming, gapless looping, volume ducking, cross-platform |
| [`audioplayers`](https://pub.dev/packages/audioplayers) | Sound effects (alternative) | Lightweight, good for fire-and-forget short sounds |
| [`soundpool`](https://pub.dev/packages/soundpool) | Sound effects (Android-optimized) | Pre-loads sounds into memory for zero-latency playback; ideal for tap feedback |

**Recommended setup**: Use `just_audio` for narration and music (two separate player instances). Use `soundpool` (Android) or `audioplayers` (cross-platform) for SFX, pre-loading common sounds at app startup for instant response to taps.

---

## 5. Offline-First Architecture

The concept note states: "Offline-first: All core content available without internet; sync progress when connected" and emphasizes this is "critical for rural Cambodia" (Sections 9, 6). This section explains how to architect for that.

### Asset Bundling Strategy

```
┌─────────────────────────────────────────────┐
│  APK / App Bundle (target: < 50 MB)         │
│                                             │
│  ├── Core app code                          │
│  ├── Rive character files (~500 KB)         │
│  ├── Lottie UI animations (~300 KB)         │
│  ├── Default language voice pack            │
│  │   └── Khmer: ~15–20 MB (AAC 64kbps)     │
│  ├── Sound effects (~2 MB)                  │
│  ├── Background music tracks (~3 MB)        │
│  └── Starter content (PreK + Grade 1)       │
│      └── Activity definitions (JSON, ~1 MB) │
│                                             │
│  Total: ~30–40 MB                           │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  Downloadable Content Packs (via CDN)       │
│                                             │
│  ├── Grade 2–3 content pack (~5 MB ZIP)     │
│  ├── Grade 4–5 content pack (~5 MB ZIP)     │
│  ├── English voice pack (~15 MB ZIP)        │
│  └── Bonus sticker packs (~2 MB each)       │
└─────────────────────────────────────────────┘
```

The app ships with enough content to be immediately useful without any download. Additional grades and languages are fetched as ZIP packs when the device is online, then stored locally for offline use.

### Local Storage

| What | Technology | Why |
|------|-----------|-----|
| User settings (volume, language, child profile) | **Hive** (key-value store) | Ultra-fast, no native dependencies, works offline, simple API |
| Progress, scores, activity completion | **Drift** (SQLite wrapper) | Relational queries (e.g., "which Grade 2 activities has this child completed?"), type-safe, migrations |
| Content manifest (which packs are installed) | **Drift** | Query which activities/assets are available locally |
| Audio & animation files | **File system** | Stored in app's documents directory (or SD card — see Section 6) |

### Audio File Organization

```
app_documents/
├── voice/
│   ├── km/                    ← Bundled with APK (default language)
│   │   ├── numbers/
│   │   ├── instructions/
│   │   └── encouragement/
│   └── en/                    ← Downloaded language pack
│       ├── numbers/
│       ├── instructions/
│       └── encouragement/
├── sfx/                       ← Bundled with APK
│   ├── tap.m4a
│   ├── correct.m4a
│   ├── try_again.m4a
│   └── reward_chime.m4a
└── music/                     ← Bundled with APK
    ├── theme_playful.m4a
    └── theme_calm.m4a
```

### Data Sync Architecture

The app is **local-first**: all writes go to the local database immediately. When a network connection is available, changes sync in the background.

```
┌──────────┐       ┌──────────────┐       ┌───────────────┐
│  App UI  │──────▶│  Local DB    │──────▶│  Sync Queue   │
│          │       │  (Drift)     │       │  (pending     │
│  Writes  │       │              │       │   changes)    │
│  happen  │       │  Reads are   │       │               │
│  here    │       │  always      │       │  Drains when  │
│          │       │  local       │       │  online       │
└──────────┘       └──────────────┘       └───────┬───────┘
                                                  │
                                          ┌───────▼───────┐
                                          │   Firebase    │
                                          │   Firestore   │
                                          │               │
                                          │  Cloud backup │
                                          │  & cross-     │
                                          │  device sync  │
                                          └───────────────┘
```

- **Why Firebase Firestore for v1**: Free tier is generous (1 GB storage, 50K reads/day), Flutter SDK is mature, offline persistence is built in, and it handles conflict resolution automatically.
- **What syncs**: Child progress (scores, completed activities, unlocked stickers), parental settings. Voice/animation assets do NOT sync — they are downloaded separately from the CDN.
- **Conflict resolution**: Last-write-wins for settings; for scores, keep the higher value; for activity completion, once completed it stays completed.

### Content Updates Without App Store Releases

Activities in Domrey STEM are **data-driven**: each activity is defined by a JSON/YAML manifest that specifies the activity type, numbers involved, voice clips to play, and animations to trigger. This means you can add new activities or fix content bugs by pushing a new content pack to the CDN — no app update required.

```json
{
  "activity_id": "add_01_03",
  "type": "addition_puzzle",
  "grade": 1,
  "operands": [3, 4],
  "voice_instruction": "voice/km/instructions/inst_add_two.m4a",
  "voice_numbers": ["voice/km/numbers/num_03.m4a", "voice/km/numbers/num_04.m4a"],
  "reward_animation": "lottie/star_burst.json",
  "character_state": "encouraging"
}
```

The app checks for content updates on launch (if online) and downloads incremental packs in the background.

---

## 6. Low-End Device Considerations

Cambodia context: many families use affordable Android phones with 2–3 GB RAM, 16–32 GB storage, and mid-range processors. The concept note highlights "Small app size for low-storage devices common in Cambodia" and "direct APK distribution for schools without Play Store access" (Sections 9, 13).

### Target Device Specs

| Spec | Minimum Target | Comfortable Target |
|------|---------------|-------------------|
| RAM | 2 GB | 3 GB |
| Storage (free) | 500 MB available | 2 GB available |
| Processor | Quad-core 1.3 GHz | Octa-core 1.8 GHz |
| Android version | Android 8.0 (API 26) | Android 10+ |
| iOS version | iOS 14 | iOS 16+ |

### APK Size Optimization

- **Target**: Initial install < 50 MB
- **Android App Bundle (AAB)**: Upload AAB to Google Play. Google generates optimized APKs per device — a phone only downloads the resources it needs (correct screen density, CPU architecture). This can reduce download size by 20–40% compared to a universal APK.
- **APK for side-loading**: For schools and NGOs distributing via USB drives, build a universal APK. Use `flutter build apk --split-per-abi` to produce architecture-specific APKs (~30 MB each instead of ~60 MB for a fat APK).

### Storage Management

- **SD card support**: Many low-end Android devices rely on SD cards. Use `path_provider` to check for external storage and offer the user a choice of where to store downloaded content packs.
- **LRU cache eviction**: If device storage is low, offer to remove content packs for grades the child has completed. Keep a "recently used" list and suggest removing the least recently accessed packs.
- **Storage-aware downloads**: Before downloading a content pack, check available storage. If insufficient, show a friendly message explaining what can be removed to make space — never silently fail.
- **Lazy asset loading**: Don't load all Rive/Lottie files into memory at once. Load only the assets needed for the current activity, and dispose of them when navigating away.

### Performance Guidelines

- Test on a real low-end device (e.g., a ~$80 Android phone) throughout development, not just on emulators
- Keep the widget tree shallow — deeply nested widgets hurt rebuild performance
- Use `const` constructors wherever possible in Flutter
- Profile with Flutter DevTools to catch jank (frames exceeding 16ms)
- Avoid loading full-resolution images — pre-scale assets to the maximum display size needed

### Side-Loading Distribution

For the concept note's "direct APK distribution for schools without Play Store access" (Section 13):

- Host the APK on a simple website or share via USB/SD card
- Include a version-check endpoint so the app can notify users of updates
- Provide clear Khmer-language installation instructions (with screenshots) since side-loading requires enabling "Install from unknown sources"

---

## 7. Admin Backend & Content Management

The mobile app is offline-first and talks directly to Firebase for progress sync. However, managing content packs, seasonal character themes, and challenge difficulty requires an **admin dashboard** — a separate web application used by the Domrey STEM team (not end users).

### Why an Admin Backend Is Needed

| Admin Need | Example |
|-----------|---------|
| Content pack management | Upload new activity ZIPs, audio files, Rive character files |
| Seasonal character themes | Swap characters for Khmer New Year, Water Festival, Pchum Ben |
| Challenge complexity tuning | Adjust operand ranges, time limits, difficulty curves per grade |
| Subscription management | View subscriber status, manage telco billing integration (see [Section 8](#8-telco-integration--subscription-billing)) |
| Analytics review | View usage data, progress distributions, popular activities, subscription metrics |

### Architecture: Django Admin + Firebase for Mobile

The admin backend and the mobile app serve different audiences and have different requirements. They are best kept as **separate systems connected by shared storage**.

```
┌─────────────────────────────────────────────────────────┐
│  ADMIN SIDE (2–5 users: team + partner)                 │
│                                                         │
│  Django + Django Admin + PostgreSQL                      │
│  Hosted on: Railway / Render / Fly.io ($5–15/month)     │
│                                                         │
│  ├── Content pack CRUD (upload Rive, audio, JSON)       │
│  ├── Character theme manager (seasonal skins)           │
│  ├── Challenge config editor (difficulty parameters)    │
│  ├── Subscriber management & telco billing API          │
│  └── Analytics dashboard (usage, subscriptions)         │
│                                                         │
│  Publishes to:                                          │
│  ├── S3 / Firebase Storage → CDN (asset files)          │
│  └── Firestore (content manifests, active theme,        │
│      challenge configs)                                 │
└────────────────────────┬────────────────────────────────┘
                         │
                         │  publishes content packs + configs
                         ▼
┌─────────────────────────────────────────────────────────┐
│  CDN + Firebase Firestore                               │
│  (the bridge between admin and mobile)                  │
│                                                         │
│  ├── CDN: asset ZIPs (audio, Rive, images)              │
│  ├── Firestore: content manifests, active theme,        │
│  │   challenge configs, feature flags                   │
│  ├── Firestore: child progress (synced from app)        │
│  └── Firestore: subscriber entitlements (per user)      │
└────────────────────────┬────────────────────────────────┘
                         │
                         │  app reads configs + downloads assets
                         ▼
┌─────────────────────────────────────────────────────────┐
│  MOBILE APP (thousands of users)                        │
│                                                         │
│  Flutter + Firestore SDK + Local DB (Drift)             │
│                                                         │
│  ├── Sign-in via phone OTP (telco numbers only)         │
│  ├── On launch: check Firestore for active theme,       │
│  │   new content packs, updated challenge configs       │
│  ├── Check subscription entitlement (cached locally)    │
│  ├── Download new assets from CDN if needed             │
│  ├── All gameplay is local/offline                      │
│  └── Sync progress to Firestore when online             │
└─────────────────────────────────────────────────────────┘
```

### Why Django (and Why Not the Alternatives)

**Django Admin is the key advantage.** By defining your data models (content packs, themes, challenge configs), Django automatically generates a full CRUD admin interface — forms, list views, search, filters, and role-based permissions. This gives you 80% of your admin dashboard essentially for free.

| Option | Verdict | Reasoning |
|--------|---------|-----------|
| **Django on simple hosting** | **Recommended** | Django Admin covers most admin needs out of the box; Python ecosystem is strong for analytics; simple to deploy on Railway/Render/Fly.io for $5–15/month |
| Django + Zappa on AWS Lambda | Not recommended | Django is too heavy for Lambda — cold starts of 3–10 seconds make the admin feel sluggish; API Gateway has a 10 MB upload limit (bad for asset packs); package size limits get tight |
| Firebase Cloud Functions alone | Not recommended | No built-in admin UI — you'd build everything from scratch; 10 MB upload limit; JavaScript/TypeScript only |
| Both combined | Not recommended | Two cloud providers, two billing systems, two deployment pipelines — excessive operational overhead for a small team |

### Key Django Models

These are the core data models the admin dashboard manages:

#### Content Packs

```python
class ContentPack(models.Model):
    name = models.CharField(max_length=200)           # "Grade 2 Addition Pack"
    grade = models.IntegerField()                      # 2
    version = models.IntegerField(default=1)
    zip_file = models.FileField(upload_to='packs/')    # Uploaded asset ZIP
    manifest = models.JSONField()                      # Activity definitions
    is_published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    published_at = models.DateTimeField(null=True, blank=True)
```

#### Seasonal Themes — Festival & Event System

Cambodia has several major cultural events throughout the year. The admin must be able to design, preview, and activate themed experiences for each — changing character costumes, environment decorations, welcome messages, and voice narrations — all from the Django admin console.

**Cambodian Festival Calendar (recurring annually):**

| Festival | Typical Dates | Theme Elements |
|----------|--------------|----------------|
| **Khmer New Year** (ចូលឆ្នាំថ្មី) | Apr 13–16 | Lanterns, lotus flowers, pagoda settings, traditional costumes |
| **Pchum Ben** (ភ្ជុំបិណ្ឌ) | Sep–Oct (lunar) | Pagoda offerings, candles, rice cakes, respectful/gentle tone |
| **Water Festival** (បុណ្យអុំទូក) | Nov (lunar) | Boat races, river scenes, festive colors, fireworks |
| **Chinese New Year** | Jan–Feb (lunar) | Red & gold decorations, dragon/lion dance, family gatherings |
| **International Children's Day** | Jun 1 | Playful extra content, special stickers, celebration |
| **Independence Day** | Nov 9 | National colors, flags, patriotic elements |

**Django Models:**

```python
class FestivalEvent(models.Model):
    """Defines a recurring cultural event (e.g., Khmer New Year happens every year)."""
    name = models.CharField(max_length=200)            # "Khmer New Year"
    name_km = models.CharField(max_length=200)         # "ចូលឆ្នាំថ្មី"
    description = models.TextField(blank=True)
    icon_emoji = models.CharField(max_length=10)       # "🏮"

    class Meta:
        ordering = ['name']

    def __str__(self):
        return f"{self.icon_emoji} {self.name}"


class SeasonalTheme(models.Model):
    """A specific themed experience for a festival event in a given year."""
    class Status(models.TextChoices):
        DRAFT = 'draft'            # Being designed
        READY = 'ready'            # Assets complete, awaiting activation
        ACTIVE = 'active'          # Currently live in the app
        ARCHIVED = 'archived'      # Past event, no longer active

    event = models.ForeignKey(FestivalEvent, on_delete=models.CASCADE)
    year = models.IntegerField()                       # 2027
    status = models.CharField(max_length=20, choices=Status.choices, default='draft')

    # Schedule
    active_from = models.DateTimeField()               # Apr 13, 2027 00:00
    active_until = models.DateTimeField()              # Apr 16, 2027 23:59
    auto_activate = models.BooleanField(default=False) # Auto-activate on active_from
    activated_by = models.ForeignKey('auth.User', null=True, blank=True, on_delete=models.SET_NULL)
    activated_at = models.DateTimeField(null=True, blank=True)

    # Welcome & messaging
    welcome_title = models.CharField(max_length=200)         # "Happy New Year!"
    welcome_title_km = models.CharField(max_length=200)      # "សួស្តីឆ្នាំថ្មី!"
    welcome_message = models.TextField(blank=True)           # Shown on home screen
    welcome_message_km = models.TextField(blank=True)

    # Visual theming
    primary_color = models.CharField(max_length=7, default='#fbbf24')    # Hex color
    secondary_color = models.CharField(max_length=7, default='#f59e0b')
    background_gradient = models.JSONField(default=dict)                 # {"start": "#fbbf24", "end": "#fff7ed"}
    banner_image = models.FileField(upload_to='themes/banners/', blank=True)

    # Priority (if two themes overlap, higher wins)
    priority = models.IntegerField(default=0)

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = ['event', 'year']
        ordering = ['-year', '-priority']

    def __str__(self):
        return f"{self.event.name} {self.year} [{self.status}]"


class ThemeCharacterCostume(models.Model):
    """A themed costume/skin for a character during a festival."""
    theme = models.ForeignKey(SeasonalTheme, on_delete=models.CASCADE, related_name='costumes')
    character_name = models.CharField(max_length=100)          # "Domrey", "Neary", "Rithy"
    rive_skin_name = models.CharField(max_length=100)          # Skin ID in the .riv file
    rive_file = models.FileField(upload_to='themes/characters/', blank=True)  # Override .riv if needed
    preview_image = models.ImageField(upload_to='themes/previews/', blank=True)
    description = models.CharField(max_length=200, blank=True) # "Domrey in sampot and krama"

    def __str__(self):
        return f"{self.character_name} — {self.theme}"


class ThemeEnvironment(models.Model):
    """Decorative elements and background assets for a theme."""
    theme = models.ForeignKey(SeasonalTheme, on_delete=models.CASCADE, related_name='environments')
    screen = models.CharField(max_length=50)                   # "home", "gameplay", "celebration"
    background_asset = models.FileField(upload_to='themes/backgrounds/')
    decoration_overlay = models.FileField(upload_to='themes/decorations/', blank=True)
    lottie_effect = models.FileField(upload_to='themes/lottie/', blank=True)  # e.g., floating lanterns
    description = models.CharField(max_length=200, blank=True)


class ThemeVoiceLine(models.Model):
    """Festival-specific voice narration clips."""
    theme = models.ForeignKey(SeasonalTheme, on_delete=models.CASCADE, related_name='voice_lines')
    line_type = models.CharField(max_length=50)                # "welcome", "encouragement", "celebration"
    text = models.TextField()                                  # "Happy New Year! Let's count lanterns!"
    text_km = models.TextField()                               # Khmer version
    audio_file = models.FileField(upload_to='themes/voice/')   # Themed voice clip
    character = models.CharField(max_length=100, default='Domrey')

    def __str__(self):
        return f"[{self.line_type}] {self.text[:50]}"


class ThemeActivity(models.Model):
    """Special themed activities available only during the festival."""
    theme = models.ForeignKey(SeasonalTheme, on_delete=models.CASCADE, related_name='activities')
    name = models.CharField(max_length=200)                    # "Count the Lanterns"
    name_km = models.CharField(max_length=200)
    description = models.CharField(max_length=300)
    icon_emoji = models.CharField(max_length=10)
    activity_manifest = models.JSONField()                     # Standard activity config (operands, type, etc.)
    sort_order = models.IntegerField(default=0)
```

**Admin Workflow:**

```
┌─────────────────────────────────────────────────────────────────┐
│  Theme Creation & Activation Workflow                           │
│                                                                 │
│  1. Admin creates a SeasonalTheme (e.g., "Pchum Ben 2027")     │
│     └── Status: DRAFT                                           │
│                                                                 │
│  2. Designer uploads assets via Django admin:                   │
│     ├── Character costumes (Rive skins or override .riv files)  │
│     ├── Environment backgrounds & decorations per screen        │
│     ├── Lottie effects (floating candles, lantern animations)   │
│     ├── Themed voice lines (recorded, uploaded as .m4a)         │
│     └── Themed activities (JSON manifests)                      │
│                                                                 │
│  3. Admin previews the theme in a staging build of the app      │
│     └── Status: READY                                           │
│                                                                 │
│  4. Admin clicks "Activate" (or auto_activate fires on date):   │
│     ├── Status → ACTIVE                                         │
│     ├── Theme config published to Firestore                     │
│     ├── Assets pushed to CDN (if not already uploaded)          │
│     └── App picks up new theme on next launch/sync              │
│                                                                 │
│  5. After the event ends (active_until passes):                 │
│     ├── Status → ARCHIVED (automatic via management command)    │
│     └── App reverts to default theme                            │
│                                                                 │
│  6. Next year: admin can clone last year's theme as a starting  │
│     point, update dates and refresh assets                      │
└─────────────────────────────────────────────────────────────────┘
```

**Django Admin custom action for activation:**

```python
@admin.action(description="Activate selected theme (push to live app)")
def activate_theme(modeladmin, request, queryset):
    if queryset.count() != 1:
        modeladmin.message_user(request, "Select exactly one theme to activate.", level='error')
        return
    theme = queryset.first()
    if theme.status != SeasonalTheme.Status.READY:
        modeladmin.message_user(request, "Theme must be in READY status before activation.", level='error')
        return

    # Deactivate any currently active themes with lower priority
    SeasonalTheme.objects.filter(status='active').update(status='archived')

    theme.status = SeasonalTheme.Status.ACTIVE
    theme.activated_by = request.user
    theme.activated_at = timezone.now()
    theme.save()

    # Publish to Firestore for the mobile app
    publish_theme_to_firestore(theme)

    modeladmin.message_user(request, f"✓ {theme} is now LIVE in the app.")


@admin.register(SeasonalTheme)
class SeasonalThemeAdmin(admin.ModelAdmin):
    list_display = ['event', 'year', 'status', 'active_from', 'active_until', 'activated_by']
    list_filter = ['status', 'event', 'year']
    actions = [activate_theme]
    inlines = [CostumeInline, EnvironmentInline, VoiceLineInline, ActivityInline]
```

**Firestore document structure (published by Django):**

```json
{
  "active_theme": {
    "id": "pchum_ben_2027",
    "event_name": "Pchum Ben",
    "event_name_km": "ភ្ជុំបិណ្ឌ",
    "welcome_title": "Pchum Ben Blessings",
    "welcome_title_km": "ពរជ័យបុណ្យភ្ជុំបិណ្ឌ",
    "welcome_message_km": "សូមរំលឹកវិញ្ញាណក្សត្រ...",
    "colors": {
      "primary": "#7c3aed",
      "secondary": "#a78bfa",
      "gradient_start": "#7c3aed",
      "gradient_end": "#f5f3ff"
    },
    "costumes": {
      "Domrey": { "skin": "pchum_ben_formal", "cdn_url": "..." },
      "Neary": { "skin": "pchum_ben_dress", "cdn_url": "..." }
    },
    "environments": {
      "home": { "background": "...", "decoration": "...", "lottie": "..." },
      "gameplay": { "background": "...", "decoration": "..." }
    },
    "voice_lines": [
      { "type": "welcome", "audio_url": "...", "text_km": "..." },
      { "type": "encouragement", "audio_url": "...", "text_km": "..." }
    ],
    "activities": [
      { "name": "Count the Rice Cakes", "manifest": {...} }
    ],
    "active_until": "2027-10-06T23:59:59Z",
    "cdn_base_url": "https://cdn.domreystem.app/themes/pchum_ben_2027/"
  }
}
```

**Mobile app theme loading:**

```
On app launch / foreground resume:
  1. Read Firestore → active_theme document
  2. Compare with locally cached theme ID
  3. If changed:
     ├── Download new assets from CDN (costumes, backgrounds, voice)
     ├── Cache in app documents directory
     └── Apply theme: swap Rive skins, set background colors, load themed voice lines
  4. If active_until has passed → revert to default theme, clear cache
  5. If offline → use cached theme (or default if no cache)
```

#### Challenge Configurations

```python
class ChallengeConfig(models.Model):
    grade = models.IntegerField()
    operation = models.CharField(max_length=20)        # "addition", "subtraction", etc.
    min_operand = models.IntegerField(default=1)
    max_operand = models.IntegerField(default=10)
    num_questions = models.IntegerField(default=10)
    time_limit_seconds = models.IntegerField(null=True, blank=True)
    allow_negative_results = models.BooleanField(default=False)
    carry_borrow_enabled = models.BooleanField(default=False)
    updated_at = models.DateTimeField(auto_now=True)
```

The admin adjusts these parameters in Django Admin — for example, increasing `max_operand` from 10 to 20 for Grade 2 addition, or enabling `carry_borrow_enabled` for Grade 3. Changes are pushed to Firestore and take effect the next time the app syncs.

### Publishing Flow

```
Admin action                   What happens
─────────────────────────────────────────────────────────────
Upload content pack     →  ZIP stored in S3/Firebase Storage
                        →  Manifest written to Firestore
                        →  CDN URL recorded in Firestore

Activate seasonal theme →  Theme config written to Firestore
                        →  Rive file uploaded to CDN
                        →  App picks up new theme on next launch

Update challenge config →  New parameters written to Firestore
                        →  App reads updated config on next sync
```

### Firebase Cloud Functions: Limited Role

Use Firebase Cloud Functions only for lightweight, event-driven tasks — not as the primary backend:

- **CDN cache invalidation** when the admin publishes a new content pack
- **Aggregation jobs** (e.g., nightly rollup of usage stats from Firestore for the analytics dashboard)
- **Push notifications** to inform users of new content (optional, future)

### Hosting & Cost

The admin dashboard is a low-traffic internal tool (2–5 users). It does not need serverless scale or complex infrastructure.

| Hosting Option | Cost | Notes |
|---------------|------|-------|
| [Railway](https://railway.app) | Free tier → $5/month | Simple Git-push deploys; includes PostgreSQL |
| [Render](https://render.com) | Free tier → $7/month | Auto-deploys from GitHub; managed PostgreSQL |
| [Fly.io](https://fly.io) | Free tier → $5/month | Global edge deployment; good for teams in multiple locations |

All three support Django out of the box with minimal configuration. For asset storage, use Firebase Storage (already in the stack) or AWS S3 — either works with Django's file field via `django-storages`.

---

## 8. Telco Integration & Subscription Billing

The concept note's sustainability model (Section 13) is a **partnership with a local Cambodian mobile operator**. The app is bundled into the telco's subscription plans. This section covers the technical implementation of phone-number authentication, billing integration, and subscription lifecycle management.

### Phone-Number Authentication

During the exclusivity period, only the partner telco's subscribers can sign in.

#### How It Works

1. **User enters phone number** — the app validates the number prefix against the partner operator's known ranges (e.g., Cellcard: `078`, `088`, `089`; Smart: `010`, `069`, `070`, etc.)
2. **OTP delivery** — Django backend calls the telco's SMS gateway (or Firebase Auth's phone provider) to send a 6-digit one-time password
3. **OTP verification** — user enters the code; backend verifies and issues a session token
4. **Entitlement check** — backend queries the telco billing API to confirm the user has an active subscription; writes the entitlement status to Firestore

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  App     │───▶│  Django      │───▶│  Telco SMS   │───▶│  User's      │
│          │    │  Backend     │    │  Gateway     │    │  Phone       │
│  Enter   │    │              │    │              │    │  (receives   │
│  phone # │    │  Send OTP    │    │  Deliver OTP │    │   OTP)       │
└──────────┘    └──────────────┘    └──────────────┘    └──────────────┘
      │                │
      │  Submit OTP    │  Verify OTP
      │───────────────▶│
      │                │──── Check telco billing API
      │                │     (is this number subscribed?)
      │                │
      │◀───────────────│  Return session token
      │                │  + entitlement (active/expired/grace)
```

#### Operator Scoping

```python
# Allowed number prefixes during exclusivity period
ALLOWED_PREFIXES = {
    "cellcard": ["855078", "855088", "855089", "85512", "85517", "85577"],
    # Add other operators post-exclusivity
}

# Validate before sending OTP
def is_allowed_number(phone_number: str, partner: str) -> bool:
    return any(phone_number.startswith(p) for p in ALLOWED_PREFIXES[partner])
```

When the exclusivity period ends, the admin simply updates the allowed prefixes in Django Admin (or removes the restriction entirely) — no app update needed.

#### Why Not Firebase Auth Alone?

Firebase Auth supports phone-number sign-in, but:
- It cannot restrict sign-in to specific operator prefixes (you'd need backend validation anyway)
- It cannot query the telco billing system
- The Django backend needs to be the authority on subscription status

Firebase Auth can still be used as the underlying OTP mechanism (it handles SMS delivery, rate limiting, and abuse prevention), but the **Django backend wraps it** with operator validation and billing checks.

### Telco Billing Integration

The Django backend integrates with the telco's billing API to manage subscriptions. The exact API varies by operator, but the pattern is standard across Cambodian telcos.

#### Subscription Lifecycle

```
Subscribe          Renew              Cancel / Expire
    │                │                      │
    ▼                ▼                      ▼
┌────────┐     ┌────────────┐     ┌─────────────────┐
│ ACTIVE │────▶│  ACTIVE    │────▶│  GRACE PERIOD   │
│        │     │  (renewed) │     │  (3–7 days)     │
└────────┘     └────────────┘     └────────┬────────┘
                                           │
                                    ┌──────▼──────┐
                                    │   EXPIRED   │
                                    │  (read-only │
                                    │   access)   │
                                    └─────────────┘
```

#### Django Models for Subscription

```python
class Subscriber(models.Model):
    phone_number = models.CharField(max_length=20, unique=True, db_index=True)
    operator = models.CharField(max_length=50)         # "cellcard", "smart", etc.
    firebase_uid = models.CharField(max_length=128, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Subscription(models.Model):
    class Status(models.TextChoices):
        ACTIVE = 'active'
        GRACE = 'grace'          # Payment failed, retry window
        EXPIRED = 'expired'
        CANCELLED = 'cancelled'

    subscriber = models.ForeignKey(Subscriber, on_delete=models.CASCADE)
    plan = models.CharField(max_length=50)             # "monthly", "weekly"
    status = models.CharField(max_length=20, choices=Status.choices)
    telco_subscription_id = models.CharField(max_length=200)  # ID from telco API
    started_at = models.DateTimeField()
    expires_at = models.DateTimeField()
    grace_until = models.DateTimeField(null=True, blank=True)
    cancelled_at = models.DateTimeField(null=True, blank=True)
    updated_at = models.DateTimeField(auto_now=True)
```

#### Integration Pattern: Webhooks + Polling

Most telco billing APIs support two notification mechanisms:

1. **Webhooks (preferred)**: The telco calls a Django endpoint when a subscription event occurs (new subscription, renewal, cancellation, payment failure). Django updates the local `Subscription` record and pushes the new status to Firestore.

2. **Polling (fallback)**: A periodic Django management command (e.g., every 15 minutes via cron) queries the telco API for subscription status changes. This catches any missed webhooks.

```python
# Django webhook endpoint (simplified)
@csrf_exempt
@require_POST
def telco_webhook(request):
    payload = verify_telco_signature(request)  # Verify authenticity

    event_type = payload['event']              # "subscribe", "renew", "cancel", "payment_failed"
    phone = payload['msisdn']
    telco_sub_id = payload['subscription_id']

    subscription = Subscription.objects.get(telco_subscription_id=telco_sub_id)

    if event_type == 'cancel':
        subscription.status = Subscription.Status.GRACE
        subscription.grace_until = now() + timedelta(days=7)
    elif event_type == 'renew':
        subscription.status = Subscription.Status.ACTIVE
        subscription.expires_at = parse(payload['next_renewal'])

    subscription.save()

    # Push updated entitlement to Firestore for the mobile app
    sync_entitlement_to_firestore(subscription)

    return JsonResponse({'status': 'ok'})
```

### Entitlement Enforcement in the App

The mobile app must handle subscription status gracefully, especially offline.

#### Caching Strategy

```
On sign-in / app launch (when online):
  1. Fetch entitlement from Firestore → {status, expires_at, grace_until}
  2. Cache locally in Hive

On every app open:
  1. Read cached entitlement from Hive
  2. If status == "active" and expires_at > now → full access
  3. If status == "grace" and grace_until > now → full access + show renewal nudge
  4. If expired → show limited/free-tier content + subscription prompt
  5. If offline and cached entitlement exists → trust the cache (generous grace)
```

#### What Happens When a Subscription Expires?

- **Do NOT lock the child out abruptly.** This is a kids' app — a child mid-activity should never see a paywall screen.
- **Grace period (3–7 days)**: Full access continues. Show a gentle nudge to the parent (not the child) to renew.
- **After grace**: Degrade to a free tier — e.g., access to PreK content only, no new content packs. The child can still use what's downloaded.
- **Progress is never lost**: All scores, stickers, and completions remain regardless of subscription status.

### Content Gating by Subscription Tier

If the telco offers multiple subscription tiers (e.g., basic vs. premium), the content manifest supports a `tier` field:

```json
{
  "activity_id": "mul_03_01",
  "type": "multiplication_quiz",
  "grade": 3,
  "tier": "premium",
  "..."
}
```

The app filters available activities based on the subscriber's entitlement tier. Free-tier users see a subset; premium users see everything.

### Security Considerations

- **Server-side entitlement is the source of truth** — the app's local cache is for UX smoothness, not access control. Sensitive content (future paid features) should re-verify entitlement when online.
- **Webhook signature verification** — always validate that billing webhooks genuinely come from the telco (HMAC signature, IP allowlist, or mutual TLS).
- **Rate-limit OTP requests** — prevent abuse of the SMS gateway (Firebase Auth handles this if used as the OTP provider).
- **Do not store sensitive billing data** in the mobile app — the app only knows "active/grace/expired", never payment details.

---

## 9. Recommended Technology Stack

| Concern | Technology | Role |
|---------|-----------|------|
| **App framework** | Flutter | Cross-platform (iOS + Android) from a single codebase; strong animation support; large ecosystem |
| **Character animation** | Rive | Interactive skeletal animation with state machines; tiny file sizes; Flutter-native |
| **UI micro-animations** | Lottie (`lottie` package) | Reward animations, transitions, UI polish |
| **Audio: narration & music** | `just_audio` | Playback, looping, volume control, ducking |
| **Audio: sound effects** | `soundpool` / `audioplayers` | Low-latency pre-loaded SFX |
| **Local key-value storage** | Hive | Settings, preferences, simple state |
| **Local relational database** | Drift (SQLite) | Progress tracking, scores, content manifest |
| **Cloud sync** | Firebase Firestore | Background sync of progress; offline persistence built in |
| **Content delivery** | Firebase Hosting / CDN | Downloadable content packs and language packs |
| **Admin backend** | Django + Django Admin | Content management, seasonal themes, challenge configs, analytics |
| **Admin database** | PostgreSQL | Relational storage for admin models; managed by hosting provider |
| **Admin hosting** | Railway / Render / Fly.io | Simple, low-cost hosting for a low-traffic internal tool |
| **Asset storage** | S3 or Firebase Storage | File storage for uploaded content packs and theme assets |
| **Authentication** | Firebase Auth (phone) + Django | OTP-based phone sign-in; Django wraps with operator validation and billing checks |
| **Telco billing** | Django + telco API (webhooks) | Subscription lifecycle: activate, renew, cancel, grace periods |
| **Lip-sync (key moments)** | Rhubarb Lip Sync | Offline phoneme extraction at build time |
| **Audio processing** | FFmpeg | Batch encoding, loudness normalization |
| **State management** | Riverpod | Recommended for managing app state, audio players, and animation controllers |
| **Navigation** | GoRouter | Declarative routing with deep link support |

### Why Flutter?

The concept note considers "Flutter or React Native" (Section 9). Flutter is recommended because:

1. **Animation performance**: Flutter renders via Skia/Impeller directly to canvas, providing smoother animations than React Native's bridge-based architecture
2. **Rive integration**: Rive's Flutter runtime is first-class and officially maintained
3. **Offline capabilities**: Flutter's compilation to native ARM code means no JavaScript engine overhead, better for low-end devices
4. **Single codebase**: One codebase for both iOS and Android, with platform-specific optimizations where needed
5. **Community & ecosystem**: Large package ecosystem for audio, storage, and Firebase integration

---

## Quick Reference: Key Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Animation engine | Rive (characters) + Lottie (UI) | Small files, interactive, Flutter-native |
| Lip-sync default | Amplitude-based jaw flap | Works with any audio, zero manual effort |
| Voice-over approach | Pre-recorded Khmer | No offline Khmer TTS exists; warmth matters for kids |
| Audio format | AAC 64kbps mono (voice) | Best quality-to-size ratio; native decoding |
| Local database | Drift (SQLite) | Relational queries for progress tracking |
| Cloud backend (mobile) | Firebase Firestore | Generous free tier, built-in offline, Flutter SDK |
| Admin backend | Django on Railway/Render | Django Admin gives 80% of the dashboard for free; simple hosting for low-traffic tool |
| Admin → mobile bridge | Firestore + CDN | Admin publishes configs/assets; app reads them on sync |
| Authentication | Phone OTP (Firebase Auth + Django) | Operator-scoped sign-in; no passwords or email needed |
| Subscription billing | Django ↔ telco webhook API | Server-side entitlement; cached locally for offline grace |
| Content architecture | Data-driven JSON manifests | Update content without app store releases |
| APK size target | < 50 MB | Practical for low-storage devices and side-loading |

---

## 10. Animation Implementation References

This section provides direct links to official documentation, tutorials, and example projects for every animation technology used in Domrey STEM.

### 10.1 Rive — Interactive Character Animation (Primary)

Rive is the recommended engine for all interactive character animation in Domrey STEM. It powers Duolingo's character system at scale and is proven for kids' apps.

| Resource | Link | Description |
|----------|------|-------------|
| **Official Docs** | [rive.app/docs](https://rive.app/docs) | Primary documentation — editor, runtimes, state machines |
| **Legacy Help Center** | [help.rive.app](https://help.rive.app/) | Detailed guides (still referenced by many tutorials) |
| **Flutter Runtime** | [pub.dev/packages/rive](https://pub.dev/packages/rive) | Official Flutter package |
| **Flutter Setup Guide** | [rive.app/docs/runtimes/flutter](https://rive.app/docs/runtimes/flutter/flutter) | Getting started with Rive in Flutter |
| **Flutter Examples** | [github.com/rive-app/rive-flutter/.../example](https://github.com/rive-app/rive-flutter/tree/master/example/lib) | Code samples: state machines, simple animation, custom controllers |
| **State Machine Docs** | [help.rive.app/editor/state-machine](https://help.rive.app/editor/state-machine) | How to build state machines (idle → talking → celebrating) |
| **Runtime State Machines** | [help.rive.app/runtimes/state-machines](https://help.rive.app/runtimes/state-machines) | Controlling state machines from Flutter code |
| **React Native Runtime** | [npmjs.com/package/@rive-app/react-native](https://www.npmjs.com/package/@rive-app/react-native) | If RN is used instead of Flutter (requires RN 0.78+) |
| **Marketplace (free assets)** | [rive.app/marketplace](https://rive.app/marketplace/) | Free and paid character rigs, UI animations |
| **Awesome Rive** | [github.com/rive-app/awesome-rive](https://github.com/rive-app/awesome-rive) | Curated list of Rive examples, apps, and tutorials |
| **Duolingo Case Study** | [elisawicki.blog: How Duolingo uses Rive](https://elisawicki.blog/p/how-exactly-is-duolingo-using-rive) | Deep analysis of Duolingo's Rive implementation |
| **Hero Animations** | [rive.app/use-cases/hero-animations](https://rive.app/use-cases/hero-animations) | Use cases and patterns for app character animation |
| **Pricing** | [rive.app/pricing](https://rive.app/pricing) | Free tier includes editor + all runtimes; Cadet $9/mo; Voyager $32/mo; Enterprise $120/mo |

**Key Rive concepts for Domrey STEM:**
- **Shared skeletons + skins**: One skeleton rig → multiple characters (Domrey, Neary, Rithy) via skin swapping
- **State machines**: Define states (`idle`, `talking`, `correct_reaction`, `celebrating`, `thinking`) with transitions driven by code inputs
- **Additive blend states**: Smooth mouth-shape transitions for lip-sync (Duolingo approach)
- **Runtime skin swaps**: Seasonal themes (Khmer New Year costumes) without shipping separate .riv files

### 10.2 Spine — 2D Skeletal Animation (Alternative)

Spine is a mature alternative to Rive, widely used in games. Consider Spine if the animation team already has Spine expertise.

| Resource | Link | Description |
|----------|------|-------------|
| **Official Docs** | [esotericsoftware.com/spine-documentation](http://en.esotericsoftware.com/spine-documentation) | Full editor and runtime documentation |
| **In-Depth Guide** | [esotericsoftware.com/spine-in-depth](http://en.esotericsoftware.com/spine-in-depth) | Detailed technical walkthrough |
| **Flutter Runtime** | [pub.dev/packages/spine_flutter](https://pub.dev/packages/spine_flutter) | Official Spine Flutter package (FFI-based) |
| **Flutter Runtime Docs** | [esotericsoftware.com/spine-flutter](http://esotericsoftware.com/spine-flutter) | Setup and usage guide |
| **All Runtimes (GitHub)** | [github.com/EsotericSoftware/spine-runtimes](https://github.com/EsotericSoftware/spine-runtimes) | Source code and examples for all platforms |
| **Download / Trial** | [esotericsoftware.com/spine-download](http://esotericsoftware.com/spine-download) | Free trial for Windows, Mac, Linux |
| **Pricing** | [esotericsoftware.com/spine-purchase](https://esotericsoftware.com/spine-purchase) | Essential $69 (one-time); Professional $349 (one-time); Enterprise for $500K+ revenue businesses |

**Rive vs. Spine for Domrey STEM:**

| Factor | Rive | Spine |
|--------|------|-------|
| Flutter support | First-class, officially maintained | Official but FFI-based (slightly more complex) |
| State machines | Built-in editor + runtime API | Must be managed in application code |
| File size | 20–80 KB per character | 50–200 KB per character |
| Lip-sync support | Native (additive blend states) | Manual via animation mixing |
| Free tier | Full editor + runtimes free | Trial only; license required for production |
| Learning curve | Lower (visual state machine editor) | Higher (more powerful, more complex) |
| **Recommendation** | **Use for Domrey STEM** | Consider if team has existing Spine skills |

### 10.3 Lottie — UI Micro-Animations

Lottie handles all non-character animation: confetti, star bursts, sticker unlocks, transitions, loading states.

| Resource | Link | Description |
|----------|------|-------------|
| **Official Docs** | [airbnb.io/lottie](https://airbnb.io/lottie/) | Original Airbnb documentation |
| **LottieFiles Dev Portal** | [developers.lottiefiles.com](https://developers.lottiefiles.com/) | Comprehensive tooling and runtime docs |
| **Flutter Package** | [pub.dev/packages/lottie](https://pub.dev/packages/lottie) | Pure Dart Lottie player — `Lottie.asset()`, `Lottie.network()` |
| **React Native Package** | [github.com/lottie-react-native/lottie-react-native](https://github.com/lottie-react-native/lottie-react-native) | Legacy RN package |
| **dotLottie RN (new)** | [github.com/LottieFiles/dotlottie-react-native](https://github.com/LottieFiles/dotlottie-react-native) | New official LottieFiles player for RN |
| **Free Animation Library** | [lottiefiles.com](https://lottiefiles.com/) | 800,000+ free animations — search "confetti", "star", "success" |
| **Figma Plugin** | [figma.com/community/plugin: LottieFiles](https://www.figma.com/community/plugin/809860933081065308/lottiefiles) | Export Figma animations as Lottie |
| **Awesome Lottie** | [github.com/LottieFiles/awesome-lottie](https://github.com/LottieFiles/awesome-lottie) | Curated list of tools, plugins, and examples |

**Domrey STEM Lottie use cases:**
- Confetti / celebration particles (Screen 6)
- Star-burst when answering correctly
- Sticker unlock animation
- Progress bar completion sparkle
- Loading/transition animations
- Button tap feedback

### 10.4 Lip-Sync Implementation

| Resource | Link | Description |
|----------|------|-------------|
| **Rhubarb Lip Sync** | [github.com/DanielSWolf/rhubarb-lip-sync](https://github.com/DanielSWolf/rhubarb-lip-sync) | MIT-licensed CLI tool — analyzes audio → outputs timed mouth shapes (visemes) |
| **Rhubarb WASM Port** | [github.com/danieloquelis/rhubarb-lip-sync-wasm](https://github.com/danieloquelis/rhubarb-lip-sync-wasm) | WebAssembly port with TypeScript support |
| **Rhubarb + Spine Integration** | [rhubarb-lip-sync/.../EsotericSoftwareSpine](https://github.com/DanielSWolf/rhubarb-lip-sync/blob/master/extras/EsotericSoftwareSpine/README.adoc) | Built-in Spine integration guide |
| **Duolingo Visemes Blog** | [blog.duolingo.com/world-character-visemes](https://blog.duolingo.com/world-character-visemes/) | How Duolingo designed 20+ mouth shapes per character |
| **Rive Character Animation Breakdown** | [dev.to: Rive character animation — production-ready](https://dev.to/uianimation/rive-character-animation-for-mobile-apps-a-production-ready-design-and-state-machine-breakdown-5e3m) | State machine design for mobile character animation |
| **AI Avatar with Rive + Voice** | [dev.to: AI avatar assistant using Rive](https://dev.to/uianimation/how-to-build-a-production-ready-ai-avatar-assistant-using-rive-voice-ai-and-api-integration-2026-580i) | Full guide to voice-driven Rive characters |

**Recommended lip-sync architecture for Domrey STEM:**

```
Build Time (offline):
  Audio file (.m4a) → Rhubarb Lip Sync → viseme timeline (.json)

Runtime (on device):
  1. Play audio clip (just_audio)
  2. Read viseme timeline JSON
  3. On each timestamp: set Rive state machine input → mouth shape
  4. Rive additive blend state smoothly transitions between shapes

Fallback (for non-critical voice lines):
  1. Play audio clip
  2. Sample amplitude at 15–20 Hz
  3. Map amplitude (0–1) to Rive jaw-open numeric input
```

### 10.5 Confetti & Particle Effects

| Platform | Package | Link | Notes |
|----------|---------|------|-------|
| **Flutter** | `confetti` | [pub.dev/packages/confetti](https://pub.dev/packages/confetti) | Most popular; control velocity, angle, gravity |
| **Flutter** | `flutter_confetti` | [pub.dev/packages/flutter_confetti](https://pub.dev/packages/flutter_confetti) | Custom particle builders (emoji shapes) |
| **Flutter** | `easy_conffeti` | [pub.dev/packages/easy_conffeti](https://pub.dev/packages/easy_conffeti) | Multiple styles: fountain, explosion, fireworks |
| **React Native** | `react-native-confetti-cannon` | [npmjs.com/package/react-native-confetti-cannon](https://www.npmjs.com/package/react-native-confetti-cannon) | Stable, 117K+ downloads |
| **React Native** | `react-native-fast-confetti` | [github.com/AlirezaHadjar/react-native-fast-confetti](https://github.com/AlirezaHadjar/react-native-fast-confetti) | Fastest (Skia Atlas API) |
| **Alternative** | Lottie confetti | [lottiefiles.com search: confetti](https://lottiefiles.com/) | Pre-made animations — lower CPU than particle systems |

---

## 11. Team Composition & Development Workflow

### 11.1 Development Approach: AI-Augmented with Claude Code

Domrey STEM uses **Claude Code** (Anthropic's AI coding agent) as the primary engineering tool. This significantly reduces the team size needed while maintaining high code quality.

**What Claude Code handles:**
- Flutter app code generation and iteration
- Django admin backend scaffolding and API development
- Test writing (unit, widget, integration tests)
- CI/CD pipeline configuration (GitHub Actions)
- Firebase configuration and security rules
- Code review and refactoring
- Documentation generation

**Official Claude Code resources:**

| Resource | Link | Description |
|----------|------|-------------|
| **Claude Code Docs** | [docs.anthropic.com/en/docs/claude-code](https://docs.anthropic.com/en/docs/claude-code/overview) | Official documentation |
| **Claude Code GitHub** | [github.com/anthropics/claude-code](https://github.com/anthropics/claude-code) | Source, issues, releases |
| **Claude API Docs** | [docs.anthropic.com/en/docs](https://docs.anthropic.com/en/docs) | Anthropic API reference |
| **Claude Agent SDK** | [docs.anthropic.com/en/docs/agents](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview) | Building custom agents for automation |

### 11.2 Team Roles & Skills

The lean team structure below reflects AI-augmented development. Claude Code replaces much of what would traditionally require 2–3 additional developers.

#### Core Team (4–5 people)

| # | Role | Responsibility | Key Skills | Getting Started |
|---|------|---------------|------------|-----------------|
| 1 | **Product Lead / Project Manager** | Product vision, stakeholder management, telco partner liaison, user research, sprint planning | Product management, stakeholder communication, Cambodian market knowledge | [Atlassian Agile Guide](https://www.atlassian.com/agile), [Product School](https://productschool.com/free-resources) |
| 2 | **Full-Stack Developer (AI-Augmented)** | App development using Claude Code, backend API, Firebase setup, CI/CD, deployment | Flutter, Python/Django, Firebase, Git, Claude Code proficiency | [Flutter Docs](https://docs.flutter.dev/), [Django Tutorial](https://docs.djangoproject.com/en/5.0/intro/tutorial01/), [Firebase Flutter Codelab](https://firebase.google.com/codelabs/firebase-get-to-know-flutter) |
| 3 | **Character Animator / Rive Designer** | Character design, Rive animation, state machines, seasonal skins, lip-sync mouth shapes | Rive Editor, 2D character design, animation principles | [Rive Learning](https://rive.app/learn), [Rive YouTube](https://www.youtube.com/@Rive_app), [Rive Community](https://rive.app/community) |
| 4 | **UI/UX Designer** | App UI design, user testing with kids, Lottie animations, accessibility | Figma, mobile design, child UX research, Lottie/After Effects | [Material Design](https://m3.material.io/), [Figma Learn](https://help.figma.com/), [Designing for Kids (Nielsen Norman)](https://www.nngroup.com/articles/children-websites/) |
| 5 | **Khmer Content & Voice Specialist** | Khmer script content, voice recording, cultural review, curriculum alignment | Native Khmer, education curriculum knowledge, audio recording/editing | [Audacity Manual](https://manual.audacityteam.org/), [FFmpeg Docs](https://ffmpeg.org/documentation.html) |

#### Part-Time / Contract Support

| Role | When Needed | Key Skills | References |
|------|-------------|------------|------------|
| **QA Tester** | Pre-launch, each release | Mobile testing, device lab management | [Flutter Testing Docs](https://docs.flutter.dev/testing), [Firebase Test Lab](https://firebase.google.com/docs/test-lab) |
| **DevOps / Infrastructure** | Initial setup, quarterly review | Cloud infrastructure, CI/CD pipelines | [GitHub Actions Docs](https://docs.github.com/en/actions), [Railway Docs](https://docs.railway.com/) |
| **Sound Designer** | Phase 1 production | Audio production, kid-friendly SFX | [Freesound.org](https://freesound.org/), [Zapsplat](https://www.zapsplat.com/) |
| **Pedagogical Advisor** | Curriculum design phase | Early childhood education, math pedagogy | [Cambodia MoEYS Curriculum](http://www.moeys.gov.kh/) |

### 11.3 Development Workflow with Claude Code

```
┌─────────────────────────────────────────────────────────────┐
│  Developer Workflow (AI-Augmented)                           │
│                                                             │
│  1. Product Lead defines feature / user story               │
│  2. Developer opens Claude Code in project directory        │
│  3. Claude Code generates implementation:                   │
│     ├── Flutter widgets and screens                         │
│     ├── State management (Riverpod providers)               │
│     ├── Django models, views, and API endpoints             │
│     ├── Unit and widget tests                               │
│     └── Firebase security rules                             │
│  4. Developer reviews, adjusts, and commits                 │
│  5. CI/CD pipeline runs tests automatically                 │
│  6. Designer reviews UI; Animator reviews character states   │
│  7. Claude Code assists with bug fixes and iteration        │
│  8. Deploy to staging → test on devices → release           │
└─────────────────────────────────────────────────────────────┘
```

### 11.4 Key Technical References by Domain

#### Flutter App Development

| Resource | Link |
|----------|------|
| Flutter Official Docs | [docs.flutter.dev](https://docs.flutter.dev/) |
| Flutter Cookbook | [docs.flutter.dev/cookbook](https://docs.flutter.dev/cookbook) |
| Riverpod (State Management) | [riverpod.dev](https://riverpod.dev/) |
| GoRouter (Navigation) | [pub.dev/packages/go_router](https://pub.dev/packages/go_router) |
| Drift (SQLite) | [drift.simonbinder.eu](https://drift.simonbinder.eu/) |
| Hive (Key-Value Store) | [pub.dev/packages/hive](https://pub.dev/packages/hive) |
| just_audio | [pub.dev/packages/just_audio](https://pub.dev/packages/just_audio) |
| Flutter Performance Best Practices | [docs.flutter.dev/perf](https://docs.flutter.dev/perf) |

#### Backend & Infrastructure

| Resource | Link |
|----------|------|
| Django Official Tutorial | [djangoproject.com/intro/tutorial01](https://docs.djangoproject.com/en/5.0/intro/tutorial01/) |
| Django REST Framework | [django-rest-framework.org](https://www.django-rest-framework.org/) |
| Firebase for Flutter | [firebase.google.com/docs/flutter](https://firebase.google.com/docs/flutter/setup) |
| Firebase Auth (Phone) | [firebase.google.com/docs/auth/flutter/phone-auth](https://firebase.google.com/docs/auth/flutter/phone-auth) |
| Firestore Flutter | [firebase.google.com/docs/firestore/quickstart](https://firebase.google.com/docs/firestore/quickstart) |
| GitHub Actions for Flutter | [docs.github.com/actions](https://docs.github.com/en/actions) |
| Fastlane (Mobile CI/CD) | [fastlane.tools](https://fastlane.tools/) |

#### Design & Animation

| Resource | Link |
|----------|------|
| Rive Learning Hub | [rive.app/learn](https://rive.app/learn) |
| Rive YouTube Channel | [youtube.com/@Rive_app](https://www.youtube.com/@Rive_app) |
| Rive Community Files | [rive.app/community](https://rive.app/community) |
| LottieFiles Free Library | [lottiefiles.com](https://lottiefiles.com/) |
| Figma for Education | [figma.com/education](https://www.figma.com/education/) |
| Designing for Children (NN/g) | [nngroup.com: children-websites](https://www.nngroup.com/articles/children-websites/) |
| Google Material Design 3 | [m3.material.io](https://m3.material.io/) |

### 11.5 Estimated Timeline (Phase 1 — Math)

| Phase | Duration | Key Deliverables |
|-------|----------|-----------------|
| **Discovery & Design** | 4 weeks | User research, UI/UX design in Figma, character concepts, curriculum mapping |
| **Character & Asset Production** | 4 weeks | Rive character rigs + state machines, Lottie UI animations, voice recording, SFX |
| **App Development (Sprint 1–3)** | 6 weeks | Core gameplay (PreK + Grade 1-2), home screen, OTP sign-in, offline storage |
| **App Development (Sprint 4–5)** | 4 weeks | Grade 3-5 content, parent dashboard, sticker collection, telco billing integration |
| **Testing & Polish** | 3 weeks | Device testing, performance optimization, accessibility, localization QA |
| **Soft Launch** | 2 weeks | Beta with telco partner employees, feedback collection, bug fixes |
| **Launch** | — | App Store + Google Play + APK distribution |

**Total: ~23 weeks (~6 months) with a lean AI-augmented team**

---

*This guide complements the [Domrey STEM concept note](./concept-note.md). As technical decisions are validated through prototyping, this document will be updated.*
