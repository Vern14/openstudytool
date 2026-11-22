# OpenStudyTool — Full Code-Level Software Specification & Audit

> **Method note:** This document was produced by reading 100% of the Dart source in
> `D:\kuhes\openstudytool` (entry point, all ~40 services, all screens/widgets/theme, all 16
> study-suite part files, build config, manifest, and pubspec) plus the test directory.
> No documentation files, README, pitch deck, or knowledge-graph notes were consulted;
> every statement below derives from code. Line counts and identifiers are cited so an
> engineer (human or AI) can jump straight to the evidence.

---

## 1. Product Summary

**OpenStudyTool** (`pubspec.yaml`: `openstudytool 1.0.0+1`, description *"standalone study
suite app (extracted from KUHeS Hub)"*) is a **Flutter study-suite application** whose single
home screen (`lib/screens/study_suite_screen.dart`, split into 16 `part` files) lets a student:

1. **Ingest source material** — PDFs, DOCX/PPTX/XLSX/CSV, plain text, HTML, images
   (vision, no on-device OCR), audio/video (transcribed via Groq Whisper), and web pages
   (incl. Google Docs/Slides/Sheets) — or just paste text / name a topic.
2. **Generate study artifacts with AI** — exhaustive notes, Q&A notes, cheatsheets,
   executive summaries, flashcards, podcast scripts (TTS-optimized), tutorials, approach
   feedback (coach-style photo feedback that never solves the problem), and 11 quiz/exam
   formats (multiple choice, true/false, fill-in, multi-select, ordering, matching, mixed,
   exam MCQ, free response, case study, mixed exam).
3. **Study the output interactively** — streaming render with LaTeX typesetting, D3.js
   figures and Three.js simulations, typeable quiz runners with timers and auto-grading,
   flashcard flip drills, word-by-word TTS "karaoke" listening, an inline chat
   assistant grounded in the current material, and a one-tap **"Quiz me on this
   material"** button on every learn output (notes, cheatsheet, summary, podcast,
   tutorial) that turns the material into a live quiz. When the original files/text/URL
   are available the quiz is generated from them, with the notes used only to scope
   coverage. That is grounded generation, **not verification**: no supporting excerpt is
   extracted per item and nothing is checked back against the source. Notes-only runs are
   labeled **unverified** and are not auto-enrolled into review.
4. **Persist everything into a local course library** — courses → chapters → topics, with
   materials (files/links/text), saved generations, page-range book slices, coverage stats.
5. **Run a spaced-repetition & mastery loop** — Leitner-box review queue (flashcards require
   a typed recall attempt before reveal and use four grades; ~1 in 5 cards, chosen randomly
   before reveal, is a graded recall check that corrects self-graded evidence), weakness-driven interleaved
   mixed practice, "repeat drills" of flagged difficult items, per-topic mastery and
   retention dashboards **hidden behind a "Show Mastery Scores" flag** (evidence-first
   display by default; the engines keep computing underneath), diagnostic seeding via
   rewarded ad, and AI accuracy checking of saved quiz keys (flagging suspicious items into
   a `reported_items` exclusion list).
6. **Plan and focus** — a weekly timetable of study blocks, a focus timer with lock-in
   distraction blocking, check-ins that feed adherence stats and streaks.
7. **Export** — publication-quality PDFs (real fonts, typeset math, real tables, captured
   D3 figures) for every output type, gated by a rewarded ad, downloadable or shared.
8. **Back up the library** — a manual, user-initiated `.ostbak` archive (SQLite snapshot +
   sidecar books/materials + manifest) exported via the OS share sheet, restored from any
   picked file (e.g. Downloads), with a "last backed up / N new items" status row in the
   More menu and a dismissible home-screen prompt when the library changed since the last
   backup. **Individual courses can also be exported** ("Export a course" → share sheet)
   and imported back via the same restore picker as a non-destructive, id-remapped merge.
   No scheduled or cloud backups — zero recurring cost, the data stays user-owned.

**Platforms:** primarily Android (manifest, AdMob, launcher icons), with a functional
Flutter **web** build (a parallel iframe-based D3/figure pipeline, PDF downloader shim,
passkey bundle in `web/`), and desktop where webview_flutter/secure-storage/ads degrade
gracefully. There is **no backend of its own**: all AI runs through user-supplied API keys
direct from the device; Supabase is used only for storage-backed document links.

---

## 2. Runtime & Dependency Stack

Flutter SDK `^3.11.5`, Material 3. Dependencies (from `pubspec.yaml`) and their roles:

| Package | Role in this app |
|---|---|
| `google_generative_ai` | Gemini SDK (chat session only; generation uses REST) |
| `webview_flutter` | D3 chart figures, Three.js sims, offscreen PDF rasterization |
| `flutter_math_fork` | LaTeX typesetting (screen) + offscreen rasterization (PDF) |
| `pdf` | PDF document engine (`DocumentPdf`) |
| `syncfusion_flutter_pdf` | PDF text extraction for ingestion |
| `flutter_svg` | Direct SVG artwork rendering |
| `file_picker`, `image_picker`, `image` | Source material picking; image compression |
| `excel`, `archive` | XLSX parsing; DOCX (zip) unpacking |
| `sqflite`, `path_provider`, `sqflite_common_ffi` | Local SQLite library (v15) + `VACUUM INTO` snapshot backups |
| `flutter_secure_storage` | API-key storage (Keystore/Keychain) |
| `shared_preferences` | Dark mode, mastery-scores flag, backup fingerprint/prompt prefs |
| `flutter_tts` | Listen player (karaoke TTS) |
| `google_mobile_ads` | Banner/interstitial/rewarded ads (Google **test** IDs) |
| `supabase_flutter` | Storage-backed document links (anon key only) |
| `sentry_flutter` | Crash/error telemetry (via `ObservabilityService`) |
| `connectivity_plus` | Offline banner |
| `http` | REST AI transports (Gemini REST, OpenAI, Anthropic, Groq, OpenRouter, web ingest) |
| `share_plus`, `url_launcher`, `app_settings` | Sharing PDFs, opening links, app settings |
| `google_fonts` | Inter/Montserrat theming (disabled in widget tests) |
| `web` | Flutter-web iframes/shims |

Bundled assets: `assets/fonts/` (Lato Regular/Bold/Italic/BoldItalic for PDFs),
`assets/d3/` (D3 v7 source inlined into chart pages — fully offline), `assets/three/`
(Three.js UMD), `assets/images/` (app icon).

---

## 3. Codebase Layout & Architecture

```
lib/
  main.dart                     184 L   bootstrap: Sentry → ApiKeys → Ads → Supabase → app
  screens/
    study_suite_screen.dart     744 L   home screen state owner; pulls in 16 part files:
    study_suite/                        shell 2025 L, generation 915 L, input_tabs 648 L,
                                        file_ingest 772 L, ingest_views 2154 L, library_save 416 L,
                                        downloads 564 L, pdf_export 839 L, pdf_feedback 380 L,
                                        pdf_blocks 41 L, quiz_exam_figures 288 L, text_quiz 205 L,
                                        output_tts 369 L, focus_session_ads 277 L,
                                        document_tile 327 L, constants 44 L
    courses_screen.dart         287 L   library home (courses + global due/difficult counts)
    course_detail_screen.dart  1295 L   chapters→topics, mastery/evidence header, diagnostic seeding
    topic_detail_screen.dart   1674 L   topic hub: materials, study launcher, saved generations
    topic_override_fixes.dart   257 L   flagged-card sheet: accept the student's wording
    saved_generation_screen.dart 2369 L read-only saved viewer + retake system
    review_queue_screen.dart    1289 L  Leitner runner (typed flashcards, 4 grades, mixed/drill modes)
    timetable_screen.dart        622 L  weekly blocks + check-in launch
    progress_screen.dart         354 L  weekly adherence, hours, streaks, history ranges
    library_widgets.dart         198 L  MasteryBar, askTitle, shared rows
  widgets/
    study_markdown.dart        1665 L  THE shared markdown/LaTeX/table/figure renderer
    economics_chart.dart       1033 L  ChartData parser + native CustomPainter fallback
    d3_chart.dart               723 L  D3 WebView widget + shared chart HTML template
    d3_chart_web.dart/_stub            web iframe host (conditional import)
    three_sim.dart              357 L  Three.js simulation widget + page builder
    math renderer lives in services/math_image_renderer.dart
    focus_session_sheet.dart    465 L  focus setup/live views
    listen_player_card.dart     282 L  karaoke TTS player UI
    save_picker_sheet.dart      391 L  course→chapter→topic destination picker
    ai_disclaimer_badge.dart    174 L  "AI-generated • check your textbook" banner + report
  services/                    ~40 files (all singletons/static engines)
    backup_service.dart        1410 L  .ostbak create/inspect/restore + course export/import + fingerprint prompting
  theme/
    kuhes_theme.dart            264 L  brand colors + M3 themes + BuildContext color ext
    kuhes_theme_scope.dart       61 L  InheritedWidget (dark toggle + mastery-scores flag live; a11y fields stubs)
test/                           34 files covering the pure-logic layer
```

**Architecture in one paragraph:** a **single-tenant, offline-first, client-only app**.
One stateful home screen owns all study-suite state; persistence is one SQLite database
(`CourseStore.instance`, schema v15) plus managed sidecar files; backups are manual
`.ostbak` ZIP archives built by `BackupService` (no backend, no cloud). AI calls go
device→provider (no proxy), orchestrated
by `GeminiService` despite its name — it is the **multi-provider orchestrator** with a
pure, unit-tested failover policy. Rendering is funneled through one shared markdown
engine (`StudyMarkdown`) used identically by the live screen and the saved viewer, and one
shared PDF engine (`DocumentPdf`) used identically by the study screen and saved viewer.
Ad-gating wraps premium actions (PDF export, diagnostic seeding); a scroll-triggered ad
appears once per generated output, but only after the reader has scrolled deep into the
document (60% of scrollable extent with a 1200 px floor — `focus_session_ads.dart`).

---

## 4. Data Model - SQLite Schema v15 (`services/course_store.dart` + `services/course_store_migrations.dart`)

Database `openstudytool_library.db` (`currentSchemaVersion = 15`),
`PRAGMA foreign_keys = ON`. The migration chain v1→v15 is moved out of `course_store.dart`
into `services/course_store_migrations.dart` as a pure, order-dependent, column/table-existence-guarded
slice (`PRAGMA table_info` checks) so half-migrated databases never crash. Legacy binary blobs
were migrated out of the DB into managed files (`migrateBinariesToFiles`, v7) with
`path` + `size` columns added; v12 adds the `attempts.graded_by` provenance column;
v13 adds `attempts.chance` (guess probability for chance-corrected mastery; NULL for
legacy/free-recall rows, treated as 0); **v14 drops the exam-schedule tables**
(`exam_topics` then `exams`, FK order, errors swallowed so a partially migrated database
still opens); **v15 adds `attempts.answer_text` and the `accepted_answers` table** for
the one-tap override fix (additive, guarded).

### Tables

| Table | Columns (semantics) |
|---|---|
| `courses` | `id` PK, `title`, `created_at` |
| `chapters` | `id` PK, `course_id` FK→courses CASCADE, `title`, `position` |
| `topics` | `id` PK, `chapter_id` FK→chapters CASCADE, `title`, `position`, `book_id` FK→course_books SET NULL, `page_start`, `page_end` (topic can be a slice of a course book) |
| `course_books` | `id` PK, `course_id` FK CASCADE, `name`, `blob` (legacy), `path` (managed file), `page_count`, `created_at` |
| `materials` | `id` PK, `topic_id` FK CASCADE, `kind` (`file`/`link`/`text`/`book-slice` semantics), `name`, `path`, `url`, `body`, `blob` (legacy), `size`, `page_start`, `page_end`, `created_at` |
| `generated` | `id` PK, `topic_id` FK CASCADE, `kind` (library kind; quiz sub-types are distinct kinds so a true/false quiz doesn't overwrite an MCQ quiz), `title`, `content_json`, `position` (drag order), `source` (provenance, incl. `diagnostic`), `created_at` |
| `engagement` | autoinc id, `topic_id` FK CASCADE, `kind` (passive study events: views, listens…), `at` |
| `attempts` | autoinc id, `topic_id`, `generated_id`, `item_key` (e.g. `topic:<id>:q:<i>:<hash>`), `correct` (0/1), `chance` (guess probability for that item, v13: `1/options` for multiple-choice/true-false, NULL for free recall/flashcards and legacy rows → treated as 0), `confidence`, `duration_ms`, `at`, `graded_by` (`'self'` = self-rated flashcard outcome, `'typed'` = graded typed recall check, `'typed_override'` = student's "I was right" override after a failed check (counted at half weight), NULL = key-graded quiz attempt or legacy row; v12), `answer_text` (the typed wording for typed checks/overrides, v15 — feeds the one-tap override fix) |
| `review_state` | `item_key` PK, `topic_id` FK CASCADE, `generated_id` FK CASCADE, `kind` (`flashcard`/`question`), `payload_json` (self-contained card/question), `box` 1..5 (Leitner), `due_at` |
| `accepted_answers` | `item_key` + `answer` composite PK, `topic_id` FK CASCADE, `created_at` — student-approved alternative wordings (v15); capped at 3 per card, requiring critical tokens (numbers, identifiers, polarity words) and at least two content tokens; evidence sheet lists them with a remove button for undo; re-validated against the original key at grading time |
| `timetable_blocks` | `id` PK, `title`, `topic_id` FK SET NULL, `weekday`, `days` (CSV multi-day, v10), `start_min`, `end_min`, `created_at` |
| `study_checkins` | `id` PK, `block_id` FK SET NULL, `topic_id` FK SET NULL, `goal`, `started_at`, `ended_at`, `planned_min`, `actual_min`, `created_at` |
| `difficult_items` | `item_key` PK, `topic_id` FK CASCADE, `generated_id` FK CASCADE, `kind`, `payload_json`, `flagged_at` — explicit "flag for repeat drills" |
| `reported_items` | `item_key` PK, `topic_id` FK CASCADE, `reason`, `reported_at` — student-reported inaccurate items; **permanently excluded from the review queue** |

The v8 `exams` table and the v13 `exam_topics` join table no longer exist: the v14
migration drops both. Exam *content* types (`exam_free`, `exam_case`, …) are untouched —
only the exam **schedule** was removed. v15 adds `attempts.answer_text` and
`accepted_answers` for the override fix.

Indexes: `attempts(topic_id, at)`, `review_state(due_at)`, `topics(chapter_id, position)`,
`course_books(course_id)`, `engagement(topic_id, at)`,
`timetable_blocks(weekday, start_min)`, `study_checkins(started_at)`,
`difficult_items(topic_id)`, `reported_items(topic_id)`, `accepted_answers(topic_id)`.

### Domain logic on top of the schema

- **Leitner spaced repetition:** boxes 1–5 spaced **1, 3, 7, 14, 30 days**. Scheduling is
  a pure function (`nextBox(box, grade, failedCheck)` in `mastery.dart`, table-tested):
  **override → hold, failed check → box 1, otherwise Again → 1, Hard → hold the current
  box, Good → +1, Easy → +2**; key-graded quiz failures demote to box 1
  (`nextBoxForCorrectness` is the Good/Again step). A failed typed recall check resets to
  box 1 regardless of the self-rating, while an "I was right" override holds — the override
  outranks the failed check that preceded it, so a contested check neither promotes nor
  demotes the card. There is no exam-based
  interval compression: **intervals are deadline-unaware** (up to 30 days), so nothing
  reschedules material around a test date. Enrolling happens automatically
  when quiz runs or flashcard saves occur on a topic; `dueCount/dueItems` power the review
  queue; mixed-practice mode interleaves *not-yet-due* items from chosen topics, capped
  and shuffled.
- **Unknown-topic selection:** mixed practice ranks only topics with **at least five**
  recent attempts; topics below that (including zero) are treated as *unknown* — two lucky
  true/false answers must not read as 100% mastery — and are **weighted toward the fewest
  attempts and longest since last practiced** ($w = (5 - \text{attempts}) \times \text{recencyMultiplier}$,
  with recency up to 2.0x for unpracticed/older topics), so brand-new topics surface ahead of ones
  with four attempts, while retaining a **reserved share of about one slot in three**, so
  continuous practice on the same few measured topics cannot starve the neglected ones. Ranked
  picks fill the rest.
- **Chance-corrected mastery (v13):** multiple-choice/true-false attempts store the item's
  guessing probability in `attempts.chance` (flashcards/free recall store NULL → 0).
  Correction is applied **only to aggregates**, never per attempt (a 0/1 outcome cannot be
  debiased): `adjusted = max(0, (p − c̄)/(1 − c̄))`, where `p` is the weighted observed
  accuracy and `c̄` the mean chance across the same window, using **the same recency and
  per-attempt evidence weights as `p`** (`weightedChanceMean`, EWMA α = 0.3; a
  half-credit self-graded/override attempt contributes half the expected guess credit),
  so the two cannot drift apart. It requires
  `chanceCorrectionMinAttempts` (5) attempts and chance values aligned 1:1 with outcomes;
  below that, raw accuracy is shown. This stops T/F-heavy topics from looking strong
  merely because guessing is easy.
- **Attempts → mastery/retention:** `MasteryService`/`RetentionService` compute per-topic
  mastery from attempt history (correct rate × recency × coverage across library kinds);
  UI copy explicitly labels results **observational** ("these numbers reflect how you've
  studied, not an exam prediction").
- **Flashcard self-grade calibration (v12, extended in v13):** review-queue flashcard
  attempts carry `graded_by`: `'self'` for rated outcomes, `'typed'` when the
  correct/wrong outcome came from grading the student's typed answer against the key, and
  `'typed_override'` when the student taps **"I was right"** after a failed check (their
  paraphrase/reasoning was correct even though string matching disagreed). Typed grading is
  **strict only for short keys** (`isShortExpectedAnswer`: terms, numbers, ≤3 words after
  normalization — normalized equality via `gradeFillBlank`); longer keys go through
  `recallCriticalTokensAgree` **and** the deterministic fuzzy score
  (`recallMatchScore`: token coverage/precision threshold 0.6). The guard extracts
  **critical tokens** from the key — numbers, **identifier tokens** (hepatitis A vs B,
  type I vs II, factor VIII vs IX, B6 vs B12, IgA vs IgG — plain token overlap drops
  single letters), negation polarity, and curated **polarity classes** (a decrease class
  of decrease/reduce/lower/fall/diminish/attenuate/…, an increase class of
  increase/rise/raise/elevate/…, plus agonist/antagonist, proximal/distal,
  hyper-/hypo- compounds, tachycardia/bradycardia, …) — and fails the check if a
  critical token is missing, an identifier is absent, or an opposite-class word appears.
  Same-class synonyms and inflections (`decreased`/`decreasing`/`reduces`/`lowered`)
  pass, and Roman numerals accept their Arabic value ("type I" ↔ "type 1"), so a
  paraphrase is not punished for choosing a different member of the class, while a
  one-word inversion ("increases" for "decreases") can no longer ride token overlap to a
  pass. No AI judge is involved. **Bag-of-tokens limit:** role reversal ("A inhibits B"
  vs "B inhibits A") shares every token and still passes; phrase-level relations are out
  of scope and rely on item reporting.
  `CourseStore.typedCheckStats({topicId})` aggregates over enrolled flashcards (reported
  items excluded; overrides counted separately); `selfEvidenceWeight()` (mastery.dart)
  maps that to a damping weight over `(typedCorrect + 0.5·overridesCorrect) /
  (typedChecks + 0.5·overrides)`, clamped to [0.5, 1.0] and 1.0 until ≥ 3 automatic
  checks exist. An override therefore adds as much to the denominator as it can to the
  numerator: "always tap I was right" can never restore full trust, and automatic
  failures keep the weight damped. `courseTopicStats` and the topic screen build
  per-attempt accuracy *values* (1.0 typed/key-graded correct, 0.0 wrong, `weight` for
  self-graded correct, 0.5 for a typed override) that feed `topicAccuracyValues` →
  `TopicStats.mastery()`, so inflated "Knew it" ratings lose weight proportionally to
  failed recall checks while an override never outranks a first-try correct answer.
  Per-card override pressure is stored/derived as `CourseStore.cardOverrideRates()`:
  automatic typed checks vs "I was right" overrides keyed by `item_key` (override
  rows are not counted as checks, or the rate could never exceed 50%). A card is
  surfaced only at **3+ automatic checks** with overrides on at least half of them
  (`overrideRateFlagged`), so one failed check plus one override — a single anecdote —
  no longer flags a key. The flag is actionable: the topic evidence sheet lists flagged
  cards (question, key, the student's stored wording via `overrideFixCandidates`) and
  offers **Accept a wording**, which saves the student's answer to
  `accepted_answers`. The guard (`acceptedAlternativeAgrees`) is tightened:
  critical tokens (numbers, identifiers, polarity words) must be present or matched
  by a same-class synonym/inflection (only fuzzy coverage is waived); minimum content
  size is enforced (at least two content tokens, rejecting single-token shortcuts like
  `"NSAID"` for `"reduces prostaglandin synthesis"`); alternatives per card are capped
  at 3 (`CourseStore.maxAcceptedAlternativesPerCard`); and the evidence sheet displays
  currently accepted wordings with an inline remove button (`deleteAcceptedAnswer`) for
  immediate undo. `gradeRecallAttempt` takes `acceptedAnswers`: an exact normalized
  match passes only after re-verifying against the original key at grading time so a stored
  contradiction can never slip through. The review queue offers the same one-tap
  **Save my wording as accepted** right after an override.
- **Library backup & restore (`services/backup_service.dart`, 1410 L — manual only):**
  `createBackup()` runs `VACUUM INTO` on the live connection (transactional snapshot
  without blocking writers), verifies it with `PRAGMA integrity_check` opened read-only,
  bundles `library.db` + sidecar `files/books/*` + `files/materials/*` (DB-referenced paths
  *and* everything present in `ost_files/`; unreadable/missing files are recorded in
  `manifest.missingFiles`, never silently dropped) + `manifest.json`
  (`formatVersion: 1`, `schemaVersion`, `appVersion`, `createdAt`, `statsFingerprint`,
  `rowCounts`) into `<temp>/openstudytool_backup_<ts>.ostbak`, then the UI hands it to the
  OS share sheet (user saves it anywhere, e.g. Downloads). `restoreFrom()` inspects the
  manifest, rejects newer `schemaVersion` (`BackupVersionException`, "update the app"),
  other `formatVersion`s, **and `scope: 'course'` archives**, closes the `CourseStore`
  singleton, swaps the `.db` file
  (deleting stale `-wal`/`-shm`), extracts sidecars into `ost_files/` and purges orphaned
  files there that aren't in the archive (regular files only, best-effort), reopens through the
  normal migration chain (older-schema backups upgrade), rewrites
  `course_books.path`/`materials.path` to the new absolute locations (basename re-housing),
  and seeds SharedPreferences from the manifest fingerprint so a freshly restored library
  is not flagged dirty.
- **Single-course export & import (same `.ostbak` container, additive manifest fields):**
  `BackupManifest` gained `scope` (`'library'` default / `'course'`), `courseId`,
  `courseTitle`, and `contentOnly` (boolean) — older archives decode unchanged.
  `exportCourse(courseId, {contentOnly: false})` produces
  `openstudytool_course_<slug>[_content]_<ts>.ostbak`: a `VACUUM INTO` snapshot of the live DB in
  which every row not belonging to the course is deleted (FK cascades remove other
  courses' chapters/topics/materials/generated/review_state/accepted_answers/engagement/
  difficult/reported rows; `attempts` is trimmed explicitly since it has no topic FK; planner
  tables `timetable_blocks`/`study_checkins` are excluded entirely as schedule data).
  When **`contentOnly: true`**, the snapshot additionally purges all personal student history:
  `attempts` (containing typed `answer_text` and scores), `review_state`, `engagement`,
  `accepted_answers`, `difficult_items`, and `reported_items`, leaving only materials,
  chapters, topics, and generated study content for clean sharing with peers.
  The individual course detail screen (via its footer action) and More menu offer a choice between
  **Content only (share)** and **Full (with history)**.
  Exports are integrity-checked, bundled with **only the course's** referenced sidecar files (no
  `ost_files/` directory scan), and per-table `rowCounts` in the manifest. The backup
  fingerprint prefs are deliberately **not** updated (an export is not a library backup).
  `importCourse(zipPath)` opens the snapshot through the normal migration chain (so
  older-schema exports upgrade), inserts rows into the **live** database in dependency
  order (courses → chapters → course_books → topics → materials → generated →
  review_state → attempts → engagement → accepted_answers → difficult/reported) without
  closing or swapping anything, extracts sidecars **without purging** (other courses' files share those
  directories), and re-houses imported paths by basename. Ids that collide with existing
  rows (e.g. re-importing the same export) get fresh `CourseStore.newId()` values and all
  references — including `review_state`/`difficult_items`/`reported_items`/`accepted_answers`
  `item_key`s (`topic:<id>…` / `gen:<id>…` substrings) — are remapped; ids placed by the import
  itself are tracked so kept-original ids are never spuriously remapped. Autoincrement
  ids (`attempts`, `engagement`) are dropped and reassigned by the target. Returns
  `CourseImportResult` (courseId/title, `idsRemapped`, per-table counts).
- **Backup change detection (no write-path hooks):** a monotonic stats fingerprint
  (`LibraryStatsFingerprint`: COUNT + MAX(id) over `attempts`/`engagement`/`study_checkins`,
  COUNT over `courses`/`topics`/`generated`/`review_state`/`difficult_items`, encoded
  `v1:…`) is compared at home-screen load and More-menu open against the fingerprint saved
  at last backup. When dirty, the home screen shows a dismissible banner ("N new items
  since your last backup"); dismissal suppresses for the same fingerprint for 24 h
  (`shouldShowPrompt`/`dismissPrompt`). Web (`kIsWeb`) never prompts and the More-menu
  backup rows are hidden there (no `dart:io`).
- **Save flow (study screen → library):** dedupes by (topic, kind) — only a second save of
  the *exact same kind* is an upgrade/replacement; enrolls the quiz/flashcards into Leitner
  review; runs an async **AI accuracy check** (`quiz_key_check` prompt) that compares each
  key against the source and flags suspicious items into `reported_items` (excluded from
  review); records engagement.
- **Time model:** all timestamps are epoch ms (`CourseStore.nowMs()`); the store supports
  injected clocks for tests (`sqflite_common_ffi` in dev deps).

---

## 5. AI Subsystem (`services/gemini_service.dart`, 1784 L + per-provider services)

### 5.1 Key resolution (`services/api_key_store.dart`)

`ApiKeys` holds five keys loaded at startup from **secure storage** (Android
Keystore/EncryptedSharedPreferences, iOS Keychain): `gemini`, `groq`, `openRouter`,
`openAI`, `anthropic`. If a provider has no user-entered key, it falls back to its
compile-time dart-define from `defines.json` (`kGeminiApiKey`, `kGroqApiKey`,
`kOpenRouterApiKey` in `supabase_constants.dart`) — *dev convenience; user keys always
win*. **OpenAI and Anthropic have no define fallback** (user-entry only). On save, empty
strings delete the stored key; `clear()` wipes all. All AI services read ONLY these values.

> ⚠️ **Security finding (fixed in source, rotation still required):** the **hardcoded**
> `kOpenRouterApiKey` default in `supabase_constants.dart` (`sk-or-v1-66522343…`) has been
> **removed** — the constant is now a plain `String.fromEnvironment('OPENROUTER_API_KEY')`,
> so OpenRouter activates only with a define or a user-entered key. `defines.json`
> (gitignored, dev-only via `--dart-define-from-file`) still carries live-looking
> Gemini/OpenRouter/Groq keys and a Supabase anon key for dev convenience — and it holds
> the *same* OpenRouter key that previously sat in committed source, so that key must be
> treated as leaked and rotated; after rotation, update defines.json.

### 5.2 Provider fleet

| Provider | Transport | Models / notes |
|---|---|---|
| Gemini | REST (`gemini_rest.dart`, low "thinking" budget, temp 0.5, maxOutputTokens 65 536, 60 s timeout) | chain: `gemini-3.8-flash` → `gemini-3.5-flash` (deprecated but served) → `gemini-3.5-flash-lite` → `gemini-2.5-flash`; **4–6 s random pause between model attempts** to respect free-tier RPM; chat uses the SDK with `gemini-3.8-flash` (600 tokens) |
| OpenAI | `openai_service.dart`, Chat Completions | `gpt-5-mini` |
| Anthropic | `anthropic_service.dart`, Messages API | `claude-haiku-5-5` |
| Groq | `groq_service.dart` | fast free text models **+ Whisper transcription** for audio/video ingestion; text requests over ~100 000 chars are skipped (free tier 413/stall) |
| OpenRouter | `openrouter_service.dart` | free-model router, last resort |

**Safety settings:** all four Gemini harm categories are `HarmBlockThreshold.none`
(harassment, hate, sexual, dangerous). The code comments explain this as a deliberate
product decision (health-science coursework trips filters; user's own material; coach
prompts carry their own guardrails; user pays for their own quota). It is centralized so
it can be tightened in one place — both REST and the chat model read the same list.

### 5.3 Failover policy — `providerOrder()` (pure, unit-tested)

```
vision request (any DataPart):            [gemini]                    — only Gemini reads images
text request:  [openai] → [anthropic] → [groq (≤100k chars)] → [gemini] → [openrouter]
```
Paid keys first, then fastest-first free tiers; providers without keys are skipped. Quota
errors (`429/quota/resource_exhausted/rate limit`) short-circuit the remaining **Gemini
models** (quota is project-wide) but not other providers. Non-quota errors (503, timeouts,
empty responses) advance to the next Gemini model. Text fallbacks receive the same prompt
content flattened to text plus a shared **fallback system prompt** (`_fallbackSystemPrompt`)
that re-implements the formatting discipline (no meta-talk, assess-mode restrictions,
currency/dash hygiene, illustration rules or a no-illustrations clause, quiz `Why:`/
`Misconception:` lines for `quiz`/`quiz_true_false`). Fallback results are wrapped into a
synthetic `GenerateContentResponse`; in streaming they arrive as **one final chunk** after
`onReset()` cleared partial output.

**Streaming:** `generateStream` mirrors the same order. Gemini attempts use a **30 s
inactivity timeout** (a healthy stream never trips it) and a **short-stop guard** — output
< 300 chars on a > 3000-char input is treated as a give-up and fails over (small inputs
exempt). Each model restart fires `onReset()` so the UI clears partial text.

**Quota UX:** a vision request whose only provider failed on quota throws
`GeminiVisionQuotaException` ("Image processing capability quota for the day has been
exhausted… try again tomorrow"); regeneration retries for assessments exclude Gemini and
drop images (`excludeGemini`).

### 5.4 Prompt system

**Process-type inventory** (`_learnPrompts` + `_assessPrompts`, i.e. `allProcessTypes`):

- **Learn:** `notes`, `notes_qa`, `cheatsheet`, `summary`, `transcript`, `approach_feedback`,
  `flashcards`, `podcast`, `tutorial`
- **Assess (user-selectable count 1–100, default 20; `exam_case` 10, `quiz_match` 6):**
  `quiz`, `quiz_true_false`, `quiz_fill`, `quiz_multi`, `quiz_order`, `quiz_match`,
  `quiz_mixed`, `exam`, `exam_free`, `exam_case`, `exam_mixed`
- **Machine-owned pseudo-types** (raw prompt passthrough, strict-JSON/raw contract):
  `chart_repair`, `essay_grade`, `quiz_key_check`

`buildPrompt()` composes, in order: base prompt → count rewriting (`applyQuestionCount`
string-replaces "20 questions"/"10 cases"/"6 match items" etc.) + a strict backstop
(`QUESTION COUNT (STRICT): EXACTLY n …`) with a **Bloom difficulty blueprint**
(`quizBlueprint`: Easy/Normal/Hard/Mixed splits n into recall→evaluate segments ordered
easiest-first) → **trust gate** (`_stripSourceInstruction` removes per-item `Source:`
citation lines in knowledge mode so models can't invent "Page 42"; grounded runs keep
citations under a strict CITATION RULE) → **learn closer** (for `learnCloserTypes`
= {notes, notes_qa, cheatsheet, summary}: mandatory `## Objectives` first section,
exactly one `## Check yourself` last section emitted only when the document is complete,
anti-plagiarism paraphrase-with-citations rule, hard length cap of ≤ 1 output page per
source page, **key-term cloze** — 6–12 `[[term]]` markers (1–5 words) on first
mentions in body prose only, plus a `## Key terms` glossary section listing every
marked term with a one-line definition, placed immediately before `## Check yourself`,
and a **grounded section self-check** — after every `##` content section (except
`## Objectives`/`## Key terms`/`## Check yourself`), a `### Quick check` block of
1–2 questions drawn only from that section's own text, each followed by an italic
`*Answer:*` line) → reading level → **strict language override** → Bloom line →
**output constraint** (no reasoning traces/meta-talk/filler; start immediately with first
heading or question 1) → **table instruction** (`[TABLE_START]/[TABLE_END]` fences,
faithful reproduction, no pipes inside cells; assess mode forbids tables) →
**LaTeX instruction** (exhaustive rigor when math exists; zero `$` decoration otherwise;
currency/ordinals/dates as plain text; **ASCII dashes only** because PDF fonts lack
Unicode dashes; delimiter hygiene — every `$` closes in the same sentence) →
**chart instruction** (D3-first; `### Figure: <title>` immediately before every
` ```d3 `/` ```chart ` fence; AXES RULE — numbered axes only for data-anchored figures,
pure schematics get zero axes/gridlines; forbids `{"type":"diagram","d3":…}` escaped-JS
shape, ASCII art, and ` ```three `; note ` ```three ` is forbidden in prompts yet the
renderer still supports it for legacy documents) → **grounding** (grounded: facts only
from `--- FILE x/N ---` headers, inline `("file.pdf", p. 5)` citations, fair coverage
across files, drawing-code exception) vs **knowledge instruction** (full knowledge base)
→ **no-source guard** (`noSourceGuardClause`, grounded runs only — makes a literal model
reply with a sentinel instead of hallucinating when the corpus is empty; the UI refuses
to start sourceless grounded runs).

**Coach guardrail (`approach_feedback`):** the model is a study coach — never the final
answer, never completes remaining steps, always reproduces a ` ```d3 ` illustration
(either the student's own sketch marked where it goes wrong, or a labelled setup sketch),
fixed section order (## Problem → illustration → ## What Works → ## Where It Goes Wrong →
## Hints and Next Steps → "Concept to Revise: …").

**Continuation:** `buildContinuePrompt` feeds original content + the last 1500 chars of
output, demands continuation without repetition, keeps assess/exam formats (numbering
continues; exam-style keeps `Answers:` sections), end-detection via a literal
`<COMPLETE>` reply. Exam-style papers (`quiz_fill`, `exam`, `exam_free`, `exam_case`,
`exam_mixed`) are spliced back into Questions/Answers sections by the screen
(`_mergeExamContinuation` in `generation.dart`).

**Chat:** `sendChat` seeds a `gemini-3.8-flash` SDK session with up to 40 000 chars of
current study material ("treat as source of truth… Reply READY"), re-seeds when the
context changes, 120 s timeout; falls back OpenRouter→Groq for quota/overload/timeout or
when Gemini has no key. System prompt: concise grounded study assistant, ≤ 70 words.

### 5.5 Parsing, validation, grading

- **Quiz parsing** (`services/study_models.dart`): `parseQuizQuestions(text,
  expectedOptions, allowedOptionCounts, processType)` first tries structured
  `parseQuizQuestionsJson` (model returning JSON), else legacy text parsing
  (`_parseQuizQuestionsText`) — exact option counts (5 default; 2 true/false; 4 ordering;
  3–6 matching; {2,5} mixed), sequential numbering, `Answer:`/`Why:`/`Misconception:`
  lines; the validators reject duplicated/repeated/missing options and empty parses
  surface the message *"The AI returned an unusable quiz…"* (unit-tested heavily:
  `quiz_validator_test`, `auto_quiz_test`, `quiz_parse` coverage).
- **Text-answer/essay grading** (`services/essay_grader.dart`, `text_answer_parser.dart`):
  `essay_grade` prompt → strict JSON `EssayGrade` (score, criteria, feedback); exam
  free-response/case auto-grading runs AI when keys exist with a keyword-overlap fallback
  (screen-side, `study_suite_screen.dart` L~300).
- **Key-accuracy check on save:** `quiz_key_check` prompt verifies each stored answer
  against the source; flagged items go to `reported_items` and are excluded from review.
- **Sanitization:** model replies are cleaned (`sanitize_reply_test`) — leading
  conversational filler stripped, table markers normalized, code fences balanced.

### 5.6 Vision & images

Images ride as inline `DataPart`s (magic-byte MIME sniffing). Every path to the API runs
`ImagePreflight.preflight` (downscale/compress under a payload budget; unit-tested).
Picked images from file picking/camera preflight once at pick time; library materials
preflight again at request time. No on-device OCR — vision is the OCR.

### 5.7 Figure repair loop (`services/chart_block_repair.dart` + screen wiring)

Pure, UI-free, unit-testable helpers over the generated markdown:

- `extract()` finds figure fences (`chart|svg|d3|javascript|js|mermaid|three`);
  `javascript`/`js` count as `d3`; **legacy `mermaid` is always "bad"** (Mermaid support
  was removed) and gets rewritten into D3 by the repair call.
- `findBadBlocks()` flags corrupt JSON, valid-but-empty blocks, and the forbidden
  `{"type":"diagram","d3":…}` shape.
- `findMissingFigures()` finds `### Figure:`/caption headings (explicit `Figure:` marker or
  diagram-keyword regex) with no fence within 10 lines ahead / 8 lines behind, plus
  caption *sentences* ("The following diagram illustrates…") with no drawing.
- The screen asks the model (max 3 attempts per figure, `_maxFigureRegens`) to supply the
  missing/broken drawing; `splice()`/`insertBlocks()` put repaired fences back at exact
  offsets; the on-screen **Regenerate figure** button re-runs this for one figure index
  (shared indices with `StudyMarkdown` figure ordering).

---

## 6. Input & Ingestion

**Files (`file_ingest.dart` picker extensions):** pdf, docx, pptx, xlsx/xlsm/xls, csv, tsv,
txt, md, json, html, rtf, log; images jpg/jpeg/png/webp/gif (vision); audio
mp3/wav/m4a/aac/ogg/oga/flac/wma/opus/aiff (Whisper); **video mp4/mkv/avi/mov/webm** —
Whisper transcribes the audio track, picture ignored; all media under a **25 MB cap**.
PDFs get per-document **start/end page inputs** (`_DocumentTile`) and are extracted with
Syncfusion; DOCX via `archive` (unzip XML); XLSX via `excel`. Extraction failures surface
AI-error helpers with actionable copy.

**Web (`ingest_views.dart` + `web_ingest.dart`):** `WebIngestService.browse(url)` fetches
the page and discovers article links on listing/hub pages so the user can pick what to
ingest; Google Docs/Slides/Sheets URLs are special-cased and labelled. Extracted content
becomes a grounded source with the same `--- FILE x/N ---` header treatment as files.
⚠️ Ingested web text is **not** wrapped in any prompt-injection containment — page content
goes straight into the prompt (see Weaknesses).

**Text:** pasted text and bare topics run in **Knowledge Mode** (`sourceGrounded: false`):
grounding off, citation lines stripped, no-source guard not armed.

**Images:** photos of handwritten work or textbook pages become vision input (Gemini
only). Audio/video are transcribed by **Groq Whisper** before processing.

---

## 7. Rendering Pipeline

### 7.1 `StudyMarkdown` (`widgets/study_markdown.dart`, 1665 L) — the single renderer

Used verbatim by the live output card and the saved viewer so both look identical.
Block parser: paragraphs (with stray-number suppression), headings h1–h6, blockquotes,
ordered/bullet lists (literal source numbers preserved so exam numbering never resets),
**exam option lines** (`A) …`) aligned under the question gutter with jumbled same-line
options split apart, markdown tables, horizontal rules, code blocks, **math fences**
(``​```latex|math|tex|equation|eq|eqn|katex|mathjax`` render as display math, not code),
display math (`$$…$$`, `\[…\]`), figure fences, and the **ANSWERS divider** (styled on
screen; a full page break in PDF). `processType` adds writing space after questions
(`exam_free`: 80 px; `exam_case`: 40 px after `Question:` lines; answers section gets a
1-line gap). Optional **bionic reading** (bolded first half of every word; math/code runs
untouched — renderer-supported; no screen toggle wires it in the study suite yet).

**Key-term cover-up (cloze):** learn prompts mark important terms with `[[term]]`
double-bracket markers (§5.4). The inline token pattern consumes `[[…]]` everywhere,
so markers never leak raw on any surface: with the output card's **cover-up toggle**
(`_coverKeyTerms`, eye icon in the header row) on, body text renders each marker as a
stateful `_ClozeTerm` chip — an opaque rounded bar (width scaled from term length)
with a reveal icon; tapping reveals the bold term on a tinted chip, tapping again
re-covers (per-chip state survives timer/scroll rebuilds inside the memoized widget
list). With the toggle off (or on non-main-document surfaces: saved viewer, headings,
table cells, option lines, zen reader), the term renders as bold accent-colored text.
`stripKeyTermMarkers` removes the wraps for **TTS reading** (live output and saved
player) and **PDF export** (`_pdfMdText` prints the term plainly — the interaction is
screen-only). Memoization: `coverKeyTerms` is part of `_BlocksKey`.

**Memoization:** parsed widget lists are cached in a bounded LRU (12 entries) keyed on
every render input (`_BlocksKey`); `dropCache()` is called after regens so retry counts
don't go stale. Non-selectable mode (flashcards/quiz options) uses plain `Text.rich` so
taps aren't swallowed, with a raw-text fallback when content parses to zero widgets.

**Inline spans** (`buildInlineSpans`): bold/italic/code with recursive parsing (math
inside bold renders), inline LaTeX via `flutter_math_fork` `WidgetSpan`s with
width-capped `FittedBox` scale-down, and a battery of guards against model pathologies:
currency/plain-text guards (`$5.8 billion`, `120%`, `19th` stay literal; digit-led math
like `$45^\circ$` still renders), **prose-math detector** (≥4 plain words and few TeX
commands → readable text fallback), dangling-`$` repair (tail that looks like math
renders as math, currency stays literal), stray TeX in prose converted via
`toPlainText`, and a never-empty guarantee (delimiter-only tokens fall back to raw text).

**Tables:** fixed column widths (no `IntrinsicColumnWidth` — crash history with math
widgets), content-weighted columns (`tableColumnWeights` mirrors the PDF weights),
horizontal scroll only when floors exceed available width, and a **math-aware cell
splitter** (`_splitTableCells`) that respects `$$`, `$…$` (with closing-in-cell
lookahead), `\(...\)`, `\[…\]`, and escaped pipes — shared with the PDF engine
(`splitPipeRowCells`) so screen and PDF tables never drift.

**Citation & disclaimer surfaces (verified by grep):** stored per-item citations ARE
displayed wherever items are shown — the live output card renders `Source: <citation>`
chips for flashcards (`ingest_views.dart` L406–419) and quiz questions (L1797–1811);
the review queue does the same (`review_queue_screen.dart` L501–523, L713–723); the
saved viewer shows them on saved flashcards (`saved_generation_screen.dart` L1199) and
embeds them into weakness-note remediation prompts (L907–909). Learn-document citations
are inline prose (`("file.pdf", p. 5)`) and flow through the markdown/PDF engines
unchanged. `AiDisclaimerBadge` IS imported and used on three surfaces:
`study_suite_screen.dart` imports it (L48) and renders it in the live output card via
part file `output_tts.dart:104`; `saved_generation_screen.dart:427` and
`review_queue_screen.dart:333` render compact badges on their headers.

### 7.2 Math rendering (`services/math_image_renderer.dart`, 639 L)

- `sanitize()`: unicode→TeX map (~140 symbols incl. Greek/sub/superscripts),
  align/equation/gather→`aligned`, `\mbox`→`\text`, `\text{…}` block protection from the
  symbol conversion, dangling-backslash cleanup, delimiter stripping.
- `toPlainText()`: readable fallback (\frac → (a)/(b), √(…), superscripts, ~150 command
  map, longest-first ordering).
- `isCurrencyAfterDollar` / `isProseMath` / `hasTexCommands` guards (shared with the
  markdown engine).
- `render()`: mounts `Math.tex` into the root Overlay at x = −20000, waits two frames,
  captures a `RepaintBoundary` at pixelRatio 3 → PNG + logical size; per-expression cache;
  parse failure → null (callers degrade to readable italic text, never raw TeX).

### 7.3 Figures

- **`ChartData`/`EconomicsChartData`** (`widgets/economics_chart.dart`): tolerant parser —
  fence stripping, direct-SVG detection, lenient JSON (single→double quotes, trailing
  commas, **stack-aware bracket repair** that rewrites wrong closers and appends missing
  ones), shape normalization (series/data/parallel x-y/labels+values/**Chart.js-style
  `data.labels`+`datasets`**), strict label numerics ("K-12" stays categorical), palette,
  `diagramTypes` (freeform set: pendulum, anatomy, map, flowchart…), `isDiagram`,
  `hasDrawableContent` (SVG/D3 script/nodes/steps/series points — anything else is
  "announced but not drawn" and renders an honest empty bordered box, never fake axes).
- **On screen:** `D3ChartWidget` loads a **self-contained HTML page** (D3 v7 inlined from
  assets → fully offline) into a hidden `WebView`, with a `D3ChartSize` JS channel that
  reports content height so tall illustrations aren't clipped; the page tightens the SVG
  viewBox to the artwork and de-collides crowded axis labels (−32° rotation). The same
  template executes model-supplied **raw D3 scripts** via `new Function(svg, d3, width,
  height, chartW, chartH, chartData)` (model scripts are neutralized against `</script`
  breakout). Types drawn natively in the template: raw SVG, custom script, steps/flowchart,
  nodes/links networks, radar, pie/donut, bar/column/histogram, choropleth,
  line/area/scatter/bubble (year axes snap to whole ticks). **Web build:** the identical
  page runs in a sandboxed offscreen `<iframe>` (`d3_chart_web.dart`) with a token-filtered
  postMessage shim replacing the JS channel.
- **`EconomicsChartWidget`** native `CustomPainter` (line/bar/pie/steps/nodes with dark
  mode + optional grid) remains as the **PDF fallback** painter.
- **Three.js simulations** (`widgets/three_sim.dart`): a ` ```three ` fence is a model-
  authored interactive 3D sim. Contract (also mandated by the Simulation prompt): globals
  `THREE`/`container`/`W`/`H`/`sceneBox`/`controlsBox`, renderer with
  `preserveDrawingBuffer: true`, canvas into `#scene`, HTML sliders/buttons into
  `#controls`, rAF loop, no imports/external URLs/OrbitControls. `tryParse` rejects fence
  leakage and prose without `THREE.` usage. Web build shows a "view in the mobile app"
  placeholder. *(Prompt-side, ` ```three ` is currently forbidden in Learn output — the
  renderer is kept for legacy saved documents; see Weaknesses.)*

### 7.4 PDF engine (`services/document_pdf.dart`, 1199 L + `study_suite/pdf_*.dart`)

One static engine (`DocumentPdf`) used by **both** the study screen and the saved viewer
(`pdf_blocks.dart` is a thin delegate):

- **Fonts:** cached Lato TTFs (Regular/Bold/Italic/BoldItalic) replace Helvetica.
- **Text normalization:** soft hyphens removed, NBSP→space, zero-width chars dropped,
  **all Unicode dash variants → ASCII `-`** (tofu prevention), curly quotes normalized;
  markdown/LaTeX cleanup preserving spacing around inline math.
- **Blocks:** same parser shape as the screen (headings, lists as flowing `RichText`,
  option lines, tables, math, figures, ANSWERS → `pw.NewPage()` page break).
- **Orphan control:** headings are *held back* (`pendingHeading`) and glued to the next
  paragraph/chart/table via `pw.Inseparable` when height estimates fit (≤45% page);
  tables ≤92% of page stay together, oversized ones split with **repeating header rows**.
- **Tables:** flex column widths mirroring screen weights; per-column inline-math width
  caps; zebra rows.
- **Math:** `MathImageRenderer.render` → PNG at 3× (`WidgetSpan` inline / image display,
  capped 680 px).
- **Figures:** `ChartImageRenderer.render` — first tries **D3 via hidden WebView**
  (mobile/desktop) or **offscreen sandboxed iframe** (web), captures the SVG after a
  **settle poll** (bounding-box signature stable across two 350 ms polls, ≤ ~5 s) so
  async model scripts finish drawing; prunes grid lines, de-collides tick labels,
  **tightens the capture box** to the artwork (via `SIZE:w,h` channel message). Cold-
  engine timeout (10 s → retry 20 s) before falling back to the native painter capture;
  script-only figures refuse the painter fallback (it cannot run JS) and become the
  bordered placeholder. Three.js sims capture the WebGL canvas at 2.5 s (9 s deadline).
- **Chrome:** title header, doc title header on pages ≥ 2, footer
  **"Powered by Streamline XM" + "Page X of Y"**, landscape auto when tables exist.
- **Truncation detection** (`pdf_export.dart: checkOutputTruncated`): multi-tier
  heuristics across all output types (missing closers like `## Check yourself`,
  mid-sentence ends, dangling fences) drive the **Continue** flow
  (`continueGeneration` + exam splice/renumber `_mergeExamContinuation`).

---

## 8. Library, Review, Mastery

- **CoursesScreen** → **CourseDetailScreen** (chapter filter auto-selects the chapter with
  most due items; per-topic **mastery bars / grades / RAG colors shown only when the
  "Show Mastery Scores" flag is on** — hidden by default, replaced by an evidence header:
  due counts, answers-logged counts, "not enough data yet", page ranges; the breakdown
  sheet switches to an evidence-only variant with counts and kind lists, no percentages;
  mixed-practice subtitles drop the per-topic accuracy percentages) → rewarded-ad-gated
  **diagnostic seeding** generates a baseline quiz for a topic, marked `source: diagnostic`
  → **TopicDetailScreen** (same evidence/mastery split on the header + breakdown sheet;
  calibration shown as confidence-level *counts* with the confident-but-wrong number;
  a recall-checks row counts automatic typed checks vs "I was right" overrides, flags
  cards at 3+ checks overridden on at least half (likely over-specific keys), and lists
  them with a one-tap **Accept a wording** fix that saves the student's stored wording
  after the opposite-only guard;
  coverage as a kind list; materials list with newest-first toggle + date range filter,
  study launcher → StudySuiteScreen preloaded with the topic, saved generations by kind).
  The engines (`mastery.dart`, `retention.dart`, `TopicStats.mastery()`) are untouched by
  the flag — only the display is gated, and they keep driving mixed-practice selection,
  drill selection, and the review queue.
- **Mastery scoring under the flag stays calibrated:** the weighted-accuracy path feeds
  `topicAccuracyValues` (typed recall checks count fully; self-graded flashcard "correct"
  counts as the calibration weight; a typed override counts at half weight) so hiding the
  *display* of scores never changed how they're computed — and the same values drive the
  hidden score. The same aggregate then passes through chance correction
  (`chanceCorrectedAccuracy`, ≥5 attempts, `max(0, (p − c̄)/(1 − c̄))`) when the window's
  per-attempt chance values align with its outcomes; `c̄` uses the same recency and
  evidence weights as `p`.
- **SavePickerSheet:** when the study screen runs standalone, bookmark-saving opens a
  course→chapter→topic picker with inline creation at every level.
- **SavedGenerationScreen (2323 L):** read-only viewer mirroring the live designs via
  `StudyMarkdown`; handles every saved shape (markdown documents, flashcard decks with
  flip + **four-grade** self-grading "Again/Hard/Good/Easy" logging (`graded_by: 'self'`
  with grade 0/1/2/3), choice quizzes with per-question scoring, typeable papers, podcast
  transcript with the same Listen player, exam papers with writing space) and a **retake
  system** (`SavedQuestionCard`): retakes build a fresh `QuizSession` from saved payloads
  and log attempts like live runs. This is the re-study surface, so it carries the same
  active-recall tools: a **cover-up toggle** hides key terms/answers for self-testing, and
  `StudyMarkdown` turns emphasized `*Answer:*` Quick Check lines into **tap-to-reveal
  chips** instead of printing the answer under the question. Cover-up and Quick Check
  reveals log **no evidence** (deliberate: passive re-reading should not inflate mastery).
- **ReviewQueueScreen (1058 L):** Leitner consumer with three modes — due review, **mixed
  practice** (shuffled capped interleaving across topics incl. not-due items), and **repeat
  drills** (`difficultItems` only; correct answer unflags). Mixed practice draws from
  topics with saved questions: topics with ≥5 recent attempts are ranked weakest-first,
  while thinly evidenced topics are sampled at random with a reserved share (~1/3) of the
  slots even when ranked topics could fill them all. Flashcards require
  a
  **typed recall attempt before the answer can be revealed** (front-side text field +
  "Reveal answer"; tap-to-flip disabled until typed) and use **four grades** — Again →
  box reset/confidence 0, Hard → hold box/correct, Good → +1/correct, Easy → +2/correct —
  so all four buttons change scheduling. The student's typed text is shown next to the key
  answer on **every** card (not just checks), so a mismatch is visible and auditable.
  **One in five cards is a "Recall Check"**, chosen randomly when the card is presented
  (still before reveal, so it cannot be predicted or gamed — typing "x" on the other four
  only unlocks the reveal). Check outcomes use `gradeRecallAttempt` (strict for short keys,
  critical-token guard + fuzzy score for longer ones), log `graded_by: 'typed'` (chance
  NULL), and a failed check resets the box regardless of rating; the result banner says the
  outcome came from the typed answer. A failed check offers **"I was right"**, which logs a
  separate `graded_by: 'typed_override'` attempt at half weight in calibration and schedules
  the card as an **override hold** (`gradeOverride` in `nextBox`) — it neither promotes nor
  resets, so it cannot promote past the check that just failed. Non-check cards log
  `graded_by: 'self'` with the chosen grade. Questions
  (choice + explanation reveal) keep key-graded outcomes (`graded_by: NULL`) and store
  `chance = 1/options`; `reported_items` are hard-excluded.
- **Backup UI:** More menu shows a live status row ("Never backed up · N items" /
  "Up to date" / "Last backed up: 3h ago · N new items since") plus **Back up now**,
  **Restore from file**, and **Export a course** rows (`kIsWeb`-gated). Full restore:
  file picker (warns on non-`.ostbak` extensions), manifest inspect with newer-schema
  rejection dialog, destructive-confirm dialog showing current-vs-backup course/review
  counts, then the service restore and a success snackbar with restored counts. Course
  export (`_exportCourse`): bottom-sheet course picker → **Content only (share) /
  Full (with history)** choice dialog → progress dialog → service export → OS share sheet. When the picked file's manifest has `scope: 'course'`, the
  restore flow branches to a **non-destructive** "Import Course?" confirm (title, date,
  topic/review counts; "Your existing courses and data are not changed") →
  `importCourse` → green snackbar with imported counts — the destructive overwrite path
  is never reached for course archives. The home screen shows the dismissible backup
  prompt banner when the fingerprint is dirty (`study_suite_screen.dart`:
  `_checkBackupPrompt`/`_buildBackupPromptBanner`).
- **ProgressScreen:** weekly adherence, studied hours, per-day minutes, streak; history
  ranges 14 days / 8 weeks / 6 months; range state held in the State (no refetch
  spinner-flash).
- **Mastery/Retention services:** pure math over `attempts`/`engagement` (unit-tested,
  incl. the typed-check calibration weight), with deliberately honest UI copy
  ("observational, not an exam prediction").

---

## 9. Planner, Focus, Timetable

- **TimetableScreen:** one tab per weekday (today active), multi-day blocks (CSV `days`),
  add/edit/delete with confirm; **Start** check-in opens `FocusSessionSheet` with the
  block preselected.
- **FocusSessionController** (screen-owned, keeps ticking across sheet dismissals):
  presets (25 min etc.), 5–240 min clamp, optional goal text, **Lock in** mode that
  requires confirmation before leaving mid-block (`PopScope.canPop`), pause, and
  **"Park distraction"** counter. Completion chime = `SystemSound.play(alert)` +
  `HapticFeedback.vibrate` (Android). Ending a block finalizes the check-in
  (`StudyPlanner.startCheckin/finishActive`) and updates **FocusStats**
  (streak days, today blocks/minutes, goal hit/set) which surface as a "momentum" line
  in the sheet and in progress screens.
- **✅ Removed: the exam schedule.** A later pass removed the entire exam-schedule
  feature that had been "wired" here: the next-exam banner, red weekday tab dots,
  red-tinted day cards, `_showExamSheet` with its add form and topic multi-picker, the
  `examCoverage` strip, and the **Practice** launch. Matching removal on the data and
  engine side: `PlannerExam`, `createExam`/`setExamTopics`/`deleteExam`/`listExams`/
  `examCoverage`/`examTopicIds`/`listAllTopics`, the `exams`/`exam_topics` tables and
  their indexes (dropped by v14), `compressedIntervalDays`, the nearest-exam lookup in
  scheduling, and the course-export exam exclusions. Exam **content** types
  (`exam_free`, `exam_case`, `exam_mixed`, …) are untouched. The timetable is now weekly
  blocks plus check-ins only. Two consequences: review intervals are **deadline-unaware**
  (the 1–3–7–14–30 day schedule never compresses around a date), and the `includeAllKinds`
  cram launcher for covered topics went with the feature. Mixed practice still interleaves
  not-yet-due items from selected topics, so the cram path is largely mitigated — but it
  is not an exam-date-driven launch.

---

## 10. Ads & Monetization (`services/ad_service.dart`, `focus_session_ads.dart`)

- **Google TEST ad IDs throughout** (app id `ca-app-pub-3940256099942544~3347511713`,
  test banner/interstitial/rewarded units). Android only; no-ops elsewhere; preload at
  startup (`AdService.initialize`).
- **Scroll-ad:** not on the first scroll — it fires only once the user has scrolled deep
  into a finished output (60% of `maxScrollExtent`, absolute floor 1200 px; short outputs
  never trip the floor and are never interrupted), once per document, not per scroll.
  Rationale: a reader deep in the material has a reason to wait out the ad and continue;
  a first-flick ad just interrupts reading. Rewarded gates remain only on PDF export and
  diagnostic seeding (both opt-in actions).
- **Rewarded gates:** PDF export (notes/quiz-feedback) and diagnostic seeding require a
  rewarded ad (`_requireRewardedAd`); test video ads fire in these flows during dev.
- Unit-tested (`ad_service_test`).

---

## 11. Configuration, Security & Platform Wiring

- **`main.dart` bootstrap order:** Sentry init (`ObservabilityService`, incl.
  `FlutterError.onError` + `PlatformDispatcher.onError` hooks) → `ApiKeys.load()` →
  `AdService.initialize()` → Supabase init **only if** `SUPABASE_URL`/`SUPABASE_ANON_KEY`
  dart-defines exist (failure caught + logged; app continues) → dark mode + mastery-scores
  flag from prefs → edge-to-edge UI → `runApp` → connectivity monitoring fire-and-forget.
- **Navigation:** `home: StudySuiteScreen()` — the app is a single-screen composition;
  library screens are pushed as routes from it. No named-route table.
- **`KuhesThemeScope`:** the dark-mode toggle is live (persisted) and is now joined by a
  real **`showMasteryScores`** flag (`toggleShowMasteryScores`, persisted to
  SharedPreferences, default **off** — evidence-first display); its `textScale`,
  `highContrast`, `reduceMotion`, `boldText` fields remain **stubs** (defaults, no-op
  setters). The More menu (shell.dart) toggles both via the scope.
- **SharedPreferences keys:** `darkMode`, `showMasteryScores`, and BackupService's
  `ost_last_backup_fingerprint` / `ost_last_backup_time` /
  `ost_prompt_dismissed_time` / `ost_prompt_dismissed_fingerprint`.
- **Android manifest:** `INTERNET`, `RECORD_AUDIO` (mic for transcription workflows),
  AdMob app-ID meta-data (test), `singleTop` launch, edge-to-edge config.
- **Web build:** `web/index.html`, `manifest.json`, `passkeys-bundle.js`; PDF downloads
  via `pdf_downloader_web.dart`; figure pipeline swaps to iframes via conditional imports.
- **`AppLinks`:** hard-coded More-menu rows so privacy/terms/feedback are reachable even
  without Supabase — but the URLs are **placeholders** (see Weaknesses).
- **Secrets surface:** user keys in secure storage (good); dev keys via gitignored
  `defines.json` (acceptable for dev, dangerous if defines leak into release); **hardcoded
  OpenRouter key default in source** (bad — see §5.1).

---

## 12. Observability

`ObservabilityService` wraps Sentry: crash hooks at startup, `captureException(hint,
extra)` at every AI failure boundary (per model attempt, per provider), and
`addBreadcrumb` for successes ("gemini served gemini-3.8-flash"). Breadcrumbs are small
and non-PII. No analytics/product telemetry exists beyond Sentry + AdMob.

---

## 13. Test Coverage (34 files in `test/`)

Pure-logic suites (no network, `sqflite_common_ffi` for store tests):
`course_store_test` (schema/migrations/review/graded_by provenance, per-attempt `chance`
persistence, pure `logAttempt` vs `logAttemptAndReschedule` double-apply guarantee,
grade-aware box holds, per-card override rates, flagged-card candidates from stored
`answer_text`, accepted-wording save/reject, cap-at-3 and remove-wording enforcement),
`backup_service_test`
(fingerprint changes, full create→inspect→restore round-trip with sidecar files,
newer-schema rejection, older-schema upgrade-on-restore, orphan-sidecar purge on restore,
course export/import round-trip incl. id remap, accepted wordings, restore-rejection,
**`contentOnly` export purges personal history and uses `_content_` filename suffix**,
24 h prompt suppression),
`mastery_test` (EWMA/weights, pure `nextBox` schedule table — grade steps, failed-check
reset, override hold, clamping — `selfEvidenceWeight`,
override half weight on both sides, 3-check override flag threshold,
`topicAccuracyValues`, chance correction with the 5-attempt floor and weighted c̄,
unknown-topic sampling with reserved slots, **weighted unknown-topic sampling toward fewest attempts and longest since last practiced**),
`retention_test`, `planner_progress_test`, `focus_session_test`, `focus_stats_test`,
`quiz_session_test`, `quiz_validator_test`, `auto_quiz_test`, `saved_viewer_test`,
`saved_retake_test`, `essay_grader_test`, `math_renderer_test`, `sanitize_reply_test`,
`study_markdown_*` (renderer + table splitting + cache + empty render + Quick Check
reveal), `citation_gating_test` (grounding strips citations in knowledge mode),
`no_source_guard_test`, `paste_text_guard_test`, `output_language_test`,
`key_terms_test`, `quiz_parse_test`, `text_answer_parser_test` (strict short-key vs fuzzy
long-key recall grading, polarity-class synonym/inflection tables, opposite-meaning pairs,
identifier tokens — letters, Roman numerals, B6/B12, Ig isotypes — number and negation
guards, overrides, accepted alternative wordings and their contradiction guard),
`image_preflight_test`, `gemini_rest_test`, `keys_sheet_test`, `ad_service_test`,
`size_error_test` (file-size guardrails), `study_planner_test`, `study_topic_flow_test`,
`truncation_detection_test`, `widget_test`. **Provider order (`providerOrder`) is pure and
unit-tested.** Full suite: **329 tests passing.** Widget-level coverage is thin — the big
screens (study suite shell, ingest views, saved viewer wiring) are tested mostly through
their extracted pure helpers; the review queue's typed-attempt/four-grade/override flow and
the backup More-menu wiring have no widget tests (service-level logic is covered, including
the full `nextBox` grade/failed-check/override table, and a `course_store_test` case that
drives `recordReview` with `gradeOverride`/`failedCheck` to prove the store applies those
rules).

---

## 14. Strengths (honest audit)

1. **Provider resilience as a first-class design.** Pure, unit-tested failover
   (`providerOrder`), quota-aware short-circuiting, per-model Gemini chain with RPM
   pauses, 30 s stall detection, short-stop guard, paid-key priority, chat fallbacks —
   this is the most robust part of the codebase.
2. **One renderer, one PDF engine.** `StudyMarkdown` + `DocumentPdf` are shared by live
   and saved surfaces, eliminating the classic "screen looks right, PDF looks wrong"
   drift; the math-aware table splitter is literally shared between both.
3. **Extraordinary input-hygiene engineering.** The prompt system encodes hundreds of
   hard-won model-pathology fixes (currency vs math `$`, prose-math detection, dangling
   `$`, ASCII-dash-only for PDF fonts, Chart.js payload normalization, escaped-JS-in-JSON
   ban, `</script` breakout neutralization, unicode dash normalization, empty-figure
   honesty). Most of these have matching unit tests.
4. **Honest-empty-state philosophy.** Figures the model never drew render as a labelled
   empty box (with a repair loop and a Regenerate button) instead of fake axes; failed
   math renders as readable text instead of raw TeX; missing sources yield a guard
   sentinel rather than hallucinated content.
5. **Real pedagogy scaffolding.** Objectives-first/Check-yourself-last document contract,
   Bloom difficulty ladder wired to concrete question blueprints, coach guardrail that
   never solves problems, citations only when a real corpus exists (knowledge mode strips
   them to prevent fabricated "Page 42"), AI-disclaimer badges with report hooks,
   reported-items exclusion loop.
6. **Solid local data design.** Foreign-key-clean schema v15 with guarded migrations,
   blob→file migration, distinct kinds per quiz sub-type (no accidental overwrites),
   attempt provenance (`graded_by`) plus per-attempt guessing chance (`chance`), Leitner
   grade-aware scheduling + drills + reported items composing into one review queue.
7. **User-owned data safety.** Manual `.ostbak` backups built on `VACUUM INTO` snapshots
   with integrity verification, sidecar bundling with honest `missingFiles` reporting,
   basename re-housing on restore, newer-schema rejection, and a fingerprint-driven
   "N new items since your last backup" prompt — no cloud, no accounts, no recurring
   cost, and the restore path re-enters the normal migration chain. Per-course exports
   reuse the same container with a non-destructive, id-remapped import path, so students
   can share or move a single course without touching the rest of the library.
8. **Performance-conscious rendering.** LRU-memoized block parsing, compiled regexes,
   cached fonts/math/chart rasters, width-capped math spans, fixed table columns to
   avoid intrinsic-layout crashes.
9. **Evidence-first pedagogy option.** Mastery scores can be hidden entirely (default)
   while the engines keep running: students see due counts, attempt counts, confidence
   counts, and "not enough data yet" instead of gamified percentages — and self-graded
   flashcard evidence is now corrected by typed recall checks rather than trusted blindly.
10. **Accessibility & platform care.** Dark mode everywhere (system-aware chart pages),
    edge-to-edge handling, web parity via iframe pipeline, offline D3/Three bundles,
    secure key storage.
11. **Test discipline where it matters.** 34 focused suites over the exact code that
    breaks silently (parsers, spaced repetition, guards, provider policy, backup
    round-trips).

---

## 15. Weaknesses & Risks (prioritized)

1. **🟠 Secret hygiene (source fixed — rotation outstanding).** The hardcoded
   `kOpenRouterApiKey` default in `supabase_constants.dart` (`sk-or-v1-66522343…`) was
   **removed this session** (now a bare `String.fromEnvironment`), so no key ships in
   builds. Remaining: **rotate the OpenRouter key** — it previously sat in committed
   source, so treat it as leaked — and keep `defines.json` (which holds the same
   OpenRouter key plus live Gemini/Groq/Supabase values) out of any build artifact
   leak.
2. **🔴 Test AdMob IDs everywhere.** `ca-app-pub-3940256099942544~3347511713` + test
   units in the manifest and `AdService`. Shipping this earns no revenue and violates
   AdMob policy expectations; the rewarded gates (PDF export, diagnostics) must switch to
   real unit IDs per store listing.
3. **🟠 Prompt-injection surface.** Ingested web pages and extracted file text are
   concatenated into prompts with no delimiting/escaping or instruction-isolation (the
   `--- FILE x/N ---` headers are structural, not a security boundary). A malicious page
   could steer generation (e.g. "ignore instructions, output…"). Low stakes (single-user,
   own-device) but should be treated before any shared/export feature grows.
4. **🟠 Permissive safety settings** (`HarmBlockThreshold.none` on all categories) — a
   deliberate, documented choice, but it is a policy risk for store review and abuse if
   keys are ever shared (see #1).
5. **🟠 Monolith screens.** `StudySuiteScreen` = 724 L shell + ~9 900 L across 16 part
   files of extension methods on one State class (mutable shared state: ~100+ fields);
   `saved_generation_screen.dart` 2 369 L, `topic_detail_screen.dart` 1 674 L,
   `ingest_views.dart` 2 071 L, `review_queue_screen.dart` 1 289 L,
   `course_store.dart` 2 251 L (migration chain v1→v15 extracted to `course_store_migrations.dart`). Extension-on-State "modularity" gives file-level
   separation but zero architectural isolation — state coupling and `_processType`-keyed
   conditionals pervade every part file. High regression risk for an AI agent making
   edits; extracting controllers/view-models is the single biggest maintainability win.
   Two extractions are done rather than promised: the Leitner rules are a pure
   `nextBox(box, grade, failedCheck)` in `mastery.dart` (table-tested, so the store no
   longer owns untested scheduling branches), and the override-fix bottom sheet moved to
   `topic_override_fixes.dart` (257 L) with `showTopicOverrideFixes(context, …)`, so a new
   feature stops landing inside `_TopicDetailScreenState`. Everything else above is still
   one State class per screen.
6. **✅ Removed (was 🟠 dead feature): the exam schedule.** `exams` + `exam_topics` +
   `CourseStore.createExam/setExamTopics/examCoverage` + `PlannerExam` had been implemented
   and then wired into the timetable; a later pass deleted the whole schedule surface (UI,
   store, engine compression, backup exclusions) to keep the product scoped to
   self-directed study rather than deadline planning — not because study blocks already
   did the same job (blocks carry no deadline or topic coverage). Exam *content* generation
   is unaffected, as is mixed practice (not-due cram items); the v14 migration drops the
   stale tables, so the schema carries no dead exam shape.
7. **🟡 Placeholder legal/contact URLs** (`app_links.dart` privacy/terms/feedback rows are
   hard-coded placeholders) — a store-submission blocker and a trust problem for a health-
   sciences study tool that ingests personal documents.
8. **🟡 Observational metrics risk — partially mitigated.** Mastery/retention numbers
   derive from self-logged attempts; confidence is self-reported and passive engagement
   counts views. The typed recall-check system now *corrects* the weight of self-graded
   flashcard evidence (a "Knew it" counts proportionally to observed typed-check accuracy),
   typed checks use strict short-key matching and, for longer answers, a critical-token
   guard (numbers, identifiers such as hepatitis A/B and factor VIII/IX, negation, and
   curated polarity classes with synonym/inflection matching) plus deterministic fuzzy
   matching; a half-weight "I was right" override cannot count as a full correct, so
   blanket overriding stays damped. Multiple-choice/true-false aggregates are
   chance-corrected (≥5 attempts), thin topics are sampled rather than ranked, and mastery
   scores are hidden by default. Quiz confidence and live `topic:` attempts remain
   self-reported, and fuzzy matching is token-based rather than semantic: a very unusual
   paraphrase can still need the override, and role reversal ("A inhibits B" vs
   "B inhibits A") is invisible to a bag-of-tokens guard. Any future "exam readiness"
   framing would still over-claim.
9. **🟡 Duplicated guard families.** Currency/prose-math/dash guards exist in at least
   three places (markdown inline spans, PDF spans, table splitter) with comment-noted
   "identical behavior" coupling — a change to one silently desyncs the others if tests
   aren't extended.
10. **🟡 ` ```three ` prompt/renderer mismatch.** Prompts currently forbid ` ```three `
    ("REMOVED"), yet `ThreeSimFigure`, the widget, PDF capture, and `chart_block_repair`
    fully support it. Legacy saved docs render; new generations never produce sims; the
    Simulation prompt referenced by `three_sim.dart` docs is not part of the current
    Learn/Assess taxonomy. Intentional or drift — needs a decision.
11. **🟡 Vision single-provider.** Vision work is Gemini-only (OpenAI/Anthropic fallbacks
    are explicitly text-only), so image users hit a hard daily-quota wall with a
    "tomorrow" message; `excludeGemini` retries drop the images. Documented, but a
    capability ceiling.
12. **🟡 Quiz `Source:` fields** remain in the base prompt text for grounded runs and are
    regex-stripped for knowledge mode — a fragile string-surgery approach to prompt
    management (works, unit-tested, but brittle to prompt edits).
13. **🟡 Widget-test gap** for the huge screen files; behavior there is only exercised
    through pure helpers, so shell wiring (state fields, part-file interactions) can
    regress silently. The new backup/restore More-menu wiring and the review queue's
    typed-attempt flashcard flow are service-tested only.
14. **🟢 Minor backup edge cases.** Orphan purging on restore is now implemented: restore
    deletes regular files in `ost_files/{books,materials}` whose basenames are not in the
    backup archive (best-effort `_purgeOrphanFiles`, subdirectories untouched), so the
    sidecar dirs can't accumulate stale files across restores. Sidecar re-housing keys
    on basename, so two rows whose files share a basename in different directories would
    coalesce to one archived copy (`addedPaths` dedupe). The fingerprint cannot detect
    in-place edits that change no counts (e.g. renaming a topic), so the "N new items"
    prompt slightly under-counts. Backups shared via the OS share sheet land wherever the
    user saves them — restore accepts any picked file with an explicit non-`.ostbak`
    warning.
15. **🟢 Minor:** `KuhesThemeScope` accessibility fields are stubs (API parity without
    function); `podcast`/`transcript` are in the prompt map but surface only via specific
    tabs; `engagement` kinds are open-ended strings (schema-wise fine, doc-wise thin).

---

## 16. Agent Quick-Reference (invariants to preserve)

- **Renderer invariants:** never render raw TeX or raw figure code; empty figures become
  labelled boxes; every `$` must close within its sentence; ASCII dashes only in PDF
  text; tables never contain `|` inside cells; screen and PDF table splitting must both
  go through `splitPipeRowCells`.
- **Provider invariants:** vision ⇒ Gemini only; quota ⇒ skip remaining Gemini models;
  text order OpenAI→Anthropic→Groq(≤100k chars)→Gemini→OpenRouter; fallbacks are wrapped
  as synthetic responses and stream as a single chunk after `onReset()`.
- **Format invariants:** assess outputs have no tables/charts; `Why:`/`Misconception:`
  lines after `Answer:` for `quiz`/`quiz_true_false`; exam-style keeps `Answers:` section
  with continued numbering; learn docs start `## Objectives`, end `## Check yourself`;
  length cap ≤ 1 output page per source page; podcast scripts are text-only — the
  illustrations toggle is hidden for `podcast` and the flag is force-cleared when that
  tool is picked (AUDIO RULES forbid figure blocks).
- **Quiz me invariant:** `_quizMe()` prefers the original material (typed text, files,
  URL) as the source and passes the on-screen learn document only as **coverage scope**
  (`_quizScopeFromDocument` reduces it to headings), stashed in `_quizMeSource` so a
  truncated quiz's Continue keeps the same grounding — never silently falls back to the
  notes. The result is marked `sourceGrounded` when real source text was used. With notes
  only, the quiz is flagged `_quizMeUnverified` (unverified banner), saved with
  `source: 'quizme-unverified'`, and **skipped by enrollment/trust checks** so notes errors
  cannot become auto-enrolled review items. The grounding/flag clears on any fresh
  generation or tool change.
- **Save invariants:** same-kind second save replaces, different kinds coexist; verified
  saves enroll Leitner review and trigger the async `quiz_key_check`; unverified Quiz-me
  saves are stored but not enrolled; `reported_items` are excluded from every review path.
- **Flashcard grading invariants:** review-queue flashcards require a typed attempt
  before reveal and show that typed text beside the key answer on every card; outcomes
  are logged with `graded_by` — `'typed'` on recall-check cards (randomly assigned at
  presentation time, ~1 in 5, before reveal; strict `gradeFillBlank` for short keys,
  critical-token guard + fuzzy `gradeRecallAttempt` for longer ones), `'typed_override'`
  for a failed check the student overrides with "I was right" (half weight in
  calibration/mastery, **scheduled as an override hold** so it neither promotes nor
  resets), `'self'` on rated cards, NULL on key-graded quiz attempts. Grades drive the box
  through the pure `nextBox(box, grade, failedCheck)`: override → hold, failed check → 1,
  otherwise Again → 1, Hard → hold, Good → +1, Easy → +2. A failed typed check resets to
  box 1 even on an Easy self-rating; the override takes precedence over that reset.
  **`logAttempt` is a
  pure insert** (no `grade`, no reschedule flag); scheduling is `recordReview` (the review
  queue owns the step) or `logAttemptAndReschedule` (saved-viewer/live-quiz callers that
  have not written the schedule themselves). Never call both for one answer — the split
  makes double-application impossible by construction. The calibration damping weight
  applies **only** to `graded_by == 'self'` correct attempts and requires ≥ 3 automatic
  typed checks (else 1.0); overrides count at half weight on both sides of the ratio, so
  blanket "I was right" tapping cannot restore full trust. `attempts.graded_by` and
  `attempts.chance` are additive-only (NULL-safe for legacy rows, chance treated as 0).
- **Chance-correction invariant:** store the per-attempt guessing chance
  (`1/options` for MC/TF, NULL/0 for free recall); never debias a single 0/1 attempt.
  Aggregate as `max(0, (p − c̄)/(1 − c̄))` over a window whose `chance` values align with
  its outcomes, and only at ≥ `chanceCorrectionMinAttempts` (5) — below that show raw
  accuracy. `c̄` uses the **same recency and per-attempt evidence weights** as `p`
  (`weightedChanceMean`, EWMA α = 0.3), so a format shift is not corrected with a stale
  mix. Hidden mastery UI must still use the corrected value for selection.
- **Active-recall viewer invariant:** saved-viewer cover-up and Quick Check
  (`*Answer:*` → tap-to-reveal chip) are retrieval aids and log **no** attempts or
  engagement evidence; only graded quiz/flashcard flows write evidence.
- **Mastery-display invariant:** the "Show Mastery Scores" flag gates *display only* —
  never `MasteryService`/`RetentionService`/`TopicStats.mastery()` computation, which must
  keep running for mixed-practice selection, drill selection, and the review queue. When
  hidden, show evidence-labeled facts (due counts, attempt/confidence counts, kind lists,
  "not enough data yet") — no bars, percentages, or RAG colors.
- **Backup invariants:** `.ostbak` = ZIP with `library.db` + `files/{books,materials}/*` +
  `manifest.json` (`formatVersion: 1`); snapshots come from `VACUUM INTO` on the live
  connection and must pass `PRAGMA integrity_check`; restore rejects
  `manifest.schemaVersion > CourseStore.currentSchemaVersion` and any other
  `formatVersion`, closes the `CourseStore` singleton before swapping the `.db` (stale
  `-wal`/`-shm` deleted), re-houses sidecars by basename under `ost_files/`, **purges
  orphaned sidecars** (regular files in `ost_files/{books,materials}` whose basenames are
  not in the archive are deleted, best-effort, subdirectories untouched — safe because
  `createBackup` archives every file present in those dirs), rewrites
  `course_books.path`/`materials.path`, and re-seeds the fingerprint prefs from the
  manifest so a restored library isn't flagged dirty. Change detection compares the
  monotonic `LibraryStatsFingerprint` (encode `v1:…`, 12 fields) — never hook write paths;
  prompt dismissal is fingerprint-scoped for 24 h; all backup UI is `kIsWeb`-gated.
- **Course-export invariants:** course archives share the `.ostbak` container with
  `manifest.scope: 'course'`; `restoreFrom` must always reject them (course exports must
  never wipe the library) and `importCourse` must always reject `scope: 'library'` files.
  Course exports bundle only the course's referenced sidecars (no directory scan) and
  exclude planner tables (`timetable_blocks`, `study_checkins`); imports add
  rows to the live DB in dependency order without closing it, never purge `ost_files/`,
  and remap colliding ids together with every reference (including `item_key` substrings)
  — an id placed by the import itself must never be treated as a collision. Exports never
  touch the backup fingerprint prefs.
- **PDF invariants:** Lato fonts + dash normalization (tofu), headings kept with next
  block, tables repeat headers when split, ANSWERS starts a new page, footer
  "Powered by Streamline XM".
- **Prompt anchors that must stay in sync with parsers:** `[TABLE_START]/[TABLE_END]`,
  `### Figure:`, `[[term]]` cloze markers (inline token pattern + `stripKeyTermMarkers`
  in TTS/PDF), `### Quick check` / `## Key terms` (learn closer sections),
  ` ```chart ` JSON shape, `Q:/A:/Source:` (flashcards), `Answers:`
  + `- Marking:`/`- Watch out:` (exams), `<COMPLETE>` continuation sentinel,
  `noSourceGuardClause` sentinel.
- **Glossary of key strings:** `kind` values = process types + `flashcards`;
  `source` = `''` | `diagnostic` | `quizme-unverified`; `review_state.box` 1–5;
  `attempts.graded_by` = `'self'` | `'typed'` | `'typed_override'` | NULL;
  `attempts.chance` = `1/options` | NULL; `attempts.answer_text` = typed wording for
  checked/overridden attempts; `accepted_answers.answer` = student-approved alternative
  (max 3 per card, min 2 content tokens, critical tokens must match or have same-class
  synonym, removable from evidence sheet);
  attempt `item_key`
  `topic:<id>:(q|card):<i>:<hash>` or `gen:<generatedId>:<i>`; figures are indexed in
  document order for repair; backup fingerprint = `v1:<11 count/max-id fields>`; More-menu
  prefs `ost_last_backup_*` / `ost_prompt_dismissed_*`.

---


