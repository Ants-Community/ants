# ANTS — BRAND.md

*The mark, the wordmark, and how to use them. Released under CC0.*

---

## 1 · Mark

- **Direction:** Cluster · Variant D · asymmetric
- **Composition:** five ants from above, no rotational symmetry
- **Construction:** 100 × 100 viewBox
- **Colour:** black on paper (`#0E0E0C` on `#F4F1E9`)
- **Reverse:** paper on ink for dark surfaces
- **Stroke:** single weight; no gradients, no shading, no outlines

The five placements (centre-of-mass at ≈ 49.6, 50.8):

| # | x  | y  | rot   | scale |
|---|----|----|-------|-------|
| 1 | 46 | 50 |  −8°  | ×1.05 |
| 2 | 24 | 30 | −72°  | ×0.85 |
| 3 | 70 | 26 | +38°  | ×0.80 |
| 4 | 76 | 62 | +105° | ×0.85 |
| 5 | 32 | 76 | −148° | ×0.85 |

## 2 · Bare body — favicon master

A single ant, no legs. Used at 16 / 24 / 32 px (favicon), at app-icon sizes
below 32 px, and on embroidered patches under 3 cm. **Not** a reduction of
the full cluster — it is its own master, with proportions matched so the
brand reads as the same thing at every scale.

## 3 · Type

| Use      | Family            | Weight | Notes                          |
|----------|-------------------|--------|--------------------------------|
| Wordmark | IBM Plex Sans     | 500    | All caps · tracking 0.16 em    |
| Display  | IBM Plex Sans     | 500    | Sentence case · tracking −0.01 em |
| Body     | IBM Plex Sans     | 400    | 14–16 px · line-height 1.55    |
| System   | IBM Plex Mono     | 400/500| Code · captions · commits      |

No third family.

## 4 · Colour

| Token  | Value                       | Use                |
|--------|-----------------------------|--------------------|
| Ink    | `#0E0E0C`                  | Mark on paper      |
| Paper  | `#F4F1E9`                  | Background         |
| Rule   | `rgba(14,14,12,0.14)`      | Hairlines, borders |
| Muted  | `rgba(14,14,12,0.55)`      | Secondary text     |

Mono first. Colour exists only as a system reference and is not part of the
brand identity at v0.1.

## 5 · Lockup

- **Horizontal:** mark + wordmark, baseline-aligned; the wordmark sits at
  the height of the mark's central ant.
- **Stacked:** mark above, wordmark below; tracking widens to 0.22 em.
- **Descriptor** (`A distributed-AI protocol`): institutional applications
  only — whitepaper covers, conference signage, README hero. Never on the
  everyday lockup.

### Clear-space

`X` = height of the mark. No element may sit within `½ X` of the
lockup's bounding box.

### Minimum sizes

| Variant            | Pixel  | Print  |
|--------------------|--------|--------|
| Horizontal lockup  | ≥ 96 px wide  | ≥ 18 mm |
| Stacked lockup     | ≥ 64 px       | ≥ 14 mm |
| Mark only (full)   | ≥ 32 px       | ≥ 8 mm  |
| Mark only (bare)   | ≥ 16 px       | ≥ 4 mm  |

## 6 · Don't

- Rotate, distort, skew, or reflow the cluster.
- Re-colour the mark in anything other than ink or paper.
- Outline the mark.
- Apply gradients, shadows, glow, or any depth effect.
- Anthropomorphise — no eyes, faces, expressions, bows.
- Set the wordmark in any typeface or weight other than Plex Sans 500.
- Use the descriptor lockup below 44 px high.

## 7 · Files

| File                  | Purpose                              |
|-----------------------|--------------------------------------|
| `ants-mark.svg`      | Master · full cluster · 100 × 100 vb |
| `ants-mark-bare.svg` | Bare body · favicon master           |
| `ants-lockup-h.svg`  | Horizontal lockup                    |
| `ants-lockup-s.svg`  | Stacked lockup                       |
| `ants-avatar.svg`    | 512 × 512 social avatar              |
| `ants-og.svg`        | 1200 × 630 Open Graph image          |

All files use `fill="currentColor"` so they paint with CSS `color` —
one source serves both ink-on-paper and paper-on-ink contexts.

## 8 · Credits & licence

Designed for ANTS · Round 1–3, 2026. Released under **CC0** — public
domain dedication. Reuse without attribution.

The brief that produced this identity is also CC0; reuse it for your own
open-source projects.
