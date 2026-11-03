# D1 — Game Design Document

Nov 3, 2026 · @GUB

## Overview

**Beat the Target** is a 3D dice-betting game: bet money, roll physics-simulated dice, and win if the total reaches the target. It runs in the browser from a single file, `index.html` (Three.js 0.160 for rendering, cannon-es 0.20 for physics).

| Item | Value |
| --- | --- |
| Genre | Push-your-luck dice betting |
| Players | 1 (solo) or 2 (hot-seat, shared table) |
| Starting money | $10,000 per player |
| Goal (solo) | Grow your money; best total is saved as a high score |
| Goal (2 players) | Last player with money wins; first to go broke loses |
| Dice types | d4, d6, d8, d10, d20 |

`index_gub_roller.html` is a separate, earlier prototype ("Dice Roller") and is not part of this game.

## Core loop

Each round moves through four phases: **bet → rolling → decide → resolve**. All costs are paid when the dice are first thrown.

1. **Bet.** A new target is shown. The player sets the bet and picks options: difficulty, dice type, number of dice, exploding dice, penalty dice and rerolls. The Roll button shows the total upfront cost.
2. **Roll.** Press Roll (or Space), or flick any die by hand. The upfront cost is deducted and the payout is locked in.
3. **Settle.** The game waits until all dice sleep (max 8 s). Dice resting on an edge get a nudge. With exploding dice on, each max face spawns a gold bonus die, then the game waits again.
4. **Decide.** If the total is below the target and rerolls remain, the player selects dice and rerolls them, or takes the loss.
5. **Resolve.** Total ≥ target wins the locked payout; otherwise the money spent is lost. The result is logged and a new round starts.

The total is the sum of regular and bonus dice minus the sum of penalty dice. The game is over when the active player's money is below the minimum bet.

## Rules, costs and payouts

Win = final total ≥ target. The payout is a total return (stake included), priced from the real odds at 95% house return.

### Target

The target sits in a band between the lowest and highest total the regular dice can roll (penalty dice not counted). One random draw is made per round, so changing dice or difficulty moves the target within the band but cannot re-draw it.

| Difficulty | Band (% of the way from min to max total) |
| --- | --- |
| Easy | 15–35% |
| Medium (default) | 35–55% |
| Hard | 55–75% |
| Very hard | 75–100% |

Example: 2d6 can roll 2–12, so Medium gives a target of 6–7.

### Bet and options

| Option | Range | Cost |
| --- | --- | --- |
| Bet | $500–$5,000, step $100, never above current money | The bet itself |
| Dice type | d4, d6, d8, d10, d20 | Fee of 0%, 10%, 20%, 30%, 50% of the bet |
| Number of dice | 1–5 (default 2) | Free; the target scales with it |
| Exploding dice | On / off | 50% of the bet |
| Penalty dice | 0–2, any dice type | Free; raises the payout |
| Rerolls | 0–3, bought before rolling | 25% of the bet, doubling each: 25%, 50%, 100% |

If the player can no longer afford the extras (after raising the bet or losing money), rerolls are dropped first, then exploding dice.

### Payout

```latex
\text{multiplier} = \min\left(50,\ \frac{0.95}{P(\text{win})}\right)
```

```latex
\text{payout} = \lfloor \text{bet} \times (M_{plain} + (M_{penalty} - M_{plain}) \times \text{penaltyScale}) \rfloor
```

- P(win) is computed exactly from the dice distribution, ignoring rerolls and explosions (those are what the player pays extra for).
- M plain is the multiplier without penalty dice; M penalty is with them. penaltyScale defaults to 100%.
- Bigger dice win more often against their scaled target, cost more and pay less.
- Net result of a round = payout (if won) − everything spent that round.

### Rerolls and explosions

- Rerolls are only offered when the roll misses the target. The player picks any dice (regular, bonus or penalty) to reroll.
- Exploding dice: every regular or bonus die showing its max face adds one gold bonus die, which can explode too. Max 6 bonus dice per round. Penalty dice never explode.
- When a target is out of reach without explosions, the summary says "only exploding dice can reach it".

### End of game

- Solo: the game ends when money drops below the minimum bet. The best money ever reached is saved in the browser.
- 2 players: when the active player can't cover the minimum bet, the other player wins.

## Special mechanics

### Physical dice and hand throws

- Dice are real physics bodies in a walled tray. Press Roll to toss them all, or grab a die and flick it.
- A flick faster than 4 units/s throws: before betting it rolls all dice; in the decide phase it rerolls only that die. A slower release just sets the die down with the same face up.
- In the decide phase, a click (under 6 px of movement) selects or deselects a die for reroll.
- Faces are read from the die's orientation once it rests. Opposite d6 faces sum to 7.

### Penalty dice

- Red dice whose value is subtracted from the total. Count 0–2 by default (hard cap 10 via debug); their type is chosen separately from the regular dice.
- They lower the win chance, so the odds-based payout goes up. The debug Payout raise scales how much of that increase is applied.

### Flip the table

- A "Flip the table" button sits on the 3D view, available any time except while dice roll or a die is held.
- The tray tips over in 0.7 s, walls vanish, dice fly off and the bet chips scatter and bounce.
- Flipping in the decide phase gives up the round: it resolves as a loss. Flipping during betting costs nothing.
- The main button becomes "Put the table back"; the tray returns and play continues. Hint text: "Dice everywhere. Feel better?"

### Tray sizing

- The tray grows with the dice on it (regular + penalty), measured in d6 widths: width = 8 + 1 per die, depth = 6 + 0.5 per die.
- The camera reframes to keep the tray and the chips in view, backing off on narrow screens.

### Bet chips

- The bet is shown as chip stacks in front of the tray, biggest chips first: $10,000 black, $2,500 green, $1,000 blue, $500 red, $100 white.
- Up to 10 chips per stack and 8 stacks across.

## Two-player mode

Two players share one table and take turns; each has their own $10,000 wallet. Everything else (options, target, dice) is shared, and the active player's wallet pays and collects.

- **Win:** the player keeps the table and chooses: **Roll again** or **Pass to** the other player.
- **Loss:** the table moves to the other player automatically.
- **Broke:** if the active player can't cover the minimum bet, the game ends and the other player wins.
- Switching between 1 and 2 players starts a new game (both wallets reset). It is locked while rolling or deciding.
- The active player's wallet is highlighted, buttons are prefixed with their name, and history entries are tagged P1 / P2.
- Passing is not allowed while the table is flipped.

## UI and controls

The screen has a settings panel in four numbered groups, the 3D table below it, and two action buttons. Settings lock once the dice are thrown.

| Panel group | Contents |
| --- | --- |
| 1. Money & Best | Wallet(s), best money, player count, bet slider |
| 2. Difficulty | Easy / Medium / Hard / Very hard, with the current target range |
| 3. Target | Target, current total, roll breakdown (e.g. "4 + 5 − 2 = 7") |
| 4. Dice | Type, count, exploding, penalty count and type, rerolls, cost summary |

| Input | Action |
| --- | --- |
| Primary button / Space | Roll · Reroll selected · New game · Put the table back (by context) |
| Secondary button | Take the loss (decide phase) · Pass to the other player (after a win) |
| Click a die | Select / deselect it for reroll |
| Flick a die | Throw all dice (bet) or reroll that die (decide) |
| Flip the table | Flip the tray (see Special mechanics) |

Feedback:

- The cost summary shows upfront cost, dice fee, base win chance and the payout before each roll.
- A hint line under the buttons explains the next possible action.
- Roll history (collapsible menu) keeps the last 10 rounds as "total/target ±net", green for wins and red for losses.
- Dice colours by type: d4 green, d6 white, d8 blue, d10 purple, d20 orange; penalty dice red, bonus dice gold.

## Debug settings and tuning

A collapsible Debug menu exposes the balance values. Every change is saved in the browser and "Reset debug settings" restores the defaults.

| Debug group | Setting | Default |
| --- | --- | --- |
| 1. Dice colors | Body colour per dice type | d4 green, d6 white, d8 blue, d10 purple, d20 orange |
| 2. Chips colors | Value and colour of the 5 chip types | $10,000 / $2,500 / $1,000 / $500 / $100 |
| 3. Difficulty | From–to band per difficulty | 15–35 / 35–55 / 55–75 / 75–100% |
| 4. Bet | Min and max bet | $500 / $5,000 |
| 5. Dice count | Min and max regular dice | 1 / 5 (hard cap 20) |
| 6. Dice cost | Fee per dice type, % of bet | 0 / 10 / 20 / 30 / 50% |
| 7. Penalty | Payout raise; max penalty dice | 100%; 2 (hard cap 10) |
| 8. Tray size | Base width, width per die, base depth, depth per die (d6 widths) | 8, +1, 6, +0.5 |

Fixed in code (no debug control):

| Constant | Value |
| --- | --- |
| Starting money | $10,000 |
| House return | 95% of fair odds |
| Max payout multiplier | ×50 |
| Exploding dice price | 50% of bet |
| Reroll price | 25% of bet, doubling each |
| Max rerolls | 3 |
| Max bonus dice per round | 6 |
| Roll settle timeout | 8 s |
| Table flip duration | 0.7 s |
