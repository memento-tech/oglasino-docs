# Oglasino — Versioning Runbook

Simple spec. How each repo is versioned, when to bump, and what happens when you do.

---

## 1. The rule (all six repos)

| Repo                     | Scheme  | First value | Where it lives                      |
| ------------------------ | ------- | ----------- | ----------------------------------- |
| oglasino-backend         | SemVer  | `1.0.0`     | `pom.xml`                           |
| oglasino-web             | SemVer  | `1.0.0`     | `package.json`                      |
| oglasino-expo            | SemVer  | `1.0.0`     | `app.config.ts` (marketing version) |
| oglasino-router          | integer | `v1`        | `package.json` + `CHANGELOG.md`     |
| oglasino-firestore-rules | integer | `v1`        | `package.json` + `CHANGELOG.md`     |
| oglasino-image-router    | integer | `v1`        | `package.json` + `CHANGELOG.md`     |

**Independent.** Each repo bumps on its own. There is no lockstep — backend can be at `1.4.0` while web is at `1.0.2`. That's fine and expected.

---

## 2. SemVer in one line (backend, web, expo)

`MAJOR.MINOR.PATCH` → e.g. `1.0.0`

- **PATCH** (`1.0.0` → `1.0.1`) — a bug fix, nothing new.
- **MINOR** (`1.0.0` → `1.1.0`) — a new feature, nothing breaks.
- **MAJOR** (`1.0.0` → `2.0.0`) — a big or breaking change.

When unsure, it's a PATCH.

---

## 3. The integer repos (router, firestore-rules, image-router)

The simplest case. No SemVer, just a counter.

**How:** change the number in `package.json` from `"1"` to `"2"`, add one line to `CHANGELOG.md` (`## v2 — <what changed> (<date>)`).

**When:** each time you deploy a meaningful change to that worker / ruleset.

**What happens:** nothing automatic. It's a record for you, not wired to anything.

---

## 4. backend and web

Also simple — the version is a label.

**How:**

- backend: edit `<version>` in `pom.xml`.
- web: edit `"version"` in `package.json`.

**When:** at each release you want to mark (use the SemVer rule above to pick PATCH/MINOR/MAJOR).

**What happens:** nothing automatic at runtime. (Note: the backend version is _not_ shown by any API endpoint today — it lives only in the build artifact. If you ever want `/actuator/info` to report it, that's a separate small change.)

---

## 5. expo — the one to understand

Expo has **three** version numbers. You only ever touch **one**.

| Number                | Example             | Who sets it                    | You touch it?      |
| --------------------- | ------------------- | ------------------------------ | ------------------ |
| **Marketing version** | `1.0.0`             | You, in `app.config.ts`        | **YES — this one** |
| Build number          | auto                | EAS, every build               | No                 |
| runtimeVersion        | = marketing version | Automatic (policy: appVersion) | No                 |

So: **you change the marketing version, the other two follow.**

The build number going up every build is normal — ignore it. runtimeVersion automatically equals your marketing version, so when you set `1.0.1`, the runtime becomes `1.0.1` too, by itself.

---

## 6. expo — OTA vs new build (the important part)

There are two ways to ship a change to the phone app. Pick one per change.

### A) OTA update — instant, no store review

- **Keep the marketing version the SAME.**
- We run `eas update --channel production` (together — this is a server command).
- Reaches everyone on that version on their next app open.
- **Use for:** JS-only fixes, hotfixes, the kill-switch, text/style tweaks.

### B) New store build — slow, goes through store review

- **Bump the marketing version** (e.g. `1.0.0` → `1.0.1`).
- Build + submit to the store.
- **Use for:** native changes — a new permission, a new native library, an Expo SDK upgrade, or a planned feature release.

### The trap (read this twice)

OTA updates only reach apps with the **same** marketing version (because runtimeVersion = marketing version).

- If you push an OTA **without** changing the version → it reaches your live users. ✅
- If you **bump** the version **and** push an OTA → the OTA does **not** reach your existing users, because they're still on the old version. ❌

**So:**

- OTA-only fix → **do NOT bump the version.**
- Store release → **DO bump the version.** That release starts fresh; OTAs you push afterward target the new version only.

---

## 7. Quick decision guide for expo

> I changed something. What do I do?

- **Only JS/TS changed** (logic, text, styles, a bug fix) → **OTA.** Keep version. Run `eas update`.
- **Anything native changed** (added a permission, a native package, upgraded Expo SDK) → **new build.** Bump version. Submit to store.
- **Emergency: turn something off fast** → **OTA / kill-switch.** Keep version. Run `eas update` with the fix.
- **Planned release of new features** → **new build.** Bump version (MINOR or MAJOR).

If you're not sure whether a change is "native": if you only edited files under `src/` and didn't add a package or change `app.config.ts` native settings, it's almost certainly OTA-safe.

---

## 8. What depends on the version

- **Force-update gate:** the backend compares your app's **marketing version** against the ceiling/floor. So the version you ship is the number that gate sees. (This works today, with or without OTA.)
- **runtimeVersion:** follows the marketing version automatically. This is what makes OTA reach the right phones. (Active once the OTA config — currently in progress — is committed.)
- **Store URLs:** the force-update screen sends users to the store. Android URL is wired. iOS URL is still pending the App Store Apple ID (tracked in issues.md).

---

## 9. One-time note: OTA is being wired now

The OTA part of section 6 becomes active once the in-progress OTA config (install `expo-updates`, set channels, `runtimeVersion: appVersion`) is committed and the first OTA-capable build is in the store. Until then, **every** expo change ships as a new store build (section B), because there is no OTA channel to publish to yet.

Sections 1–5 (the version numbers themselves) apply right now.
