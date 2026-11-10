# kharnage-replays

Tacticus replay folders (one per playlist), the walkthrough site in `site/`, and the `raidman-replay-gallery` skill in
`.claude/skills/raidman-replay-gallery/` that builds the pages.

## When the user says a quest battle is a "100%": verify it, don't take it on faith

A claim like "I got 100% on L2" is the user's memory of the in-game result, and it can be wrong. Before adding the replay
as a "100%" card (or counting it in `roster_top`), check it against the battle's bonus requirements:

1. Build the page and read the card. The builder checks each bonus restriction against the units actually deployed
   (`No-Physical Damage lineup`, `No-Chain`, `No-Eviscerating`, ...) and shows a red "bonus not met" chip with the
   offending unit named and the requirement struck through.
2. Check the lightning limit: the builder prints `WARNING not a lightning victory` if the replay took more player turns
   than allowed, and the replay must end `PlayerWon` (not `Surrender`).
3. If either check fails, **tell the user what failed and which unit caused it, and do not label the replay "100%"**.
   Label it by the score the user reports (e.g. "400 out of 500") or ask.
4. Do not use `bonuses_confirmed` in `gallery.json` to silence a failing check just because the user says it was 100%.
   That key is only for a case where the builder's static check is wrong and the replay has been looked at to prove it.
   Say so to the user when using it.

Why this exists: the first "100%" Legendary 2 replay used Snotflogga and Gibbascrapz, who both deal physical damage, against
a No-Physical requirement. The builder flagged it, `bonuses_confirmed` hid the flag, and the page showed a wrong 100% until
the user replaced it with a replay that really was clean.

## Publishing

Uploading to tacticus-raidman.com and pushing to `main` (which deploys the site) are the user's decisions. Do not upload or
push unless asked in that conversation. See the skill's "Before anything public happens".
