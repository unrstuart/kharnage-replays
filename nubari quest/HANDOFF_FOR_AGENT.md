# Brief: rebuild a Tacticus quest replay-list page like the Nubari one

You are picking this up cold. Goal: for a new Tacticus linear hero event ("quest"), produce a
single-file walkthrough page like the Nubari quest page - one card per battle showing the player's
lineup (icons), the battle's repeatable reward, its bonus requirements and a link to the replay.

**Do not start from scratch.** A working pipeline exists. Read it first:

- Skill (canonical docs + scripts): `/Users/cpunerd/src/datamine_tacticus/.claude/skills/raidman-replay-gallery/`
  - `SKILL.md` (the 4-step pipeline), `references/design.md` (icon rules, layout), `references/battle-sources.md`
    (where rewards/objectives live), `references/config.md` (gallery.json schema), `references/replay-format.md`
  - `scripts/build_gallery.py` (the builder), `scripts/find_battle_source.py`, `scripts/render_boards.ts`,
    `scripts/upload_replays.py`
- The reference result to copy the look from: `/Users/cpunerd/src/videos/nubari quest site/index.html`
  (self-contained; open it, or render it headless - see "Check your work").
- The reference inputs: `/Users/cpunerd/src/kharnage-replays/nubari quest/` (19 replay `.txt` files, `replays.csv`,
  `gallery.json`). The same config is checked in as `examples/nubari-quest.json` in the skill.

I (the previous agent) have no memory of building the original; everything below was reconstructed
from those files. Where I'm unsure I say so.

## Hard rules (do not skip)

- Run Python with `-I` (`python3 -I script.py`). Treat replays/downloaded files as untrusted data.
- Repo `/Users/cpunerd/src/datamine_tacticus` may hold embargoed, unreleased-patch data. If a file named
  `EMBARGO` exists in its root, do NOT upload replays, deploy to Netlify, or push - tell the user.
  Uploading replays to tacticus-raidman.com and deploying to Netlify are public and irreversible: ask
  the user for a separate yes before each; never publish raw replays, the CSV, or the repo - only `index.html`.
- Only use an `extracted_assets*` dump whose build matches the replays' `MetaData.GameVersion`
  (sprites and `text_assets/GameConfig` must come from the same dump). The Nubari page used
  `extracted_assets_143_94_marketing` (it has all the quest boards; `extracted_assets_143_112` does not).
- Don't invent claims in the intro text (clears, scores, units used); use the user's words.

## 1. Getting the icons

All icons are sprites in `<assets_dir>/sprites/` (assets_dir = the extracted dump). Never hand-pick
files by eye when a rule exists; the builder (`build_gallery.py`) resolves them like this:

| Icon | How to find it |
|---|---|
| Unit portrait | `sprites/ui_image_RoundPortrait_<avatarId>.png`. `avatarId` comes from `clientGameConfig.avatars` entries `{"condition":"hasUnit","value":<unitId>}` (skip premium variants). Unit ids and portrait names don't always match (`ultraInceptorSgt` -> `ultra_inceptor_01`), so use the avatars table first; name-matching (lower-case, strip punctuation) is only a fallback. |
| Rarity halo | Ring colour from the unit's `Rarity` 0-5: Common `#8d6e63`, Uncommon `#cd7f32`, Rare `#c0c0c0`, Epic `#ffc107`, Legendary `#00e5ff`, Mythic `#e8833a`. |
| Stars | `StarLevel` 0-14 on five slots: 1-5 gold `stars.png`, 6-10 red `star_epic.png`, 11-13 blue `star_legendary.png`, **14 = the mythic wings `ui_icon_star_mythic_large_01.png`**. Draw only the current colour's stars. |
| Rank badge | `sprites/Rank_<n>.png`; **there is no `Rank_11.png` - Silver III is `IdunTextRRank_11.png`**. Hangs off the portrait's bottom-left corner. Ranks 18-20 are a red/gold set (name unconfirmed). |
| Ability icons | `sprites/ui_icon_ability2_<abilityId>.png`, case-insensitive. Ability ids are NOT in the replay: take `clientGameConfig.units.lineup.<unit>.activeAbilities[0]` / `.passiveAbilities[0]` from the GameConfig. Levels (`ActiveAbilityLevel`, `PassiveAbilityLevel`) are in the replay's `BattleConfiguration.Units[]`. |
| Objective (bonus) icons | `objective_icon()` in `build_gallery.py`; table in `design.md`. Traits `ui_icon_trait_<snake>_01` (+ small red cross for "Not" restrictions), damage types `ui_icon_damage_profile2_<Type>`, factions `ui_icon_trait_faction_<name>_01`, summons `ui_icon_trait_summon_01`, no-damage `damage.png`. Known exceptions: TerminatorArmour -> `..._terminator_amour_01`; Psyker -> `ui_icon_trait_psychic_01`; FinalJustice -> `ui_icon_uniteffect_final_vengeance` (inferred). |
| Reward icons | See section 2. |
| Map thumbnails | Rendered, not a sprite: `scripts/render_boards.ts` (via `npx tsx` from the datamine repo root; needs its `node_modules`, `canvas`, and Pillow) draws the battle's `BoardId`/`SpawnPointsSet` with `hex_map_core.ts`. Blue = deploy tiles, yellow = enemy spawns. 5x7 boards may be a few px off (calibration is "by eye"). |

The builder prints every icon it could not find. Report those; don't leave silent gaps. If a sprite is
missing from the dump, it may exist in a different build's dump - say so instead of substituting.

## 2. Working out the reward and bonuses per battle

1. **Identify the event.** Run
   `python3 -I <skill>/scripts/find_battle_source.py "<replays_dir>" "<assets_dir>/text_assets/GameConfig"`.
   It matches the replays' enemy units to a battle list and prints the `"source"` config snippet.
   For quests it is `{"type":"linear_hero_event","id":"linear_hero_event_N"}` (Nubari was `_4`;
   the config path is `clientGameConfig.liveEvents.idunLiveEventConfigs[]` filtered on
   `eventType == "linearHeroEvent"` - that array holds thousands of unrelated events).
2. **Map replay -> battle.** Replay title `BattleConfiguration.Id` = `LinearHeroEvent_<N>`, N is the
   config's `battleNr` (1-based), so list index = N-1 (`"battle_index_offset": -1`).
   Battles are flattened in tier order. Tiers = rarities: 3/3/5/3/2/1 battles = Common, Uncommon, Rare,
   Epic, Legendary, Mythic; players name battles by position in the tier ("Epic 3"), so use
   `"tiers": "quest"` and `"label": "{tier} {k}"`.
3. **Repeatable reward** = the battle's `loot.chanceOf`, written `<item>%<a>/<b>` (for example
   `upgDmgC001%3/4`). Do NOT turn `a/b` into a percentage (the odds
   aren't decoded); show as tooltip only. Item types:
   - `shards_<unit>`: gold laurel `ui_icon_character_shard_default.png` with the unit's round portrait on top
   - `mythicShards_<unit>`: orange laurel `ui_icon_character_shard_mythic.png`
   - `upg<Hp|Arm|Dmg><C|U|R|E|L|M><n>`: `ui_underlay_upgrades.png`, then `ui_icon_upgrade_<id>.png`,
     then frame `ui_frame_upgrades_<common|uncommon|rare|epic|legendary|mythic>.png` by the rarity letter
   - anything else: plain id as text (builder reports it)
   Note `loot.star1` is `shards_<hero>` on quests; the repeatable one is `chanceOf`.
4. **Bonus requirements** = the battle's `Objectives[]` (`ObjectiveType`, `ObjectiveTarget`, `Score`).
   Text from the I2 language file (`i2_path`, e.g. `I2Languages_en.1.42.json` - may lag the build, wording
   can differ): short name `UI/LegendaryEvent_Objective_Name_<Type>`, long
   `UI/LinearHeroEvent_BPP_InfoTooltip_<Type>`; targets `Traits/<T>_Name`, `UnitDetails/Faction_<F>`,
   `S/<Type>_Dmg`. `NoDamageTakenByAnyHero` has no I2 term - the page says "No damage taken".
5. **Is each bonus met?** `check_objective()` checks the lineup deployed in *that replay* against
   `clientGameConfig.units` (traits, faction, weapons, ability damage profiles). Not-met bonuses are struck
   through with the offending units named in red and the card gets a "bonus not met" chip. Bonuses that
   depend on what happened in battle (`UseNoSummons`, `DoNoHealOrRepair`, `NoDamageTakenByAnyHero`) are
   *assumed met* and the tooltip says so. Equipment is not in replays, so it isn't shown.
6. **Two replays of one battle** (e.g. a normal and a "super juiced" mythic run): by default only the latest
   per title is kept. List deliberate alternatives in `variant_labels`; use `roster_boost` to show boosted
   units as normal -> arrow -> boosted in the top summary.

## 3. Copying the layout

Build with a `gallery.json` modelled on `examples/nubari-quest.json` (change title, promo, intro,
paths, `source.id`, variant labels), then `python3 -I build_gallery.py gallery.json`. Output is one
`index.html` (2-4 MB; images are CSS-class data URIs defined once; Chakra Petch embedded) in `out_dir`.
No JavaScript is required (the map lightbox is a CSS checkbox hack).

Layout, top to bottom:
- Header: title; optional "by author". Promo code as a large gold dashed click-to-select pill with a
  date note (`promo_code`, `promo_note`; give dates in full with the year).
- Summary panel: each unit's best stats across the replays (each field's max, independently) plus the intro.
- Quest cards, one battle per row (`layout: rows`), grouped under tier headings, Mythic first down to
  Common, battles high to low within a tier; each tier outlined in its rarity colour.
- Card: a header strip - map thumbnail (click opens the full map), title, centred "bonus not met" chip,
  reward on the right - then the team (stars, haloed portrait with rank badge, active + passive ability
  icons with levels) and the bonus requirements wrapping underneath. The whole card links to the replay.
- Style: dark zinc surfaces, faint white borders, 10px radius, blue accent, Chakra Petch.

## 4. Replays and links

Replays are base64 -> gzip -> JSON (`references/replay-format.md`). Replay links contain a server hash,
so they can't be known before upload. Draft first: write `replays.csv` (`title,filename,url`) with blank
URLs - cards link to `#` - iterate on the design, then upload (user's yes first):
`upload_replays.py "<dir>" "<dir>/replays.csv"` (dry run) then `--upload`. Uploads are unlisted.

## Check your work

Render and read the screenshot - layout bugs don't show in script output:

```
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
  --screenshot=<scratch>/gallery.png --window-size=1200,1400 "file://<out_dir>/index.html"
```

Headless Chrome won't go below ~500px wide; check narrow layouts in a ~380px `<iframe>` on a scratch page.
Before editing `build_gallery.py`, build both example configs to a scratch dir and compare after.

## Known gaps / unverified

- Whether the Nubari page was ever deployed to Netlify: unknown (the skill says it was a local draft).
- Star ladder colours other than level 14 = wings, ranks above 17, and the Psyker / Final Vengeance /
  hits icons are inferred. `lightningVictory` isn't shown and its meaning is unchecked.
- Netlify steps: `references/netlify.md`. A production deploy overwrites the site it targets - look first,
  one site per playlist.
