# XAFFX

Affiliate product discovery platform — YouTube Shorts-style vertical video feed for browsing and buying products on Meesho. Revenue via affiliate commissions. Installable as a PWA / Light App.

## Architecture

- **Entry point**: `index.html` — vanilla HTML/CSS/JS, no build system
- **PWA**: `manifest.json` + `sw.js` for installable light app support
- **Icons**: `icons/` directory with 192px and 512px PNG app icons
- **Legacy**: `prototype.html` — original single-file prototype
- **Video**: YouTube IFrame API embedding Shorts/Reels content
- **Data**: Mock product JSON (will connect to Meesho affiliate API later)
- **Target**: Mobile-first, phone browsers

## Key Design Rules

1. **Mobile-first, always.** Every UI change must work on phone screens (320px–414px wide). Test at small viewport sizes before confirming.
2. **Everything inside the player-wrap.** All overlays (product card, buttons, panels, gradients) are children of `.player-wrap` and positioned absolutely within it. Nothing should overflow or sit outside the video container.
3. **Player-wrap uses `aspect-ratio: 9/16`** to match YouTube Shorts. Max-width 420px, max-height 100vh. This ensures overlays always align with the visible video area at any window size.
4. **YouTube chrome is hidden** via iframe scaling (oversized iframe + overflow:hidden) and gradient overlays at top/bottom. Do not rely on playerVars alone — they don't hide channel icons.
5. **Responsive sizing**: Use `clamp()` or viewport units for font sizes, padding, and spacing so the product card scales down gracefully at small viewports.
6. **No categories/chips on top.** Removed — keep the top clean. Only a mute/unmute icon button sits in the top-right.
7. **Product card at bottom**: Single dark card with product name, price (prominent), MRP strikethrough, discount %, rating, and Buy Now button. Product details on left, buy button on right, vertically centered.
8. **Slide-out panel**: Swipe left or tap Buy Now to open the detail panel (80% width, max 300px) inside player-wrap with full product info, images, badges, social proof, buy link, and share button.

## What NOT to Do

- Don't add UI outside `.player-wrap` (except the mute button which is fixed-position)
- Don't use fixed `px` font sizes — use `clamp()` so text scales with viewport
- Don't assume desktop viewport — always verify at narrow widths
- Don't add YouTube branding, channel info, or controls — keep them hidden
- Don't over-engineer — this is a prototype, keep it simple
