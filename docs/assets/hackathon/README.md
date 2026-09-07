# Hackathon figures

Four figures for the submission README, authored with the `diagram-design` skill:

| Figure | Purpose | Canvas |
|---|---|---|
| `hero` | Product identity and sprint context | 960 × 360 |
| `learning-loop` | Run → evidence → evaluation → optional learning → next run | 960 × 800 |
| `learning-policy` | Candidate admission, bounded trial, promotion, fallback and waiting | 960 × 816 |
| `architecture` | Execution, authenticated owner feedback, background learning and persistence | 960 × 736 |

Each `.html` is an editable, standalone source. Its `.svg` export preserves the accessible title
and description and embeds the font subsets needed by that figure. GitHub can therefore display
it as an image without downloading fonts. HTML previews load the same fonts from Google Fonts.

The diagrams summarize `docs/ARCHITECTURE.md` §14; they do not replace the normative contract.
The overview omits retention windows, inference accounting and individual persistence tables.
The policy figure omits rejection before trial and detailed rollback triggers; the README and
algorithm ADR explain those branches.

## Visual system

The palette adapts the existing orange swift-claw identity to a warm editorial canvas.
It is local to these figures; no global diagram-design profile is required.

| Role | Token |
|---|---|
| Paper | `#F7F5EF` |
| Ink | `#252922` |
| Secondary text | `#596056` |
| Accent | `#C44D1F` |
| Accent fill | `#FBE9DC` |
| Borders | `#DADCD2` |
| Provider calls | `#3E6770` |
| Titles | Literata 400 / 500 |
| Human-readable labels | Manrope 400 / 500 / 700 / 800 |
| Technical labels | IBM Plex Mono 400 / 500 |

Fonts are distributed under the SIL Open Font License; notices are in `fonts/`.
The hero reuses the project's existing `swift-claw-avatar-dark-512.png` artwork.

Verification covers the diagram-design accessible/single-file check, SVG XML parsing, browser
rendering as inline SVG and as an image, label bounds and README link resolution. All figures
are static and convey their meaning without motion, JavaScript or color alone.
