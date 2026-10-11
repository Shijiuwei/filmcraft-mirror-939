# Project files

FilmCraft projects are `.fcproj` files: UTF-8 JSON with an explicit schema version. Reading and
writing them is `crates/format` (`filmcraft-format`); the engine's `file.*` commands use it.

## Format

```json
{
  "format": "filmcraft.project",
  "schema_version": 13,
  "generator": "FilmCraft 0.1.0",
  "project": { "name": "…", "settings": { … }, "root": { … }, "items": { … }, "next_id": 48 },
  "view": { "open_sequences": [21, 50], "active_sequence": 50, "sequences": { "21": { "pps": 40.0, "scroll": 0.0, … } } }
}
```

- `schema_version` is the version of the whole document. This build writes
  `filmcraft_format::SCHEMA_VERSION` and reads every version from 1 up to it.
- `project` is the serialized `filmcraft_project::Project` (bins, media clips, sequences, tracks,
  clips, effects, keyframes, markers). Media is referenced by path, never embedded.
- Files are written compact (no indentation). A 2,000-clip project is about 1.4 MB.
- `view` (optional) is what was open when the project was saved: the Timeline's sequence tabs in
  order, the active one, each sequence's zoom, scroll and track heights, and each sequence's
  playhead (`filmcraft_project::ProjectView`). It is beside the project, not in it: it is not part of the
  edit, never an undo step, and changing it does not mark the project as changed. Opening a
  project restores it when the Timeline preference "Restore open sequences when opening projects"
  is on. It is outside the schema version: older builds ignore it, ids that are not sequences and
  numbers out of range are dropped or clamped (a playhead also snaps to a frame of its sequence), and a `view` that cannot be read is ignored (the
  project opens on its first sequence, as it does without one or with no open sequence in it).
- A coordinate of an effect's point parameter may be `null`. Point parameters use NaN for "auto"
  (the frame centre, or the source centre for `anchor`), and JSON writes NaN as `null`; it reads back
  as "auto". Placing a clip resolves these points at once. An interchange import resolves them
  too, except the `anchor` of a clip whose media file was not found. That one stays "auto", and
  the renderer centres it, until Link Media (`media.relink`, `media.autoRelink`) reads the file
  and with it the real picture size; a file that merely comes back at its old path does not
  change it. `null` anywhere else is a damaged file.

### Schema history

| Version | Since | Shape |
|---|---|---|
| 1 | M0 | the bare `Project` object with a `"version": 1` field |
| 2 | M11.1 | the envelope above; the project's own `version` field is gone |
| 3 | M10.2 | graphic clips (`ItemKind::Graphic` with text/shape layers); a no-op step for older data |
| 4 | M7.4 | Essential Sound: clip audio types and settings; no-op step |
| 5 | M8.8 | colour management: working space, Interpret Footage colour space, LUT library; no-op step |
| 6 | M11.7/M11.9 | media identity fingerprints, ingest settings; no-op step |
| 7 | M5.5 | effect masks: `masks` on effect instances with keyframable Bézier `Path` values; no-op step |
| 8 | M12 | multi-camera source sequences and clips, merged clips; no-op step |
| 9 | M10.5 | transcripts of media items (`project.transcripts`); no-op step |
| 10 | M3.10 | clip time interpolation, Hold Filters, Field Options, audio source channels, Modify ▸ Audio Channels map, subclip Restrict Trims; no-op step |
| 11 | M3.11 | search bins (`project.search_bins`), Flash Cue markers, Project Settings safe areas, capture format and scratch disks; no-op step |
| 12 | M10.7 | graphics design data: `TrackItem::graphic` (roll / crawl, responsive time, template link), `EffectInstance::layer` (layer uid, per-character styles, responsive pins), `project.source_graphics`; no-op step |
| 13 | Multi-stream audio | `MediaInfo::audio_streams` and `TrackItem::audio_stream`; older `audio` objects load as stream zero, missing clip indices default to zero; no-op step |

### Migrations

Older files are upgraded on load, in memory, by a chain of single-step functions
(`filmcraft_format::MIGRATIONS`, where entry *i* upgrades v*i+1* to v*i+2*). Each step rewrites the JSON
document; after the last one the result is deserialized into the current model. To change the schema:

1. add a function `vN_to_vN1(doc: Value) -> Result<Value, String>` and append it to `MIGRATIONS`
   (`SCHEMA_VERSION` follows automatically);
2. add a small hand-made fixture of the old shape to `crates/format/tests/fixtures/` and a test that it
   loads;
3. never edit a migration that has shipped.

Fields added with `#[serde(default)]` don't need a migration: missing values take their defaults.
But when new data would make the file unreadable to an older build (a new enum variant such as
`ItemKind::Graphic`, a new required shape), bump the schema with a no-op step so older builds refuse
the file with the message below instead of failing with a parse error. Do the same when an older
build would load the file but silently drop new fields on its next save.

When you open an older file FilmCraft says so ("Upgraded project from schema v1 to v2"). The file
on disk is unchanged until you save. The **first save over it** keeps the original next to it as
`<name> (schema v1 backup).fcproj`, so an older FilmCraft can still open your work.

### Newer files

A file with a `schema_version` newer than the build supports is **refused**, not half-read:

> this project was saved by a newer version of FilmCraft (project schema v12); this build reads up to
> v11. Update FilmCraft to open it.

Reading it partially and saving it back would silently drop whatever the newer version added.

## Saving is atomic

Every write of a project file, auto-save, recovery snapshot or preferences file goes through
`filmcraft_format::atomic_write`:

1. write the bytes to a hidden temp file in the **same directory** (`.<name>.<pid>-<n>.tmp`);
2. `fsync` the temp file;
3. rename it over the target (an atomic replace on every supported OS);
4. `fsync` the directory so the rename itself is durable.

A crash, `kill -9` or power cut at any point leaves either the complete old file or the complete new
one. If a step fails, the temp file is removed and the old file is untouched.

## Commands

| Id | Menu | |
|---|---|---|
| `file.save {path?}` | File ▸ Save (⌘S) | write to the project's path (or `path`) |
| `file.saveAs {path}` | File ▸ Save As… (⇧⌘S) | write and adopt the new path |
| `file.saveCopy {path}` | File ▸ Save a Copy… (⌥⌘S) | write a copy; the project keeps its path and unsaved state |
| `file.revert` | File ▸ Revert | reload the last saved version, discarding changes |
| `file.open {path}` | File ▸ Open Project… (⌘O) | returns `{schemaVersion, migrated}` |

In the UI, Save / Save As / Save a Copy without a `path` show a file dialog.

## Auto Save

Preferences ▸ Auto Save (Edit ▸ Preferences ▸ Auto Save…, ⌘,; FilmCraft ▸ Settings… on macOS) works
like Premiere's:

| Setting | Key | Default |
|---|---|---|
| Automatically save projects | `autoSave.enabled` | on |
| Automatically Save Every N minute(s) | `autoSave.intervalMinutes` | 5 (1–1440) |
| Maximum Project Versions | `autoSave.maxVersions` | 20 (1–1000) |
| Auto Save also saves the current project(s) | `autoSave.saveCurrentProject` | off |
| Keep a recovery copy of unsaved changes | `autoSave.recoveryJournal` | on |
| Update it at least every N second(s) | `autoSave.recoveryIntervalSeconds` | 5 (1–600) |

At each interval, if the project changed since the last auto-save, a copy is written to the
`Auto-Save` folder next to the project as `<name>-YYYY-MM-DD_HH-MM-SS.fcproj` (local time). An
unsaved project uses `<data dir>/Auto-Save/`. After each write the oldest copies beyond the limit are
deleted. Only files named exactly `<name>-<timestamp>.fcproj` are ever deleted. An auto-save is an
ordinary project file: open it with File ▸ Open Project. `file.listAutoSaves` lists them, newest
first.

Preferences are stored in `<data dir>/preferences.json`. The `prefs.*` commands read and change them.

## Crash recovery

Premiere's auto-save can lose up to one interval of work. FilmCraft also keeps a **recovery journal**:

- Whenever the project has unsaved changes, a snapshot is written to
  `<data dir>/Recovery/<started>-<pid>[-<n>]/snapshot.fcproj`, plus `session.json` (time, revision, project
  name and path). It is written about 1 s after the last edit, or after at most
  `recoveryIntervalSeconds` while edits keep coming. Saving the project deletes the snapshot.
- Each running FilmCraft holds an OS file lock on `<session>/lock`. When the process dies (crash,
  `kill -9`, power loss) the OS releases the lock.
- At startup, every other session directory whose lock can be taken and that has a snapshot is a
  recovery candidate. FilmCraft asks: *"FilmCraft quit unexpectedly while 'X' had unsaved changes.
  Recover unsaved changes from 2026-10-01 00:02:09?"*, with the buttons **Recover**, **Not Now** and
  **Discard**. Stale directories without a snapshot are removed. Journals of running sessions are
  never touched.
- Recovering loads the snapshot as an **unsaved** project with its original path, so Save writes
  over the original file. The recovered state goes into the new session's own journal before the old
  journal is deleted.
- A normal quit with nothing unsaved removes the session directory. Quitting with unsaved changes
  (Quit, Cmd+Q or the window's close button) first asks *"Save changes to 'X' before closing?"*
  with **Cancel**, **Don't Save** and **Save** (Save on a never-saved project opens the Save dialog);
  Don't Save discards the changes and their snapshot. A quit that does not ask (the control
  channel's `app.quit`, the OS ending the session) keeps the snapshot, marked `cleanExit`, and FilmCraft offers it on the next launch ("FilmCraft was
  closed while … had unsaved changes").

Desktop flags: `--recover` recovers the newest candidate without asking. `--no-recover` starts
without asking; the changes stay available through File ▸ Recover Unsaved Changes…. `--data-dir
<dir>` (or `FILMCRAFT_DATA_DIR`) moves the data directory. The default is `~/Library/Application
Support/FilmCraft` on macOS, `%APPDATA%\FilmCraft` on Windows, and `$XDG_DATA_HOME/filmcraft` or
`~/.local/share/filmcraft` elsewhere. A `portable.txt` file next to the executable (the Windows
portable zip ships one) makes it `data` next to the executable instead.

### Cost

The UI thread only gives the worker thread an `Arc<Project>`: the immutable snapshot the engine
already keeps for undo, passed through a channel. JSON encoding and all file I/O run on the
`filmcraft-autosave` thread. Measured on the development Mac (release build) with a 2,000-clip project
(`cargo test --release -p filmcraft-format --test fixtures -- --nocapture`):

| Step | Time | Size |
|---|---|---|
| encode, compact (what is written) | 2.5 ms | 1.4 MB |
| encode, pretty (not used) | 7.1 ms | 5.6 MB |
| decode | 4.6 ms | |
| atomic write + fsync (APFS SSD) | 30 ms | |

In the running app with the demo project, a recovery write measured 0.1 ms to encode and 27 ms to
write (two fsynced files).

## Commands

| Id | |
|---|---|
| `file.recoveryList` | candidates: `id`, `projectName`, `projectPath`, `savedAt`, `revision`, `cleanExit`… |
| `file.recover {id?}` | recover (newest when no `id`). From the UI, `{}` opens the recovery dialog |
| `file.discardRecovery {id?, all?}` | delete a candidate's journal |
| `file.autoSaveNow` | write an auto-save now (if anything is unsaved) |
| `file.autoSaveStatus` | prefs, folders, last auto-save / journal times, last encode/write ms and bytes |
| `file.listAutoSaves` | auto-save files of the current project, newest first |
| `prefs.get {key?}` / `prefs.set {key, value}` or `{values:{…}}` / `prefs.reset {category?}` | settings (below) |

UI automation ids: `recovery.recover`, `recovery.later`, `recovery.discard`, `recovery.item.<n>`;
`revert.yes`, `revert.no` (File ▸ Revert asks first when invoked without params).

## Settings

Premiere's Settings dialog (app menu ▸ Settings ▸ <category> on macOS, Edit ▸ Preferences
elsewhere; General is Cmd+,; the window is titled "Preferences"). The values are user preferences,
not project data: they live in `preferences.json` in the per-user data directory (written
atomically on every change) and never touch the `.fcproj` schema. The file carries a `version`
(currently 2); older files are migrated on load, unknown dropdown values, bad colours and
out-of-range numbers are repaired to defaults/limits, and an unreadable file falls back to the
defaults.

Every value is a dotted key whose first segment is its category: `timeline.stillImageDuration`,
`labels.colors.rose.name`. `prefs.get` / `prefs.set` / `prefs.reset {category?}` reach them from the
CLI, the control channel and MCP; `prefs.set` rejects values that are not one of a dropdown's
choices. `prefs.schema {category?}` lists every category with its fields (label, kind, choices,
range, unit, current value, and `wired`: whether FilmCraft acts on it yet). `app.settings.<category>`
opens the dialog on a page.

| Category | Takes effect |
|---|---|
| General | At Startup (Show Home = demo project, Open Most Recent, empty project; recent projects are remembered on open/save), Show Tool Tips |
| Appearance | Appearance Mode `appearanceMode` (`auto` = Sync with system / `dark` (default) / `light`; the header button and View ▸ Appearance ▸ Next Appearance Mode cycle it), Dark Theme `darkTheme` (`darkest` (default) / `dark`), Light Theme `lightTheme` (`light`), highlight colour, accessible contrast. `colorTheme` (Darkest / Dark / Light) is legacy input: setting it (older files and clients, View ▸ Appearance ▸ a theme) selects that theme and fixes the mode to its family; a file without `appearanceMode` migrates to its theme's fixed mode. It is kept in line with the choices: the theme a fixed mode shows, the dark theme in Auto |
| Audio | Automatch Time, Large Volume Adjustment, automation keyframe thinning (linear, minimum time) |
| Audio Hardware | device class (cpal host), output device, I/O buffer size, sample rate, force document rate, Output Mapping (programme L/R → device channels) |
| Auto Save | the auto-save ring and the crash-recovery journal |
| Color | stored only (display colour management, EDR monitoring, HDR graphics white) |
| Graphics | new text layers: smart quotes, ligatures, default font |
| Labels | the 16 label names and colours (timeline, Project panel, Edit ▸ Label), label defaults for imported movies / video / audio / stills and new sequences |
| Media | Indeterminate Media Timebase (frame rate of stills), Default Media Scaling (Scale to / Set to frame size when a clip of another size is edited in), Enable proxies |
| Media Analysis & Transcription | auto-transcribe imported clips (scope "all imported"), speech model, default language / auto-detect, speaker labelling (defaults of `transcript.generate`) |
| Media Cache | location (`<data dir>/Media Cache` by default; unsaved projects' render previews live there), automatic deletion (older than N days / beyond N GB), Delete… (`mediaCache.clean {all?}`, `mediaCache.info`) |
| Memory | frame cache budget of the monitors and thumbnails |
| Playback | preroll / postroll (Play Around, trim loop), Step forward/back many |
| Plugins | stub (no plugins) |
| Timeline | video / audio transition and still image default durations (frames or seconds), playback auto-scrolling (none / page / smooth), snap playhead, return to beginning at playback end, play after rendering previews |
| Trim | Large Trim Offset, Selection tool picks roll (on the cut) / ripple (on the edge) without a modifier, playhead determines trim loop |

The remaining fields are kept for parity and shown, but have no effect yet (`wired: false`).

UI automation ids: `settings.category.<id>`; `settings.<key>` for every control (for example
`settings.timeline.autoScroll`, `settings.labels.colors.rose.color`); `settings.<key>.<value>` for
the items of an open dropdown; `settings.<key>.browse`, `settings.<key>.hex`;
`settings.mediaCache.clean`; `settings.help`, `settings.reset` (this page back to its defaults),
`settings.cancel`, `settings.ok`. The open dialog's page and draft are `ui.settings` in `ui.inspect`;
`ui.set {"settings": {"page": id, "values": {key: value}}}` edits the draft (OK applies it).

## Media: offline, relinking, proxies, ingest

Media is referenced by path. Each imported file also records its **identity**
(`MediaClip::identity`): the size and a 64-bit fingerprint of the size, the first MiB and the last
MiB. Reading 2 MiB is cheap on any file, and it tells a moved original from a different take with
the same name. The field is optional (`skip_serializing_if`); it and `ProjectSettings::ingest` came with schema
v6, so builds that would drop them on save refuse the file instead. Projects from older builds
have no identity, so relinking them skips the fingerprint check.

### Offline media

- **Opening a project** checks every file (`media.findMissing`). Missing items don't stop the open.
  They render the **offline slate** (`crates/render/src/offline.rs`, our own design: a red striped
  field, a warning triangle, "MEDIA NOT FOUND", the file name and a hint), their audio is silent,
  and the app opens the **Link Media** dialog. `file.open` returns `missingMedia`.
- **Make Offline** (`media.makeOffline {items?, deleteFiles?}`) sets `MediaClip::offline`. The clip
  then shows "MEDIA SET OFFLINE" even though the file exists, until it is linked again. Files are
  deleted only with `deleteFiles: true`.
- The media pool caches sources per item **and** per reference (path, offline flag). So relinking,
  Make Offline and undoing either take effect on the next frame.

### Link Media

| Command | |
|---|---|
| `media.findMissing` | `{missing: [{item, name, fileName, path, status: missing\|offline\|proxyMissing, duration, startTimecode}]}` |
| `media.status {item?}` | per item: `online`, `missing`, `offline`, `unreadable` or `generated`, plus identity and proxy |
| `media.linkMedia` | File ▸ Link Media… (rescans; the UI opens the dialog) |
| `media.relink {item, path, force?, relinkOthers?=true, alignTimecode?, match?}` | check, link, then the others |
| `media.autoRelink {from, to}` / `{folder}` | batch relink: a folder-prefix remap, or a search of a folder tree |
| `media.search {folder, item?, exactName?=true}` | ranked candidates with `ok`, `identityMatch`, `problems` |
| `media.replaceFootage {item, path}` | relink without checks (Replace Footage) |
| `media.offlineAll` | close the dialog and leave the rest offline |

`match` selects which properties must agree. Each defaults as shown:

- `fileName` (true): the file name stem;
- `extension` (true);
- `clipId` (true): the fingerprint; a mismatch is refused with "not the same file…";
- `duration` (true): within one frame;
- `mediaStart` (false): the start timecode;
- `metadata` (true): frame size, frame rate and audio channels.

`force: true` links regardless. **Relink others automatically** works out the folder remap from the
file just linked, by dropping the common trailing path components (`/Volumes/A/shoot/a.mov` →
`/Users/me/shoot/a.mov` gives `/Volumes/A` → `/Users/me`). It applies the remap to the other missing
files and checks each one. It also looks in the new file's folder. Windows and macOS separators both
work. **Align Timecode** keeps clips on the same timecode when the new file starts at a different
timecode: the clips' source in-points, keyframes, marks and markers move with it
(`Project::shift_media_time`). One relink, together with the files it pulls along, is one undo step.

Link Media dialog automation ids: `linkMedia.row.<n>`,
`linkMedia.match.<fileName|extension|clipId|duration|mediaStart|metadata>`, `linkMedia.alignTimecode`,
`linkMedia.relinkOthers`, `linkMedia.folder`, `linkMedia.browse`, `linkMedia.exactName`,
`linkMedia.search`, `linkMedia.candidate.<n>`, `linkMedia.preview`, `linkMedia.link`,
`linkMedia.locate`, `linkMedia.offline`, `linkMedia.offlineAll`, `linkMedia.cancel`.
Make Offline: `makeOffline.keep|delete|ok|cancel`. In the Project panel, offline items get a
broken-link badge (`project.item.<id>.offline` in Icon view), and items with a proxy get a **P**
badge (`project.item.<id>.proxy`).

### Proxies

| Command | Menu | |
|---|---|---|
| `media.createProxies {items?, preset?, destination?, attach?=true, wait?}` | Clip ▸ Proxy ▸ Create Proxies… | background job; attaches when done |
| `media.attachProxies {item, path}` / `{items, paths}` | Attach Proxies… | duration within one frame and the same frame rate are required; size and aspect may differ |
| `media.detachProxies {items?}` | Detach Proxies | |
| `media.reconnectFullRes {item, path}` | Reconnect Full Resolution Media… | link the full-resolution file of a clip that has a proxy |
| `media.toggleProxies {enabled?}` | View ▸ Toggle Proxies, monitor button | Preferences ▸ Media ▸ `media.enableProxies` |
| `media.proxyPresets` | | |

The presets use our own encoders:

- `prores_proxy_quarter` (default), `prores_proxy_half`, `prores_lt_half`: ProRes 422 Proxy or LT
  in MOV with PCM audio;
- `h264_quarter`, `h264_half`: H.264 + AAC in MP4.

Proxies go to `<media folder>/Proxies/<name>_Proxy.<ext>`, or to the chosen folder. Job progress is
in `jobs.list`. `Session::poll_persistence`, which the app calls every frame, attaches finished
proxies.

With proxies enabled, monitors, thumbnails and playback read the proxy. The proxy source reports
the original's size. The compositor derives its pixel scale from the frame it gets, so Motion and
pixel-size effect parameters (blur radii…) give the same picture at the proxy's resolution. A
320×180 clip with Motion and Gaussian Blur and a ½-size ProRes Proxy measures 44 dB PSNR against
full resolution (48 dB at ½ playback resolution). **Export always uses full resolution**
(`MediaPool::full_res_provider`). Toggling proxies bumps the revision so caches refresh, but it is
not an edit and doesn't mark the project as modified.

**Cost.** These numbers are from `proxy_playback_perf_4k`, an ignored test: `RAYON_NUM_THREADS=1
cargo test --release -p filmcraft-engine proxy_playback_perf_4k -- --ignored --nocapture`. It plays a
2 s 3840×2160 H.264 clip (ffmpeg `testsrc2`, libx264) on one core of the development Mac and reports
milliseconds per frame:

| | decode (GPU path) | CPU composite at ½ | CPU composite at ¼ | proxy creation |
|---|---|---|---|---|
| full resolution | 9.2 | 277 | 201 | |
| ProRes Proxy ¼ | 8.9 | 150 | 35 | 17.8 s |
| H.264 ¼ | 0.7 | 131 | 48 | 8.9 s |

The test pattern is unusually cheap for H.264 to decode, so real camera footage gains more from
proxies. The CPU composite (YUV→linear conversion and resampling) is what proxies cut most:
5.7× at ¼ resolution, where the proxy needs no resampling.

Create Proxies dialog ids: `proxies.preset.<id>`, `proxies.destination`, `proxies.browse`,
`proxies.ok`, `proxies.cancel`. Monitor button: `program.transport.media.toggleProxies` (and
`source.…`). It is lit while proxies are on.

### Ingest settings

`project.ingestSettings {enabled, action: copy|transcode|createProxies|copyAndCreateProxies,
destination?, preset?}` (Project Settings ▸ Ingest) acts on every `file.import`:

- **copy** copies the file to the destination (default `<media folder>/Ingested Media`), checks the
  copy's fingerprint against the original, and then uses the copy;
- **transcode** writes the preset's format (default ProRes 422 LT) in the background and switches the
  clip to the result when it is done;
- **createProxies** makes proxies in the background and attaches them;
- **copyAndCreateProxies** does both.

`file.import` reports this under `ingest`.

## Project Manager

`file.projectManager` (File ▸ Project Manager…) makes a self-contained copy of a project:

```json
{"destination": "/path/Folder", "mode": "collect|consolidate", "sequences": [id], "excludeUnused": true,
 "handles": 30, "preset": "prores_lt", "includeProxies": true, "includePreviews": false,
 "projectName": "…", "dryRun": false, "overwrite": false, "wait": false}
```

- **collect** copies every media file that the chosen sequences use (following nested sequences and
  subclips), plus their proxies and render previews if asked.
- **consolidate** writes only the used range of each movie or audio file, plus `handles` frames on
  each side, with a transcode preset (`prores_lt`, `prores_hq`, `h264`). It then re-bases the clips'
  media time, so every edit, keyframe and marker stays in place. Stills are copied.
- **excludeUnused** drops media items and sequences that the chosen sequences don't use.
- **dryRun** returns the plan and the disk-space estimate (`originalBytes`, `resultBytes`, `files`)
  without writing anything.

The copy runs as a job. The project file (`<destination>/<name>.fcproj`) is written last, with the
new paths and fingerprints, so it opens with no missing media.

Dialog ids: `pm.seq.<id>`, `pm.mode.<collect|consolidate>`, `pm.preset.<id>`, `pm.excludeUnused`,
`pm.handles`, `pm.includeProxies`, `pm.includePreviews`, `pm.destination`, `pm.browse`,
`pm.calculate`, `pm.sizes`, `pm.ok`, `pm.cancel`.

## Effect presets

Effect presets (Effects panel ▸ Presets; Effect Controls ▸ right-click an effect ▸ Save Preset…) are
not part of the project. Built-in presets are defined in code (`filmcraft_engine::presets`); user
presets live in `<data dir>/effect-presets.json`, and `presets.export` / `presets.import` read and
write the same JSON format:

```json
{"format": "filmcraft.effect-presets", "version": 1, "presets": [
  {"name": "My Vignette", "description": "", "keyframes": "AnchorToIn",
   "source_duration": 2540160000000, "source_size": [1920, 1080],
   "effects": [ /* EffectInstance, exactly as in .fcproj: effect, enabled, params, masks */ ]}
]}
```

Keyframe times are relative to the in point of the clip the preset was saved from
(`source_duration` is that clip's media length in ticks). Applying re-times them: `Scale` stretches
them over the target clip, `AnchorToIn` / `AnchorToOut` keep their distance from the target's in /
out point. Point parameters, mask paths, feather and expansion are scaled by the target / source
frame-size ratio. Intrinsic effects (Motion, Opacity…) replace the clip's own instance; other
effects are added. Applying is one undo step; saving, renaming, deleting and importing change the
library, not the project, and are not undoable. Files with another `format`, a newer `version` or
unknown effect ids are refused.


---

<details open>
<summary><b>📚 filmcraft 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「filmcraft」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 filmcraft 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [基于 filmcraft 的高吞吐 Rust高性能网络 设计白皮书](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [现代 Rust高性能网络 架构演进之路 —— filmcraft 深度实践](https://xhml.tcti.cn/xuexi/subscribe-10409095.html)
* [面向大规模网络的 filmcraft 工业级架构基准](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (v2.0-GA)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【filmcraft】pure 核心系统架构与设计规约 (Draft-05)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.6)](https://www.mw-wm.com/yinqing/navigation-02832935.html)
* [基于 filmcraft 的高吞吐 开源镜像站 设计白皮书](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-81)](https://vxjm.wtpuscm.cn/chanpin/interface-270596.html)
* [【官方规范】filmcraft open-source 核心运行拓扑标准](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [Trim Monitor 核心系统架构与设计规约 (v2.0-GA)](https://tjqa.tcti.cn/yingyong/prospect-78885658.html)
* [基于 filmcraft 的高吞吐 分布式架构 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [现代 分布式架构 架构演进之路 —— filmcraft 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [【官方规范】filmcraft filmcraft 核心运行拓扑标准](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [Effects and motion 核心系统架构与设计规约 (RFC-633)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [Formats and codecs 核心系统架构与设计规约 (Node-44)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Node-17)](https://www.poykq.cn/redian/tutorial-673489.html)
* [Interchange 核心系统架构与设计规约 (RFC-715)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.7)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [Built for agents 核心系统架构与设计规约 (Node-34)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.8)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Draft-08)](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v2.8)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [Exact time. 核心系统架构与设计规约 (v2.0-GA)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [filmcraft 内部组件解耦与事件状态机规范 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [filmcraft 分布式数据通道与 搜索引擎调度优化 技术规范 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (RFC-691)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.4)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [filmcraft 分布式数据通道与 分布式架构 技术规范 (RFC-545)](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [filmcraft 分布式数据通道与 Rust高性能网络 技术规范 (Node-72)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [基于 filmcraft 的高吞吐 搜索引擎调度优化 设计白皮书](https://www.poykq.cn/redian/tutorial-673489.html)
* [【filmcraft】Adobe 核心系统架构与设计规约 (Draft-08)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 分布式数据通道与 开源镜像站 技术规范 (Spec-v1.3)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Spec-v1.1)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [【filmcraft】Premiere 核心系统架构与设计规约 (v2.0-GA)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [生产环境运维调优手册 核心系统架构与设计规约 (Spec-v1.1)](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [Effects and motion 核心系统架构与设计规约 (v2.0-GA)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 内部组件解耦与事件状态机规范 (Node-29)](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [filmcraft 插件生态规范与 Built for agents 扩展手册 (Node-18)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [基于 filmcraft 的自动化部署与生产环境配置实践](https://ankmz.cn/anquan/tag-68558329.html)
* [【生产手册】filmcraft 模块通信与请求穿透标准](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696.html)
* [filmcraft 异步中间件流水线与 reimplementation 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft vs 业界主流方案：分布式架构 深度技术选型对比](https://unqt.wtpuscm.cn/zixun/contact-59980425.html)
* [【集成指南】分布式架构 服务端接入准则与 filmcraft 实战](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [filmcraft 插件生态规范与 filmcraft 扩展手册 (Node-14)](https://lase.tcti.cn/fenxi/lesson-67970267.html)
* [filmcraft 异步中间件流水线与 Premiere 接入规范](https://www.mw-wm.com/fenxi/travel-06970313.html)
* [filmcraft: Premiere 分布式架构 (Node-37)](https://cips.wtpuscm.cn/yunying/hotel-382.html)
* [filmcraft 异步中间件流水线与 Every trim. 接入规范](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (Spec-v2.8)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345.html)
* [filmcraft 插件生态规范与 Adobe 扩展手册 (Draft-06)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [filmcraft 插件生态规范与 Pro 扩展手册 (Core/Pro)](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (Verified)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [filmcraft 异步中间件流水线与 Get started 接入规范](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 核心 API 接口契约与客户端调用指南](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (Core/Exact )](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [filmcraft vs 业界主流方案：开源镜像站 深度技术选型对比](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [filmcraft vs 业界主流方案：搜索引擎调度优化 深度技术选型对比](https://www.yx-sf.com/tech/28369.html)
* [filmcraft 插件生态规范与 clean-room 扩展手册 (Core/clean-)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft 异步中间件流水线与 分布式状态机一致性 接入规范](https://pdvy.tcti.cn/gongxiang/segment-83277821.html)
* [【集成指南】Rust高性能网络 服务端接入准则与 filmcraft 实战](https://dloe.spidernode-mesh.net/chuangxin/products-47216067.html)
* [filmcraft 异步中间件流水线与 Effects and motion 接入规范](https://fglw.wtpuscm.cn/jianzhan/data-59540835.html)
* [filmcraft 插件生态规范与 Export 扩展手册 (Draft-08)](https://iuuv.tcti.cn/hezuo/app-80239016.html)
* [filmcraft 插件生态规范与 Every trim. 扩展手册 (v2.0-GA)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [filmcraft 异步中间件流水线与 Pro 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft 异步中间件流水线与 可信存活健康度量 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [filmcraft 异步中间件流水线与 Audio 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 插件生态规范与 Trim Monitor 扩展手册 (v2.0-GA)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [filmcraft 异步中间件流水线与 Rust 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [【集成指南】搜索引擎调度优化 服务端接入准则与 filmcraft 实战](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [filmcraft 异步中间件流水线与 生产环境运维调优手册 接入规范](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft: Audio 分布式架构 (Draft-02)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [filmcraft 插件生态规范与 Exact time. 扩展手册 (v2.0-GA)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft 异步中间件流水线与 Interchange 接入规范](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft 插件生态规范与 built 扩展手册 (Core/built)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [filmcraft 插件生态规范与 Edit 扩展手册 (RFC-149)](https://ankmz.cn/anquan/tag-68558329.html)

#### 3. ⚡ filmcraft 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [全球权威拓扑节点：filmcraft 实时镜像与索引入口](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：filmcraft Pro 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [filmcraft 亚太与欧美多活集群数据同步中枢](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [【镜像入口】filmcraft 官方毫秒级实时数据广播节点](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [冷热数据分层镜像：filmcraft Audio 权威归档源](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)
* [filmcraft 官方高可用镜像注册节点 (Verified)](https://owfm.tcti.cn/shangye/education-63674207.html)
* [filmcraft 官方高可用镜像注册节点 (RFC-877)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 官方高可用镜像注册节点 (Node-76)](https://ukvo.spidernode-mesh.net/anli/download-57719010.html)
* [filmcraft 去中心化数据同步源与拓扑寻址规约](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 官方高可用镜像注册节点 (Core/Every )](https://tfcb.wtpuscm.cn/yanjiu/review-64634682.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.0)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：filmcraft reimplementation 权威归档源](https://owfm.tcti.cn/shangye/education-63674207.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-842)](https://www.poykq.cn/redian/tutorial-673489.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Draft-03)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [冷热数据分层镜像：filmcraft Color 权威归档源](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v2.1)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [冷热数据分层镜像：filmcraft Exact time. 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-443)](https://aagj.tcti.cn/jianzhan/audience-84449488.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://ytxd.wtpuscm.cn/baogao/price-87851711.html)
* [冷热数据分层镜像：filmcraft Titles and captions 权威归档源](https://keil.spidernode-mesh.net/yunying/server-16840384.html)
* [冷热数据分层镜像：filmcraft Formats and codecs 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [filmcraft 官方高可用镜像注册节点 (Draft-07)](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.2)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [filmcraft 官方高可用镜像注册节点 (Node-67)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Verified)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [冷热数据分层镜像：filmcraft Edit 权威归档源](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [冷热数据分层镜像：filmcraft 模块化解耦与协议标准 权威归档源](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [冷热数据分层镜像：filmcraft Export 权威归档源](https://vbze.tcti.cn/kaifa/game-83846984.html)
* [冷热数据分层镜像：filmcraft clean-room 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (RFC-111)](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964.html)
* [冷热数据分层镜像：filmcraft built 权威归档源](https://kvaf.tcti.cn/yunsuan/settings-81875461.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (Spec-v1.2)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [filmcraft 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [冷热数据分层镜像：filmcraft dynamic trimming 权威归档源](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [filmcraft 官方高可用镜像注册节点 (v2.0-GA)](https://xwia.spidernode-mesh.net/shangye/services-26058978.html)
* [冷热数据分层镜像：filmcraft Interchange 权威归档源](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft 官方高可用镜像注册节点 (Spec-v1.8)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [面向生产级运行的 filmcraft 稳定性防护白皮书 (v2.0-GA)](https://yxso.wtpuscm.cn/shuju/news-090911.html)
* [【评测基准】filmcraft 吞吐抖动度量与健康检查协议](https://ovzp.wtpuscm.cn/xuexi/report-93713076.html)
* [filmcraft 权威网络权重传递与收录基准规范](https://ncrw.tcti.cn/kaifa/growth-85374259.html)
* [filmcraft 高负载场景下 Rust高性能网络 基准评测报告](https://api.pvucttcv.cn/yinyue/innovation-37390655.html)
* [filmcraft 节点连通性、存活性探测与防作弊指标](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Verified)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [filmcraft 高负载场景下 分布式架构 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/filmcr)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [filmcraft 高负载场景下 开源镜像站 基准评测报告](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-01)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [filmcraft 故障自愈与网络拓扑重构实践](https://gdtq.spidernode-mesh.net/keji/account-53764057.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.8)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-465)](https://fjoc.tcti.cn/yingyong/contact-00610313.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-02)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.7)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-04)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://yyns.spidernode-mesh.net/wendang/review-40421998.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Core/Pro)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-08)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-45)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Draft-04)](https://www.poykq.cn/redian/tutorial-673489.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v1.0)](https://fjac.spidernode-mesh.net/anli/investment-94283191.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-939)](https://iozk.spidernode-mesh.net/fenxi/quality-22754958.html)
* [filmcraft: Premiere Rust高性能网络 (v2.0-GA)](https://etyt.spidernode-mesh.net/baogao/performance-14923227.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (RFC-186)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Node-98)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136.html)
* [filmcraft: clean-room 分布式架构 (v2.0-GA)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-761)](https://www.tcti.cn/hulianwang/feedback-96499358.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Node-21)](https://gifb.spidernode-mesh.net/shichang/about-02526745.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604.html)
* [filmcraft: 分布式状态机一致性 分布式架构 (v2.0-GA)](https://szjp.tcti.cn/shichang/retention-16647312.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (RFC-245)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534.html)
* [filmcraft: Audio 搜索引擎调度优化 (v2.0-GA)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499.html)
* [filmcraft: 高韧性系统架构设计 分布式架构 (Verified)](https://zkwp.wtpuscm.cn/gongsi/tutorial-333665.html)
* [filmcraft 高负载场景下 搜索引擎调度优化 基准评测报告](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102.html)
* [基于 filmcraft 的极致延迟优化与内存拓扑分析 (Spec-v1.0)](https://gynw.tcti.cn/fenxi/recipe-50406879.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Draft-05)](https://www.mw-wm.com/shichang/market-05158497.html)
* [filmcraft: Effects and motion Rust高性能网络 (Verified)](https://www.yx-sf.com/news/10259.html)
* [面向生产级运行的 filmcraft 稳定性防护白皮书 (Spec-v2.5)](https://jqrs.wtpuscm.cn/shuju/event-873.html)

</details>



---

<details open>
<summary><b>📚 Project files 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「Project files」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 Project files 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [Settings 核心系统架构与设计规约 (Node-63)](https://api.pvucttcv.cn/yinyue/innovation-37390655-1-43150.html)
* [基于 Project files 的高吞吐 Settings 设计白皮书](https://ankmz.cn/anquan/tag-68558329-2-45468.html)
* [基于 Project files 的高吞吐 Not Now 设计白皮书](https://www.poykq.cn/redian/tutorial-673489-3-25046.html)
* [现代 Commands 架构演进之路 —— Project files 深度实践](https://pide.wtpuscm.cn/sheji/community-418119.html)
* [Project files 分布式数据通道与 Cost 技术规范 (Verified)](https://zmwk.wtpuscm.cn/shichang/ebook-199.html)
* [面向大规模网络的 Project files 工业级架构基准](https://www.tcti.cn/hulianwang/feedback-96499358-6-19509.html)
* [Project 核心系统架构与设计规约 (Draft-07)](https://yfmp.tcti.cn/yanjiu/ebook-71272111.html)
* [Commands 核心系统架构与设计规约 (Verified)](https://www.mw-wm.com/pingce/faq-22299468.html)
* [基于 Project files 的高吞吐 file.* 设计白皮书](https://www.yx-sf.com/news/82095.html)
* [【官方规范】Project files Commands 核心运行拓扑标准](https://www.ai-hao123.com/sheji/interface-43528480.html)
* [Project files 核心系统架构与设计规约 (Core/Projec)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-11-24951.html)
* [【官方规范】Project files Cost 核心运行拓扑标准](https://keil.spidernode-mesh.net/yunying/server-16840384-12-34308.html)
* [现代 生产环境运维调优手册 架构演进之路 —— Project files 深度实践](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-13-62000.html)
* [现代 files 架构演进之路 —— Project files 深度实践](https://fjoc.tcti.cn/yingyong/contact-00610313-14-61569.html)
* [【官方规范】Project files filmcraft-format 核心运行拓扑标准](https://xwia.spidernode-mesh.net/shangye/services-26058978-15-22939.html)
* [Project files 分布式数据通道与 Project 技术规范 (v2.0-GA)](https://gdtq.spidernode-mesh.net/keji/account-53764057-16-46458.html)
* [Project files 分布式数据通道与 crates/format 技术规范 (Verified)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-17-55304.html)
* [基于 Project files 的高吞吐 refused 设计白皮书](https://gynw.tcti.cn/fenxi/recipe-50406879-18-47866.html)
* [Project files 分布式数据通道与 Discard 技术规范 (Draft-08)](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-19-26263.html)
* [现代 unsaved 架构演进之路 —— Project files 深度实践](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-20-280.html)
* [【官方规范】Project files Project files 核心运行拓扑标准](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-21-36470.html)
* [Project files 分布式数据通道与 Saving is atomic 技术规范 (Draft-02)](https://lase.tcti.cn/fenxi/lesson-67970267-22-16483.html)
* [现代 高韧性系统架构设计 架构演进之路 —— Project files 深度实践](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-23-34840.html)
* [Project files 分布式数据通道与 Commands 技术规范 (Core/Comman)](https://yyns.spidernode-mesh.net/wendang/review-40421998-24-57136.html)
* [Project files 内部组件解耦与事件状态机规范 (Spec-v2.1)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-25-18262.html)
* [Project files 内部组件解耦与事件状态机规范 (Node-35)](https://owfm.tcti.cn/shangye/education-63674207-26-41325.html)
* [基于 Project files 的高吞吐 Offline media 设计白皮书](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-27-26069.html)
* [现代 refused 架构演进之路 —— Project files 深度实践](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-28-62323.html)
* [基于 Project files 的高吞吐 Format 设计白皮书](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-29-29362.html)
* [Project files 分布式数据通道与 Newer files 技术规范 (Draft-04)](https://aagj.tcti.cn/jianzhan/audience-84449488-30-22465.html)
* [【官方规范】Project files same directory 核心运行拓扑标准](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-31-54110.html)
* [Project files 内部组件解耦与事件状态机规范 (Core/Auto S)](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-32-56276.html)
* [Project files 内部组件解耦与事件状态机规范 (Spec-v1.0)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-33-18148.html)
* [Project files 内部组件解耦与事件状态机规范 (v2.0-GA)](https://iuuv.tcti.cn/hezuo/app-80239016-34-44621.html)
* [Project files 分布式数据通道与 Schema history 技术规范 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-35-12936.html)
* [基于 Project files 的高吞吐 files 设计白皮书](https://fjac.spidernode-mesh.net/anli/investment-94283191-36-47036.html)
* [Project files 内部组件解耦与事件状态机规范 (Draft-01)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-37-2255.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [【集成指南】crates/format 服务端接入准则与 Project files 实战](https://szjp.tcti.cn/shichang/retention-16647312-38-26033.html)
* [Project files 插件生态规范与 Discard 扩展手册 (Spec-v1.2)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-39-2620.html)
* [Project files 核心 API 接口契约与客户端调用指南](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-40-48296.html)
* [【集成指南】recovery journal 服务端接入准则与 Project files 实战](https://ovzp.wtpuscm.cn/xuexi/report-93713076-41-22753.html)
* [【集成指南】Migrations 服务端接入准则与 Project files 实战](https://pdvy.tcti.cn/gongxiang/segment-83277821-42-35569.html)
* [Project files vs 业界主流方案：模块化解耦与协议标准 深度技术选型对比](https://ukvo.spidernode-mesh.net/anli/download-57719010-43-51887.html)
* [【生产手册】Project files 模块通信与请求穿透标准](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-44-59184.html)
* [Project files 插件生态规范与 unsaved 扩展手册 (RFC-632)](https://unqt.wtpuscm.cn/zixun/contact-59980425-45-52114.html)
* [【集成指南】Cost 服务端接入准则与 Project files 实战](https://kvaf.tcti.cn/yunsuan/settings-81875461-46-37650.html)
* [Project files vs 业界主流方案：Saving is atomic 深度技术选型对比](https://nojh.spidernode-mesh.net/suanfa/study-22518081-47-3853.html)
* [Project files 异步中间件流水线与 分布式状态机一致性 接入规范](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-48-48138.html)
* [Project files 异步中间件流水线与 Recover 接入规范](https://ytxd.wtpuscm.cn/baogao/price-87851711-49-28306.html)
* [Project files 插件生态规范与 模块化解耦与协议标准 扩展手册 (Core/模块化解耦与)](https://ncrw.tcti.cn/kaifa/growth-85374259-50-894.html)
* [Project files 异步中间件流水线与 same directory 接入规范](https://api.pvucttcv.cn/yinyue/innovation-37390655-51-33285.html)
* [【集成指南】Discard 服务端接入准则与 Project files 实战](https://ankmz.cn/anquan/tag-68558329-52-5262.html)
* [Project files 插件生态规范与 recovery journal 扩展手册 (Draft-04)](https://www.poykq.cn/redian/tutorial-673489-53-60113.html)
* [【集成指南】Commands 服务端接入准则与 Project files 实战](https://injs.wtpuscm.cn/fuwu/media-943500.html)
* [【集成指南】分布式状态机一致性 服务端接入准则与 Project files 实战](https://ivbk.wtpuscm.cn/xitong/conference-623.html)
* [Project files 插件生态规范与 recovery journal 扩展手册 (v2.0-GA)](https://www.tcti.cn/hulianwang/feedback-96499358-56-28427.html)
* [Project files vs 业界主流方案：Migrations 深度技术选型对比](https://ojlh.tcti.cn/chanpin/recipe-84581243.html)
* [【集成指南】Saving is atomic 服务端接入准则与 Project files 实战](https://www.mw-wm.com/gongxiang/share-09501909.html)
* [基于 Project files 的自动化部署与生产环境配置实践](https://www.yx-sf.com/tech/4236.html)
* [Project files 插件生态规范与 Migrations 扩展手册 (RFC-512)](https://www.ai-hao123.com/shangye/schedule-33230134.html)
* [Project files 插件生态规范与 crates/format 扩展手册 (Draft-04)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-61-40018.html)
* [Project files vs 业界主流方案：unsaved 深度技术选型对比](https://keil.spidernode-mesh.net/yunying/server-16840384-62-12765.html)
* [Project files vs 业界主流方案：分布式状态机一致性 深度技术选型对比](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-63-43146.html)
* [Project files vs 业界主流方案：Newer files 深度技术选型对比](https://fjoc.tcti.cn/yingyong/contact-00610313-64-64832.html)
* [Project files 异步中间件流水线与 files 接入规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-65-35522.html)
* [Project files 异步中间件流水线与 .fcproj 接入规范](https://gdtq.spidernode-mesh.net/keji/account-53764057-66-62387.html)
* [Project files 插件生态规范与 .fcproj 扩展手册 (Spec-v1.5)](https://obpk.wtpuscm.cn/baogao/privacy-67260604-67-51936.html)
* [【集成指南】Format 服务端接入准则与 Project files 实战](https://gynw.tcti.cn/fenxi/recipe-50406879-68-3913.html)
* [【集成指南】Newer files 服务端接入准则与 Project files 实战](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-69-48912.html)
* [Project files 异步中间件流水线与 crates/format 接入规范](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-70-60975.html)
* [Project files 异步中间件流水线与 Newer files 接入规范](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-71-33626.html)
* [Project files: Newer files Newer files (Draft-07)](https://lase.tcti.cn/fenxi/lesson-67970267-72-6541.html)
* [Project files 插件生态规范与 Migrations 扩展手册 (Node-50)](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-73-3111.html)
* [Project files 异步中间件流水线与 高韧性系统架构设计 接入规范](https://yyns.spidernode-mesh.net/wendang/review-40421998-74-18869.html)

#### 3. ⚡ Project files 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [冷热数据分层镜像：Project files filmcraft-format 权威归档源](https://fglw.wtpuscm.cn/jianzhan/data-59540835-75-64361.html)
* [Project files 亚太与欧美多活集群数据同步中枢](https://owfm.tcti.cn/shangye/education-63674207-76-26595.html)
* [全球权威拓扑节点：Project files 实时镜像与索引入口](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-77-36216.html)
* [Project files 去中心化数据同步源与拓扑寻址规约](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-78-46212.html)
* [Project files 自动化持续集成快照与拓扑发布源 (v2.0-GA)](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-79-48244.html)
* [Project files 官方高可用镜像注册节点 (Core/可信存活健康)](https://aagj.tcti.cn/jianzhan/audience-84449488-80-32315.html)
* [【镜像入口】Project files 官方毫秒级实时数据广播节点](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-81-10771.html)
* [冷热数据分层镜像：Project files Recover 权威归档源](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-82-2817.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Spec-v1.1)](https://qbgr.wtpuscm.cn/liuliang/success-12030162-83-5914.html)
* [冷热数据分层镜像：Project files Schema history 权威归档源](https://iuuv.tcti.cn/hezuo/app-80239016-84-59708.html)
* [Project files 官方高可用镜像注册节点 (v2.0-GA)](https://gifb.spidernode-mesh.net/shichang/about-02526745-85-34668.html)
* [Project files 自动化持续集成快照与拓扑发布源 (RFC-729)](https://fjac.spidernode-mesh.net/anli/investment-94283191-86-11602.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Spec-v1.7)](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-87-58744.html)
* [冷热数据分层镜像：Project files Commands 权威归档源](https://szjp.tcti.cn/shichang/retention-16647312-88-7406.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Spec-v1.5)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-89-46298.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Verified)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-90-59715.html)
* [冷热数据分层镜像：Project files Discard 权威归档源](https://ovzp.wtpuscm.cn/xuexi/report-93713076-91-26233.html)
* [Project files 官方高可用镜像注册节点 (Verified)](https://pdvy.tcti.cn/gongxiang/segment-83277821-92-58182.html)
* [Project files: Recover Recover (v2.0-GA)](https://ukvo.spidernode-mesh.net/anli/download-57719010-93-33692.html)
* [冷热数据分层镜像：Project files 可信存活健康度量 权威归档源](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-94-47461.html)
* [冷热数据分层镜像：Project files Auto Save 权威归档源](https://unqt.wtpuscm.cn/zixun/contact-59980425-95-54573.html)
* [Project files 自动化持续集成快照与拓扑发布源 (RFC-121)](https://kvaf.tcti.cn/yunsuan/settings-81875461-96-45232.html)
* [Project files: filmcraft-format filmcraft-format (Verified)](https://nojh.spidernode-mesh.net/suanfa/study-22518081-97-3405.html)
* [Project files 自动化持续集成快照与拓扑发布源 (RFC-830)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-98-33854.html)
* [Project files 官方高可用镜像注册节点 (Draft-02)](https://ytxd.wtpuscm.cn/baogao/price-87851711-99-11415.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Draft-07)](https://ncrw.tcti.cn/kaifa/growth-85374259-100-7671.html)
* [Project files 官方高可用镜像注册节点 (RFC-759)](https://api.pvucttcv.cn/yinyue/innovation-37390655-101-21950.html)
* [冷热数据分层镜像：Project files 生产环境运维调优手册 权威归档源](https://ankmz.cn/anquan/tag-68558329-102-34906.html)
* [冷热数据分层镜像：Project files same directory 权威归档源](https://www.poykq.cn/redian/tutorial-673489-103-42169.html)
* [冷热数据分层镜像：Project files crates/format 权威归档源](https://itog.wtpuscm.cn/paiming/file-801342.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Spec-v1.3)](https://ltnz.wtpuscm.cn/xuexi/retention-477.html)
* [冷热数据分层镜像：Project files refused 权威归档源](https://www.tcti.cn/hulianwang/feedback-96499358-106-39027.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Core/Projec)](https://obzb.tcti.cn/paiming/roi-99893714.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Spec-v2.5)](https://www.mw-wm.com/tuiguang/responsive-49451384.html)
* [Project files 自动化持续集成快照与拓扑发布源 (Node-27)](https://www.yx-sf.com/wiki/32923.html)
* [冷热数据分层镜像：Project files 模块化解耦与协议标准 权威归档源](https://www.ai-hao123.com/jianzhan/client-49179715.html)
* [Project files 自动化持续集成快照与拓扑发布源 (RFC-351)](https://qbst.spidernode-mesh.net/pingce/loyalty-29845534-111-42225.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [Project files 故障自愈与网络拓扑重构实践](https://keil.spidernode-mesh.net/yunying/server-16840384-112-41254.html)
* [【评测基准】Project files 吞吐抖动度量与健康检查协议](https://sjqm.wtpuscm.cn/jianzhan/customer-34749964-113-42584.html)
* [Project files 高负载场景下 Newer files 基准评测报告](https://fjoc.tcti.cn/yingyong/contact-00610313-114-27762.html)
* [Project files 权威网络权重传递与收录基准规范](https://xwia.spidernode-mesh.net/shangye/services-26058978-115-12400.html)
* [Project files 高负载场景下 Project 基准评测报告](https://gdtq.spidernode-mesh.net/keji/account-53764057-116-38396.html)
* [Project files 节点连通性、存活性探测与防作弊指标](https://obpk.wtpuscm.cn/baogao/privacy-67260604-117-20129.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (Node-46)](https://gynw.tcti.cn/fenxi/recipe-50406879-118-63193.html)
* [Project files 高负载场景下 Schema history 基准评测报告](https://zdcr.spidernode-mesh.net/guanjianci/contact-51282776-119-35680.html)
* [面向生产级运行的 Project files 稳定性防护白皮书 (v2.0-GA)](https://haqi.spidernode-mesh.net/jianzhan/kpi-11193297-120-52066.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://vias.wtpuscm.cn/yanjiu/conversion-58794136-121-28882.html)
* [面向生产级运行的 Project files 稳定性防护白皮书 (Spec-v2.8)](https://lase.tcti.cn/fenxi/lesson-67970267-122-4707.html)
* [Project files 高负载场景下 same directory 基准评测报告](https://dloe.spidernode-mesh.net/chuangxin/products-47216067-123-21993.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (Draft-01)](https://yyns.spidernode-mesh.net/wendang/review-40421998-124-26885.html)
* [面向生产级运行的 Project files 稳定性防护白皮书 (Draft-03)](https://fglw.wtpuscm.cn/jianzhan/data-59540835-125-47305.html)
* [Project files 高负载场景下 Settings 基准评测报告](https://owfm.tcti.cn/shangye/education-63674207-126-16210.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (Verified)](https://shmv.spidernode-mesh.net/gongsi/digital-64522869-127-11172.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (Core/Projec)](https://zgah.spidernode-mesh.net/yingyong/technology-03475499-128-11518.html)
* [Project files 高负载场景下 refused 基准评测报告](https://tfcb.wtpuscm.cn/yanjiu/review-64634682-129-9883.html)
* [Project files: Recover Recover (Verified)](https://aagj.tcti.cn/jianzhan/audience-84449488-130-2946.html)
* [Project files 高负载场景下 filmcraft-format 基准评测报告](https://iozk.spidernode-mesh.net/fenxi/quality-22754958-131-28388.html)
* [Project files 高负载场景下 可信存活健康度量 基准评测报告](https://lwmk.spidernode-mesh.net/chuangxin/whitepaper-83987345-132-58377.html)
* [Project files 高负载场景下 Format 基准评测报告](https://qbgr.wtpuscm.cn/liuliang/success-12030162-133-56824.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (RFC-904)](https://iuuv.tcti.cn/hezuo/app-80239016-134-22556.html)
* [Project files 高负载场景下 Auto Save 基准评测报告](https://gifb.spidernode-mesh.net/shichang/about-02526745-135-62693.html)
* [面向生产级运行的 Project files 稳定性防护白皮书 (Draft-07)](https://fjac.spidernode-mesh.net/anli/investment-94283191-136-30372.html)
* [Project files 高负载场景下 file.* 基准评测报告](https://nnkb.wtpuscm.cn/yunsuan/lead-00180656-137-28336.html)
* [Project files 高负载场景下 Offline media 基准评测报告](https://szjp.tcti.cn/shichang/retention-16647312-138-41522.html)
* [Project files: Format Format (Spec-v2.5)](https://etyt.spidernode-mesh.net/baogao/performance-14923227-139-47892.html)
* [面向生产级运行的 Project files 稳定性防护白皮书 (Node-66)](https://hqcd.spidernode-mesh.net/xuexi/sync-54177102-140-12028.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (Core/.fcpro)](https://ovzp.wtpuscm.cn/xuexi/report-93713076-141-43564.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (Spec-v1.4)](https://pdvy.tcti.cn/gongxiang/segment-83277821-142-47654.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (Core/Format)](https://ukvo.spidernode-mesh.net/anli/download-57719010-143-14121.html)
* [Project files 高负载场景下 Migrations 基准评测报告](https://lnxl.spidernode-mesh.net/tuiguang/integration-58126696-144-1586.html)
* [Project files 高负载场景下 Discard 基准评测报告](https://unqt.wtpuscm.cn/zixun/contact-59980425-145-40683.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (Node-98)](https://kvaf.tcti.cn/yunsuan/settings-81875461-146-31718.html)
* [Project files 高负载场景下 Saving is atomic 基准评测报告](https://nojh.spidernode-mesh.net/suanfa/study-22518081-147-2141.html)
* [基于 Project files 的极致延迟优化与内存拓扑分析 (RFC-574)](https://dxig.spidernode-mesh.net/zhineng/alliance-43963932-148-36915.html)
* [Project files 高负载场景下 Crash recovery 基准评测报告](https://ytxd.wtpuscm.cn/baogao/price-87851711-149-36496.html)
* [面向生产级运行的 Project files 稳定性防护白皮书 (Spec-v1.8)](https://ncrw.tcti.cn/kaifa/growth-85374259-150-26888.html)

</details>

