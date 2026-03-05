# Concept Note: Domrey STEM (ដំរី) — A Cambodian Kids' Learning App

## 1. Introduction

Domrey STEM is a mobile learning app designed for Cambodian children from preschool through Grade 5 (ages 3–11). The app makes foundational mathematics fun and accessible through interactive, game-based activities — and is built to expand into Science, Technology, and Engineering in future phases.

The concept is inspired by [Math Kids — Add, Subtract, Count](https://apps.apple.com/kh/app/math-kids-add-subtract-count/id1272098657), a highly rated (4.6/5) free app targeting ages 4+ with character-driven math games, sticker rewards, and parental progress tracking.

## 2. Problem Statement

Cambodian children, particularly in rural and underserved areas, have limited access to engaging, age-appropriate educational tools in their own language. Existing apps like Math Kids are English-only and culturally generic. There is a clear gap for a localized, Khmer-language learning app that reflects Cambodian culture, curriculum standards, and the daily experiences of Cambodian kids.

## 3. Vision

To become Cambodia's go-to educational app that empowers every child to build strong STEM foundations through joyful, self-paced play — in Khmer.

## 4. Target Audience

| Segment | Age | Description |
|---------|-----|-------------|
| Preschool | 3–5 | Number recognition, counting, basic shapes |
| Grades 1–2 | 6–7 | Addition, subtraction, simple patterns |
| Grades 3–5 | 8–11 | Multiplication, division, fractions, word problems, early problem-solving |

Secondary audience: Parents and teachers who monitor progress and guide learning.

## 5. Core Features (Phase 1 — Mathematics)

Drawing from Math Kids' proven activity types and extending them for a wider age range:

### 5.1 Counting & Number Recognition
- Count objects on screen (tap or drag)
- Match numbers to quantities
- Number tracing for writing practice

### 5.2 Comparison
- Compare quantities (more/less/equal)
- Compare numbers (greater than / less than)
- Size and length comparisons

### 5.3 Addition
- **Puzzle mode**: Drag objects to complete addition equations
- **Fun mode**: Find the missing number in an equation
- **Quiz mode**: Timed/untimed assessment
- Carry-forward (multi-digit) addition for older kids

### 5.4 Subtraction
- **Puzzle mode**: Visual subtraction with objects
- **Fun mode**: Fill in the blank
- **Quiz mode**: Progress evaluation
- Step-by-step borrowing for older kids

### 5.5 Multiplication & Division (Grades 3–5)
- Visual grouping and array models
- Times-table practice games
- Division as sharing/grouping

### 5.6 Problem Solving & Logic
- Word problems with Khmer context (e.g., counting mangoes at the market)
- Number balance / scale activities
- Simple Sudoku and pattern recognition
- Sequencing and ordering

## 6. Differentiation from Math Kids

| Aspect | Math Kids | Domrey STEM |
|--------|-----------|-----------|
| Language | English only | Khmer-first, with English option |
| Age range | 4–7 (PreK–Grade 1) | 3–11 (PreK–Grade 5) |
| Cultural context | Generic Western | Cambodian settings, characters, and scenarios |
| Curriculum alignment | US-centric | Aligned with Cambodian Ministry of Education standards |
| Scope | Math only | Math first, expanding to full STEM |
| Difficulty progression | Limited levels | Adaptive difficulty across 6+ grade levels |
| Offline support | Unknown | Designed for offline-first use (critical for rural Cambodia) |

## 7. User Experience Principles

- **Kid-friendly UI**: Large buttons, bright colors, playful sound effects (inspired by Math Kids' squeaky-toy interactions)
- **Character-driven**: Original Cambodian characters that guide, encourage, and celebrate with the child
- **Reward system**: Stickers, badges, and unlockable content for completing activities
- **Minimal text**: Icon-based navigation so pre-readers can use the app independently
- **Safe environment**: No ads, no external links, no data collection from children
- **Simple sign-in**: Phone-number-based authentication scoped to the partner telco operator during the exclusivity period; no passwords or email required
- **Parental dashboard**: Progress reports, time-spent tracking, and difficulty settings

## 8. Localization & Cultural Design

- Full Khmer Unicode text and voice-over
- Illustrations reflecting Cambodian life: local fruits, animals, school settings, festivals
- Audio narration in Khmer for instructions and encouragement
- Optional English mode for bilingual learners

## 9. Technical Considerations

- **Platforms**: iOS and Android (cross-platform framework, e.g., Flutter or React Native)
- **Offline-first**: All core content available without internet; sync progress when connected
- **Lightweight**: Small app size for low-storage devices common in Cambodia
- **Accessibility**: Support for screen readers; high-contrast mode option
- **Telco billing integration**: Backend API integrates with the partner telco's billing system to handle subscription activation, renewal, and cancellation
- **Phone-number auth**: Sign-in via OTP sent to the subscriber's phone number; during the exclusivity period, restricted to the partner operator's number ranges

## 10. Future Phases (STEM Expansion)

| Phase | Subject | Example Activities |
|-------|---------|-------------------|
| Phase 2 | Science | Animal classification, plant life cycles, simple experiments |
| Phase 3 | Technology | Basic coding logic (sequencing, loops), digital literacy |
| Phase 4 | Engineering | Building challenges, bridge/structure puzzles, simple machines |

Each phase follows the same play-based, Khmer-first design principles.

## 11. Impact Goals

- Improve numeracy outcomes for Cambodian children in target age groups
- Provide equitable access to quality learning tools regardless of geography or income
- Support parents and teachers with visibility into children's learning progress
- Build a foundation for broader STEM literacy in Cambodia

## 12. Success Metrics

- Number of active users (daily/monthly)
- Average session duration and frequency
- Skill progression rates (pre/post assessments within the app)
- Geographic distribution of users (urban vs. rural reach)
- Parent/teacher satisfaction ratings
- Subscription conversion rate and churn rate (telco bundle subscribers)
- Revenue per subscriber and lifetime value

## 13. Sustainability & Distribution

### Commercial Model: Telco Partnership

The primary monetization strategy is a **partnership with a local Cambodian mobile operator**. Domrey STEM is bundled into the telco's subscription plans, providing a sustainable revenue stream while keeping the app accessible to families.

- **Subscription bundle**: The app is offered as part of the telco's family/education subscription tier. Parents subscribe via their phone account — no credit card needed, billing is handled by the operator.
- **Exclusivity period**: During the initial partnership, sign-in is scoped to the partner operator's phone numbers. This gives the telco a value-add for their subscribers and provides Domrey STEM with a guaranteed distribution channel.
- **Post-exclusivity**: After the locked period, the app may open to other operators or offer direct subscriptions.
- **Billing integration**: The app's backend integrates with the telco's billing API to handle subscription activation, renewal, cancellation, and grace periods. No in-app purchase through Apple/Google is needed during the telco-exclusive phase.

### Additional Sustainability

- Potential funding: NGO partnerships, Ministry of Education collaboration, grant funding, CSR sponsors
- A free tier with limited content may be offered to non-subscribers to drive conversion

### Distribution

- App Store and Google Play (primary)
- Direct APK distribution for schools without Play Store access
- Pre-installation on telco-branded devices (negotiated with partner)

---

*This concept note is a living document and will be refined as user research, curriculum review, and stakeholder feedback are incorporated.*
