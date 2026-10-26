# Saboot System Architecture & Technical Defense Blueprint

> **Saboot** ("Proof") is an offline-first disaster crop-loss evidence layer designed for smallholder farmers and revenue/insurance officers. It inverts traditional post-disaster loss assessment: instead of waiting weeks for field inspectors while physical flood evidence recedes, Saboot starts an automated evidence clock the moment remote sensors detect an event, guides tamper-evident, hash-chained photo capture offline, and generates an open, cryptographically sealed 253-byte proof bundle verifiable without a backend server.

---

## 1. High-Level System Architecture

### 1.1 The Core Problem Solved

1. **The Fading Evidence Window**: Following flash floods or localized calamities, floodwaters recede within 24 to 72 hours. Silt dries, debris is cleared, and rotting crops are removed. By the time revenue patwaris or insurance loss assessors reach remote plots, physical proof of inundation has vanished, resulting in disputed or rejected claims.
2. **Asymmetric Burden & The 72-Hour Trap**: Under institutional schemes (such as the localized calamity provision of India's PMFBY or state disaster relief codes), farmers face a strict 72-hour intimation deadline. Yet smallholders are often unaware that the official clock has started, lack mobile connectivity in submerged areas, and possess no legally defensible method to record loss.
3. **Field Assessment Bottlenecks**: Surveyors are overwhelmed by thousands of simultaneous claims across an entire district, lacking objective criteria to prioritize field visits.
4. **The Saboot Solution**:
   - **Continuous Sensor Telemetry**: Detects flood conditions by combining automated rain gauge thresholds (Rule A), Sentinel-1 synthetic aperture radar (SAR) flood masks against crop baselines (Rule B), and river gauge/dam discharge rates with hydrodynamic travel lag (Rule C).
   - **Proactive Vernacular Warning**: Triggers an automated evidence clock at the *earliest* qualifying sensor signal, initiating native-language voice alerts (Hindi, Punjabi, English) before water recedes.
   - **Zero-Trust, Offline-First Capture**: Guides the farmer (or a village *Sarpanch* / CSC operator via Helper Mode) to take three geocoded photos. The client hashes the images using SHA-256, binds them into an append-only cryptographic chain, and signs the bundle locally using an in-browser ECDSA P-256 key via the WebCrypto API.
   - **Hyper-Compact, Serverless Transfer (Feature K)**: The entire proof bundle compresses to ~253 bytes (~338 Base64URL characters), formatted as a Base45 QR code or a URL `#k=` fragment that never touches a server over the wire.
   - **Ranked Worklist & Cryptographic Provenance**: Provides officers with an objective triage worklist based on evidence ambiguity, urgency, plot area, and integrity flags, ensuring Saboot acts as an immutable evidence layer—**never deciding money or claims**.

---

### 1.2 Tech Stack Breakdown & Engineering Justifications

| Layer | Technology | Engineering Justification & Trade-offs |
|---|---|---|
| **Frontend Framework** | **Preact 11.x** + `@preact/signals` | **Sub-3KB Footprint & Surgical DOM Reactivity**: React + ReactDOM add ~40KB+ of minified overhead. In remote 2G/3G rural networks, minimal JavaScript payload is critical to maintain First Contentful Paint (<1.5s) and Time-to-Interactive. Preact Signals enable fine-grained, direct DOM mutations for high-frequency simulation ticks (1x, 10x, 100x replay speeds) without triggering full Virtual DOM re-renders. |
| **Language & Tooling** | **TypeScript 7.x** + **Vite 8** | **Type-Level Cryptographic & Binary Rigor**: Big-endian binary serialization (`DataView`), bitwise packing (origin flags, photo kinds), SEC1 elliptic curve points, and rule math demand strict compile-time type verification. Vite provides Rollup-based tree shaking and sub-second Hot Module Replacement (HMR). |
| **Cryptography & Integrity** | **Native WebCrypto API** (`SubtleCrypto`) | **Zero-Dependency Asymmetric Cryptography**: Utilizes hardware-accelerated, browser-native primitives (`ECDSA P-256`, `SHA-256`). Eliminates heavy third-party crypto bundles (e.g., Forge, Web3, TweetNaCl), reducing attack surface and bundle size while executing in secure browser execution contexts. |
| **Local Storage / Persistence** | **IndexedDB** (`src/lib/db.ts`) | **Asynchronous Binary Storage & Offline Queueing**: `localStorage` is synchronous, blocks the main thread, and is capped at ~5MB of UTF-16 text. IndexedDB allows asynchronous storage of raw image `Blobs`, structured hash-chain event logs (`FocusEvent`), and outbox sync queues. Includes in-memory fallback for private browsing. |
| **Concurrency Control** | **Web Locks API** (`navigator.locks`) | **Cross-Tab Transactional Integrity**: Prevents race conditions and chain bifurcation if a farmer or helper opens multiple browser tabs simultaneously. Device log mutations acquire the `saboot-log` mutex before appending to the hash-chain. |
| **Transfer & Barcodes** | **Base45**, `uqr`, `jsqr` | **Maximum Density & Air-Gapped Transmission**: Base45 encodes binary payloads using the QR Code alphanumeric mode, achieving higher data density than Base64. `uqr` generates micro-footprint SVG/Canvas QR codes client-side (~0.1KB), and `jsqr` executes client-side computer vision scanning from real-time webcam video streams. |
| **Multi-Sensor Fusion Engine** | **Deterministic Pure Functions** (`src/sim/rules.ts`) | **Explainability & Non-Hallucinatory Logic**: Rule A (Rain + Terrain), Rule B (SAR change), and Rule C (Hydrologic stage) are pure mathematical functions with zero DOM, clock, or network side-effects. Shared identically between client runtime, simulation engine, and real-data validation harnesses. |
| **Cloud / Hosting** | **Static Edge PWA** (Netlify / Pages) + **Service Worker** | **Total Disaster-Resilience (No Runtime Server)**: Deployed as static immutable assets protected by strict Content Security Policies (`default-src 'self'`). The Service Worker (`sw.js`) caches all application assets, fonts, and sound bites on first load, enabling complete offline functionality during communications blackouts. |

---

### 1.3 The 4-Box Mental Model

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 THE 4-BOX MENTAL MODEL                                 │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 1. INGESTION   ──> Remote Telemetry (Rain Gauges, Radar SAR, River Levels)             │
│                    + Local Sensors (Camera VideoStream, Geolocation, Device Timestamp) │
│                                                                                        │
│ 2. PIPELINE    ──> Deterministic Multi-Rule Fusion Engine (Rules A, B, C)              │
│                    + Canvas 1280px Normalization + SHA-256 Digest Hashing              │
│                    + Append-Only Hash Chaining + ECDSA P-256 WebCrypto Signing         │
│                    + SEC1 Point Compression & Big-Endian Binary Encoding               │
│                                                                                        │
│ 3. STATE       ──> Browser IndexedDB (`kv`, `photos`, `outbox`)                        │
│                    + In-Memory Reactive Signals (`logs`, `t`, `model`, `kpis`)         │
│                    + Web Locks Concurrency Isolation (`navigator.locks`)               │
│                                                                                        │
│ 4. OUTPUT      ──> 253-Byte Base45 QR Code / `#k=` URL Fragment Payload                │
│                    + Ranked Officer Worklist (Ambiguity, Urgency, Area, Flags)         │
│                    + Tamper-Evident Verification Status ("Unchanged" vs "Altered")     │
│                    + Draft Girdawari Assessment List (CSV / JSON with Provenance)      │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### 1.4 End-to-End Data Flow Architecture

#### Mermaid Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Sensor as Satellites & Gauges
    actor Farmer as Farmer / Helper
    participant PWA as Saboot Client (Browser PWA)
    participant Crypto as WebCrypto & IndexedDB
    participant QR as Air-Gapped Code (~253 B)
    actor Officer as Revenue Officer / Patwari

    Note over Sensor, PWA: Phase 1: Event Detection & Clock Trigger
    Sensor->>PWA: Rain (>65mm/24h), Radar (water > expected), River (danger level)
    PWA->>PWA: Rule Engine (A, B, C) fires; starts clock at EARLIEST signal
    PWA->>Farmer: Automated Voice Alert (Punjabi / Hindi / English)

    Note over Farmer, Crypto: Phase 2: Offline Proof Capture & Sealing
    Farmer->>PWA: Open Camera (Wide shot, Close-up, Landmark)
    PWA->>PWA: Downscale to 1280px via HTML5 Canvas (JPEG 0.8)
    PWA->>PWA: Capture GPS Location + Device Timestamp
    PWA->>Crypto: Compute SHA-256(PhotoBytes) -> 32-byte digest
    Crypto->>Crypto: Append to Hash Chain: link[i] = SHA-256(link[i-1] || record[i])
    Crypto->>Crypto: Generate/Retrieve Device ECDSA P-256 KeyPair
    Crypto->>Crypto: Sign (Body || ChainHead || CompressedPubKey) via SubtleCrypto
    Crypto->>PWA: Persist Photos & Chain to IndexedDB ('saboot' DB)

    Note over PWA, QR: Phase 3: Binary Encoding & Payload Generation
    PWA->>QR: Pack big-endian binary struct (253 bytes)
    PWA->>QR: Encode to Base45 (QR Alphanumeric) / Base64URL (#k= link)

    Note over QR, Officer: Phase 4: Verification & Triage
    Officer->>QR: Scan QR code via Webcam or paste Base64URL code
    QR->>PWA: Parse Binary Payload -> Extract Timestamps, Hashes, PubKey, Sig
    PWA->>PWA: Recompute Hash Chain Head from Photo Digests & Seal Record
    PWA->>Crypto: Decompress P-256 point & subtle.verify(signature)
    alt Verification Succeeded
        PWA-->>Officer: "Integrity Verified: Unchanged since sealing"
        PWA->>Officer: Insert into Ranked Worklist (Ambiguity, Urgency, Area, Flags)
    else Verification Failed (Tampered Bit)
        PWA-->>Officer: "Verification Failed: Chain or Signature mismatch"
    end
```

#### ASCII Data Flow Diagram

```
[ RADAR / GAUGES / RAIN ]
           │
           ▼
┌─────────────────────────┐
│ Multi-Sensor Rules      │ ──> Starts Evidence Clock at Earliest Signal
│ (src/sim/rules.ts)      │ ──> Plays Vernacular Voice Alert
└─────────────────────────┘
           │
           ▼
┌─────────────────────────┐
│ Camera Viewfinder (PWA) │ ──> Captures 3 Photos (Wide, Close, Landmark)
│ (src/farmer/screens/    │ ──> HTML5 Canvas Resizes to 1280px JPEG
│  Capture.tsx)           │ ──> Fetches Geolocation (Lat/Lon)
└─────────────────────────┘
           │
           ▼
┌─────────────────────────┐
│ Cryptographic Sealer    │ ──> Hashes Photo Bytes: SHA-256
│ (src/k/payload.ts)      │ ──> Builds Hash-Chain: link_i = SHA-256(link_{i-1} || rec_i)
│                         │ ──> Signs: ECDSA P-256 over Body + Head + PubKey
└─────────────────────────┘
           │
           ▼
┌─────────────────────────┐
│ IndexedDB Storage       │ ──> Stores Raw Blobs in 'photos'
│ (src/lib/db.ts)         │ ──> Stores Chained Events in 'kv' via Web Locks
└─────────────────────────┘
           │
           ▼
┌─────────────────────────┐
│ 253-Byte Binary Payload │ ──> Base45 QR Code / URL #k= Fragment
│ (Air-Gapped Transfer)   │ ──> Zero Server Uploads (Images stay on phone!)
└─────────────────────────┘
           │ (Scanned via Webcam)
           ▼
┌─────────────────────────┐
│ Officer Triage Console  │ ──> Recomputes Chain Head + Verifies Signature
│ (src/dashboard/         │ ──> Computes Multi-Factor Priority Score (0-100)
│  Dashboard.tsx)         │ ──> Emits Draft Girdawari Assessment List (CSV/JSON)
└─────────────────────────┘
```

---

## 2. File-by-File Blueprint

### 1. `src/k/payload.ts` — Cryptographic Evidence Engine
- **Purpose**: Implements Feature K: binary serialization, hash-chain generation, SEC1 elliptic curve point compression, and ECDSA P-256 signature generation and verification for tamper-proof evidence packets. It ensures an evidence payload packs into ~253 big-endian bytes (~338 Base64URL characters), small enough for high-density QR scanning without server interaction.
- **Key Entry Functions**:
  - `makeCode(input: KInput, key: CryptoKeyPair): Promise<string>`: Encodes photo metadata, timestamps, and coordinates into binary, computes the hash-chain head, signs the payload, and outputs Base64URL text.
  - `verifyCode(code: string): Promise<KResult>`: Parses the Base64URL payload, verifies structural constraints, recomputes the SHA-256 chain head, decompresses the public key, and verifies the ECDSA signature.
  - `chainHead(b: Uint8Array): Promise<Uint8Array>`: Iteratively reduces photo and seal records into an immutable 32-byte SHA-256 chain root.
  - `decompress(c: Uint8Array): Uint8Array`: Performs point decompression over the NIST P-256 prime field ($\mathbb{F}_P$) to reconstruct the uncompressed 65-byte EC point for WebCrypto ingestion.

### 2. `src/sim/rules.ts` — Deterministic Event Detection Engine
- **Purpose**: Defines pure, deterministic rule functions combining meteorological, hydrologic, and radar sensor inputs to establish the official start of a disaster event without relying on black-box AI. It enforces the conservative principle that the evidence clock starts at the earliest qualifying signal, preventing farmers from being penalized by sensor lag.
- **Key Entry Functions**:
  - `ruleA(r24, lowLying, th): RuleAResult`: Evaluates 24-hour cumulative rainfall against IMD heavy-rain thresholds combined with DEM topographic depression flags.
  - `ruleB(i, th, latencyH): RuleBResult`: Calculates Sentinel-1 SAR radar change detection against crop-specific baselines (accounting for standing water in irrigated paddy).
  - `ruleC(i, gauges, th): RuleCResult`: Evaluates upstream river stage and dam discharge series against danger thresholds, factoring in hydrodynamic flood wave lag.
  - `combine(a, b, c, baselineWetCrop): Combined`: Computes multi-sensor consensus, sets event start time to the minimum timestamp among all valid signals, and outputs a confidence grade (`high`, `medium`, `low`).

### 3. `src/farmer/focus.ts` — Append-Only Hash-Chained Device Store
- **Purpose**: Manages the local farmer state, recording all device actions (photo snaps, location fixes, consent grants, and sealing) into an append-only SHA-256 hash-chained log stored in IndexedDB. It also isolates multi-farmer sessions during "Helper Mode", ensuring each farmer's consent and evidence remain bound to an independent, tamper-evident cryptographic chain.
- **Key Entry Functions**:
  - `addShot(kind, blob, w, h, geo, synthetic, helper): Promise<void>`: Hashes a captured photo, writes the binary blob to IndexedDB, and commits an append-only `FocusEvent` link.
  - `seal(queued): Promise<string>`: Finalizes the farmer's capture session, computes the seal record, triggers cryptographic signing, and stages the packet for transmission.
  - `startHelperSession(s: HelperSession): Promise<void>`: Configures an isolated operational context for a village helper (Sarpanch / CSC operator) while preserving farmer-specific consent requirements.
  - `loadFocus(): Promise<void>`: Hydrates the reactive Preact signal state from IndexedDB on application initialization.

### 4. `src/farmer/screens/Capture.tsx` — Offline Camera & Viewfinder Pipeline
- **Purpose**: Handles hardware media stream negotiation, real-time video preview, client-side canvas downsampling, and geolocation acquisition for field evidence collection. It enforces strict provenance tagging (`live`, `sim`, `hand`) so that synthetic testing or manual entries cannot masquerade as genuine camera captures.
- **Key Entry Functions**:
  - `Viewfinder({ onShot, onFail, onClose })`: Mounts the HTML5 `<video>` element, binds the rear-facing camera (`facingMode: "environment"`), and draws video frames to an offscreen canvas at a normalized 1280px resolution.
  - `Capture()`: Orchestrates the 3-step capture workflow (Wide shot, Close-up, Landmark), initiates background geolocation pre-warming, and dispatches blobs to `addShot()`.

### 5. `src/dashboard/Triage.tsx` & `src/dashboard/Dashboard.tsx` — Officer Assessment & Triage Console
- **Purpose**: Powers the revenue officer and loss assessor console, transforming raw evidence packets into an objectively ranked inspection worklist and publishing village-by-village draft girdawari lists. It prioritizes field staff visits without ever rendering automated claim verdicts or displaying rupee compensation figures.
- **Key Entry Functions**:
  - `whyParts(r: Row)`: Deconstructs a plot's priority into an explainable, 4-factor score: 35% Evidence Ambiguity + 30% Urgency (time elapsed in window) + 15% Plot Area + 20% Integrity Flags.
  - `csv(list: Row[])`: Generates provably auditable CSV exports of the assessment worklist, stamped with rule versions, scenario SHA-256 hashes, and execution timestamps.
  - `Dashboard()`: Coordinates the multi-view interface (Claims Map, Triage Queue, Draft List, Impact Metrics) and listens for imported QR packets from webcams or peer devices.

---

## 3. The Hackathon Judge Defense Cheat Sheet

### 3.1 The 12-Year-Old Analogy: "The Tamper-Proof Wax Seal"

> *"Imagine you take a photograph of your flooded bicycle. If you email that photo to someone days later, they might ask: 'Did this happen today, or was this taken last year? Did you edit it in Photoshop?'*
>
> *Now imagine that the moment you snap the photo, your phone puts the picture into a metal lockbox, stamps the exact GPS coordinates and time onto it, and drips hot wax over the lock with a unique signet ring that only your phone owns. You don't send the heavy bicycle or the photo across the village; you only show the tiny wax seal.*
>
> *If an inspector looks at the wax seal, they can instantly verify that the box was sealed at that exact hour, in that exact field, and that nobody has opened or changed a single speck of dust inside. If anyone tampers with even one pixel of the photo, the wax seal shatters into pieces. That is Saboot."*

---

### 3.2 Top 5 Toughest Technical Questions & Senior-Engineer Scripts

#### Q1 (Security / Anti-Spoofing): "What prevents a farmer or bad actor from spoofing GPS coordinates, device time, or taking a photo of an old flood on a tablet screen?"
> **Script**:
> *"That is the exact distinction between **data integrity** and **ground truth**. Saboot is architected around cryptographic integrity: we prove mathematically that a specific set of bytes, timestamps, and coordinates have not been altered since the moment of signing.
>
> To mitigate spoofing at the capture boundary:
> 1. **Time-Skew Detection**: The packet records the unverified client clock (`deviceMs`) alongside the remote sensor's event timestamp. When the officer's console imports the packet, it stamps the officer's wall-clock receipt time. Packets displaying anomalous drift between event start, sealing time, and receipt time are flagged with an integrity penalty in our triage algorithm.
> 2. **Multi-Shot Spatial Consistency**: We require three distinct perspectives: a wide landscape shot, a close-up crop view, and a permanent village landmark. Forging all three under consistent lighting and angle constraints requires sophisticated physical staging.
> 3. **Provenance Flags**: The format embeds origin bits directly into the signed body (`live` camera vs `sim` vs `hand`). A modified APK or developer-mode spoof could lie to the client, but it cannot alter the record post-signing.
>
> Crucially, Saboot does not grant automated payouts. It ranks inspection priorities. If an integrity flag is raised or photos appear suspect, the officer's worklist prioritizes that plot for a mandatory physical visit."*

---

#### Q2 (Architecture / Scalability): "Why build this as an entirely client-side, serverless application instead of using a standard cloud backend with PostgreSQL and microservices?"
> **Script**:
> *"Because in a Category 4 cyclone or a massive monsoon flood, centralized cloud backends fail at the physical layer: cell towers lose grid power, fiber backhauls are severed, and rural bandwidth collapses to zero.
>
> If an evidence system requires an active HTTP POST connection to upload multi-megabyte JPEG images to AWS S3, the system is dead on arrival during the exact 72-hour window when evidence must be captured.
>
> By utilizing a serverless Progressive Web App architecture:
> 1. **Zero-Connectivity Survival**: The Service Worker serves the entire application and cryptographic engine directly from local cache.
> 2. **Zero Ingress Bandwidth**: Photos remain on the device's IndexedDB storage. The only payload that moves off the phone is our 253-byte signed binary struct. That fits into a single Base45 QR code, allowing peer-to-peer air-gapped transmission to an officer's laptop webcam even with zero cellular coverage.
> 3. **Infinite Scaling & Zero Cloud OpEx**: During an event affecting 100,000 farmers simultaneously, our backend compute cost is zero dollars, and our database throughput is zero IOPS. The computational load of hashing and signing is distributed across 100,000 client devices at the edge."*

---

#### Q3 (Edge Cases / ML Perception): "Radar SAR cannot penetrate dense crop canopies, and irrigated paddy fields are flooded intentionally. How does your detection logic avoid catastrophic false positive rates?"
> **Script**:
> *"This is why we explicitly rejected a black-box end-to-end computer vision model in favor of a deterministic, multi-sensor fusion engine with crop-aware baselines.
>
> In `src/sim/rules.ts`, Rule B specifically implements baseline excess subtraction:
> $$\text{Excess} = \frac{\text{Observed} - \text{Expected}}{1 - \text{Expected}}$$
> For dry-standing crops like cotton or maize, the expected baseline water is zero. For kharif paddy, which is intentionally submerged during transplanting, the baseline parameter represents the expected standing water fraction for that specific phenological stage. Only SAR backscatter drops exceeding that baseline contribute to flood detection.
>
> Furthermore, Rule B never acts in isolation. To attain high confidence, the system requires independent multi-sensor corroboration: Rule A (IMD rainfall crossing 65mm in 24 hours on topographically depressed plots) or Rule C (Central Water Commission hydrologic river stage crossing danger levels with travel lag). If only radar fires on a known wet crop, the event confidence is explicitly downgraded to `low`, preventing false alarm cascades."*

---

#### Q4 (Data Synchronization & Concurrency): "If a village helper captures evidence for 50 different farmers on a single phone while offline, how do you prevent data loss, storage exhaustion, and race conditions?"
> **Script**:
> *"We engineered Helper Mode specifically for rural shared-device realities through three architectural safeguards:
>
> 1. **Isolated Hash-Chains per Farmer**: A helper does not append multiple farmers into one shared log. Each farmer session creates an independent, isolated append-only chain in IndexedDB, initiated by an explicit cryptographic consent record (`FocusEvent` of type `consent`). A farmer's proof is legally independent and never co-mingled with their neighbor's data.
> 2. **Web Locks Mutex**: In browser environments where multiple tabs or service workers might attempt background storage writes, all IndexedDB mutations run through the Web Locks API (`navigator.locks.request('saboot-log')`), serializing transactions and eliminating chain bifurcation.
> 3. **Two-Stage Bandwidth Synchronization**: When the helper eventually reaches an area with marginal 2G connectivity, Saboot executes a two-stage sync. Stage 1 transmits only the 32-byte SHA-256 fingerprint packet, fixing the official server receipt timestamp. Stage 2 queues the heavy image blobs in an IndexedDB outbox, transmitting them lazily in the background only when high-throughput Wi-Fi or 4G is detected."*

---

#### Q5 (Technology Justifications): "Why did you choose Preact and the WebCrypto API instead of React and established blockchain/Web3 crypto libraries like ethers.js or TweetNaCl?"
> **Script**:
> *"Every dependency in this codebase was audited against strict rural operational constraints: bundle size, memory footprint, and cryptographic compliance.
>
> 1. **Preact over React**: React with ReactDOM introduces approximately 42KB of minified, gzipped JavaScript before a single line of application code executes. Preact provides full Virtual DOM compatibility and hooks at just 3KB. Coupled with `@preact/signals`, we achieve direct, reactive DOM text updates for our simulation clock without incurring Virtual DOM diffing overhead on low-end budget smartphones with $1GB of RAM.
> 2. **WebCrypto over External Crypto Libraries**: Libraries like `ethers.js` or `tweetnacl` add anywhere from 30KB to 200KB to the bundle and execute cryptographic primitives in user-space JavaScript, which is vulnerable to side-channel attacks and execution throttling. The native `globalThis.crypto.subtle` API is baked into the browser engine in C++, executes in constant time with hardware acceleration, and adds exactly zero bytes to our network payload.
> 3. **NIST P-256 Standardization**: We chose ECDSA over the P-256 curve (secp256r1) because it is natively supported by W3C WebCrypto across every modern mobile browser (Chrome for Android, Safari, Firefox), whereas Ed25519 or secp256k1 support in WebCrypto remains fragmented or nonexistent without third-party WASM shims."*

---

## 4. Git Pull Request Workflow (Antigravity Terminal Commands)

To commit this architecture documentation to the repository and open a professional Pull Request:

```bash
# 1. Create and switch to a dedicated documentation branch
git checkout -b docs/architecture-and-analysis

# 2. Stage the architecture documentation file
git add ARCHITECTURE.md

# 3. Create a conventional commit
git commit -m "docs: add system architecture, data flow diagrams, and judge reference"

# 4. Push the branch to GitHub
git push -u origin docs/architecture-and-analysis

# 5. Create the Pull Request using GitHub CLI
gh pr create --title "docs: comprehensive architecture and data flow documentation" --body "Analyzed repository structure, mapped execution flow, and documented technical architecture for hackathon evaluation."
```

