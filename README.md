# Dissecting a Scale App That Ships More Ad Code Than Actual Features

**Target:** OKOK International v3.1.51 (com.chipsea.btcontrol)  
**Tags:** Android, Reverse Engineering, Malvertising, Privacy, Smali Patching  
**AI%:** 30% used to find files and methods, code changes and so were all manual!**

---

## How It Started

\>be me  
\>bought a Bluetooth body scale. Nothing fancy — just wanted to track my weight, BMI, body fat, the usual.  
\>downloaded the companion app, OKOK International, paired the scale, stepped on it and... fullscreen video ad.  
\>cool.   
\>closed it.  
\>banner ad.   
\>swiped to check my history — nope, watch this 30 second reward video first.  
\>mfw

Tried to just *see my weight* and honestly spent more time fighting ads than standing on the damn scale.

I'm not exaggerating. The experience was genuinely worse than browsing a sketchy streaming site with no adblocker. This is an app with over a million downloads on the Play Store. By Chipsea Technology. For a >*bathroom scale*.

At that point something clicked. This wasn't just annoying — I wanted to know what was actually going on under the hood. So I pulled the APK, threw it into apktool, and what I found was way worse than I expected.

This is the story of how a frustrating weigh-in turned into a full reverse engineering session, a security audit, and a patched APK with zero ads. Buckle up.

---

## Part 1: What The Hell Is In This App

### Eight. Entire. Ad SDKs.

I cracked open the decompiled source expecting maybe AdMob and one Chinese ad network. Standard stuff for a free app. Instead I found this:

| SDK | Files | Purpose |
|-----|-------|---------|
| **Mintegral (MBridge)** | 4,000+ smali files | Chinese ad network, fullscreen/video |
| **BigoAds** | 1,799+ files | Video and interstitial ads |
| **ByteDance/Pangle** | 326+ files | TikTok's ad platform |
| **Vungle** | 214+ files | Video ad network |
| **Google AdMob** | Bundled | Banner and interstitial |
| **AppLovin** | Bundled | Fullscreen ads |
| **IronSource** | Bundled | Ad mediation |
| **Fyber** | Bundled | Ad mediation |

All of them running at the same time, managed by **TopOn/AnyThink** — an ad mediation layer that literally auctions your eyeballs to eight companies every time it needs to show you something. The app runs a real-time bidding war on your phone. For a scale app. Let that sink in.

And the kicker? Mintegral alone — just one of the eight — has more source files than the entire actual health/scale codebase from Chipsea. The ad infrastructure is the app. The scale stuff is a side feature.

### Why This Isn't Just "Annoying"

>**every single ad impression is an attack surface.**

I'm not being dramatic. When an ad SDK loads a creative, it's pulling remote HTML/JS content from third-party servers and rendering it inside a WebView on your device. That's remote code execution with extra steps. It's called **malvertising** and it's been a thing for years — but nobody talks about it on mobile because "it's just ads bro."

Here's the actual kill chain:

1. Ad SDK pings the ad network: "give me something to show"
2. Ad server returns a creative — usually a mini webpage with JS
3. That creative renders **inside the app**, with WebView access
4. If the creative is compromised? Phishing redirects, drive-by downloads, browser exploits. All from inside a "legitimate" app.
5. User taps anywhere to dismiss the ad — whoops, that was a click. Now you're on some sketchy landing page.

Now multiply that by **eight SDKs**, all firing constantly, fullscreen interstitials you literally cannot skip. Every time you open this scale app, you're rolling the dice eight times.

The OKOK app specifically hits you with:
- **Splash ads on every launch** — can't skip, can't avoid
- **Fullscreen interstitials** between screens — just navigating triggers them
- **Forced reward videos** — want to use a basic feature? Watch this first.
- **Persistent banners** — always there, always watching

### It Gets Worse: Dynamic Code Loading

While digging through the ad SDKs I found something that made me actually uncomfortable:

```
ByteDance/Pangle → InMemoryDexClassLoader (loads DEX bytecode straight from memory)
Facebook Ads     → DexClassLoader (loads DEX files from disk)
```

Two of these ad SDKs can **download and execute new Java code on your phone at runtime**. No app update needed. The app you installed from the Play Store is not necessarily the app that's running right now. Google Play Protect literally flags this behavior as potentially harmful, and yet here it is, in a scale app with a million installs.

The app you think you installed is just a shell. The ad SDKs can morph after the fact.

### The Cherry On Top: Your Data

On top of all the ad garbage, OKOK requests permissions that have absolutely zero business being in a bathroom scale app:

| Permission | Justification for a scale app? |
|---|---|
| `READ_CONTACTS` |  None |
| `RECORD_AUDIO` |  None |
| `CAMERA` |  None (maybe QR scan, but CameraX isn't even used) |
| `READ_CALENDAR` / `WRITE_CALENDAR` |  None |
| `READ_PHONE_STATE` |  None |
| `PACKAGE_USAGE_STATS` |  None |
| `ACCESS_FINE_LOCATION` |  Required for BLE scanning on older Android, but also fed to ad SDKs |

Why does a scale app need my contacts? My microphone? My camera? My calendar? It doesn't. These permissions exist because the ad SDKs demand them for behavioral targeting. More data about you = higher bid prices = more money per impression. You're not the user. You're the product.

Oh, and the app also ships three analytics SDKs that phone home to three different countries:
- **Umeng** (Alibaba) — Chinese servers. Ships a native lib literally called `libumeng-spy.so`. I wish I was making this up.
- **Yandex AppMetrica** — Russian servers. Runs in its own process (`:AppMetrica`).
- **Google Analytics / Firebase** — US servers.

So yeah. Your weight data, body fat percentage, and BMI measurements live alongside tracking infrastructure that reports to China, Russia, and the US simultaneously. For a bathroom scale. Cool.

At this point I wasn't even mad anymore. I was just... fascinated. Time to fix it.

---

## Part 2: Ripping It Apart

### The Toolkit

Nothing fancy. Standard Android RE setup:

- **apktool** v2.x — APK decompilation and recompilation
- **keytool** (JDK 21) — keystore generation
- **zipalign** (Android SDK build-tools r34) — APK alignment
- **apksigner** (Android SDK build-tools r34) — APK signing
- **VS Code** — smali editing and analysis

### Step 1: Crack It Open

```powershell
apktool d okok-international-3-1-51.apk
```

Standard decompilation. Out comes the full app:
- `AndroidManifest.xml` — readable XML
- `smali/`, `smali_classes2/` through `smali_classes10/` — Dalvik bytecode in smali format
- `res/` — resources (layouts, drawables, etc.)
- `lib/arm64-v8a/` — native libraries
- `assets/` — raw assets

10 DEX files worth of smali. Ten. This thing is *massive* for what it does. Most of it is ad code.

### Step 2: Finding The Brain

I started grep'ing around for ad-related classes and pretty quickly mapped out the whole system. It's actually well-organized (credit where it's due, I guess):

```
TopOnAdManager (com.chipsea.code.ad)
├── Initializes TopOn/AnyThink SDK with APP_ID and APP_KEY
├── Loads and shows: Banner, Reward Video, Splash ads
├── Placement IDs:
│   ├── Banner:       n682afc6f83ee0
│   ├── Reward Video: n682afc6fc1f68
│   └── App Open:     n682afc7022331
│
├── GoogleAdManager (com.chipsea.code.ad)
│   └── Loads Google AdMob banners for specific pages
│
├── GoogleMobileAdsConsentManager (com.chipsea.code.ad)
│   └── GDPR consent flow (controls whether ads can be requested)
│
├── AppOpenManager (com.chipsea.btcontrol.ad)
│   └── Google AdMob app-open ads (fullscreen on launch)
│
└── AdManager (com.chipsea.code.bean)
    └── Ad display config — decides whether to show ads
        based on language and region settings
```

The `getAdConfig()` method caught my eye — a ~660-line monster with a sparse-switch checking dozens of language codes to decide who gets ads. Some locales get hit, some don't. Interesting targeting choices in there.

### Step 3: The Plan

The goal: **kill every ad without breaking anything else.**

I went with **early return injection** — the cleanest approach for smali patching. You don't delete methods (that crashes callers), you don't NOP out instructions (fragile). You just replace the method body with an immediate return. The method still exists, still has the right signature, callers are happy. It just does nothing.

For void methods it's dead simple:
```smali
.locals 0
return-void
```

For methods returning `boolean`:
```smali
.locals 1
const/4 v0, 0x0    # return false
return v0
```

`.locals 0` is always valid — you're telling Dalvik "this method uses zero registers" which is the safest possible floor. No register conflicts, no type mismatches, no verification errors. Clean.

### Step 4: Surgery — File by File

#### 4.1 TopOnAdManager — The Brain

**Path:** `smali/com/chipsea/code/ad/TopOnAdManager.smali`

This is ground zero. Every ad operation in the app flows through this class. I gutted eight methods:

```
initAd()              → return-void  (kills SDK initialization)
loadBannerAd()        → return-void  (no banner loading)
loadRewardAd()        → return-void  (no reward video loading)
loadSplashAd()        → return-void  (no splash ad loading)
toLoadAd()            → return-void  (generic ad loader, dead)
showBannerAd()        → return-void  (can't show what didn't load)
showRewardAd()        → return-void  (same)
showSplashAd()        → return-void  (same)
```

The big one here is `initAd()`. With that returning void, the TopOn SDK never boots up. The APP_KEY and APP_ID never leave the device. None of the eight ad networks get activated. This alone nukes ~90% of the ad system. The rest is defense in depth — belt and suspenders.

#### 4.2 AppOpenManager — The Splash Screen Hijacker

**Path:** `smali_classes3/com/chipsea/btcontrol/ad/AppOpenManager.smali`

This bastard is responsible for the fullscreen ad you see the *instant* you open the app. Implements `ActivityLifecycleCallbacks` and `LifecycleObserver` to hook into every app launch.

```
fetchAd()             → return-void  (don't fetch app-open ads)
showAdIfAvailable()   → return-void  (don't show them either)
isAdAvailable()       → return false (tell callers "no ad ready")
```

Gone. `isAdAvailable()` always returns false now — "nah bro, no ads here."

#### 4.3 GoogleAdManager — Banner Boy

**Path:** `smali/com/chipsea/code/ad/GoogleAdManager.smali`

Handles AdMob banner loading per page. Each screen in the app has a page ID and this class loads the corresponding banner.

```
checkAndLoadAd()      → return-void  (don't check, don't load)
```

One method, one return-void. Bye banners.

#### 4.4 GoogleMobileAdsConsentManager — The Gatekeeper

**Path:** `smali/com/chipsea/code/ad/GoogleMobileAdsConsentManager.smali`

GDPR consent manager. Controls whether Google ads can even be requested. If `canRequestAds()` says no, Google's entire ad pipeline stays off.

```
canRequestAds()       → return false (always: "no, you cannot request ads")
```

Now it always says no. Thanks GDPR for giving us a convenient kill switch lol.

#### 4.5 AdManager — The Config Bean

**Path:** `smali_classes3/com/chipsea/code/bean/AdManager.smali`

This is the ad configuration class. `getAdConfig()` is a ~660-line sparse-switch nightmare that checks your locale to decide if you should see ads. I replaced the whole thing:

```smali
.method public getAdConfig()Z
    .locals 1
    const/4 v0, 0x1    # true = skip ad loading
    return v0
.end method
```

Also killed `initMobileAds()` for good measure. No premium features, paywalls, or subscription logic was touched — strictly the ad delivery pipeline. The scale works, the Bluetooth works, the data works. Just no ads.

### Step 5: Stitch It Back Together

```powershell
# Rebuild APK from modified smali
apktool b okok-international-3-1-51 -o okok-no-ads.apk

# Generate signing keystore
keytool -genkeypair -v -keystore okok-sign.jks -keyalg RSA -keysize 2048 -validity 10000 -alias okok -storepass android -keypass android -dname "CN=OKOK, OU=Dev, O=Dev, L=City, ST=State, C=BR"

# Align the APK (required before apksigner)
zipalign -v 4 okok-no-ads.apk okok-no-ads-aligned.apk

# Sign with v1 + v2 + v3 schemes
apksigner sign --ks okok-sign.jks --ks-key-alias okok --ks-pass pass:android --key-pass pass:android --out okok-final.apk okok-no-ads-aligned.apk

# Verify
apksigner verify -v okok-final.apk
# → Verified using v1 scheme (JAR signing): true
# → Verified using v2 scheme (APK Signature Scheme v2): true
# → Verified using v3 scheme (APK Signature Scheme v3): true
```

~138 MB APK, signed and verified. Installed it, opened the app and... silence. No splash ad. No banner. No reward video. Just my weight. The way it should've been from day one.

I honestly sat there for a second just... appreciating how fast the app loads without ads. It's a completely different experience.

---

## Part 3: While I Was In There...

Since I already had the codebase open, I figured why not check if this thing has any other skeletons in the closet. Spoiler: it does.

### SSL? What SSL?

```
HttpsEngine.smali → calls initSSLALL() before every request
OkHttpUtil$TrustAllCerts → implements X509TrustManager, accepts literally any certificate
```

I had to read this twice. The app's HTTP engine **trusts every SSL certificate**. Every single one. Self-signed, expired, wrong domain — doesn't matter. `TrustAllCerts`. That's the actual class name.

This is MITM 101. Sit on the same coffee shop Wi-Fi as someone using this app, intercept their traffic to `chips-cloud.com`, and you can read all their health data in transit. Weight, body composition, account info. All of it.

But wait, it gets better. Some endpoints don't even bother with HTTPS:

```
http://image.chips-cloud.com/          → profile images
http://run.chips-cloud.com/            → cloud file downloads  
http://www.tookok.cn/license/okok/     → legal agreements
http://172.31.10.160/v3                → hardcoded internal IP (???)
```

That last one. A hardcoded **private IP address**. In a production app. On the Play Store. With a million downloads. I don't even know what to say. Someone pushed their dev config to prod and nobody caught it. Ever.

### The Good News

No `Runtime.exec()`, no `ProcessBuilder`, no `su` checks. The app doesn't try to root your phone or run shell commands. So at least there's that.

It also doesn't actually *use* most of those scary permissions — no SMS reading, no call log access, no audio recording. The permissions are there for the ad SDKs, not for active exploitation. Still bad practice, but not actively malicious.

### The Dynamic Code Loading (Again)

Already covered this above but worth repeating in the security context: ByteDance and Facebook's ad SDKs can pull down and execute new DEX code at runtime. That's a real trust boundary violation that Play Protect flags. In a health app. Cool cool cool.

### Three Countries Tracking You

Already mentioned the analytics, but framing it as a security finding:
- **Umeng** (Alibaba) → Chinese servers, ships `libumeng-spy.so`
- **Yandex AppMetrica** → Russian servers, runs in a separate process (`:AppMetrica`)
- **Firebase/Google Analytics** → US servers

Your bathroom scale app reports your behavior to Alibaba (China), Yandex (Russia), and Google (US). Three separate analytics pipelines. `libumeng-spy.so` is literally in the native libs folder. Umeng named their tracking library "spy." I can't make this stuff up.

---

## The Takeaway

OKOK International isn't a trojan. It won't steal your bank password or brick your phone. But it's a textbook example of what happens when developers treat users as revenue streams instead of people.

8 ad SDKs. Broken SSL. Permissions for contacts, camera, and mic in a *scale app*. Dynamic code loading. Analytics phoning home to three continents. All for an app whose entire job is "show me a number when I step on a thing."

The fix took an afternoon. 5 files, ~15 methods, zero functionality lost. The scale still works. Bluetooth still works. BMI and body fat still calculate. The app just doesn't spend 80% of its time trying to monetize you anymore.

What started as me being annoyed at ads while trying to weigh myself turned into one of the most eye-opening teardowns I've done. If a random scale app is this bad, imagine what's lurking in the other million apps on the Play Store that nobody's looked at.

Stay curious. Check your permissions. And maybe think twice before installing yet another "free" app.

— n3sec

---

## Legal Disclaimer

This write-up is published as **security research** under responsible disclosure principles.

- **No modified APK or proprietary code is distributed.** Only patch descriptions and technical analysis are provided. Readers must obtain the original app through official channels (Google Play Store) and apply any modifications themselves on their own devices.
- **No DRM, license checks, or payment systems were bypassed.** The patches exclusively target third-party advertising SDK entry points. All premium/VIP features, subscription flows, and in-app purchases remain completely intact and unmodified.
- **No servers were accessed without authorization.** The patches *prevent* network calls to ad servers — they don't access anything new.
- **No circumvention of access controls.** The advertising SDKs are third-party components, not access control mechanisms. Disabling them is functionally equivalent to using a network-level ad blocker (e.g., Pi-hole, AdGuard, DNS-based blocking) — a practice widely recognized as legal.
- **Reverse engineering for security research** is protected under the DMCA §1201(f) (US), EU Directive 2009/24/EC Article 6, and equivalent provisions in most jurisdictions when conducted for interoperability, error correction, or security analysis purposes.
- **This is not legal advice.** Consult a qualified attorney in your jurisdiction if you have concerns about the legality of modifying software on your own devices.

*This research was conducted for educational and security awareness purposes on a personally owned device. The author does not encourage or endorse software piracy, redistribution of copyrighted material, or circumvention of legitimate payment systems.*
