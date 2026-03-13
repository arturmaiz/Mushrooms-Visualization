# Mushroom Body Explorer — PWA Implementation Plan

## Context

The user wants a Progressive Web App (PWA) where:
- A cool mushroom logo appears as an icon on users' phone/browser homescreens
- Tapping it opens an interactive 3D human body explorer
- Users can switch between three mushroom profiles (Cordyceps, Lion's Mane, Reishi)
- Each profile highlights specific organ systems with color-coded 3D animations
- A sidebar explains health benefits per mushroom
- The app is embeddable for publishers (wellness platforms, health sites)

Repository starts empty (only LICENSE). Pure HTML/CSS/JS + Three.js via CDN — no build tools.

---

## File Structure

```
Mushrooms-Visualization/
├── index.html              # Single-page app shell, importmap, PWA registration
├── manifest.json           # PWA manifest (homescreen install, icons)
├── service-worker.js       # Offline cache-first strategy
├── css/
│   └── styles.css          # Dark theme, per-mushroom CSS vars, responsive layout
├── js/
│   ├── app.js              # Bootstrap, event wiring, state machine
│   ├── scene.js            # Three.js scene, camera, renderer, lights
│   ├── body.js             # Procedural human body + organ mesh builder
│   ├── organs.js           # Organ highlight controller (tweens, activation)
│   ├── animations.js       # Main loop, per-mushroom animation drivers, particles
│   ├── mushrooms.js        # Data: profiles, colors, organ targets, benefits
│   └── transitions.js      # Cross-fade logic between mushroom profiles
└── icons/
    ├── icon-generate.html  # Dev utility: renders SVG → exports PNG sizes
    ├── icon-72.png
    ├── icon-96.png
    ├── icon-128.png
    ├── icon-144.png
    ├── icon-152.png        # Used for <link rel="apple-touch-icon">
    ├── icon-192.png
    ├── icon-384.png
    ├── icon-512.png
    └── maskable-512.png    # Android adaptive icon (15% safe zone padding)
```

---

## PWA Setup

### `manifest.json`
```json
{
  "name": "Mushroom Body Explorer",
  "short_name": "MycoBody",
  "description": "Interactive 3D visualization of medicinal mushroom effects on the human body",
  "start_url": "/index.html?source=pwa",
  "display": "standalone",
  "orientation": "any",
  "background_color": "#0a0a0f",
  "theme_color": "#ff6b35",
  "icons": [
    { "src": "icons/icon-72.png",      "sizes": "72x72",   "type": "image/png" },
    { "src": "icons/icon-96.png",      "sizes": "96x96",   "type": "image/png" },
    { "src": "icons/icon-128.png",     "sizes": "128x128", "type": "image/png" },
    { "src": "icons/icon-144.png",     "sizes": "144x144", "type": "image/png" },
    { "src": "icons/icon-152.png",     "sizes": "152x152", "type": "image/png" },
    { "src": "icons/icon-192.png",     "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "icons/icon-384.png",     "sizes": "384x384", "type": "image/png" },
    { "src": "icons/icon-512.png",     "sizes": "512x512", "type": "image/png" },
    { "src": "icons/maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ],
  "categories": ["health", "education"]
}
```

### `index.html` Key Tags
```html
<link rel="manifest" href="manifest.json">
<link rel="apple-touch-icon" href="icons/icon-152.png">
<meta name="theme-color" content="#ff6b35" id="theme-meta">

<script type="importmap">
{
  "imports": {
    "three": "https://cdnjs.cloudflare.com/ajax/libs/three.js/r134/three.module.min.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.134.0/examples/jsm/"
  }
}
</script>
<script type="module" src="js/app.js"></script>
```

### `service-worker.js`
```
CACHE_NAME = 'myco-body-v1'
PRECACHE_URLS = [ '/', '/index.html', '/manifest.json', '/css/styles.css',
                  '/js/app.js', '/js/scene.js', '/js/body.js', '/js/organs.js',
                  '/js/animations.js', '/js/mushrooms.js', '/js/transitions.js' ]
install  → cache.addAll(PRECACHE_URLS)
activate → delete caches with key !== CACHE_NAME
fetch    → cache-first; on miss: fetch + cache.put
```
Three.js CDN is NOT precached (cross-origin); app degrades gracefully offline (shows 2D message).

---

## HTML DOM Structure

```
<body class="theme-cordyceps">
  <header>
    <h1>Mushroom Body Explorer</h1>
    <nav id="mushroom-switcher">
      <button data-mushroom="cordyceps" class="active">Cordyceps</button>
      <button data-mushroom="lionsmane">Lion's Mane</button>
      <button data-mushroom="reishi">Reishi</button>
    </nav>
  </header>
  <main>
    <aside id="sidebar">
      <div id="mushroom-name"></div>
      <div id="mushroom-tagline"></div>
      <ul id="benefits-list"></ul>
      <div id="targeted-systems"></div>
      <div id="embed-cta">...</div>      <!-- hidden when embedded in iframe -->
    </aside>
    <section id="canvas-container">
      <canvas id="body-canvas"></canvas>
      <div id="loading-overlay">Loading 3D Model...</div>
    </section>
  </main>
  <div id="transition-overlay"></div>    <!-- full-screen flash on mushroom switch -->
</body>
```

---

## Three.js Scene (`js/scene.js`)

```js
renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true })
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
renderer.shadowMap.enabled = true
renderer.shadowMap.type = THREE.PCFSoftShadowMap
renderer.toneMapping = THREE.ACESFilmicToneMapping
renderer.toneMappingExposure = 1.2

camera = new THREE.PerspectiveCamera(45, aspect, 0.1, 100)
camera.position.set(0, 1.6, 4.5)

controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true
controls.dampingFactor = 0.05
controls.minDistance = 2
controls.maxDistance = 8
controls.target.set(0, 1.0, 0)

ambientLight  = new THREE.AmbientLight(0x111122, 0.6)
mainLight     = new THREE.DirectionalLight(0xffffff, 1.0)   // pos (3,5,3), castShadow
rimLight      = new THREE.DirectionalLight(0x4444ff, 0.4)   // pos (-3,3,-2), cool back rim
organSpotlight = new THREE.SpotLight(...)  // color+intensity driven per-mushroom
```

---

## Procedural Human Body (`js/body.js`)

### Body Part Geometries (all named meshes in `bodyGroup`)

| Part         | Geometry                              | Position (local)    |
|--------------|---------------------------------------|---------------------|
| Head         | SphereGeometry(0.25)                  | (0, 2.25, 0)        |
| Neck         | CylinderGeometry(0.1, 0.12, 0.15)     | (0, 2.0, 0)         |
| Upper Torso  | CylinderGeometry(0.38, 0.32, 0.7)     | (0, 1.6, 0)         |
| Lower Torso  | CylinderGeometry(0.32, 0.28, 0.4)     | (0, 1.2, 0)         |
| Pelvis       | CylinderGeometry(0.28, 0.22, 0.2)     | (0, 0.95, 0)        |
| L/R Upper Arm| CylinderGeometry(0.09, 0.09, 0.4)     | (±0.52, 1.75, 0)    |
| L/R Lower Arm| CylinderGeometry(0.07, 0.07, 0.38)    | (±0.58, 1.3, 0)     |
| L/R Upper Leg| CylinderGeometry(0.13, 0.13, 0.5)     | (±0.18, 0.6, 0)     |
| L/R Lower Leg| CylinderGeometry(0.09, 0.09, 0.48)    | (±0.18, 0.05, 0)    |
| L/R Foot     | BoxGeometry(0.12, 0.08, 0.22)          | (±0.18, -0.2, 0.05) |

**Base body material**: `MeshStandardMaterial({ color: 0x2a2a3a, roughness: 0.7, transparent: true, opacity: 0.82 })`

### Organ Overlay Meshes (inside body, `depthWrite: false`, default `opacity: 0`)

| Organ            | Geometry                           | Position            |
|------------------|------------------------------------|---------------------|
| lung-left        | SphereGeometry(0.13) scaleX=0.7    | (-0.14, 1.55, 0.05) |
| lung-right       | SphereGeometry(0.13) scaleX=0.7    | (0.14, 1.55, 0.05)  |
| heart            | SphereGeometry(0.09)               | (-0.05, 1.6, 0.05)  |
| brain            | SphereGeometry(0.22) scaleY=0.85   | (0, 2.25, 0)        |
| neural-spine     | CylinderGeometry(0.04, 0.04, 1.1)  | (0, 1.55, -0.12)    |
| kidney-left/right| SphereGeometry(0.07)               | (±0.12, 1.35, -0.05)|
| gut              | TorusGeometry(0.18, 0.06)          | (0, 1.3, 0.05)      |
| liver            | SphereGeometry(0.1) scaleX=1.4     | (0.12, 1.45, 0.05)  |

All organ meshes use `MeshStandardMaterial` with `emissive` color and `emissiveIntensity` animated.
`depthWrite: false` prevents z-fighting between overlapping transparent organ meshes.

---

## Mushroom Data (`js/mushrooms.js`)

```js
export const MUSHROOMS = {
  cordyceps: {
    id: 'cordyceps',
    displayName: 'Cordyceps',
    tagline: 'Oxygen, Energy, Endurance',
    accentColor: '#ff6b35',
    glowColor: '#ffaa44',
    particleColor: '#ff8c42',
    themeClass: 'theme-cordyceps',
    targetOrgans: [
      { id: 'lung-left',    intensity: 1.0, animStyle: 'breathe' },
      { id: 'lung-right',   intensity: 1.0, animStyle: 'breathe' },
      { id: 'heart',        intensity: 0.7, animStyle: 'pulse' },
      { id: 'kidney-left',  intensity: 0.5, animStyle: 'pulse' },
      { id: 'kidney-right', intensity: 0.5, animStyle: 'pulse' },
    ],
    benefits: [
      'Increases ATP energy production',
      'Improves oxygen utilization (VO2 max)',
      'Supports adrenal and kidney function',
      'Enhances athletic endurance',
      'Regulates cortisol stress response',
    ],
    animationDriver: 'energize',
  },
  lionsmane: {
    id: 'lionsmane',
    displayName: "Lion's Mane",
    tagline: 'Cognition, Nerve Growth, Clarity',
    accentColor: '#7eb8f7',
    glowColor: '#a0d4ff',
    particleColor: '#5c9fe0',
    themeClass: 'theme-lionsmane',
    targetOrgans: [
      { id: 'brain',        intensity: 1.0, animStyle: 'neural-fire' },
      { id: 'neural-spine', intensity: 0.8, animStyle: 'wave-down' },
    ],
    benefits: [
      'Stimulates Nerve Growth Factor (NGF)',
      'Supports memory and concentration',
      'Reduces brain fog and mental fatigue',
      'May protect against neurodegenerative decline',
      'Enhances myelin sheath regeneration',
    ],
    animationDriver: 'neural',
  },
  reishi: {
    id: 'reishi',
    displayName: 'Reishi',
    tagline: 'Calm, Sleep & Immunity',
    accentColor: '#9b59b6',
    glowColor: '#c39bd3',
    particleColor: '#8e44ad',
    themeClass: 'theme-reishi',
    targetOrgans: [
      { id: 'neural-spine', intensity: 0.9, animStyle: 'slow-wave' },
      { id: 'brain',        intensity: 0.5, animStyle: 'calm-pulse' },
      { id: 'liver',        intensity: 0.7, animStyle: 'steady-glow' },
      { id: 'gut',          intensity: 0.6, animStyle: 'steady-glow' },
    ],
    benefits: [
      'Activates parasympathetic nervous system',
      'Promotes deep sleep and circadian rhythm',
      'Powerful immunomodulating beta-glucans',
      'Liver protection and detox support',
      'Reduces anxiety via triterpene compounds',
    ],
    animationDriver: 'calm',
  },
};
```

---

## Organ Controller (`js/organs.js`)

```
OrganController class:
  organMeshMap: Map<id, THREE.Mesh>       // populated by body.js during construction
  activeTweens: Map<id, TweenState>

  registerOrgan(id, mesh)                 // called from body.js
  activateProfile(mushroomData, duration) // entry point for mushroom switch
    → fade out all organs NOT in new profile (duration/2)
    → fade in all organs IN new profile (duration)
  fadeMeshIn(id, targetIntensity, ms)     // lerp opacity 0→0.6, emissiveIntensity 0→target
  fadeMeshOut(id, ms)                     // reverse
  update(delta, elapsed)                  // advance tweens + dispatch animStyle functions
```

### Animation Style Functions (called every frame per active organ)

| Style         | Behavior                                                              |
|---------------|-----------------------------------------------------------------------|
| `breathe`     | `scale.y = 1 + 0.08 * sin(elapsed * 1.4)` — lung expansion rhythm   |
| `pulse`       | `emissiveIntensity = base + 0.3 * abs(sin(elapsed * 2.2))` — heartbeat|
| `neural-fire` | Random emissive spikes via noise, cycling neuron spark PointLight     |
| `wave-down`   | Traveling emissive peak down spine Y axis at 0.8s period             |
| `slow-wave`   | Low frequency sine, period ~4s                                        |
| `calm-pulse`  | 0.3–0.5 Hz glow, very subtle                                          |
| `steady-glow` | Constant emissive ±5% breathing                                       |

---

## Animation Drivers (`js/animations.js`)

### Main Loop
```js
export function startAnimationLoop(scene, camera, renderer, organController, particleSystem) {
  const clock = new THREE.Clock();
  function tick() {
    requestAnimationFrame(tick);
    const delta = clock.getDelta();
    const elapsed = clock.getElapsedTime();
    controls.update();
    organController.update(delta, elapsed);
    particleSystem.update(delta, elapsed);
    renderer.render(scene, camera);
  }
  tick();
}
```

### Particle System
- `~200 Points` per mushroom, distributed in cylinder around body (r=0.8, h=3.0, y-center=1.2)
- `AdditiveBlending`, `depthWrite: false` → additive brightening = bioluminescent look
- Per-driver behavior:
  - `energize` (Cordyceps): fast upward rise 1.2 units/s
  - `neural` (Lion's Mane): arc toward head region
  - `calm` (Reishi): lazy spiral drift

### Per-Mushroom Driver Behaviors

| Driver    | Heart | Lungs | Camera       | Particles       |
|-----------|-------|-------|--------------|-----------------|
| `energize`| 1.2Hz | 0.25Hz| slow rotate  | fast rise       |
| `neural`  | —     | —     | static       | arc to head     |
| `calm`    | —     | —     | gentle sway  | slow spiral     |

Lion's Mane `neural-fire`: random `PointLight` inside brain flashes 80ms on, every 300–800ms random interval.

---

## Transition System (`js/transitions.js`)

**Total wall time: ~700ms**

| Phase       | Duration | Actions                                                         |
|-------------|----------|-----------------------------------------------------------------|
| Fade-out    | 300ms    | `#transition-overlay` → opacity 1; organs fade to 0            |
| Swap        | 0ms      | Update body class → CSS vars shift; update sidebar DOM content  |
| Fade-in     | 400ms    | Organs fade in; overlay fades out; particle color lerps 600ms   |

- `<meta id="theme-meta">` `content` updated to new `accentColor`
- `isTransitioning = true` in `app.js` blocks re-clicks during transition

---

## CSS Design (`css/styles.css`)

### CSS Custom Properties + Theme Classes
```css
:root {
  --bg-primary: #0a0a0f;
  --bg-secondary: #12121a;
  --bg-card: #1a1a28;
  --text-primary: #e8e8f0;
  --text-secondary: #9090b0;
  --border-color: #2a2a40;
  --accent: #ff6b35;
  --accent-glow: rgba(255, 107, 53, 0.25);
  --btn-active-bg: #ff6b35;
}
body.theme-lionsmane { --accent: #7eb8f7; --accent-glow: rgba(126,184,247,0.2); ... }
body.theme-reishi    { --accent: #9b59b6; --accent-glow: rgba(155,89,182,0.2);  ... }
```

### Responsive Grid
```css
body  { display: grid; grid-template-rows: 60px 1fr; height: 100dvh; }
main  { display: grid; grid-template-columns: 320px 1fr; }
@media (max-width: 768px) {
  main { grid-template-columns: 1fr; grid-template-rows: 1fr 240px; }
}
```

### Switcher Buttons
```css
#mushroom-switcher button { border-radius: 20px; border: 1px solid var(--border-color); }
#mushroom-switcher button.active { background: var(--btn-active-bg); box-shadow: 0 0 12px var(--accent-glow); }
```

### Embeddable Support
```css
.embedded #embed-cta { display: none; }
```
`app.js` adds class `embedded` to `<body>` when `window.self !== window.top`.
Publisher embed: `<iframe src="index.html?embed=1&mushroom=reishi" width="900" height="600">`
postMessage API exposed: `iframe.contentWindow.postMessage({ mushroom: 'reishi' }, '*')`

---

## Icon Generation (`icons/icon-generate.html`)

Developer-only utility (never served by PWA). Renders mushroom SVG to `<canvas>` → download buttons for each size.

**Icon Design**: Stylized mushroom silhouette with multicolor gradient (orange → blue → violet diagonal sweep) on `#0a0a0f` dark circle background. Subtle `shadowBlur` glow ring around cap. Three mycelium dots below stem.

**`maskable-512.png`**: Same graphic with 15% padding on all sides (Android safe zone). Background fills full 512×512.

Required sizes: 72, 96, 128, 144, 152, 192, 384, 512, maskable-512.

---

## Implementation Order

1. `icons/icon-generate.html` → export all PNG icons
2. `manifest.json` + `service-worker.js` → PWA shell
3. `index.html` + `css/styles.css` → full layout, theme classes (no JS, test in DevTools)
4. `js/scene.js` → Three.js scene (verify rotating cube in canvas)
5. `js/body.js` → procedural body geometry (verify anatomical form)
6. `js/mushrooms.js` → data constants
7. `js/organs.js` → organ highlight controller
8. `js/animations.js` → main loop + animation drivers + particles
9. `js/transitions.js` → cross-fade polish
10. `js/app.js` → bootstrap all modules, wire switcher buttons, URL param handling
11. Embed mode: `?embed=1` detection + postMessage API

---

## Verification

1. Open `index.html` → 3D body visible, rotating, no console errors
2. Click each mushroom → correct organs light up, sidebar updates, distinct animation style
3. Mobile Chrome → "Add to Home Screen" → icon appears on homescreen
4. Tap homescreen icon → opens in standalone mode (no browser chrome)
5. Offline (DevTools → Network: offline) → reload → app works (service worker)
6. `<iframe src="index.html?embed=1&mushroom=lionsmane">` → Lion's Mane pre-selected, CTA hidden
7. Lighthouse PWA audit → installability ✓, offline ✓, icons ✓
