# StickerSwap — Product Spec (v1)

**One-liner:** Paste-a-link sticker swapping for Panini collectors. Two people, two lists, instant "you give / you get" — no accounts, no server.

**Status:** Locked for build. Album: FIFA World Cup 2026™ (Panini).

---

## 1. Problem

Two collectors each have a stack of duplicates. Today they compare by reading lists aloud, scrolling photos, or using heavyweight platforms (accounts, listings, shipping). The actual need is a 30-second intersection of two lists. Nothing lightweight does this well.

## 2. Value proposition

**Speed to intersection.** From "here's my link" to "we swap these 14 for those 9" in under a minute, on a phone, over WhatsApp. Everything that doesn't serve that moment is out of scope.

## 3. Non-goals (v1)

- Parallels / chase / special-finish variants (free-text "extras" note only)
- Fairness scoring, valuations, or trade suggestions beyond the raw intersection
- Multi-way (3+ person) trades
- Accounts, chat, history, marketplace, shipping
- Any server-side component
- Camera scan of album pages (v2 spike — see Appendix A)

## 4. Users & core scenario

Collector A fills in their collection once (~2–3 min). They tap **Share**, send the link to Collector B. B opens it; if B has already entered their own collection (localStorage), the match view renders immediately. If not, B enters theirs first, then sees the match. They negotiate the actual swap themselves, in person or in chat.

## 5. Data model

Per collector, per album:

| Field | Type | Notes |
|---|---|---|
| `albumId` | string | e.g. `fwc2026` — versioned config key |
| `have` | bitset over all sticker slots | tapped = owned |
| `dupes` | map `{ stickerIndex: count }` | count = spare copies beyond the one in the album (×1 spare, ×2 spares…) |
| `extras` | string ≤ 200 chars | free text for parallels/oddities |
| `name` | string ≤ 30 chars, optional | shown to the other party in match view |

`missing` is always **derived** (complement of `have`) — never stored.

**Invariant:** `dupes` keys ⊆ `have` set. Untapping a sticker clears its dupe count.

### Album definition (config, not code) — ✅ LOCKED

Confirmed against the physical album and the official checklist. **980 stickers total:**

- `00` — Panini logo foil (1 sticker)
- `FWC1–FWC19` — specials: emblem ×2, mascots, slogan, ball, 3 host-country foils, FIFA Museum history ×11
- 48 teams × 20 stickers, grouped A–L (4 teams each). Per team: slot **1** = team logo foil, slot **13** = team photo, remaining 18 = players.

Sticker IDs are `CODE + number` (`MEX7`, `FWC3`, `00`) — the album's own scheme, so collectors can type/read them natively. Flat index for the bitmap: specials first (`00`, `FWC1..19`), then teams in group order A–L, slots 1–20 each → indices `0…979`, HAVE bitmap = 123 bytes = **164 base64url chars**. Full definition lives in `fwc2026-album-config.json` (team codes, names, group colors sampled from the album).

**Out of scope:** Coca-Cola promo stickers `CC1–CC12` and all regional parallels — free-text extras field only.

## 6. State, persistence, sharing

- **Persistence:** full state serialized to `localStorage` on every mutation (debounced 300 ms). Key: `stickerswap:{albumId}`.
- **Sharing:** state encoded in the **URL hash** (never query params — hash stays client-side):

```
https://…/#a=fwc2026&v=1&n=Marcos&h=<base64url bitmap>&d=<dupes>&x=<extras>
```

- `h`: bitset of `have`, one bit per slot, byte-packed → base64url. ~1000 slots ≈ 125 bytes ≈ **~170 chars**.
- `d`: dupes in collector shorthand, run-length friendly: `12,40-46,88x3` (range = all ×1 unless `xN`).
- `v`: encoding version. Decoder must reject unknown versions with a friendly "ask your friend to update the link" message.
- Full link comfortably under 400 chars → safe for WhatsApp, SMS, QR.

**Opening a link never overwrites local state.** Incoming data is held as "the other person" only.

## 7. UX — three screens

Single-page app, bottom tab bar: **Album · Dupes · Swap**.

### Screen 1 — Album (HAVEs)

- 12 group blocks (A–L) + specials block, using Panini's section colors for instant recognition.
- Numbered chips, ≥ 40 px touch targets, ~8 per row, sticky section headers while scrolling.
- Tap toggles have/missing (filled vs hollow).
- Accelerators:
  - **Long-press + drag** paints a contiguous range in one gesture.
  - **Invert section** button per block (for "I have all of Group E except two").
- Live counter in header: `487 / 980 · 52 missing in view`.

### Screen 2 — Dupes

- Shows only HAVE stickers. Tap cycles count: — → ×1 → ×2 → ×3 → —.
- **Paste box** accepting collector shorthand (`12, 40-46, 88x2`), tolerant of spaces/commas/newlines; parse errors highlighted inline, valid entries applied anyway.
- Header counter: `61 dupes (74 total spares)`.

### Screen 3 — Swap

Two modes, auto-selected:

- **No incoming link:** big **Share my list** button → copies URL + native share sheet. Secondary: **Show QR**.
- **Opened via someone's link:** match view.
  - Column **You get**: their dupes ∩ your missing, with their counts.
  - Column **You give**: your dupes ∩ their missing, with your counts.
  - Totals row: `You get 14 · You give 9`. No fairness judgment — humans negotiate.
  - Each column copyable as plain text shorthand (for pasting into the chat where the deal happens).
  - Banner with the sender's `name` and their `extras` note, if present.

### Empty/edge states

- Match view with zero overlap: say so plainly, still show both totals.
- Link with same data as local (opened own link): detect and show Share mode with a hint.
- Malformed/truncated hash: friendly error, never crash, local data untouched.

## 8. Matching logic (normative)

```
youGet  = { s ∈ theirDupes | s ∉ yourHave }
youGive = { s ∈ yourDupes  | s ∉ theirHave }
```

Counts displayed but not netted — a ×3 dupe can serve future trades; the tool doesn't allocate.

## 9. Technical approach

- **One static HTML file** (inline CSS/JS, no framework, no build step). Hostable on GitHub Pages / any static host.
- No network calls at runtime. No analytics in v1.
- Vanilla JS + CSS grid; virtualized rendering not needed if sections render lazily on scroll (IntersectionObserver).
- Encoding module isolated + unit-testable: `encode(state) → hash`, `decode(hash) → state`, round-trip tested.
- Mobile-first (~380 px). Desktop is "works, not optimized."

## 10. Risks

| Risk | Mitigation |
|---|---|
| Wrong album ranges → all numbers off | Transcribe from physical album index; verify section boundaries against photos before build |
| Chip grid feels sluggish on cheap phones | Lazy-render sections; no per-chip event listeners (event delegation) |
| Link mangled by messaging apps | base64url alphabet only; QR fallback |
| User clears browser data | "Copy my backup link" = same share link doubles as backup/restore |
| Scope creep toward marketplace | This spec's §3. Re-read it. |

## 11. Success criteria

- New user: empty grid → shared link in ≤ 3 minutes.
- Link recipient: open → match view in ≤ 10 seconds (returning) / ≤ 3 minutes (first-time).
- Two real collectors complete a physical swap negotiated only via the tool + one chat thread.

## 12. Milestones

1. **M0 — Album config:** ✅ DONE — `fwc2026-album-config.json`, verified against album photos + official checklist (980 stickers).
2. **M1 — Grid + persistence:** Screen 1 with localStorage.
3. **M2 — Dupes:** Screen 2 incl. shorthand parser.
4. **M3 — Encode/share/compare:** Screen 3, round-trip tests, QR.
5. **M4 — Field test:** two real collections, one real swap.

---

## Appendix A — Scan-assist (v2 spike, not committed)

**Goal:** photograph a page, pre-fill that page's chips; user confirms.

**Why it's not v1:** no off-the-shelf library does this. Slot numbers remain printed and visible whether or not a sticker is placed, so OCR alone cannot distinguish filled from empty.

**Approach worth a spike:**
1. **Tesseract.js** (digits-only) finds printed slot numbers → anchor points.
2. **OpenCV.js** homography aligns photo to a per-page reference template.
3. Per-slot classifier: placed stickers are foil/gloss → high specular variance vs. known flat placeholder art. Simple per-region contrast/texture diff first; only escalate if that fails.
4. Output = suggestions only; user always confirms on the grid.

**Costs:** ~10 MB lazy-loaded WASM; per-page templates; lighting/angle tuning.
**Spike protocol:** run the heuristic on owner-supplied photos (1 empty page, 1 partial, 1 full). Promote to roadmap only if ≥ 95 % slot accuracy on those samples with zero tuning per photo.
