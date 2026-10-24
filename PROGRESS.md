# PROGRESS

Living state file for this project. Written so a cold-start session with zero
conversation history can pick work up from here. Update it at every milestone.

---

## What this is

An Android-only on-screen keyboard (Input Method Editor) for **9-inch and
11-inch tablets**. Not a phone keyboard scaled up — the whole design assumes a
tablet held or propped, typed on with two thumbs.

Three things make it different from a stock keyboard:

1. **Posture-adaptive layout** — one unified narrow board in portrait, two
   clusters split to the bottom corners in landscape, chosen automatically from
   device tilt and the geometry of the first few touch contacts. No settings
   toggle.
2. **Grounded adaptive decoding** — instead of resizing predicted keys, shift
   the *probability boundary* between geometrically adjacent keys, and decode
   whole words rather than snapping each tap to one letter.
3. **Voice dictation that actually works** — Whisper on Groq with a constrained
   cleanup pass, replacing Android's built-in recogniser.

---

## Current state — Phase 1 complete: baseline build GREEN

- Forked `HeliBorg/HeliBoard` → `orev7s/HeliBoard`
- Cloned locally; remotes `origin` (fork) and `upstream` (HeliBorg) both set
- Base commit = `e97acdbe` — upstream `main`, 2026-10-18
- **`assembleDebug` builds with zero source changes.**
  Output: `app/build/outputs/apk/debug/HeliBoard_4.3-beta1-debug.apk`
  27,254,502 bytes (26 MB), package `helium314.keyboard.debug`,
  versionCode 4300, minSdk 21, targetSdk 37.
- **No application code has been modified.** Everything so far is environment.

## Environment (this sandbox)

| Thing | State |
|---|---|
| Machine | 2 vCPU Xeon Platinum, **4 GB RAM, zero swap**, 30 GB disk |
| Location | Singapore (Alibaba Cloud); Turkey RTT ~140-180ms |
| JDK | Temurin **17.0.21** at `/home/wuying/jdk17` — system JVM is JRE-only with no `javac` |
| Android SDK | `/home/wuying/android-sdk` — cmdline-tools `16111833`, platform-tools `37.0.1`, build-tools `37.0.0`, `platforms/android-37.0`, NDK `28.0.13004108` |
| Gradle | 9.7.1 via wrapper |
| gh CLI | 2.45.0, authed as `orev7s`, scopes `repo workflow read:org gist` |
| Project config | `compileSdk 37`, `minSdk 21`, `targetSdk 37`, Compose BOM 2025.11.01 |

### Build command that works

4 GB RAM with **no swap** means an unconstrained build gets the JVM OOM-killed
inside `minifyDebugWithR8`. This invocation succeeds:

```bash
export JAVA_HOME=/home/wuying/jdk17
export ANDROID_HOME=/home/wuying/android-sdk
export PATH="$JAVA_HOME/bin:$ANDROID_HOME/platform-tools:$PATH"
./gradlew assembleDebug --no-daemon --max-workers=1 \
  -Dorg.gradle.jvmargs="-Xmx2g -XX:MaxMetaspaceSize=512m"
```

- Cold build ~8m30s (17 tasks executed). Expect 1-3 min incremental.
- **Run `./gradlew --stop` before rebuilding.** A stale daemon holding ~1.2 GB is
  what killed the first attempt.
- Do **not** disable debug minification. It exists to keep the APK under the
  25 MB ceiling, which also keeps it small over a slow link.
- Note the SDK CLI renamed the platform package: it is `platforms/android-37.0`,
  not `platforms;android-37`.

### CI is not available

GitHub Actions refuses to start jobs: *"your account is locked due to a billing
issue."* This is account-level and blocks every repo. Local builds are the only
path until it is cleared in GitHub billing settings.

## Repo landmarks

```
app/                 main module (IME lives here)
  src/main/java/helium314/keyboard/   core IME source (Kotlin)
gradle/              version catalogs
.github/workflows/   build-debug-apk.yml, build-test-auto.yml
layouts.md           layout documentation
LICENSE              GPL-3.0
```

## Decisions made, with reasons

| # | Decision | Why |
|---|---|---|
| 1 | Fork HeliBoard, don't start from scratch | IME plumbing (`InputConnection`, composing regions, insets, password fields, autofill, OEM quirks) is ~20% of the code and ~80% of the bugs |
| 2 | Kotlin | Kotlin-first Android toolchain |
| 3 | Custom `View` + `Canvas` for the key surface; Compose only for settings/overlays | Compose adds recomposition cost on the one path where ~8ms touch-to-pixel matters. Raw `MotionEvent` gives `getTouchMajor`/`getPressure`/`getSize` for palm rejection |
| 4 | Split layout in **landscape only**, never portrait | Measured: split cuts discomfort 4.5 → 2.7 in landscape but changes nothing in portrait (2.9 → 2.8). Portrait split = pure learning cost, zero benefit |
| 5 | Voice = Whisper `large-v3-turbo` on Groq | $0.04/hr of audio. Owning the pipeline beats `SpeechRecognizer`, which is a black box |
| 6 | Cleanup pass emits **edit operations**, not free text | Constrains the model so it physically cannot rewrite the user's words. The failure mode people hate is "it changed what I said" |

## Open decisions (unresolved, deliberate)

- **GPLv3 — the big one.** HeliBoard is GPL-3.0. Forking and building privately
  is fine. *Distributing* anything means publishing source. Options: (a) go open
  source and monetise the server-side voice service, (b) pick a permissively
  licensed base, (c) clean-room the IME scaffolding. **Must be settled before
  significant code is written.**
- On-device cleanup model choice (Qwen3 0.6B / Gemma 3n E2B / Llama 3.2 1B).
- Whether to keep or delete HeliBoard's existing prediction engine.

## Next steps, in order

1. ~~Get a baseline build green~~ **DONE** — see build command above.
2. Measure stock keypress→glyph latency on real hardware. **Do not start feature
   work until there is a baseline number to regress against.** Needs `adb` against
   a real tablet. Only the APK (26 MB) ever needs to cross the network — never the
   290 MB repo.
3. Voice pipeline: push-to-talk → Groq Whisper → `commitText`. Smallest shippable
   differentiator; build it before the hard layout research.
4. Posture detection + split layout.
5. Adaptive boundary allocation and word-lattice decoding.

## Reference numbers worth not re-deriving

- Split vs unified, discomfort (1–10): landscape **2.7 vs 4.5**; portrait 2.8 vs 2.9.
  Typing speed 113 vs 127 char/min — but Apple's split keys were physically
  smaller, so that gap is at least partly a key-size artefact.
  Source: Trudeau et al., PLoS ONE 2013, PMC3694062.
- Thumb reach ≈ 105mm from the bottom corners. On a 248mm landscape tablet that
  leaves ~70mm in the centre unreachable; on a 208mm tablet, only ~30mm. This is
  why split matters far more on 11" than on 9".
- Groq: `whisper-large-v3-turbo` $0.04/hr, `whisper-large-v3` $0.111/hr, 25MB/request.
- `gpt-oss-20b` on Groq: ~3.09s to first token — it is a *reasoning* model, so it
  is the wrong tool for a 40-token cleanup. Needs ~16GB for edge deployment.

## Resuming cold

```
git clone https://github.com/orev7s/HeliBoard.git
cd HeliBoard
git remote add upstream https://github.com/HeliBorg/HeliBoard.git   # if missing
git log --oneline -20      # what has actually happened
```

Sync with upstream without a local clone:

```
gh api -X POST repos/orev7s/HeliBoard/merge-upstream -f branch=main
```
