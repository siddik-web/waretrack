# WareTrack

**Play it:** https://siddik-web.github.io/waretrack/

A 3D warehouse strategy game that runs in the browser. Manage a distribution center like a strategy game: trucks dock, forklifts put pallets away and pick orders, problems pop up, and you decide how to handle them. Around the warehouse is a town you can buy and build.

Everything lives in a single file, `index.html`. It uses [three.js r128](https://threejs.org/) from cdnjs and Google Fonts. There's no build step and no backend.

## Play locally

Open `index.html` in a browser, or serve the folder:

```bash
npx serve .
# or
python3 -m http.server 8000
```

## Features

- **Warehouse simulation:** trucks queue at the gate, back into docks and unload. Forklifts route through aisles, lift to rack levels, charge their batteries, and pick orders to outbound staging.
- **Shifts and scoring:** 8-hour shifts (about 8 minutes at 1× speed) with three goals, stars, order combos and a points ledger.
- **Five warehouses in one live world:** Northgate DC, Riverside Hub, Eastport Cold Chain, Southfield Cross-Dock and Westgate Robotics Hub sit side by side on one road network. You run one site in full detail; the other four keep working on their own, with trucks queuing, docking, unloading or loading outbound freight, and yard forklifts moving pallets. Each site has its own building style (gable roof, cold store with silos, sawtooth cross-dock, solar-roofed robotics hub). Unlock each one with 2 stars at the previous site.
- **Problems with choices:** breakdowns, spills, rush orders, late trucks, order surges and low stock.
- **Progression:** XP and 8 ranks, 24 achievements, a daily challenge with streaks, upgrades, cosmetics, and a build mode for each warehouse.
- **Town planner:** buy plots, lay roads, and put up houses, shops, offices, depots and parks. Buildings linked to the warehouse by road earn credits, and shops you build become delivery customers. You can also buy the empty Unit 7 to run a second warehouse.
- **Traffic rules:** right-hand driving, lane-following, traffic lights with amber and all-red phases, safe following distance and left-turn yielding. The game's trucks and vans obey them too.
- **Inspector and selection:** tap a truck, forklift, dock bay, charger or pallet to see its details. Selected items get blue corner brackets, a label, and a route line with a destination pin. The follow camera tracks moving vehicles.
- **Site switcher and notifications:** the camera flies between warehouses from the top bar (between shifts, the site you fly to becomes the one you run), or zoom out to a network overview with map pins for every site. Tap any truck, forklift, dock or building at another site to inspect it; the bell lists problems, achievements and promotions.
- **Training shift, sound and juice:** a guided first shift, synthesized sound effects and music, floating points and confetti.

## Controls

| Input | Action |
|---|---|
| Drag / pinch / scroll | Pan and zoom |
| Tap a truck or forklift | Inspect it |
| `Space` | Pause or resume |
| `1` / `2` | Choose an option on a problem card |
| `B` | Build mode |
| `T` | Town planner (`Ctrl/Cmd+Z` to undo) |
| `M` | Menu |
| `/` | Search |

## Saves

Progress (stars, credits, upgrades, builds, town, achievements) is stored in the browser's `localStorage` under `waretrack.game.v2`. Clearing site data resets it. There is also a reset button under Menu → Profile.

## Deploy

It's a static site, so any static host works (Vercel, Netlify, GitHub Pages). With Vercel, import the repo with no framework preset and no build command.
