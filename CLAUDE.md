# Strata — project brief

Working notes for whoever picks this up. Read top to bottom once; after that
the sections stand alone.

---

## 1. How to work with Tad

- **Be direct and brief.** He dislikes verbosity. Lead with the answer, cut the
  preamble, skip the summary of what you just did.
- **He leads with his own framework.** When he brings an analysis, extend or
  complete it — don't explain basics back to him. He's a University of Chicago
  student working across math, CS, social science, and classics. Assume
  competence; he'll tell you when something's unfamiliar.
- **Push back when he's wrong.** The most useful thing in this project so far
  has been honest negative assessment (see §6). If a design change will make
  the game worse, say so and say why. Do not soften findings to be
  encouraging.
- **State what's being sacrificed.** When shipping something incomplete, name
  the gap explicitly rather than letting it pass silently.
- **Ask before large rewrites.** He's learning the stack as he goes; a
  refactor he can't read is worse than messy code he can.

### The agent system

Tad works multi-perspective. When he asks for a full review, run these roles
explicitly and let them disagree with each other — the value is in the
conflict, not the consensus:

| Role | Job |
|---|---|
| **Brainstormer** | Generate and branch ideas fast; drop them just as fast |
| **Researcher** | Find real, reputable, well-cited papers; summarize what matters |
| **Debby Downer** | Why this fails. Ranked by probability, no hedging |
| **CopyCat** | Comparable companies — founded when, by whom, worth what |
| **Lawyer** | Every legal exposure, no box left unticked |
| **Technomaniac** | Build it. Deep technical proficiency assumed |
| **Advertiser** | Demographics, channels, what's worked for others |
| **Monetizer** | Extract revenue; optimize placement and pricing |
| **Quality Assurer** | Refuses to call a demo a product; names every sacrifice |
| **Intern** | Asks the naive question everyone else skipped |
| **Manager** | Decides. Big-picture, practical, harsh, forces collaboration |

Don't run all eleven for a small question. Run the two or three that
actually bear on it.

---

## 2. What the game is

A vertical dropper. You fall down a shaft, steering left/right, threading gaps
in horizontal rock ledges, collecting coins. Depth in metres is the score.

**The hook is not falling — it's the crusher.** A ceiling descends behind you,
slightly slower than terminal velocity. Clip a ledge and you stall; the
crusher eats your slack. Every mistake is visibly costing you. This is what
makes the revive offer feel earned rather than extracted, and it's the one
mechanic that separates this from forty other games in the genre.

Reference points: Helix Jump, Doodle Jump (inverted), Downwell.

### Current state

Web release, 19 Sep 2026. `index.html` (~520 lines) plus `manifest.json`,
`sw.js` and three icon PNGs. Canvas 2D, no dependencies, no build step.
Live at https://tadobrien34-coder.github.io/strata/ once Pages is enabled.

**Tad's decision, 19 Sep:** the game is done. He's building a portfolio of
small apps under one company hoping one sticks; this one is "basically good"
and he doesn't want to think about it further. Don't propose features. If he
comes back, it's for the App Store build (§4) or a bug.

Implemented:
- Procedural strata streaming; rows generate ahead of camera, cull behind crusher
- Five depth bands (TOPSOIL / LIMESTONE / GEODE / MAGMA / THE CORE) with
  interpolated palettes — the world visibly changes as you descend, which is
  the cheapest possible progression and makes screenshots at 800m look
  different from 50m
- Coins, debris particles, screen shake, player trail
- `navigator.vibrate` haptics, deliberately asymmetric
- Free continue on the death screen, once per run. Labelled plainly, no ad
  framing, since there is no ad. The rewarded-video SDK plugs in at
  `revive.onclick` when there is one.
- Audio (added 14 Sep 2026): Web Audio synthesis, no files. Coin blip that
  climbs a semitone per coin in a streak (capped at 8, reset by a ledge hit),
  ledge thud (noise + sine), death crunch (noise + descending saw), revive
  rise, and a low crusher drone that swells over the last ~560px of slack.
  Mute toggle bottom-right, persisted in `localStorage` under `strata.mute`.
  AudioContext is created on the first button tap (iOS requirement).
  Coin sound is a two-note ring (B5 → E6) since 18 Sep; the original swept
  blip was rejected as too blippy, and the square-wave layer was dropped
  19 Sep as "staticy" — sines only now, with a 5ms attack so it can't click.
  Confirmed working on Tad's iPhone.
- Music (19 Sep): generative, in Web Audio, no file — so nothing to license.
  Four-chord pad (Am7 · Fmaj7 · Cmaj7 · G6, 8s per chord) of detuned
  triangle pairs through a lowpass, plus a sparse pentatonic plink every ~2s.
  `setDanger(0..1)` is driven from the crusher gap every frame: it opens the
  filter, adds 6.5Hz tremolo, fades in a sawtooth a tritone above the root,
  and a heartbeat at ~140bpm. Plinks stop while danger is high. `MUSIC_VOL`
  is the ceiling. Starts with the AudioContext on first tap; the mute
  button covers it.
- Volume sliders on the start screen (19 Sep): MUSIC and SOUND, 0–100,
  persisted as `strata.musicVol` / `strata.sfxVol` (defaults .7 and 1).
  Effects and the drone go through `sfxBus`, music through its own bus, both
  under `master`, which the mute button controls.
- Space or Enter starts and restarts
- Coin bank: total coins across runs, shown on the start and death screens
- Skins shop (18 Sep): 25 ball skins in a `SKINS` array. 19 purchasable at
  Tad's tiers (10×50, 5×100, 3×500, 1×1000 coins) and 5 earned by best depth
  (220/480/760/1080/1500m), which are never sold. Skins set ball and trail
  colour; `rim` adds a stroke for dark balls; `fx:'glow'` adds a halo;
  `fx:'molten'` cycles colour and sheds embers. Owned set and equipped id
  persist in `localStorage` (`strata.owned`, `strata.skin`).
- Daily shaft (19 Sep): one seed per local calendar day, day 1 = 19 Sep 2026,
  mulberry32 PRNG feeding `build()` only. The canvas has a fixed logical width
  of 360 (scaled to the frame) so the layout is identical on every phone.
  Daily best persists for the day (`strata.dailyBest`, `strata.dailyDay`).
  Death screen shows "DAILY #N · BAND" and a Share button: `navigator.share`
  on phones, clipboard on desktop. This is §9, and it's built.
- Swept ledge collision (19 Sep): tests the ball's whole path for the frame.
- Reduced-motion: no screen shake when the OS asks. Haptics toggle bottom-left,
  hidden where `navigator.vibrate` doesn't exist (iOS).
- PWA: manifest, icons, network-first service worker. Installs to the home
  screen, plays offline, no browser chrome.

### Tuning — treat as load-bearing

Everything that ramps is a function of metres (`m`), so a level system can
swap curves without a rewrite. Reworked 18 Sep after Tad's playtest: the
old fixed constants put the top of the shaft at end-game pace.

```
PPM   = 6      pixels per metre
GRAV  = 950    (was 1500; ~0.7s from a stall back to terminal)
STEER = 28     lerp rate toward pointer     (was 17)
KEYSPD= 680    px/s for arrow keys         (was 430)
R     = 13     player radius
GAPVAR= .25    per-row gap jitter, ±25%
REACT = .2     reaction allowance, seconds
HSPEED= 520    px/s a thumb covers; below KEYSPD so keyboard has leeway

vmax(m)         = min(780, 470 + m*0.3)          terminal velocity
rowGap(m)       = 340                            ledge spacing, flat (Tad: speed is the ramp, not density)
gapWidth(m)     = max(100, 190 - m*0.05) × (1 ± GAPVAR)   (ball is 26px; gaps sit ≥40px off the walls)
reach(m)        = max(50, (rowGap/vmax − REACT) × HSPEED)   max sideways shift between consecutive gaps
crusherSpeed(m) = vmax(m) × min(.82, .55 + m*0.0004)   (cap was .9 until 19 Sep)
HITWIN = 5     seconds; two ledge hits inside this window = death
NEAR   = 60    px; the crusher snaps to this close behind the ball on every hit
```

Reaction time per ledge (rowGap/vmax): 0.72s at the top, 0.44s from ~1030m.

**Every ledge is physically possible by construction.** Tad's rule, 18 Sep:
"every ledge should be possible, with a tiny bit of leeway." `reach(m)`
limits how far sideways the next gap may sit from the previous one — 272px
at the top, ~125px deep — so the gap *drifts* rather than jumps. Verified
with a simulated thumb (0.2s reaction, 520px/s): zero stalls to 2500m; a
400px/s thumb also zero; 0.3s reaction dies around 2400m. The sacrifice is
horizontal variety deep down, which is the right trade.

**The crusher ratio is the game.** Crusher speed is a *fraction* of terminal
velocity, 55% at the top rising to 90% by ~875m (ratio unchanged 18 Sep pm; the
stall-budget figures below predate the flat rowGap and read slightly harsh now), so falling clean always
outruns it at every depth and only stalls close the distance. Measured 18 Sep
(sim, 360×640): holding the ball on rock at every ledge from the top survives
8 stalls / 9s; stalling at one ledge in three reaches ~960m; from 800m on,
one stall from a standing start is fatal. Tune the ratio last, alone, and
playtest on a phone.

**Every hit is a near-miss.** Tad's reference is Subway Surfers: stumble and
the cop and dog are right on your heels, then fall back as you run clean. After
a ledge hit, while the ball is slower than the crusher, the crusher rushes in
(exponential approach, most of the distance in ~0.25s) to NEAR px behind the
ball and holds there through the stall and the re-acceleration (~1s at depth);
it cannot take you on its own. Once the ball outruns it the gap opens (sim: 60px at the hit, ~330px
three seconds later at 800m+, ~510px at the top). The only kill is the second
hit inside the window. Accumulated slack no longer matters and the crusher's
own speed only governs how fast it recedes.

**Ledges: one hit stalls, two hits inside 5s kills.** Tad's rule, 19 Sep.
Lethal-on-first-hit was tried 18 Sep and rejected. The 19 Sep version: a hit
stalls you (vy capped at 110 for 0.42s) and the crusher eats the slack; a
second hit within `HITWIN` drops the crusher onto you on the spot. The ball's
eye stays red for the window so the player can see they're on their last
hit. The crusher cap came down from 90% to 82% of terminal at the same time
so a lone stall is survivable at every depth (sim: one stall at 1500m from
terminal leaves ~890px of slack after 6s clean). The crusher still catches
anyone who stalls repeatedly at >5s spacing deep down, just more slowly.

---

## 3. Environment

- **Windows 11.** Project at `C:\Users\tadob\OneDrive\Desktop\strata`
- Python installed via winget but **not on PATH** — both `python` and
  `python3` hit the Store stub. Use the full path:
  `C:\Users\tadob\AppData\Local\Programs\Python\Python312\python.exe -m http.server 8000`
- **No Node installed yet.** `winget install OpenJS.NodeJS.LTS` when needed
- Git: `https://github.com/tadobrien34-coder/strata`, public, main branch,
  deployed by GitHub Pages from main / root.
- Phone testing: laptop at `192.168.1.216`, phone opens
  `http://192.168.1.216:8000` on the same wifi. Firewall must allow port 8000
  on Private networks.

### Confirmed finding: iOS has no web haptics

Tested on Tad's iPhone over LAN. Nothing fires — no coin tick, no ledge
thud, no death pulse. Safari does not implement `navigator.vibrate`. This is
expected behaviour, not a bug, and it means **the central premise of the idea
is currently unverified on half the market.**

Untested: whether haptics fire correctly on Android. Worth five minutes on any
borrowed device before committing to a native rewrite.

---

## 4. Goals, in Tad's order

1. **Ship to the App Store**
2. **Sound and haptics working**
3. **Customization** (skins, trails) — after the above, not before

### Agreed plan

1. **Audio first.** Web Audio synthesis, no sound files: bright short blip for
   coins, low thud for ledges, descending tone for death, plus a mute toggle.
   Works on iOS today, testable immediately, keeps the single-file property.
   The research says juiciness comes from *redundant multimodal* feedback —
   right now there's one channel, and sound is the one that actually works on
   his device.
2. **Apple Developer enrollment** ($99/yr) in parallel — approval takes time
   and gates everything.
3. **Persistence** — done 18 Sep. Best depth and a coin bank survive refresh.
   Bank is the cosmetics currency for the shop; it adds the delta on a
   revived death, not the full run again.
4. **Capacitor wrap + native haptics.** Capacitor's Haptics plugin calls
   Apple's real feedback generators, so the existing canvas game gets genuine
   iOS haptics without a Unity rewrite.
5. **Customization.** Cosmetics only. Ball skins shipped 18 Sep. Themes
   (alternate band palettes) and trails are the obvious next cosmetics.
6. **Leaderboards, once public.** Daily and monthly. Cheapest compliant path
   is Game Center via a Capacitor plugin: Apple holds the identity, no server
   to run, no PII stored by us, which keeps the COPPA / age-law profile clean.
   A self-hosted board means accounts, moderation of names, and a privacy
   policy that actually says something. Decide when the App Store build
   exists, not before.

**Building on Windows:** Xcode is macOS-only and Apple requires its
certificates for signing. Route around it with a cloud CI service —
Codemagic, Bitrise, Capawesome or Expo EAS all keep Mac fleets, compile and
sign, and can push straight to App Store Connect. Codemagic's free tier is
500 macOS minutes/month. What you cannot do this way is interactive debugging
on a simulator; UChicago lab Macs cover that if it's ever needed.

**Known risk:** Apple rejects thin web wrappers under the minimum-functionality
rule, and enforcement has tightened. A canvas game with native haptics,
offline play and no browser chrome usually clears review, but this is a real
risk, not a hypothetical. **Fallback if rejected twice: port to Godot.** Free,
exports to iOS, and the game is small enough that a port is a weekend.

---

## 5. Hard constraints

These came out of the legal and research review. Don't relitigate them without
a strong reason.

- **No randomized rewards.** No loot boxes, no mystery chests, no gacha.
  Zendle & Cairns (2018, *PLOS ONE*, n=7,422) found loot box spending links to
  problem gambling severity at η²=0.054 versus η²=0.004 for non-randomized
  purchases — the randomization specifically is what drives it. Regulatory
  temperature is high and Tad is a small operator.
- **Coins buy cosmetics, never power.** The moment coins buy advantage, the
  depth leaderboard compares wallets instead of skill and the game is dead.
- **This project stays separate from Tad's other venture.** He has a separate
  social-casino/slots project. No shared branding, no cross-promotion, no
  overlapping mechanics. The compliance profiles are incompatible and mixing
  them endangers both.
- **Child-directed status must be decided before ad integration.** The amended
  COPPA Rule reached full compliance on 22 April 2026, adds biometric
  identifiers as personal information, and sharply limits sharing children's
  data with third parties for advertising. A cartoon falling-ball game is
  presumptively child-attractive. If it's child-directed: contextual ads only,
  no IDFA/AAID, much lower eCPM.
- **App-store age laws are live.** Texas's App Store Accountability Act is
  operative; Alabama's starts 1 Jan 2027; Utah's compliance deadline is
  6 May 2027 with a private right of action. These apply to all apps available
  to residents, not just child-directed ones. Developers must accept age-bracket
  data from app stores, act on it, use it only for compliance, and re-obtain
  parental consent after significant changes to terms, privacy policy or
  monetization.
- **IAP must go through Apple.** App Store §3.1.1. No side-channel payments.
- **VIP = auto-renewing subscription** → ROSCA rules: price, term and
  cancellation disclosed before purchase; cancelling as easy as signing up.
- **Ship privacy policy, ATT prompt, CCPA opt-out and a deletion path on day
  one**, not after launch.
- **"Strata" is a placeholder.** USPTO clearance in classes 9 and 41 before
  spending anything on branding.

---

## 6. What the research actually says

Three papers, all peer-reviewed, all well past 10 citations.

**Singhal & Schneider (2021), CHI — "Juicy Haptic Design."** Haptic
embellishments are vibrations that reinforce information *already delivered
visually*; juicy haptics are deliberately excessive positive feedback. Two
studies. Combining haptics with multimodal juicy feedback significantly
improved enjoyability, appeal and immersion. → Vibration must accompany a
visual and a sound, not substitute for one.

**Lomas et al. (2017), CHI — "Is Difficulty Overrated?"** Three experiments,
20,000+ sessions. When difficulty was *blindly assigned*, the easiest levels
were most motivating; moderate difficulty won only when players chose it
themselves. Moderate novelty optimal. Suspense in close games increased
motivation. → Resist making the gaps punishing. Depth already lets players
self-select difficulty. Lean on suspense — near-misses with the crusher.

**Zendle & Cairns (2018), PLOS ONE.** See §5.

Industry context, not academic: hyper-casual retention runs around 5.9% D7 and
1.38% D30. That's the gravity this is fighting.

---

## 7. The honest assessment

Kept here because it's more useful than optimism, and because it was true
before the game turned out to be fun.

Ranked by probability of killing this:

1. **User acquisition is the actual business.** Voodoo tests roughly 2,000
   prototypes a year with millions in CPI spend. Tad has one prototype and no
   ad budget.
2. **D30 around 1%.** Even a hit dies in a month without a meta layer.
3. **The genre is post-peak** — incumbents moved to hybrid-casual because pure
   hyper-casual LTV collapsed.
4. **Haptics are half-dead on iOS** (now confirmed empirically).
5. **Clonable in a weekend.** No moat, no IP, no network effect.
6. **Vertical shafts don't screenshot well** — the depth-band palettes are the
   partial mitigation.

**Comparables:** Voodoo (2013, Alexandre Yazdi & Laurent Ritter, valued above
$2bn, 8bn+ downloads); Rollic (Dec 2018, Burak Vardal + two cofounders, 80%
acquired by Zynga for $168m cash within two years); Ketchapp (2014, the Morcos
brothers, sold to Ubisoft 2016); Lima Sky / Doodle Jump (2009, the Pušenjak
brothers, 200m+ downloads, never sold).

The Rollic line is the one to study: three people, no IP, 24 months, $168m.
Their method was screening hundreds of concepts monthly and keeping only those
showing high retention and viral spread on TikTok. The product was never the
asset — the testing pipeline was.

**Framing Tad has landed on:** he's treating this like the lottery — building
it to learn, not needing the money back. That's the right posture. Even if the
game earns nothing, it teaches creative testing, ad mediation, cloud CI and
app-store compliance, all of which his other venture needs.

---

## 8. What's deliberately missing

Don't let anyone call the current build a product.

- iOS silent switch mutes Web Audio entirely. No in-game workaround; worth a
  one-line hint if players report no sound.
- Persistence is `localStorage` only (`strata.best`, `strata.bank`, `strata.mute`).
  Fine for the web build; the Capacitor wrap should keep working since the
  WebView has localStorage, but confirm it survives an app update.
- No difficulty tuning against real players; gap width and crusher speed are
  guesses and need remote config plus an A/B test
- No colourblind check on the magma band. Reduced motion and the haptics
  toggle are done.
- No analytics. **Measure session length, D1, and continues-per-run — not
  revenue.** If D1 is under 35% no monetization scheme saves this.

---

## 9. The one feature that matters most

**Daily seeded shaft.** Everyone falls the identical level that day; one
shareable end card showing depth and band name — "847m, THE CORE" — designed
to be screenshotted. It's the only feature on any list here that generates
distribution rather than consuming it, and it costs almost nothing to build.
**Built 19 Sep.** See §2.

Everything else on the roadmap helps the game retain. This one helps it
spread.

---

## 10. Compliance (19 Sep 2026 audit)

Tad asked for a twenty-point legal/accessibility pass. The game's position is
that it collects nothing: no accounts, analytics, ads, cookies, purchases,
emails, third-party code or web fonts. Verified by grep: zero external URLs
in `index.html`, `sw.js`, `manifest.json`; ten `strata.*` localStorage keys,
all settings or progress. Don't break this without rewriting the policies.

Shipped: `privacy.html`, `terms.html` (includes refund and cookie sections),
Privacy · Terms · Reset-all-data footer on the start screen, canvas
`role="img"` + description, decorative icons `aria-hidden`, shop cards are
real `<button>`s with `aria-pressed` and labels, focus moves into the shop
and back out, locked cards dim only the swatch so labels keep 5.2:1
contrast. Measured contrast: dim text 5.2:1, gold 11.6:1, bone 15.7:1.

**Tad must fill in** the three highlighted placeholders in both legal pages:
legal/business name, city + state/country, contact email. Governing law is
the same field. Until then they are drafts.

Not applicable today and why: cookie banner (no cookies, storage is strictly
functional), form consents (no forms), fake reviews / hidden fees (nothing
sold, no reviews), unsubscribe (no email), age-of-consent (no data
collected from anyone), SDK audit (none present), asset licences (icons,
music and effects are original and generated in this repo; fonts are the
OS's own). The trademark note in §5 stands: "Strata" is uncleared.

**When ads or IAP arrive, all of this changes:** consent management platform
for GDPR/ePrivacy, ATT prompt on iOS, an explicit not-child-directed
declaration to the ad network, COPPA/age-law handling per §5, and both
legal pages rewritten first.

---

## 11. Decisions and remaining questions

**Structure: endless for now, levels possibly later.** Build the depth curve
and the daily seeded shaft against an endless model. Don't design anything
that a later level system would have to tear out — in particular, keep depth
bands data-driven rather than hardcoded to an infinite shaft, so a 1000m
bottom can be dropped in without a rewrite.

**Audience: not child-directed, and that has to be made true by design.** Tad's
position is "just a fun game," which is the ambiguous middle and the most
dangerous place to sit — the FTC decides child-directedness on subject matter,
visuals, characters and actual audience evidence, not on intent. Direction
agreed: push the art *older*. Lean into the menace the crusher and the MAGMA
and CORE bands already carry. Target a 12+ App Store rating with a full ad
stack. Avoid the combination that draws enforcement: bright, cheerful, 4+
rating, behavioural ads.

This is also the better marketing angle — "fall into hell" reads faster on
TikTok than "collect coins."

Still open:
- Spend ceiling before revenue. The $99 Apple fee is unavoidable; everything
  past that is a choice.
- Whether haptics fire correctly on Android — five minutes on a borrowed
  device, and it decides whether a native rewrite is worth it.
