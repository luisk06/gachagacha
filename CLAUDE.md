# ガチャガチャ — Capsule toy web app

A single-page gachapon (capsule toy) game. Prizes are 200円, 300円, 400円 or 500円. The goal is fun, tactile "juice": sound, motion and haptics on every action.

- **Repo:** https://github.com/luisk06/gachagacha
- **Live:** GitHub Pages from `main` / root. The URL should be https://luisk06.github.io/gachagacha/ — confirm under repo Settings → Pages.
- **Deploy:** push to `main`. Pages redeploys in about 1–2 minutes.

## Constraints (from the user)
- Simple: one file, `index.html`, with inline CSS and JS. No build step, no framework, no database, no backend.
- **All UI text is in Japanese**, using the words Japanese gacha players and capsule-toy machines actually use, not literal translations (see Terminology).
- Prices and prizes are written as `500円`, never `¥500`.

## Structure of `index.html`
- `<style>`: palette tokens on `:root` (`--ink`, `--red`, `--yellow`…), followed by the machine, capsule, sidebar and reveal-overlay styles. Font: M PLUS Rounded 1c from Google Fonts.
- Markup: the machine (dome, collar, body with marquee, 投入口 coin slot, crank, prize panel, chute), the coin button, the sidebar cards, the reveal overlay, `#flash` and the `#fx` canvas.
- Script (one IIFE), in order:
  - `TIERS`: prize table (yen, rank, Japanese label, color, weight)
  - `Sfx`: every sound, synthesized with the Web Audio API (no audio files)
  - `fx`: canvas particles (confetti, stars, shards, coins)
  - dome capsule layout (`ROWS`/`SLOTS`)
  - the state machine and its handlers
  - wiring and init

## Game flow (state machine)
`idle → coin → ready → turning → dropping → dropped → revealing → revealed → cooldown → idle`

1. **Insert coin:** the coin flies into 投入口.
2. **Turn the crank:** drag clockwise (it only turns forward, like a ratchet) or tap it to turn automatically. Four clicks per full turn.
3. **Capsule drops** into the 取り出し口 (exit chute).
4. **Tap the capsule 3 times** to crack it open. For SR and SSR prizes, light glows through the cracks from the 2nd tap (like gacha games' 確定演出) and the capsule shakes and charges up before bursting.
5. **Prize reveal:** flash, rays, confetti, fanfare. SSR also shakes the screen and rains coins.
6. **Collect** (〇〇円を受け取る) adds to the total, then the **60 s cooldown** starts.

Space/Enter advances whichever step is current. Haptics use `navigator.vibrate`, which works on Android only.

## Prizes and odds (`TIERS`)
| 円 | Rank | Label | Weight |
|---|---|---|---|
| 200 | R | レア | 25 |
| 300 | N | ノーマル | 37 |
| 400 | SR | スーパーレア | 24 |
| 500 | SSR | スーパースーパーレア (shout: 大当たり！！) | 14 |

The average payout is about 327円 per play. The user set these odds on 2026-10-18 and asked for the 200/300 labels to be swapped so the most common prize is N. Each tier's `lvl` (0 = N … 3 = SSR) drives the reveal effects and sounds; the array index only drives price order, counts and history.

## Cooldown
- `COOLDOWN_MS = 60 * 1000`. It starts on collect.
- The end time is stored as `stats.cooldownUntil` in localStorage, so a reload doesn't skip it.
- 記録をリセット clears winnings and history only, not the cooldown, so it can't be used to skip the wait.
- During the wait, the coin button shows `あと N秒` with a fill bar and the marquee lights go dark. The last 5 seconds beep. The end plays a chime and shows `準備OK！`.

## Persistence
Key `gachagacha-v1` in localStorage holds `{ total, plays, counts[4], history[≤14], muted, cooldownUntil }`. Every read and write is wrapped in try/catch. Data is per browser only and never shared.

## Terminology (researched, keep consistent)
- Steps: コインを入れてね → ハンドルを右に回してね → カプセルが出たよ！取り出し口からとってね → タップしてカプセルを開けよう！ → もう一回！/あと一回！
- 投入口 (coin slot), 取り出し口 (exit chute), ハンドル (crank; some machines say レバー)
- Rarity ladder N / R / SR / SSR. 大当たり means jackpot.
- 排出率 is the colloquial word for odds. 提供割合 is the formal in-game label; switch to it if a more official tone is wanted.
- Sidebar: 獲得金額, 〇回・平均〇円, 景品と排出率, ガチャ履歴, 記録をリセット

## Audio notes (important)
- iPhone mutes Web Audio while the silent switch is on. The fix in `Sfx`: set `navigator.audioSession.type = 'playback'` (iOS 17+). As a fallback, loop an inaudible WAV in an `<audio>` element (`primeSession`), and pause it when the page is hidden or the game is muted. **The user confirmed this works on iPhone Chrome.**
- Every `pointerdown`/`touchend`/`keydown` calls `Sfx.unlock()`, because iOS suspends audio after app switches or screen lock.

## Testing gotchas (Claude's preview pane)
- The pane pauses `requestAnimationFrame` and WAAPI `onfinish` while hidden. Timing-critical logic therefore uses `setTimeout`/`setInterval`, not `onfinish`. In tests, shim it with `window.requestAnimationFrame = f => setTimeout(() => f(performance.now()), 16)`.
- After editing the file, reload by navigating to `file:///C:/Users/luism/Documents/Claude/Projects/gachagacha/index.html`. `location.reload()` keeps showing the old snapshot.
- To force a jackpot in tests: `Math.random = () => 0.999`.
