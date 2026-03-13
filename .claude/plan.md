# Mushroom Body Explorer — PWA Implementation Plan

## Context

The user wants a Progressive Web App (PWA) where:
- A cool mushroom logo appears as an icon on users' phone/browser homescreens
- Tapping it opens an interactive 3D human body explorer
- Users can switch between three mushroom profiles (Cordyceps, Lion's Mane, Reishi)
- Each profile highlights specific organ systems with color-coded 3D animations
- A sidebar explains health benefits per mushroom
- The app is embeddable for publishers (wellness platforms, health sites)

The repository is currently empty (only a LICENSE file). Everything is built from scratch using pure HTML/CSS/JS + Three.js via CDN — no build tools required.

---

## File Structure

```
Mushrooms-Visualization/
├── index.html              # Main app shell (PWA entry point)
├── manifest.json           # PWA manifest (homescreen install, icons)
├── service-worker.js       # Offline caching
├── css/
│   └── styles.css          # Dark theme, mushroom color schemes, animations
├── js/
│   ├── app.js              # Main controller: mushroom switching, UI logic
│   ├── scene.js            # Three.js scene setup, camera, lighting, renderer
│   ├── body.js             # Procedural human body + organ meshes
│   ├── mushrooms.js        # Mushroom data: colors, targeted organs, benefits
│   └── transitions.js      # Smooth animation transitions between profiles
└── icons/
    ├── icon-192.svg        # SVG mushroom logo (source)
    ├── icon-192.png        # PNG for PWA manifest
    └── icon-512.png        # PNG for PWA manifest
```

---

## PWA Setup

### `manifest.json`
```json
{
  "name": "Mushroom Body Explorer",
  "short_name": "ShroomX",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0a0a0f",
  "theme_color": "#1a1a2e",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### `service-worker.js`
- Cache-first strategy for all assets on install
- Serves cached content when offline

### Icons
- SVG mushroom cap drawn with `<path>` elements — a stylized cap + stem + gills
- Inline JavaScript uses Canvas API to rasterize SVG → PNG at 192×512px
- Alternatively: embed base64 PNG directly in manifest (canvas-drawn at build time)

---

## Three.js Scene (`scene.js`)

```
Scene setup:
- Renderer: WebGLRenderer, antialias, transparent background
- Camera: PerspectiveCamera, fov 45, positioned at (0, 0, 5)
- OrbitControls (CDN): enable damping, limit polar angle
- Lighting:
    - AmbientLight (dim, #0a0a1a)
    - PointLight (front, changes color per mushroom)
    - RimLight (back, subtle white glow)
- Post-processing: UnrealBloomPass for organ glow effect (via CDN EffectComposer)
- Animation loop: requestAnimationFrame, slow body rotation (0.002 rad/frame)
```

---

## Procedural Human Body (`body.js`)

The body is built from Three.js primitive geometries merged/positioned as named meshes:

| Mesh name        | Geometry                          | Position           |
|------------------|-----------------------------------|--------------------|
| `body-torso`     | BoxGeometry(1, 1.4, 0.5)          | (0, 0, 0)          |
| `body-head`      | SphereGeometry(0.35)              | (0, 1.1, 0)        |
| `body-neck`      | CylinderGeometry(0.12, 0.15, 0.3) | (0, 0.75, 0)       |
| `body-pelvis`    | BoxGeometry(0.9, 0.4, 0.45)       | (0, -0.85, 0)      |
| `arm-left`       | CylinderGeometry(0.1, 0.08, 1.1)  | (-0.65, 0, 0)      |
| `arm-right`      | CylinderGeometry(0.1, 0.08, 1.1)  | (0.65, 0, 0)       |
| `leg-left`       | CylinderGeometry(0.13, 0.1, 1.3)  | (-0.25, -1.55, 0)  |
| `leg-right`      | CylinderGeometry(0.13, 0.1, 1.3)  | (0.25, -1.55, 0)   |
| **`organ-brain`**   | SphereGeometry(0.28)           | (0, 1.1, 0)        |
| **`organ-lungs`**   | Two EllipsoidGeometry          | (±0.22, 0.1, 0)    |
| **`organ-heart`**   | SphereGeometry(0.12)           | (-0.12, 0.2, 0.1)  |
| **`organ-nervous`** | TubeGeometry along spine       | (0, 0→-0.8, 0)     |
| **`organ-gut`**     | TorusGeometry(0.18, 0.07)      | (0, -0.3, 0)       |
| **`organ-adrenal`** | Two SphereGeometry(0.06)       | (±0.12, -0.1, 0)   |

All organ meshes start as `MeshStandardMaterial` with low emissive (dark base).
Body (non-organ) meshes use translucent `MeshPhysicalMaterial` (opacity 0.15) to let organs show through.

---

## Mushroom Data (`mushrooms.js`)

```js
const MUSHROOMS = {
  cordyceps: {
    name: "Cordyceps",
    tagline: "Energy & Respiratory Power",
    themeColor: "#ff6b2b",    // orange-amber
    accentColor: "#ffd166",
    particleColor: "#ff8c42",
    targetOrgans: ["organ-lungs", "organ-adrenal", "organ-heart"],
    animation: "pulse",       // organs pulse rhythmically
    benefits: [
      { title: "Lung Capacity", body: "Increases oxygen uptake and VO2 max..." },
      { title: "Energy Metabolism", body: "Boosts ATP production in mitochondria..." },
      { title: "Adrenal Support", body: "Regulates cortisol for sustained energy..." }
    ]
  },
  lionsmane: {
    name: "Lion's Mane",
    tagline: "Brain & Neural Clarity",
    themeColor: "#7b2fff",    // violet-purple
    accentColor: "#c77dff",
    particleColor: "#9d4edd",
    targetOrgans: ["organ-brain", "organ-nervous"],
    animation: "ripple",      // neural ripple waves from brain down spine
    benefits: [
      { title: "Neurogenesis", body: "Stimulates NGF (nerve growth factor)..." },
      { title: "Cognitive Focus", body: "Enhances memory and concentration..." },
      { title: "Mood & Anxiety", body: "Supports serotonin pathway regulation..." }
    ]
  },
  reishi: {
    name: "Reishi",
    tagline: "Calm, Sleep & Immunity",
    themeColor: "#00b4d8",    // deep teal-blue
    accentColor: "#90e0ef",
    particleColor: "#48cae4",
    targetOrgans: ["organ-brain", "organ-gut", "organ-nervous"],
    animation: "breathe",     // slow breathing wave across whole body
    benefits: [
      { title: "Sleep Regulation", body: "Modulates GABA receptors for deep sleep..." },
      { title: "Immune Modulation", body: "Activates NK cells and macrophages..." },
      { title: "Stress Adaptation", body: "Triterpenes calm the HPA axis..." }
    ]
  }
};
```

---

## Transition Animation System (`transitions.js`)

When user switches mushroom:

1. **Fade-out phase** (300ms): Lerp emissive of all active organs → 0, theme color fades
2. **Swap phase** (0ms): Set new mushroom data, update sidebar text, change `--theme-color` CSS var
3. **Fade-in phase** (600ms): Lerp organ `emissiveIntensity` from 0 → 1.8 on targeted organs
4. **Continuous animation** (looping):
   - `pulse`: `Math.sin(time * 2) * 0.4 + 1.4` scale + emissive flicker on organ meshes
   - `ripple`: Expanding `RingGeometry` particles from brain along spine
   - `breathe`: Whole-body Y-scale oscillation `Math.sin(time * 0.8) * 0.02`
5. **Particle system**: Per-mushroom colored `Points` geometry floating around targeted organs

---

## UI Layout (`index.html` + `styles.css`)

```
┌─────────────────────────────────────────────┐
│  HEADER: "Mushroom Body Explorer" logo+title │
├────────────────┬────────────────────────────┤
│                │                            │
│   3D CANVAS    │     SIDEBAR                │
│  (Three.js)    │  [mushroom icon + name]    │
│                │  [tagline]                  │
│                │  [benefit cards ×3]         │
│                │                            │
├────────────────┴────────────────────────────┤
│  SWITCHER: [Cordyceps] [Lion's Mane] [Reishi]│
└─────────────────────────────────────────────┘
```

- **Dark theme**: `#0a0a0f` background, `#1a1a2e` panels
- **CSS custom property** `--theme-color` updates per mushroom → used for borders, glow, accents
- **Switcher buttons**: Pill-shaped, active state glows with `--theme-color` box-shadow
- **Sidebar**: Fixed on desktop, bottom sheet on mobile
- **Responsive**: Canvas full-screen on mobile, sidebar overlays
- **Embeddable**: `<iframe src="index.html?mushroom=cordyceps">` with URL param to preload profile

---

## Implementation Steps

1. `icons/icon-192.svg` — SVG mushroom logo (cap, stem, gills, spore dots)
2. `icons/icon-192.png` + `icon-512.png` — Canvas-rasterized PNGs from SVG
3. `manifest.json` — PWA manifest with icons, display: standalone
4. `service-worker.js` — Cache-first offline support
5. `css/styles.css` — Full dark theme, CSS vars, responsive layout, transitions
6. `js/mushrooms.js` — Mushroom data constants
7. `js/scene.js` — Three.js scene, renderer, camera, lights, bloom
8. `js/body.js` — Procedural body + organ meshes factory
9. `js/transitions.js` — Transition engine + per-mushroom animation loops
10. `js/app.js` — Init everything, wire up switcher UI, handle URL params
11. `index.html` — HTML shell linking all the above + Three.js CDN

---

## Verification

1. Open `index.html` in browser → app loads, 3D body visible, rotating
2. Click each mushroom button → organs light up in correct color, sidebar updates, animation plays
3. On mobile Chrome: "Add to Home Screen" prompt → install → icon appears on homescreen
4. Tap homescreen icon → app opens in standalone mode (no browser chrome)
5. Offline: disable network → reload → app still works (service worker cache)
6. Embed test: `<iframe src="index.html?mushroom=lionsmane">` → loads with Lion's Mane pre-selected
