# kharnage-replays

Tacticus battle replay files (`*.txt`, base64 -> gzip -> JSON) behind the walkthrough pages at the
`kharnage-replays` Cloudflare Pages site. One folder per playlist, each with its replays,
`replays.csv` (Raidman upload URLs) and, where one exists, the `gallery.json` the page was built from.

The pages are built by the `raidman-replay-gallery` skill in the datamine_tacticus repo
(`.claude/skills/raidman-replay-gallery/`); its `examples/*.json` configs point at these folders.
