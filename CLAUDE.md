# Raider Runner — Project Context

## What this is

A single-file HTML5 canvas game (`index.html`) set in a Fallout-inspired post-apocalyptic world. The entire game — engine, rendering, audio, UI, and assets — lives in one file (~1900 lines). There are no build tools, no dependencies, no npm. Open in a browser and it runs.

## Architecture

**One file, one canvas.** All game logic is imperative drawing into an 800×450 canvas (`W=800, H=450, GROUND=370`). The game loop runs `update()` then `draw()` via `requestAnimationFrame`.

**State machine:** `state` is one of `'title' | 'playing' | 'vault' | 'dead'`. The top of `update()` branches on this. The Pickle 3000 menu (`pickle3000Open=true`) intercepts `playing` state and `return`s early to freeze gameplay — it is NOT a separate state.

**Input system:**
- `keys{}` — held state (truthy while key is down)
- `pressed{}` — single-frame edge triggers (cleared at end of each `update()` branch)
- Touch: `activeTouches` Map translates touch events into `keys`/`pressed` entries
- Gamepad: `pollGamepad()` maps gamepad buttons to the same `keys`/`pressed` dicts

## Key constants and globals

```
W=800, H=450, GROUND=370   — canvas dimensions and floor Y
GRAVITY=0.58               — applied to player.vy each frame
VAULT_ITEM_H=52            — shop item row height in vault
VAULT_ITEMS_VISIBLE=4      — visible rows in vault shop
PICKLE_VISIBLE=5           — visible rows in Pickle 3000 lists
```

Global game state: `player`, `enemies[]`, `bullets[]`, `particles[]`, `pickups[]`, `vaultDoors[]`, `squishDecals[]`, `scroll`, `score`, `highScore`, `caps`, `gameFrame`, `diffTick`, `spawnTimer`, `worldSpeedMult`, `silencerTicks`.

## Entity classes

| Class | Type string | Notes |
|---|---|---|
| `Player` | — | Single instance; `player.crouching` shrinks hitbox from h=64 to h=44; `player.feetY = player.y + player.h` |
| `RadRoach` | `'radroach'` | Ground enemy; can be squished by jump-stomp; drops 5–12 caps |
| `MallRat` | `'mallrat'` | Melee charger; drops 15–25 caps |
| `Raider` | `'raider'` | Shoots horizontal bullets at `GROUND-52`; drops 25–40 caps |
| `Bullet` | — | `owner='player'` or `'enemy'`; raider bullets travel at y=GROUND-52, which is above a crouching player |
| `Particle` | — | Visual only |
| `Pickup` | — | Silencer item on ground |
| `VaultDoor` | — | Spawns periodically; triggers vault state on interaction |

## Crouch mechanic

Crouching sets `player.h = 44` (from 64). **Critical:** when height changes, `player.y` is adjusted by the delta (`this.y += this.h - newH`) to keep feet planted at GROUND. Without this the player lifts off the ground, `onGround` goes false, and the crouch immediately releases — the flicker bug.

Raider bullets spawn and travel at `y = GROUND - 52`. A standing player's rect top is `GROUND - 64`; a crouching player's rect top is `GROUND - 44`. Bullets pass over crouching players because the bullet y (`GROUND-52`) is above the crouching hitbox top (`GROUND-44` ... `GROUND`).

## Weapons

```js
const WEAPONS = [
  { id:'pistol', label:'10mm PISTOL',   dmg:25, cd:14, spd:13, bcolor:null,      pellets:1, spread:0,   cost:0   },
  { id:'laser',  label:'LASER PISTOL',  dmg:18, cd:7,  spd:17, bcolor:'#ff44ff', pellets:1, spread:0,   cost:60  },
  { id:'shotgun',label:'SHOTGUN',       dmg:35, cd:28, spd:10, bcolor:'#ffaa44', pellets:3, spread:2.2, cost:80  },
  { id:'rifle',  label:'HUNTING RIFLE', dmg:55, cd:40, spd:15, bcolor:'#ffff55', pellets:1, spread:0,   cost:110 },
];
```

`currentWeaponIdx` — active weapon index. `ownedWeapons` — Set of owned indices (starts `{0}`). Weapons sold in both the Vault shop and Pickle 3000 WEAP tab. Duplicate-purchase guard: check `ownedWeapons.has(idx)`, not `currentWeaponIdx===idx`.

## Pickle 3000 menu

Opened/closed with `P`. Freezes gameplay via early `return` in `update()`. Four tabs (index 0–3):

| Tab | Label | Content |
|---|---|---|
| 0 | STAT | Live stats: HP, weapon, enemy count, caps, score, owned clothing |
| 1 | WEAP | `WEAPONS` array — buy or equip |
| 2 | CLTH | `CLOTHING_ITEMS` — one-time purchase, applies `apply()` to player |
| 3 | ITEM | `SHOP_ITEMS` filtered to consumables (`weaponIdx===undefined`) |

Navigation: `A/D` or `←/→` cycle tabs; `W/S` or `↑/↓` scroll list; `Enter/Space/Z` buy/equip; `Escape/P` close.

`pickle3000Open`, `pickle3000Tab`, `pickle3000Sel`, `pickle3000Scroll`, `pickle3000Msg`, `pickle3000MsgTimer` are module-level state. Reset on close; `pickle3000Open=false` on player death.

## Vault shop

State `'vault'` — entered via `VaultDoor`. Shows `SHOP_ITEMS`. Navigation: `W/S`, buy with `Enter/Space`. `buyShopItem(i)` handles purchase. Repurchase guard: `ownedWeapons.has(item.weaponIdx)`.

## Clothing

`CLOTHING_ITEMS` — each has `id`, `label`, `desc`, `cost`, `apply()`. `ownedClothing` is a Set of owned IDs. Each item is a one-time purchase; `apply()` modifies `player.maxHp` (and sometimes `player.hp`).

## Sound

All audio synthesized via Web Audio API. `ac()` returns the shared `AudioContext` (lazy-created to satisfy browser autoplay policy). All `sfx*()` functions are hoisted — safe to call before textual definition. `silencerTicks > 0` mutes `sfxShoot()` and `sfxEnemyShoot()`.

## Rendering notes

- Canvas coordinate system: `(0,0)` = top-left; `y` increases downward
- `image-rendering: pixelated` CSS enables crisp nearest-neighbor upscaling
- **Never use `roundRect()` in sprite draw functions** — it produces anti-aliased curves that look blurry when upscaled with pixelated rendering. Use `fillRect()` only for pixel-art sprites
- Pip-Boy aesthetic: green phosphor `#00ff44` / `#00cc33` on near-black `#0a0800`. CRT scanlines via CSS `repeating-linear-gradient`
- `drawT51()` / `drawT51Crouch()` — player sprite functions; crouch version uses horizontal thighs + vertical shins forming 90° L-corners at the knee joints
- Screen shake: `shakeX`/`shakeY` globals, applied via `ctx.translate()` at draw start, decremented by `tickShake()`

## Responsive / mobile

Canvas CSS: `width: min(100vw, calc(100svh * 1.7778)); height: auto; max-height: 100svh;` — fills viewport at 16:9 with no hard desktop cap.

Portrait phone: `#rotate-hint` overlay shown via `@media (max-width: 600px) and (orientation: portrait)`.

Touch controls: drawn in `drawTouchControls()` only when `isTouchDevice()` is true. Virtual buttons defined in `TB` object using canvas-space coordinates (800×450 space).

## Difficulty scaling

`diffTick` increments each frame during play. `getSpeedMult(dt)` returns enemy speed multiplier. `spawnEnemy()` uses weighted random selection — early game favors radroaches, late game favors raiders. `worldSpeedMult` slows everything when a vault door is near.

## What to avoid

- Do not introduce external files, build steps, or module imports — everything stays in `index.html`
- Do not use `roundRect()` for pixel-art sprite drawing — use `fillRect()` only
- Do not add a new game `state` string for overlays that can be handled with a boolean flag (see Pickle 3000 pattern)
- When changing player hitbox height, always adjust `player.y` by the delta to keep feet at GROUND
- `pressed{}` is cleared at the end of every `update()` branch — consuming inputs in a menu requires `delete pressed[k]` before returning, not after
