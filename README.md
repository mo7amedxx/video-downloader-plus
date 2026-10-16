# Video Downloader Plus

> An open-source desktop video and audio downloader powered by `yt-dlp`, with a redesigned interface and automatic downloads after you enter a link — no paid subscription or app-imposed daily quota.

**Video Downloader Plus** is an independent derivative of [Open Video Downloader](https://github.com/jely2002/youtube-dl-gui), with a redesigned interface and its own identity. It is **not an official product** of the Open Video Downloader developers or 4K Download, and it does not use the official 4K Video Downloader Plus logo.

## GitHub repository description

Copy this into the repository’s **About → Description** field:

> Open-source video and audio downloader powered by yt-dlp. Starts downloads directly after link entry, with no subscription or app-imposed daily quota.

**Suggested topics:** `video-downloader` `yt-dlp` `tauri` `vue` `rust` `open-source` `linux`

## Features

- **Automatic downloads:** paste or enter a link; the app fetches its information and starts downloading without requiring a separate start button.
- **Smart Mode:** monitor the clipboard and automatically add and download copied links.
- **Video and audio downloads** from websites supported by `yt-dlp`.
- **Playlists, formats, quality, and subtitles**, subject to the options available in the upstream project and on each source website.
- **Compact download queue** with status and progress, plus additional download-management tools.
- **No paid subscriptions or app-imposed daily quota.** Source websites may still impose their own access or rate limits.

## Install on Debian / Ubuntu

Download the appropriate `.deb` file for your computer from the project’s [Releases](https://github.com/OWNER/REPOSITORY/releases) page, then run this command from the download directory:

```bash
sudo apt install ./video-downloader-plus_3.2.1_amd64.deb
```

The currently available package targets **Linux amd64 / x86_64**. Replace `OWNER/REPOSITORY` in the Releases link with your GitHub account and repository name after publishing.

## Usage

1. Open Video Downloader Plus.
2. Copy a video or playlist link.
3. Click **Paste Link**. If the clipboard does not contain a valid link, the app shows a field where you can enter one.
4. The app retrieves the video information and starts the download automatically.
5. Enable **Smart Mode** to automatically add and download links you copy afterward.

You can change the download folder and format or quality options in the app settings and download tools.

## Build from source

### Requirements

- Node.js 24 or newer.
- Rust and Cargo.
- Tauri system dependencies for your distribution, including GTK, WebKitGTK 4.1, and AppIndicator libraries on Debian/Ubuntu.

On Ubuntu/Debian, common build dependencies can be installed with:

```bash
sudo apt update
sudo apt install build-essential curl wget file libssl-dev libgtk-3-dev \
  libayatana-appindicator3-dev librsvg2-dev libwebkit2gtk-4.1-dev patchelf pkg-config
```

### Run the development build

```bash
npm ci
npm run tauri dev
```

### Build a Debian package

```bash
npm run tauri -- build --bundles deb
```

The package is usually created under:

```text
src-tauri/target/release/bundle/deb/
```

## License and attribution

This project is based on [Open Video Downloader](https://github.com/jely2002/youtube-dl-gui). It retains the upstream license and notices; see [`LICENSE`](./LICENSE) and the bundled third-party license files before redistributing. The upstream project is licensed under the **GNU Affero General Public License v3.0 or later (AGPL-3.0-or-later)**. Modified redistributions must comply with the license terms, including applicable notice and source-availability requirements.

Use this tool only to download content you own or are authorized to download. Users are responsible for respecting copyright, source-site terms, and applicable laws.


## Codebase guide

This section describes how the source is organized and how a link travels through the app. It is a practical map of the implementation, not a line-by-line listing of every dependency-generated file.

### Technology stack

- **Frontend:** Vue 3 single-file components, TypeScript, Vite, Tailwind CSS 4, and DaisyUI.
- **Application state:** Pinia stores.
- **Routing and translations:** Vue Router and Vue I18n; translation files are in `src/locales/`.
- **Desktop shell and native operations:** Tauri 2 with a Rust backend.
- **Download engine:** `yt-dlp` and other managed runtime tools. The Rust side schedules processes, parses their output, and sends events to the frontend.

### Link-to-download flow

```text
Paste Link / link field / Smart Mode / bottom URL bar
        ↓
URL parsing and batch dispatch (frontend)
        ↓
Media store creates a queue group and requests metadata
        ↓
Tauri command `media_info` → fetch pipeline / yt-dlp
        ↓
Metadata and state events update the Pinia stores
        ↓
Direct-download path waits until the group is ready
        ↓
Tauri command `media_download` → download pipeline / yt-dlp
        ↓
Progress, completion, and diagnostic events update the queue UI
```

The direct-download behavior is wired through the following frontend entry points:

- `src/components/TheHeader.vue`: the link-entry form, the **Paste Link** action, import handling, and Smart Mode. Typed links and valid clipboard links are sent through the immediate-download path; Smart Mode calls `mediaStore.addAndDownload()` when it detects a new copied URL.
- `src/components/TheFooter.vue`: the bottom URL field calls `addUrlBatchAndDownload()` for valid links.
- `src/stores/media/media.ts`: `addUrlBatchAndDownload()` queues the URL, waits for the group to reach a ready state, and then calls `downloadGroup()` with that group's chosen options. `downloadGroup()` invokes the Rust command `media_download`.

Playlist handling, selection rules, state transitions, errors, and progress are part of the same pipeline. The UI is not a mock downloader: it calls the existing Tauri commands and listens for backend updates.

### Frontend source map

| Path | Responsibility |
| --- | --- |
| `src/main.ts` | Creates and mounts the Vue application and registers its main plugins. |
| `src/App.vue` | Root component; initializes app-level stores/listeners and routes to installation when runtime tools need setup. |
| `src/router.ts` | Declares routes for the download queue, media details, settings, preferences, and installation. |
| `src/layouts/AppLayout.vue` | Shared desktop layout: top command area, main route view, updater, and bottom toolbar. |
| `src/components/TheHeader.vue` | Menu bar, Paste Link action, Smart Mode, link input, settings access, and URL-file import. |
| `src/components/TheFooter.vue` | Bottom navigation/URL bar and expandable global queue/download controls. |
| `src/views/app/HomeView.vue` | Renders the ordered queue, empty state, and drag-and-drop states. |
| `src/components/media-card/MediaCard.vue` | Renders a queue row, thumbnail, active processing step, and per-item actions. |
| `src/components/media-card/steps/` | UI for fetching, configuring, downloading, pausing, finishing, or reporting errors for an item. |
| `src/components/media-card/actions/` | Per-item actions such as download, pause/resume, retry, remove, preferences, and metadata. |
| `src/components/settings/`, `src/views/app/settings/` | Settings pages and their individual tabs. |
| `src/components/base/` | Shared form controls, buttons, progress, select fields, and layout primitives. |
| `src/locales/` | Localized UI strings in JSON files. |
| `src/tauri/types/` and `src/tauri/listeners/` | Frontend TypeScript contracts and event listeners for the native backend. |

### Frontend state and utility code

- `src/stores/media/media.ts` coordinates URL ingestion, media metadata, playlist expansion, automatic downloads, pause/resume, and queue cleanup.
- `src/stores/media/group.ts` stores and orders queue groups; `state.ts`, `progress.ts`, `size.ts`, `options.ts`, and `destination.ts` hold the related status, progress, media-size, format, and output-folder state.
- `src/stores/settings.ts`, `preferences.ts`, `binaries.ts`, `updater.ts`, `stronghold.ts`, `watchClipboard.ts`, `dragDrop.ts`, and `toast.ts` manage application settings and supporting UI state.
- `src/helpers/` contains focused utilities for URL parsing, playlists, formats, network settings, progress, diagnostics, subtitles, and output options.
- `src/composables/` contains reusable Vue behavior, including clipboard polling, theme handling, dialogs, drag-and-drop, diagnostics, and opening links.

### Rust/Tauri backend map

The Rust application lives in `src-tauri/`:

| Path | Responsibility |
| --- | --- |
| `src-tauri/src/main.rs` and `lib.rs` | Start Tauri, initialize plugins and shared state, create dispatchers, and register native commands. |
| `src-tauri/src/commands/` | Frontend-callable Tauri commands for media, configuration, preferences, binaries, logging, notifications, updater, and secure storage. |
| `src-tauri/src/commands/media/` | Implements the `media_info`, `media_playlist_expand`, `media_download`, and media-size operations. |
| `src-tauri/src/scheduling/` | Fetch/download queues, concurrency limits, dispatchers, pipelines, and group-state handling. |
| `src-tauri/src/runners/` | Builds `yt-dlp` arguments, runs and supervises processes, resolves overrides, and assembles output templates. |
| `src-tauri/src/parsers/` | Parses `yt-dlp` JSON/output for media metadata, playlists, formats, tracks, errors, and progress. |
| `src-tauri/src/models/` | Rust data structures used for downloads, errors, parsed metadata, progress, and event payloads. |
| `src-tauri/src/binaries/` | Checks, downloads, and extracts the external runtime binaries used by the app. |
| `src-tauri/src/state/` | Persists application configuration and preferences. |
| `src-tauri/src/stronghold/` | Secure state for authentication-related secrets. |
| `src-tauri/src/window.rs`, `tray.rs`, `menu.rs`, `paths.rs` | Desktop window, system tray, native menus, and platform path management. |
| `src-tauri/capabilities/` | Tauri permissions and capability declarations for the frontend. |
| `src-tauri/tauri.conf.json` | Product metadata, window configuration, build commands, bundling targets, icons, and plugin configuration. |

### Styling and app assets

- `src/app.css` defines the design tokens, shared utility styling, and the redesigned menu, command bar, compact queue rows, and bottom navigation.
- `public/icon.png` is the frontend icon; `src-tauri/icons/` contains platform-specific icon assets used by installers and desktop environments.
- `index.html` is the frontend entry document.

### Useful project scripts

```bash
npm run dev          # Start the Vite frontend development server
npm run build        # Type-check and build the app and Tauri isolation page
npm run lint         # Run ESLint
npm run test:unit    # Run Vitest unit tests
npm run test:e2e     # Run Playwright end-to-end tests
npm run tauri dev    # Run the desktop application in development mode
npm run tauri -- build --bundles deb  # Build a Debian package
```

`npm run tauri dev` and native package builds require the Rust toolchain and the platform dependencies required by Tauri. For package/license generation, also follow the repository's package scripts and required development tools.
