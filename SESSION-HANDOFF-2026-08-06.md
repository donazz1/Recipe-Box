# Recipe Box — Session Handoff (2026-08-06)

Comprehensive state dump for resuming this work in a new Claude Code session if needed. Written because this session may not persist. Paste this whole file into a new session to get it back up to speed.

---

## 1. What Recipe Box is

A private family recipe manager. Single-file web app (`recipe-box.html`) hosted on GitHub Pages at `https://donazz1.github.io/Recipe-Box/recipe-box.html`, backed by Firebase (Firestore + Authentication). No build step — the whole app is one HTML file with inline CSS/JS, deployed by dragging the file into the repo folder and pushing.

**Firebase project:** `recipe-box-e91fb`
**Firestore structure (per user):** `users/{uid}/recipes`, `users/{uid}/grocery`, `users/{uid}/contacts`, `users/{uid}/inbox`, `users/{uid}/trash`, plus a `users/{uid}` profile doc.
**Recipe field schema:** `title` (string), `categories` (string[], 1-2 of Breakfast/Lunch/Dinner/Snack/Dessert/Other), `qualifiers` (string[]), `serving` (string), `calories`/`protein`/`fat`/`carbs` (number|null), `macrosEstimated` (boolean), `ingredients`/`instructions` (string[]), `sourceUrl` (string), `image` (data URL or null, capped ~300KB JPEG, 900px max width), `ownerUid`/`ownerName` (original creator, preserved through re-shares), `sharedFromUid`/`sharedFromName` (immediate sender, for the "from X" badge).

**Auth model:** password-based (not passwordless email-link — deliberately avoided due to iOS "Add to Home Screen" storage isolation breaking that flow). Each user has their own private library, not a shared global store.

## 2. Deployment workflow (unchanged all session)

1. Drag `recipe-box.html` into the repo folder in Finder.
2. Commit + push via **GitHub Desktop** — Terminal `git push` fails with an auth error every time (`could not read Username for 'https://github.com'`), only GitHub Desktop has valid credentials on this Mac. `gh auth login` was identified as the likely fix (would also authenticate git for Terminal pushes) but the user deferred actually doing it — still outstanding, not blocking, GitHub Desktop works fine as the workaround.
3. GitHub Pages CDN can lag a few minutes behind a push — verify via `curl -s https://api.github.com/repos/donazz1/Recipe-Box/commits/main` (commit SHA on github.com), not just GitHub Desktop (which only confirms the local commit).
4. Cloud Function changes (password reset sender): Google Cloud Console → Cloud Run → `send-custom-sign-in-link` → Edit and deploy new revision → Source tab.
5. Firestore rules: Firebase Console → Firestore Database → Rules tab (browser editor, no CLI).

## 3. Current git state (as of this handoff)

- `origin/main` is at commit `80d94f9` ("RB Sous Chef Icon + Intro Update").
- **Local is 2 commits ahead, NOT YET PUSHED:**
  - `78260bb` "RB Help Voice + Nav Bar Fix Update"
  - `108b58b` "RB Sous Chef Audio Freeze Fix" (the bug fix from tonight, see §7)
- **`privacy.html` exists on disk but has never been committed at all** (untracked file). Needs `git add privacy.html` + commit + push.
- **Action needed:** push all of the above via GitHub Desktop before any of it is live or testable in the wrapper (the wrapper loads the live site, not local files).

## 4. Features built this session (chronological)

1. **Recipe ownership tagging** — `ownerUid`/`ownerName` (original creator, survives every re-share) kept deliberately separate from `sharedFromName` (immediate sender, powers the "🎁 from X" badge). Backfilled automatically in `normalize()`. Preserved (never overwritten) through invite-copy and "Send to..." accept flows. Added an owner/person filter row next to category filters (auto-hidden until 2+ distinct owners exist in a library) and an always-visible attribution badge on every card.
   - **Incident:** this feature was silently wiped out by a later, unrelated commit (`6993f9a`, "RB VPP Recipe Import Addr Update") that appears to have been made from a stale local copy of the file. Discovered and re-applied. **Lesson: always verify current file state before building on top of it — don't assume prior session work survived.**
2. **Sous Chef** — a narrated, hands-on walkthrough library (own screen, reached via a 👨‍🍳 corner icon on Home or a Menu item). Evolved through two versions:
   - v1 (rejected by user): static highlight + narration over a frozen screen. Not good enough — user wanted the app to actually *operate itself* while narrating.
   - v2 (current): 5 modules (Home & Library, Add a Recipe, Cook Mode, Grocery List, Invite & Sharing), each with real scripted DOM actions (clicking real nav buttons/tabs/cards) timed against narration playback, "Play full tour" (chains all 5) and individual "Play walkthrough" per module. Engine: `SOUS_CHEF_MODULES` array with `enter()` + `beats` (timed action list), `playModule(i, chain)`.
   - Voice: Voicebox local API (`http://127.0.0.1:17493`), profile_id `24d1cbf6-67fa-4911-ac9d-8a5ae87a754e` ("Don" cloned voice), instruct: *"patient, clear, engaging teacher — explains ideas simply, warm slight smile, pauses to let points land."* Audio files at `audio/tour/*.wav`.
   - **Known Voicebox bug, always check for it:** the cloned voice frequently ad-libs an unrelated intro sentence ("Effortless. So take a breath, settle in...") before the real script. Fix: transcribe with faster-whisper (`WhisperModel("small")`, word timestamps), find where the real script starts (search first 30-40 words), cut with ffmpeg (`-ss <time> -c copy`). Always re-transcribe the trimmed result to confirm before shipping.
3. **Help-screen narration** — the pre-existing "🔊 Read this aloud" button used to use the browser's generic `speechSynthesis`; converted to the same pre-recorded Voicebox pattern, one clip per section (13 sections), chained via `onended`. Files at `audio/help/*.wav`. Same voice/instruct as Sous Chef, for consistency.
4. **Corner icon** — replaced a 💪 "strong-arm" branding badge in the Home screen header with 👨‍🍳, now doubling as the Sous Chef launch button (`#sousChefBadge`).
5. **Nav bar CSS fixes:**
   - Bottom nav bar "floating to the top" during scroll on iOS — fixed with `transform:translateZ(0)` + `-webkit-backface-visibility:hidden` (forces its own compositing layer, the standard fix for this WebKit quirk).
   - Taps feeling unresponsive/"sticky" — fixed by adding `touch-action:manipulation` globally (removes double-tap-zoom gesture ambiguity/delay).
   - **Neither fix has been confirmed on a real device yet** — only reasoned from known WebKit behavior.
6. **Account deletion** (Profile → Delete my account) — required by Apple Guideline 5.1.1(v) for any app with account creation. Confirmed via code check that this was genuinely missing before tonight. Flow: password re-entry + typing "DELETE" to confirm → deletes all Firestore subcollections (`recipes`, `grocery`, `trash`, `contacts`, `inbox`, chunked in batches of 450) → deletes the profile doc → `reauthenticateWithCredential` + `deleteUser` on the Firebase Auth user (Firestore data must be wiped *before* deleting the Auth user, since the session can't write/delete anything once signed out).
7. **Privacy policy page** (`privacy.html`, repo root, standalone static page, not yet committed — see §3) — required by App Store Connect. Content: what's collected (account email, recipe content, contacts), that Anthropic's API is used for AI parsing with the user's own key, that Resend/Cloud Run only handles password-reset email, no analytics/ads/tracking (confirmed via code grep). **Has a placeholder — "[add your contact email here]" — still needs a real email filled in before this is truly final.**
8. **Sous Chef audio freeze bug fix** (tonight's most recent fix, committed as `108b58b`, not yet pushed) — see §7 below for full detail.

## 5. Design decisions made but NOT YET BUILT

- **Auto-share**: when someone connects to another user, prompt them (at connection time, not buried in settings) whether to auto-receive that person's new recipes going forward. **Off by default** (explicitly decided — "there may be many users"). Toggleable anytime by the recipient.
- **Decoupled "Connect" flow**: currently, becoming someone's contact + getting their library only happens via the invite link at signup. Plan: add a standalone way to connect at *any time*, by entering the other person's **email** (user explicitly chose email over a generated code, after hearing the tradeoff — code would avoid an email-enumeration privacy leak, but user prioritized familiarity). This needs to be brokered safely server-side (a small Cloud Function, same pattern as the existing password-reset function) rather than a raw client-side Firestore email lookup, to avoid exposing "does this email have an account" to anyone who tries.
- **"Send my whole library" bulk action** — a new on-demand button (distinct from the existing one-time signup snapshot and the existing one-at-a-time "Send to...") to push everything to an existing contact whenever, not just at signup.
- None of §5 has been implemented yet. This is the next real feature work on the Recipe Box side once the wrapper/App Store track is further along.

## 6. iPhone App Store wrapper — full status

### Architecture
Native shell (WKWebView) pointing at the **live** GitHub Pages URL (`https://donazz1.github.io/Recipe-Box/recipe-box.html`), **not** a bundled local copy — so ordinary content/feature updates keep shipping via the normal drag-and-push workflow without needing a new App Store review. Only changes to the native shell itself would need resubmission.

### Project location and structure
`/Users/Don/GitHub/RecipeBox-iOS/` — a **separate** git-uninitialized folder, sibling to `/Users/Don/GitHub/Recipe-Box/`, not inside it (Xcode build artifacts shouldn't mix with the web app's repo).

- `project.yml` — xcodegen spec. App name `RecipeBoxApp`, product name "Recipe Box", bundle ID **`com.donazz.recipebox`** (placeholder, not yet reserved in App Store Connect), iOS 16.0 deployment target, `DEVELOPMENT_TEAM` currently blank (needs the Apple Developer Team ID once enrollment is approved).
- `RecipeBoxApp.xcodeproj` — generated from `project.yml` via `xcodegen generate` (run again if `project.yml` changes).
- `RecipeBoxApp/RecipeBoxAppApp.swift` — SwiftUI app entry point. Sets `AVAudioSession` category to `.playback` on launch — this is the fix so Cook Mode's read-aloud (and Sous Chef/Help audio) doesn't silently respect the phone's physical mute switch. **Not yet confirmed on a real device** — the Simulator has no physical mute switch, so this can only be truly tested on real hardware.
- `RecipeBoxApp/WebView.swift` — the actual WKWebView wrapper (`UIViewRepresentable`). Key behaviors:
  - `allowsInlineMediaPlayback = true`, `mediaTypesRequiringUserActionForPlayback = []` — lets audio autoplay without a tap, matching Safari's behavior for this site.
  - `webView.isInspectable = true` (iOS 16.4+) — lets Safari's Develop menu attach a live inspector (Safari → Develop → [Simulator name] → Recipe Box) to see real console errors. Added specifically to debug the freeze (§7).
  - `decidePolicyFor navigationAction` — normal link navigations to the app's own host (`donazz1.github.io`) stay in the webview; anything else opens via `UIApplication.shared.open(url)` (hands off to Safari) instead of navigating the main webview away.
  - `createWebViewWith` (WKUIDelegate) — catches `window.open(url, '_blank', ...)` calls specifically (this is what Recipe Box's "Watch video" button actually uses), opens via Safari, returns `nil` so no dead-end second webview is ever created.
- `RecipeBoxApp/ContentView.swift` — trivial SwiftUI wrapper hosting `WebView()`.
- No app icon set yet (deferred — not required to build/run in Simulator, will be needed before real App Store submission; Recipe Box's existing embedded base64 icon is the intended source, not yet extracted/resized).

### Build tooling status
- **Xcode**: was not installed at session start (only Command Line Tools) — installed during this session (free, via Mac App Store).
- **`xcode-select`**: was pointing at Command Line Tools instead of full Xcode — fixed via `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer` (confirmed working: `xcodebuild -version` reports Xcode 26.6).
- **iOS Simulator runtime**: was not installed (fresh Xcode ships with zero runtimes) — downloaded via `xcodebuild -downloadPlatform iOS` (~8.5GB, took a while). Confirmed installed: iOS 26.5 runtime, devices available include iPhone 17, iPhone 17 Pro, iPhone 17 Pro Max, iPhone 17e, iPhone Air.
- **xcodegen**: installed via `brew install xcodegen` (was missing, Homebrew itself was present).
- **`gh` (GitHub CLI)**: installed but never authenticated (`gh auth status` → not logged in). User started `gh auth login` guidance but deferred actually completing it. Not blocking — GitHub Desktop still works for pushes.

### KNOWN BROKEN TOOL — read this before trying to use the Simulator MCP tool
The `mcp__Claude_Code_iOS_Simulator__control` tool's actions (`attach`, `launch`, `screenshot`, `tap`, `text`, etc.) **all fail identically** with: *"Xcode is installed but not selected. Run `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`..."* — **even after that command has been run and independently confirmed working** (`xcodebuild -version`, `xcode-select -p` both correct). This persisted even after fully quitting and reopening the Simulator.app itself. Working theory: the MCP server backing this tool cached the "not selected" state before the fix was applied, and only a full restart of the Claude Code session itself (not just the Simulator app) would cause it to recheck. **This was not resolved as of this handoff** — the user was mid-decision about whether to restart the session to fix it.

**Working fallback used all session, fully functional, just not automatic:**
- Build: `mcp__Claude_Code_iOS_Simulator__build` tool (a *different* tool from `control`) **does work** — `{"action":"build","project_path":"/Users/Don/GitHub/RecipeBox-iOS/RecipeBoxApp.xcodeproj","scheme":"RecipeBoxApp"}`, then poll `{"action":"build_status","build_id":"..."}`. Omit `device`/`udid` params — passing them causes a false "No booted simulator" error even when one genuinely is booted (this is also part of the same stale-state issue, apparently only affecting `control`-adjacent device-lookup logic).
- Boot a simulator: `xcrun simctl boot "iPhone 17"` (device names or UDIDs both work with plain `simctl`).
- Open the actual visible Simulator window: `open -a Simulator` — **important**: plain `simctl` commands (boot/install/launch) do NOT open a visible window on their own; the device runs "headlessly" until `Simulator.app` is explicitly opened this way.
- Install: `xcrun simctl install "iPhone 17" "<path to .app from build_status output>"`
- Launch: `xcrun simctl launch "iPhone 17" com.donazz.recipebox`
- Terminate (before reinstalling an update): `xcrun simctl terminate "iPhone 17" com.donazz.recipebox`
- Screenshot: `xcrun simctl io "iPhone 17" screenshot <path>.png` — works reliably, this is how every verification screenshot this session was taken.
- **No working way found to synthesize taps/text input into the Simulator without the broken `control` tool.** All interactive testing this session was done by the user manually clicking in the Simulator window, with Claude verifying via the screenshot command above.

### Build/test status as of this handoff
- **First build: succeeded on the first real attempt** (33s, 1 harmless warning about AppIntents metadata, irrelevant).
- Installed and launched successfully in the iPhone 17 simulator.
- **Confirmed working (visually verified via screenshot):** app launches, shows the real live Recipe Box sign-in screen correctly (fonts, layout, gradient icon all correct); user signed in with their **real** account (not a throwaway test account — be careful not to test destructive features like account deletion for real); Home screen loaded with real recipe data, photos, category tags, ownership badges, nav bar correctly pinned at bottom (not floating); Sous Chef audio confirmed playing in the correct cloned voice.
- **Confirmed broken, now fixed (not yet re-verified):** a real freeze bug — reported as: after navigating through Menu screens while Sous Chef audio was playing, the app became fully unresponsive (no scroll, couldn't see top/bottom of screen, unresponsive to input). See §7 for the root cause and fix — **the fix has been committed (`108b58b`) but not yet pushed live, and not yet re-tested** since the wrapper loads the live site, not local files.
- **NOT yet tested at all:** Import recipes file picker (explicitly flagged in the original handoff docs as a must-work item), Watch video external-open behavior (also explicitly flagged as must-work), Cook Mode's mute-switch audio behavior (can't be tested in Simulator at all, needs real device), the account-deletion screen (deliberately not tested for real since the signed-in account is the user's genuine real account).

## 7. The Sous Chef audio freeze — root cause and fix (this session's most recent work)

**Symptom reported by user:** after interacting with Sous Chef / navigating menu screens while audio was playing, the entire app became unresponsive — no scrolling, couldn't see top or bottom of the screen, wouldn't resize.

**Root cause identified by code review** (not yet confirmed via live reproduction — Web Inspector was set up for this but not used before the handoff):
In `playModule(i, chain)` (the Sous Chef engine), the shared `<audio>` element's `.src` was being reassigned and `.play()` called again **without first pausing/resetting the previous playback or clearing its `onended` handler**, and the `.play()` promise was never awaited or caught. If a user taps into a new walkthrough while a previous one's audio is still mid-decode, the browser aborts the in-flight play request — an unhandled promise rejection. WKWebView is known to handle rapid audio-element `.src` swaps under an in-flight decode poorly at the native media-pipeline level (AVFoundation-backed), which can manifest as broader UI unresponsiveness, not just an audio glitch — consistent with what was reported.

**Fix applied** (in `recipe-box.html`, committed as `108b58b`, NOT yet pushed — see §3):
- `playModule()`: now calls `scAudioEl.onended = null; scAudioEl.pause();` **before** reassigning `.src`, fully detaching the previous playback first.
- `scAudioEl.play()` calls (in both `playModule` and `toggleSousChefPlayPause`) now have `.catch(err => console.error(...))` so rejections are handled instead of silent/unhandled.
- The identical pattern was found and fixed the same way in the Help-screen narration engine (`playHelpSection`), which uses the same chained-audio approach, as a preventive measure (lower risk there since there's only one global toggle button, not multiple simultaneous trigger points, but cheap and correct to fix regardless).

**Status: fix is written and committed locally, syntax-checked, but NOT YET verified against the real freeze** (needs: push live → reload the wrapper → deliberately try to re-trigger the freeze → confirm it no longer happens, ideally with Safari's Web Inspector open to catch anything else).

## 8. VPP (Video → Recipe) — the companion Mac app, and "Plan B"

**What VPP is (as of this session, verified from Cortex docs, not assumed):** a **local Node/Express server** on Don's Mac (`localhost:3847`, started via `npm install && npm start`), with a disk-based library (`data/library.json` + `data/images/`), Gemini API key stored in browser localStorage. **Not currently a packaged/installable app at all** — distributing it to anyone else today would mean handing them a Node project and terminal commands, not viable for non-technical users. Repo: `/Users/Don/Projects/Reciepe Video Post-Processor` (historical typo in the folder name, kept as-is).

**Original plan (superseded):** a file-only bridge — VPP exports a `{ "recipes": [...] }` JSON file (documented schema matches Recipe Box's own recipe fields, see §1), user manually gets that file onto their phone (AirDrop/Files/email) and imports it via Recipe Box's existing "Import recipes" screen. This was documented in `/Users/Don/Projects/Reciepe Video Post-Processor/IPHONE_WRAPPER_HANDOFF_PROMPT.md` and three Cortex notes (`Video → Recipe`, `Video → Recipe Export Bridge`, and updates to `Recipe Box`) — **these docs are now partially outdated**, superseded by Plan B below, though the underlying recipe field schema they document is still exactly correct and still the source of truth.

**Plan B (current, agreed, VPP-side agent has confirmed alignment and been greenlit to start building):**
1. **VPP becomes a real, distributable Mac app** — packaged via **Electron** (chosen specifically because VPP is already Node/Express + browser UI, mapping over directly rather than requiring a rewrite). Gemini API key moves from localStorage to macOS Keychain (each user needs their own key — never ship Don's). Code-signed and notarized using the **same** Apple Developer Program membership being set up for the iPhone app (one $99/year account covers unlimited apps across iOS *and* Mac, confirmed via research). Direct notarized .dmg/.zip distribution for now, not the Mac App Store (lighter process, sufficient for a small personal-network audience).
2. **VPP signs in directly to the user's own existing Recipe Box account** (Firebase Auth, same email/password — no new account type) and pushes recipes **directly into Firestore** over the network via a new "Send to Recipe Box" action, instead of only producing a file. The existing JSON export stays too, as a manual fallback — not removed.
3. Pushed recipes land through the **same duplicate-aware bulk-import path** Recipe Box's existing Import feature already uses (title + sourceUrl matching, skip duplicates) — explicitly **not** the one-at-a-time person-to-person sharing inbox, which isn't built for VPP's volume (can be hundreds/thousands of recipes at once).
4. **No Firestore security rule changes anticipated** on the Recipe Box side — VPP signing in as a user and writing to `users/{their-own-uid}/recipes` is the identical pattern to that user's own browser writing to their own data today (same UID, same existing rules).
5. **VPP's own separate "Sous Chef" feature** (it already has one, per the Cortex docs) gets narration via the **same Voicebox pipeline** (same "Don" profile, same generation/trim process) but a **different, more energetic instruct string** specifically for VPP: *"an upbeat, enthusiastic chef-teacher — genuinely excited about the food, warm and encouraging, guides you through each step like a favorite cooking-show host, makes you want to keep listening."* (An alternative voice — xAI's Grok Voice API, specifically the "Rex" voice — was researched and explicitly rejected in favor of voice-identity consistency across Don's whole app family and reusing the already-proven pipeline; Grok Voice costs ~$0.05-0.08/min if ever reconsidered.)

**VPP-side status as of this handoff:** VPP's own Claude Code / Cursor agent confirmed explicit alignment on all of Plan B (goals 1, 2, 3 above), was in the process of updating its own Cortex/VPP docs to remove file-only language, and was **greenlit to begin actual implementation** (starting with Goal 1, Electron packaging) — greenlight given, not yet confirmed as started/complete as of this handoff. **Mid-session mix-up worth knowing about:** at one point the user pasted output that was actually from an unrelated project ("InstaDoc," a Tauri-based document converter) instead of VPP's actual response — caught and corrected; just be aware this kind of copy-paste cross-contamination between the user's concurrent Cursor sessions has happened once already.

## 9. Apple App Store submission — full picture

**Cost:** $99/year, **per Apple Developer account, NOT per app** (confirmed via research) — covers unlimited apps across iOS App Store, Mac App Store, and Mac notarization, no separate fees. If the membership lapses, apps get pulled from the store until renewed — an ongoing cost of staying listed, not just a one-time entry fee.

**Enrollment:** Individual (not Organization) — no business entity, no D-U-N-S number, matches personal/non-commercial use. Link: `developer.apple.com/programs/enroll`. **Status as of this handoff: user was walked through starting enrollment; not confirmed complete/approved.** Approval can take up to ~48 hours for Individual.

**Two confirmed, code-verified requirements, both now built (see §4):**
1. Account deletion inside the app (Guideline 5.1.1(v)) — done.
2. A live, reachable privacy policy URL — page written (`privacy.html`), **not yet committed/pushed, has a placeholder contact email still to fill in** (see §3).

**Real, inherent review risk, mitigated but not eliminated:** Guideline 4.2 ("Minimum Functionality") specifically targets apps that are just a website in a wrapper — reviewers test by putting the device in Airplane Mode and checking for a native-feeling offline state vs. a blank/broken page. The native shell work (real nav handling, external link handoff, audio session config) is exactly the mitigation for this, but it's ultimately a human reviewer's judgment call, not something that can be guaranteed in advance.

**Monetization research (in case revisited later):**
- Most recipe apps are free-with-ads or freemium; the well-known paid exception (Paprika) is one-time-purchase, not subscription, ~$5-30.
- Even a purely voluntary in-app "tip" must legally go through Apple's In-App Purchase system (confirmed via research, including a real case of a rejected "Buy Me a Coffee" *link*) — Apple takes 30% the first year, 15% after, per subscriber-year, under the Small Business Program. A $1.99/year tip nets roughly $1.39-1.69 after Apple's cut.
- No fee waiver exists for a personal/non-commercial app — only verified nonprofits, accredited schools, and government entities qualify.
- If ever wanting to comp specific people (family) instead of charging them: **Family Sharing** (native Apple mechanism, automatic for people in the same Apple ID family group, up to 6), **App Store Connect promo/offer codes** (for anyone else), or simplest of all, a custom `freeAccess` boolean on the user's own Firestore profile that the app's own paywall logic checks — this last one involves no Apple mechanism at all and is fully within Recipe Box's own control.

**Real-device testing:** the Simulator cannot test the mute-switch audio behavior at all (no physical switch) — this specifically needs a real iPhone. Testing your own app on your own physical device does **not** require the paid Developer Program membership yet — Apple allows free "personal team" signing for this, with a certificate that needs re-trusting every 7 days until the paid membership is active. This was offered to the user as a next step, not yet done.

## 10. Immediate next steps, in priority order

1. **Push pending work live**: commit `privacy.html` (currently untracked — needs `git add`), then push all pending commits (`78260bb`, `108b58b`, plus the new privacy.html commit) via GitHub Desktop.
2. **Fill in the real contact email** in `privacy.html` (currently a placeholder).
3. **Re-verify the Sous Chef freeze fix** — force-quit and relaunch the wrapper app (picks up the pushed fix automatically since it loads the live site), then deliberately try to re-trigger the freeze (tap between walkthroughs quickly, interrupt one mid-playback). If it still happens, get the real console error via Safari's Web Inspector (Safari → Develop → [Simulator] → Recipe Box) this time, since that wasn't captured before this handoff.
4. **Test Import recipes** (Menu → Import recipes → file picker) — never tested, explicitly flagged as must-work.
5. **Test Watch video** (open a recipe with a source video, tap Watch video, confirm it opens Safari and you can return) — never tested, explicitly flagged as must-work.
6. **Visually confirm** the account-deletion screen/button exist (Profile screen) without actually completing it on the real signed-in account.
7. **Decide**: restart this Claude Code session to attempt fixing the broken Simulator `control` tool (would let a future session drive taps/typing directly instead of relying on the user's manual clicks + screenshot verification), or keep working with the manual fallback indefinitely — both are viable, this is a convenience tradeoff, not a blocker.
8. **Once ready**: connect a real iPhone and test on real hardware, especially the mute-switch behavior, using free personal-team signing (doesn't need paid enrollment approval yet).
9. **Confirm Apple Developer Program enrollment** actually completed/approved (status unconfirmed as of this handoff).
10. Longer-term, not blocking the wrapper: build the auto-share feature (§5), add a real app icon asset, set up the App Store Connect listing (screenshots, description, privacy questionnaire) once the app itself is fully verified.

## 11. Key file/path reference

| What | Path |
|---|---|
| Recipe Box web app | `/Users/Don/GitHub/Recipe-Box/recipe-box.html` |
| Privacy policy page | `/Users/Don/GitHub/Recipe-Box/privacy.html` (uncommitted) |
| This handoff doc | `/Users/Don/GitHub/Recipe-Box/SESSION-HANDOFF-2026-08-06.md` |
| iOS wrapper project | `/Users/Don/GitHub/RecipeBox-iOS/` (project.yml, RecipeBoxApp.xcodeproj, RecipeBoxApp/*.swift) |
| VPP repo | `/Users/Don/Projects/Reciepe Video Post-Processor` |
| VPP handoff doc (partially outdated, schema still correct) | `/Users/Don/Projects/Reciepe Video Post-Processor/IPHONE_WRAPPER_HANDOFF_PROMPT.md` |
| Cortex vault root | `/Users/Don/Library/Mobile Documents/com~apple~CloudDocs/PERSONAL/Cortex` |
| Cortex: Recipe Box project note | `Projects/Recipe Box.md` |
| Cortex: Video → Recipe project note | `Projects/Video → Recipe.md` |
| Cortex: Export Bridge schema note | `Resources/Knowledge/Video → Recipe Export Bridge.md` |
| Live site | `https://donazz1.github.io/Recipe-Box/recipe-box.html` |
