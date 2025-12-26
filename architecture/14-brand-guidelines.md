# 14 - StateRelay Brand Guidelines

## Quick Identity

| Element | Value |
|---------|-------|
| **Name** | StateRelay |
| **Tagline** | One state. Every platform. |
| **One-liner** | Virtual devices that relay automation state between ecosystems. |

---

## Logo System

### Logo Concept

**Relay Node mark**: Central node + 2–3 connected nodes. Flat, geometric, no gradients.

### Logo Variants

| Variant | Use Case |
|---------|----------|
| **Primary lockup** | Icon + wordmark "StateRelay" |
| **Icon-only** | Favicon, app icons, platform tiles |
| **Light version** | White mark on dark background |
| **Dark version** | Primary mark on light background |
| **1-color** | Black/white for docs, print, watermark |

### Clear Space & Sizing

- **Clear space**: Keep at least 1× the icon's node diameter around the logo on all sides
- **Minimum sizes**:
  - Icon-only: 16px min (must remain legible)
  - Lockup: 120px width min on web

### Placement

| Context | Usage |
|---------|-------|
| Marketing | Lockup in header/hero |
| Dashboard | Icon-only in top-left nav; wordmark on login page |

### Logo Don'ts

- ❌ Don't add gradients, glows, bevels
- ❌ Don't rotate/skew
- ❌ Don't change node count/geometry per use
- ❌ Don't co-brand by placing Alexa/IFTTT logos beside StateRelay

### File Naming

```
staterelay-logo-lockup.svg
staterelay-logo-icon.svg
staterelay-logo-light.svg
staterelay-logo-dark.svg
staterelay-favicon.svg
```

---

## Color Palette

### Core Palette

| Color | Name | Hex | Usage |
|-------|------|-----|-------|
| ![#1F2A44](https://via.placeholder.com/20/1F2A44/1F2A44.png) | **Deep Slate Blue** | `#1F2A44` | Nav, headers, hero backgrounds, key surfaces |
| ![#3FB8C8](https://via.placeholder.com/20/3FB8C8/3FB8C8.png) | **Relay Cyan** | `#3FB8C8` | Primary buttons, links, focus rings, selection states |
| ![#F5A623](https://via.placeholder.com/20/F5A623/F5A623.png) | **Signal Amber** | `#F5A623` | Warnings/attention only (≤5% of any screen) |
| ![#F7F9FC](https://via.placeholder.com/20/F7F9FC/F7F9FC.png) | **Soft Cloud** | `#F7F9FC` | Background |
| ![#0E1116](https://via.placeholder.com/20/0E1116/0E1116.png) | **Near Black** | `#0E1116` | Primary text |
| ![#6B7280](https://via.placeholder.com/20/6B7280/6B7280.png) | **Cool Gray** | `#6B7280` | Muted text, secondary elements |

### Status Colors (Product UI)

| Status | Color | Hex |
|--------|-------|-----|
| ON / Active | Green | `#22C55E` |
| OFF / Inactive | Gray | `#9CA3AF` |
| Pending / Attention | Amber | `#F59E0B` |
| Error | Red | `#EF4444` |

### CSS Variables

```css
:root {
  /* Core */
  --color-primary: #1F2A44;
  --color-secondary: #3FB8C8;
  --color-highlight: #F5A623;
  --color-background: #F7F9FC;
  --color-text: #0E1116;
  --color-muted: #6B7280;
  
  /* Status */
  --color-on: #22C55E;
  --color-off: #9CA3AF;
  --color-pending: #F59E0B;
  --color-error: #EF4444;
}
```

### Tailwind Config

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        slate: {
          deep: '#1F2A44',
        },
        cyan: {
          relay: '#3FB8C8',
        },
        amber: {
          signal: '#F5A623',
        },
        cloud: '#F7F9FC',
        ink: '#0E1116',
        muted: '#6B7280',
      },
    },
  },
};
```

---

## Typography

### Typeface

**Inter** — Website + App

```html
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

### Type Scale

| Level | Size / Line Height | Weight | Usage |
|-------|-------------------|--------|-------|
| H1 | 44px / 52px | 700 | Page titles |
| H2 | 32px / 40px | 700 | Section headers |
| H3 | 24px / 32px | 600 | Subsections |
| Body | 16px / 24px | 400–500 | Paragraphs |
| Small | 14px / 20px | 400–500 | Secondary text |
| Caption | 12px / 16px | 400 | Labels, hints |

### Typography Rules

- ✅ Favor sentence case in UI labels
- ❌ Avoid ALL CAPS except tiny tags ("BETA", "NEW")

---

## Components (UI Styling)

### Spacing & Layout

| Property | Value |
|----------|-------|
| Base spacing unit | `8px` |
| Corner radius | `8px` |
| Shadows | Subtle only (small elevation for cards/modals) |

### Buttons

| Type | Style |
|------|-------|
| **Primary** | Filled cyan `#3FB8C8` with dark text `#0E1116` |
| **Secondary** | Outline (1px) using `#1F2A44` or `#6B7280` |
| **Destructive** | Red `#EF4444` (only for irreversible actions) |

```css
.btn-primary {
  background-color: #3FB8C8;
  color: #0E1116;
  border-radius: 8px;
  padding: 8px 16px;
}

.btn-secondary {
  background: transparent;
  border: 1px solid #1F2A44;
  color: #1F2A44;
  border-radius: 8px;
  padding: 8px 16px;
}

.btn-destructive {
  background-color: #EF4444;
  color: white;
  border-radius: 8px;
  padding: 8px 16px;
}
```

### Inputs

| Property | Style |
|----------|-------|
| Background | `#F7F9FC` |
| Border | 1px muted gray |
| Focus ring | Cyan `#3FB8C8` |
| Error state | Border/text `#EF4444` |
| Warning state | `#F5A623` |

### Status Pills

| Status | Color |
|--------|-------|
| ON | Green `#22C55E` |
| OFF | Gray `#9CA3AF` |
| PENDING | Amber `#F59E0B` |
| ERROR | Red `#EF4444` |

Keep pills compact and consistent; no gradients.

---

## Iconography

- Simple line icons, consistent stroke weight
- Avoid overly playful rounded glyphs
- Use cyan for active icons, muted gray for inactive

Recommended icon libraries:
- [Lucide Icons](https://lucide.dev/)
- [Heroicons](https://heroicons.com/)

---

## Tone & Messaging

### Voice

**Clear, neutral, infrastructure-grade**

- No hype, no "magic," no "hacks"

### Preferred Phrasing

✅ "Relay device state in real time."
✅ "Trigger automations when properties change."
✅ "Platform-neutral virtual devices."

### Avoid Phrasing

❌ "Hack Alexa with IFTTT"
❌ "Proxy / emulator / workaround"
❌ "Magically sync everything"

---

## Do / Don't Examples

### Do

- ✅ Use calm whitespace and simple cards
- ✅ Use cyan for interactive emphasis
- ✅ Use amber only for attention states
- ✅ Keep partner references text-only (approval-safe)

### Don't

- ❌ Don't build a neon "smart home" aesthetic
- ❌ Don't show partner logos side-by-side with yours
- ❌ Don't overuse amber (it should feel rare)
- ❌ Don't use gimmicky copy or exclamation-heavy marketing

---

## Asset Checklist

```
staterelay-logo-lockup.svg       # Full logo with wordmark
staterelay-logo-icon.svg         # Icon only
staterelay-logo-light.svg        # White version for dark backgrounds
staterelay-logo-dark.svg         # Dark version for light backgrounds
staterelay-favicon.svg           # Favicon (16x16 optimized)
staterelay-brand-sheet.md        # This document
