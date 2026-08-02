# 4K / Anime Library Split — Migration Runbook

## Goal

- `radarr` / `sonarr` (default instances) only ever grab **1080p**.
- `radarr-4k` / `sonarr-4k` handle **4K/2160p** exclusively.
- `sonarr-anime` gets its own folder tree and quality profile.
- Existing 4K content sitting in the default instances gets moved into the
  4K folders and re-registered under the 4K instances.
- qBittorrent keeps seeding everything through the move (see below).

This is app-config + filesystem work on the NAS — none of it lives in the
`moneyball-nas` git repo except the folder listing doc, which is already
updated.

## Before you start: check your hardlink setup (matters for qBittorrent)

All the `*arr` containers and `qbittorrent` share the same `/data` mount, so
the normal setup is: qBittorrent downloads/seeds from `/data/torrents/...`,
and Radarr/Sonarr **hardlink** the finished file into
`/data/media/movies/...` (etc.) rather than copying or moving it. With a
true hardlink, both paths point at the same data on disk — moving or
renaming the *library* copy later has **zero effect** on the *torrent*
copy qBittorrent is seeding from. This is what makes the migration below
safe to do without touching qBittorrent at all.

**Verify this before migrating anything:**

1. In Radarr, Sonarr, Radarr-4k, Sonarr-4k: Settings → Media Management →
   confirm **"Use Hardlinks instead of Copy"** is enabled.
2. Spot-check a couple of existing 4K files by comparing inode/link count
   between the torrents copy and the library copy, e.g.:

   ```
   stat -c '%i %h %n' /volume2/docker/data/torrents/<file>
   stat -c '%i %h %n' /volume2/docker/data/media/movies/<file>
   ```

   If the inode numbers match and link count (`%h`) is ≥2, it's a true
   hardlink — moving the library copy is safe, **no qBittorrent changes
   needed**, skip Step 6 for that file.
3. If a file's link count is `1` (not hardlinked — it was imported via
   **Move**, meaning the library copy *is* the only copy and *is* the file
   qBittorrent is seeding), moving it via Radarr/Sonarr will leave
   qBittorrent pointing at a path that no longer exists. Those files need
   the Step 6 fix-up after they're moved.
4. Never move, rename, or delete anything directly under `/data/torrents/`
   — only touch files under `/data/media/`. That alone keeps qBittorrent
   safe in the hardlinked case.

---

## Step 1: Create the new folder

- [ ] On the NAS, create `/volume2/docker/data/media/tv-anime/`.
- [ ] Check ownership/permissions on the existing `movies-4k` folder
      (`ls -la`) and mirror them on `tv-anime` (likely `1000:100`, `775`).

## Step 2: Reconfigure root folders + quality profiles

Do this first so no *new* content lands in the wrong place while you're
migrating the backlog.

- [ ] **radarr** — root folder `/data/media/movies`. Edit its quality
      profile to only allow up to 1080p (Bluray-1080p, WEBDL-1080p,
      WEBRip-1080p, HDTV-1080p) — remove any 2160p/Remux-2160p entries.
- [ ] **radarr-4k** — root folder `/data/media/movies-4k`. Quality profile
      allows only 2160p qualities (Bluray-2160p, WEBDL-2160p, Remux-2160p).
- [ ] **sonarr** — root folder `/data/media/tv`, profile capped at 1080p.
- [ ] **sonarr-4k** — root folder `/data/media/tv-4k`, profile 2160p-only.
- [ ] **sonarr-anime** — root folder `/data/media/tv-anime` (new). Anime
      quality profile is already configured — just point the root folder
      here.

## Step 3: Migrate existing 4K movies (radarr → radarr-4k)

- [ ] In `radarr`, sort/filter Movies by the Quality column, list everything
      currently at 2160p.
- [ ] Temporarily add `/data/media/movies-4k` as an extra root folder in
      `radarr` (Settings → Media Management → Root Folders).
- [ ] Select the 4K movies → Edit → set Root Folder to
      `/data/media/movies-4k`, check **Move Files** → Save.
- [ ] Select those same movies again → Delete → **uncheck** "Delete Movie
      Files" → confirm. Removes them from `radarr`'s library, files stay.
- [ ] Remove the temporary root folder from `radarr`.
- [ ] In `radarr-4k`, add root folder `/data/media/movies-4k` (if not
      already present), then use **Library Import** ("Import Existing
      Movies") pointed at that folder to match files to the right TMDB
      entries without re-downloading. Set them Monitored.
- [ ] Spot check a few migrated movies in `radarr-4k` for correct
      file/quality recognition (no "missing" flag).
- [ ] Run Step 6 for any of these files that weren't hardlinked.

## Step 4: Migrate existing 4K TV (sonarr → sonarr-4k)

Same pattern as Step 3, using Series instead of Movies, root folder
`/data/media/tv-4k`, and Sonarr's "Import Existing Series" / Manual Import.

**Watch for mixed-quality series**: a series can have some 1080p and some
2160p episode files. A blanket per-series move only makes sense if the
series is predominantly 4K. For mixed series, move the individual episode
files instead and use Sonarr-4k's **Manual Import** on just those — review
the 4K list series-by-series before running a bulk move.

- [ ] Run Step 6 for any of these files that weren't hardlinked.

## Step 5: Migrate existing anime (sonarr/sonarr-4k → sonarr-anime)

For any anime series added before `sonarr-anime` existed:

- [ ] Set root folder to `/data/media/tv-anime` with Move Files.
- [ ] Remove from the old instance (keep files).
- [ ] Add root folder in `sonarr-anime` → Library Import.
- [ ] Run Step 6 for any of these files that weren't hardlinked.

## Step 6: Fix qBittorrent seeding paths (only for non-hardlinked files)

Only needed for files flagged in the "Before you start" check as link
count `1`.

- [ ] In qBittorrent, find the affected torrent(s) — they'll likely show as
      **Missing Files** / errored once the underlying file has moved.
- [ ] Right-click → **Set location** → point it at the file's new path
      under `/data/media/movies-4k/...`, `/data/media/tv-4k/...`, or
      `/data/media/tv-anime/...`.
- [ ] Force a recheck if qBittorrent doesn't pick it up automatically, and
      confirm it resumes seeding instead of re-downloading.
- [ ] Going forward, consider switching these imports to hardlink mode so
      this manual step isn't needed for future migrations.

## Step 7: Jellyseerr

- [ ] Settings → Services: add `radarr-4k` and `sonarr-4k` as additional
      servers with the **4K Server** toggle enabled.
- [ ] Add `sonarr-anime` as its own Sonarr server entry.

## Step 8: Jellyfin libraries

- [ ] Add `/data/media/movies-4k` as an additional folder path on the
      existing Movies library.
- [ ] Add `/data/media/tv-4k` similarly to the Shows library.
- [ ] Add `/data/media/tv-anime` — either as an extra path in Shows
      (simplest) or as a separate "Anime" library if you want a different
      metadata agent.
- [ ] Trigger a library scan.

## Step 9: Permissions verification

- [ ] `ls -la` the new/moved folders. Confirm owner `1000`, consistent
      group across the `*arr` containers (likely `100`), directory mode
      allowing group read/write (e.g. `775`).
- [ ] If anything's off, one recursive fix pass (confirm the real group ID
      first, don't assume `100`):
      ```
  chown -R 1000:100 /volume2/docker/data/media/movies-4k /volume2/docker/data/media/tv-4k /volume2/docker/data/media/tv-anime
      chmod -R 775 /volume2/docker/data/media/movies-4k /volume2/docker/data/media/tv-4k /volume2/docker/data/media/tv-anime
      ```

## Step 10: Docs

- [x] `moneyball-folders.md` updated with `tv-anime/`.

---

## Verification checklist

- [ ] Radarr/Sonarr UI shows migrated titles with correct file, no
      "missing" flag.
- [ ] qBittorrent still shows migrated torrents seeding (not errored).
- [ ] Jellyfin playback works for a migrated 4K title from its new path.
- [ ] Jellyseerr correctly routes a 4K request to the `-4k` instance.
- [ ] `ls -la` on migrated folders shows consistent ownership/permissions.
