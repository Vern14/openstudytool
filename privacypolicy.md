# Privacy Policy — OpenStudyTool

**Effective Date:** November 28, 2026  
**Developer:** Streamline XM  
**Contact:** streamlinexm@gmail.com  

OpenStudyTool ("the App") is a local-first study, spaced-repetition, and AI-assisted learning application published by Streamline XM. This Privacy Policy explains what information is processed when you use the App, how it is stored, and what third-party services are involved.

---

## 1. Local-First Data Storage

OpenStudyTool does **not** require an account and does **not** operate a central cloud database for your study library.

- **Study Library:** Courses, chapters, topics, notes, flashcards, quizzes, study plans, review history, and focus sessions are stored locally on your device in an encrypted-at-rest OS sandbox (SQLite database and app-private file storage).
- **Backups:** You may export or restore your library at any time using `.ostbak` archive files. These files are created and stored wherever you choose on your device or external storage.
- **API Keys:** If you enter your own AI provider API keys (Google Gemini, OpenRouter, Groq, OpenAI, or Anthropic), they are stored locally on your device using Android Keystore (`EncryptedSharedPreferences`) / iOS Keychain via `flutter_secure_storage`. Streamline XM never receives, sees, or stores your API keys.

---

## 2. Third-Party AI Providers (User-Keyed)

When you use AI features (such as generating study notes, flashcards, quizzes, audio transcriptions, or answer grading), the App sends your selected study material, prompts, or audio files **directly from your device** to the AI provider you configured:

- **Google Gemini** ([Google Privacy Policy](https://policies.google.com/privacy))
- **OpenRouter** ([OpenRouter Privacy Policy](https://openrouter.ai/privacy))
- **Groq** ([Groq Privacy Policy](https://groq.com/privacy-policy/))
- **OpenAI** ([OpenAI Privacy Policy](https://openai.com/policies/privacy-policy))
- **Anthropic** ([Anthropic Privacy Policy](https://www.anthropic.com/privacy))

Your requests are governed by the privacy policy and terms of the provider whose API key you use.

---

## 3. Camera, Photos, and File Uploads

OpenStudyTool requests access to your device's file picker, photo library, or camera **only when you explicitly choose** to import study materials (such as PDFs, lecture slides, images, or audio/video files).

- Selected files are copied into the App's local storage for text extraction, viewing, or playback.
- Content from these files is transmitted off-device **only** when you initiate an AI action (e.g., summarizing a PDF or transcribing an audio file) using your configured AI provider.

---

## 4. Advertising (Google AdMob)

OpenStudyTool displays ads using **Google Mobile Ads (AdMob)**. AdMob may collect and process:

- Device identifiers (such as the Android Advertising ID)
- IP address and approximate location (derived from IP)
- App interaction and ad performance diagnostics

You can reset or limit your Advertising ID in your device's system privacy settings. For more information, see [How Google uses information from sites or apps that use our services](https://policies.google.com/technologies/partner-sites).

---

## 5. Crash Reporting (Sentry)

If crash reporting is enabled in a release build, OpenStudyTool uses **Sentry** (`sentry_flutter`) to diagnose crashes and unhandled exceptions. Crash reports may include:

- Stack traces and error messages
- Device model, OS version, and app version

Crash reports are **not** used for advertising and do not intentionally include your study notes or API keys.

---

## 6. Children's Privacy

OpenStudyTool is not directed at children under the age of 13 (or the minimum age of digital consent in your jurisdiction). We do not knowingly collect personal information from children.

---

## 7. Data Deletion

Because your study library and API keys are stored locally on your device, you can delete all App data at any time by:

1. Clearing your API keys in the **Keys** tab,
2. Deleting individual courses or clearing app storage in your device settings, or
3. Uninstalling the App.

---

## 8. Changes to This Policy

We may update this Privacy Policy from time to time. The updated version will be indicated by the "Effective Date" at the top of this page.

---

## 9. Contact Us

If you have questions about this Privacy Policy or your data, contact Streamline XM at:  
**Email:** streamlinexm@gmail.com
