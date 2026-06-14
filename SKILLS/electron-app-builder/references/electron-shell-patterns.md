# Electron Shell Patterns

## Scope

Use these as architectural patterns, not as code to copy blindly. Keep downloader, broadcast, extension, sidecar, model, media-processing, or other domain-specific behavior out of the base shell unless the new app truly needs it.

## Recommended Project Layout

```text
.
|-- electron/
|   |-- main.js
|   |-- preload.js
|   |-- state.js
|   `-- modules/
|       |-- system.js
|       |-- storage.js
|       `-- <domain>.js
|-- public/
|-- src/
|   |-- index.js
|   |-- App.js
|   |-- store.js
|   |-- theme.js
|   `-- components/
|       `-- TitleBar.js
|-- package.json
|-- webpack.config.js
`-- .babelrc
```

## BrowserWindow Baseline

Use this baseline unless the app has a concrete reason to differ:

```js
const isLinux = process.platform === 'linux';

const win = new BrowserWindow({
  width: 1400,
  height: 900,
  minWidth: 1200,
  minHeight: 700,
  backgroundColor: '#0a0a0a',
  titleBarStyle: isLinux ? undefined : 'hidden',
  frame: false,
  show: true,
  skipTaskbar: false,
  icon: nativeImage.createFromPath(iconPath),
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    nodeIntegration: false,
    contextIsolation: true,
    sandbox: true,
    webSecurity: true,
  },
});
```

Important details:

- Use `frame:false` for the custom title bar.
- Avoid `titleBarStyle: 'hidden'` on Linux. It can interact badly with frameless windows and taskbar behavior.
- Keep `show:true` unless a boot/splash flow intentionally delays display.
- Use `app.setIcon(nativeImage.createFromPath(iconPath))` on Linux after `app.whenReady()` so the taskbar icon shows reliably.
- Load `dist/index.html` when packaged and `http://localhost:8081` in development.

## Main Process Flow

Use this order:

1. Import Electron, `path`, optional `fs`, shared `state`, and setup functions.
2. Acquire `app.requestSingleInstanceLock()` for normal desktop apps.
3. Register privileged schemes before app readiness if the renderer needs local media/file URLs.
4. In `app.whenReady()`, register protocol handlers, create the window, set Linux app icon, and install IPC modules.
5. On `second-instance`, restore/focus the current window.
6. On `window-all-closed`, quit except on macOS.
7. On `activate`, recreate the window if no windows remain.
8. On shutdown, unregister global shortcuts if used.

## Preload Bridge

Expose one app API, for example `window.myapp`, with strict allowlists:

```js
const invokeChannels = new Set(['minimize-window', 'maximize-window', 'close-window']);
const sendChannels = new Set([]);
const listenChannels = new Set(['download-progress']);
const listenerMap = new WeakMap();

contextBridge.exposeInMainWorld('myapp', {
  platform: process.platform,
  invoke: (channel, ...args) => {
    if (!invokeChannels.has(channel)) {
      return Promise.reject(new Error(`IPC invoke channel not allowed: ${channel}`));
    }
    return ipcRenderer.invoke(channel, ...args);
  },
  send: (channel, ...args) => {
    if (!sendChannels.has(channel)) {
      throw new Error(`IPC send channel not allowed: ${channel}`);
    }
    ipcRenderer.send(channel, ...args);
  },
  on: (channel, listener) => {
    if (!listenChannels.has(channel) || typeof listener !== 'function') return;
    const wrapped = (_event, ...args) => listener(...args);
    listenerMap.set(listener, wrapped);
    ipcRenderer.on(channel, wrapped);
  },
  removeListener: (channel, listener) => {
    if (!listenChannels.has(channel) || typeof listener !== 'function') return;
    const wrapped = listenerMap.get(listener);
    if (wrapped) {
      ipcRenderer.removeListener(channel, wrapped);
      listenerMap.delete(listener);
    }
  },
});
```

Do not expose raw `ipcRenderer`.

## Shared Main State

Keep shared handles in `electron/state.js`:

```js
module.exports = {
  mainWindow: null,
  activeDownloads: new Map(),
  activeJobs: new Map(),
};
```

Use this for background work cancellation and progress events. Do not scatter active process maps across unrelated modules.

## System IPC Module

Create `electron/modules/system.js` for:

- `minimize-window`
- `maximize-window`
- `close-window`
- `open-external`
- `open-folder`
- `select-file`
- `select-folder`
- `get-default-download-path`
- `get-disk-space` or platform-specific disk lookup

For `open-external`, validate URLs and allow only `http:` or `https:` unless the app explicitly needs another protocol.

For `open-folder`, prefer `shell.showItemInFolder(filePath)` when revealing a file, and use `shell.openPath(folderPath)` only when opening a folder directly.

## Storage IPC Module

Store user data under `app.getPath('userData')`.

Use split JSON files:

- `<app>-settings.json` for settings/preferences
- `<app>-library.json`, `<app>-gallery.json`, or another domain name for app content/state

Recommended behavior:

- `load-state`: read split files if present, reconstruct Zustand `{ state, version }`, otherwise fall back to legacy `<persist-name>.json`.
- `save-state`: parse persisted state, split settings from content, write through `*.tmp.json`, then `renameSync` to reduce corrupted writes.
- `remove-state`: remove split files and legacy file.
- `backup-state`: let the user choose a folder, create timestamped backup directory, and copy existing state files.

Renderer store should use `zustand/persist` with `createJSONStorage` that calls preload `load-state`, `save-state`, and `remove-state`.

## Local Media/File Protocols

Use a privileged custom scheme when the renderer needs local media playback, waveform decoding, or controlled file fetches:

```js
protocol.registerSchemesAsPrivileged([
  {
    scheme: 'app-media',
    privileges: {
      secure: true,
      standard: true,
      supportFetchAPI: true,
      bypassCSP: true,
      stream: true,
    },
  },
]);
```

Prefer `protocol.handle()` for new Electron code. Support HTTP range requests for audio/video so Chromium media elements can seek reliably.

Validate paths before reading. For local media, check existence and derive a conservative content type from the extension.

## Renderer Title Bar

Use a custom title bar component:

- Fixed height around `40px`.
- App logo and app name on the left.
- Version badge next to the name.
- Window controls on the right.
- `WebkitAppRegion: 'drag'` on the titlebar/title section.
- `WebkitAppRegion: 'no-drag'` on buttons and other interactive controls.

Route buttons through the bridge:

```js
window.myapp?.invoke('minimize-window');
window.myapp?.invoke('maximize-window');
window.myapp?.invoke('close-window');
```

Use a build-time version constant when possible. A clean pattern is to inject `process.env.APP_VERSION` through Webpack `DefinePlugin`, then wrap it in `src/version.js`.

## Webpack Baseline

Use:

- `entry: './src/index.js'`
- content-hashed production filenames
- `output.clean: isProd`
- `maxAssetSize` and `maxEntrypointSize` around `5 MB`
- Terser with comments disabled and console preserved
- split chunks for React/React Native Web vendor code and other common node modules
- `runtimeChunk: 'single'`
- Babel loader for JS/JSX with cache enabled
- CSS loaders
- `asset/inline` for images and SVG so packaged `file://` builds do not depend on a static server
- `react-native$` alias to `react-native-web` when using React Native Web
- dev server on `8081` unless the project needs another port

Use `DefinePlugin` for package version injection when renderer components should not import package metadata directly.

## Electron Builder Baseline

Package metadata should include:

```json
{
  "main": "electron/main.js",
  "scripts": {
    "start": "webpack serve --mode development",
    "electron": "electron .",
    "dev": "concurrently --kill-others \"npm run start\" \"wait-on http://localhost:8081 && electron .\"",
    "build": "webpack --mode production",
    "dist": "npm run build && electron-builder"
  },
  "build": {
    "appId": "com.ownername.appname",
    "productName": "AppName",
    "directories": {
      "output": "dist-bin",
      "buildResources": "build"
    },
    "files": [
      "dist/**/*",
      "electron/**/*",
      "public/**/*",
      "package.json"
    ],
    "linux": {
      "target": ["AppImage", "deb", "tar.gz"],
      "category": "Utility",
      "maintainer": "OWNER NAME <app@owner.com>"
    }
  }
}
```

Add `build/**/*`, backend folders, models, or `asarUnpack` only when the app actually ships those assets.

## Validation

Always run the project’s real build command after shell changes. For this pattern that is usually:

```bash
npm run build
```

Also run any backend syntax checks or app-specific tests if the app has them. Do not run Electron/dev server unless the user has allowed runtime testing.
