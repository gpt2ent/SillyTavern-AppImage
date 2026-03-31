# Patching SillyTavern for AppImage Deployment

This document records all the steps taken to package SillyTavern into a standalone Linux `.AppImage`.

## Current Goal
Wrap SillyTavern in an Electron app that spawns the backend server, waits for it, and then opens a dedicated window for the UI (without needing an external browser). We are targeting Ubuntu 22.04 and ensuring Chromium version stays old enough (below v123) to avoid modern glibc requirements.

## Step-by-Step Build Process

### 1. Clone the SillyTavern Repository
```bash
git clone --depth=1 https://github.com/SillyTavern/SillyTavern.git .
```

### 2. Install Node 22 (Host Requirement)
To ensure compatibility with the build tools, Node 22+ is required.
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install 22
nvm use 22
```

### 3. Install Standard Dependencies
Install the standard SillyTavern dependencies:
```bash
npm install
```

### 4. Install Packaging Dependencies
Install Electron (specifically `v29.4.0` to keep Chromium at `122.0.6261.156`), `electron-builder`, and our wait module `tcp-port-used`:
```bash
npm install --save-dev electron@29.4.0 electron-builder
npm install tcp-port-used
```
*(Note: `tcp-port-used` must be in standard `dependencies`, otherwise `electron-builder` skips it when packaging, leading to a "module not found" error on launch).*

### 5. Create the Electron Wrapper (`main.cjs`)
We created a custom entry point called `main.cjs`. It uses CommonJS syntax because SillyTavern uses ES modules (`"type": "module"`) in `package.json`, which causes conflicts if we use `.js`.

```javascript
const { app, BrowserWindow } = require('electron');
const { spawn } = require('child_process');
const path = require('path');
const tcpPortUsed = require('tcp-port-used');

let mainWindow;
let serverProcess;

const port = 8000;

function createServer() {
    console.log("Starting SillyTavern server...");
    
    // NOTE: This spawn command is currently incomplete and failing 
    // because it runs external host Node instead of resolving inside the AppImage.
    serverProcess = spawn('node', ['server.js'], {
        stdio: 'inherit',
        env: { ...process.env, PORT: port }
    });

    serverProcess.on('close', (code) => {
        console.log(`Server process exited with code ${code}`);
    });
}

async function createWindow() {
    mainWindow = new BrowserWindow({
        width: 1280,
        height: 800,
        title: "SillyTavern",
        autoHideMenuBar: true,
        webPreferences: {
            nodeIntegration: false,
            contextIsolation: true
        }
    });

    console.log(`Waiting for server to be ready on port ${port}...`);
    
    try {
        await tcpPortUsed.waitUntilUsed(port, 500, 30000);
        console.log("Server is ready. Loading UI...");
        mainWindow.loadURL(`http://localhost:${port}`);
    } catch (err) {
        console.error("Timeout waiting for server:", err);
        app.quit();
    }

    mainWindow.on('closed', function () {
        mainWindow = null;
    });
}

app.whenReady().then(async () => {
    createServer();
    await createWindow();

    app.on('activate', function () {
        if (BrowserWindow.getAllWindows().length === 0) createWindow();
    });
});

app.on('window-all-closed', function () {
    if (process.platform !== 'darwin') {
        app.quit();
    }
});

app.on('quit', () => {
    if (serverProcess) {
        console.log("Killing server process...");
        serverProcess.kill();
    }
});
```

### 6. Modify `package.json`
We patched `package.json` to point to `main.cjs` instead of SillyTavern's default `server.js`, and added the configuration instructions for `electron-builder`.

**Changes made to `package.json`:**
1. `"main": "main.cjs"`
2. Added scripts:
   - `"build:linux": "electron-builder -l"`
3. Added the build configuration object:
```json
"build": {
  "appId": "com.sillytavern.app",
  "productName": "SillyTavern",
  "files": [
    "**/*",
    "!node_modules/electron-builder/**/*",
    "!node_modules/electron/**/*"
  ],
  "directories": {
    "output": "dist"
  },
  "linux": {
    "target": "AppImage",
    "category": "Game"
  }
}
```

### 7. Compile the AppImage
```bash
npm run build:linux
```
The resulting `.AppImage` is created inside the `dist/` folder.

### 8. Fixing the `server.js` Load Error (MODULE_NOT_FOUND)
When executing the built AppImage, the server originally crashed with a `MODULE_NOT_FOUND` error because we were using `spawn('node', ['server.js'])`. This executed the host machine's external `node` executable, which blindly looked for `server.js` in the host's current directory (e.g., `dist/`) instead of digging into the AppImage's read-only `.asar` filesystem.

**The Fix:**
We updated `main.cjs` to use `child_process.fork()` instead of `spawn`, and resolved the exact internal path using `path.join(__dirname, 'server.js')`. 
- `__dirname` resolves correctly to the temporary `/tmp/.mount_XXXXXX/resources/app.asar/` directory.
- `fork()` natively spins up Electron's internally bundled Node.js environment, which perfectly understands how to read files trapped inside an `.asar` archive.

**Updated code in `main.cjs`:**
```javascript
const { fork } = require('child_process');
const path = require('path');

// Inside createServer():
const serverPath = path.join(__dirname, 'server.js');
serverProcess = fork(serverPath, [], {
    env: { ...process.env, PORT: port },
    stdio: 'inherit'
});
```

## Current Status / Known Issues
- Currently crashes on launch because `child_process.spawn('node', ['server.js'])` executes the *host's* `node` binary within the host's current working directory (e.g., `dist/`), so it cannot find `server.js` which is sealed inside the AppImage's `.asar` footprint at `/tmp/.mount_XXXXXX/resources/app.asar/server.js`. See #8 how we addressed it.
- Did not yet address user caution about saving state in ~/.local/state/SillyTavern (to point its config and data paths there). Will do it after we can launch a basic image.

### 9. Fixing the `process.chdir()` Crash (ENOTDIR) inside ASAR
SillyTavern immediately calls `process.chdir(serverDirectory)` inside `server.js`. When packaged via `electron-builder`, the `serverDirectory` string resolves to the inside of an `.asar` archive (`/tmp/.mount_XXXXXX/resources/app.asar`). 
Node.js via Electron intercepts `fs` reads to work seamlessly with `.asar` files, but the underlying OS cannot natively `chdir` (change directory) into a singular archive file. This immediately throws `Error: ENOTDIR`. 

**The Fix:** 
The most robust solution to this when dealing with an aggressively hardcoded `chdir` is to completely disable `.asar` packaging inside the `electron-builder` configuration in `package.json`. 
When ASAR is set to `false`, `electron-builder` packages the raw file tree unmodified into `app/` inside the AppImage, meaning Native Node `chdir()` works perfectly because it's navigating into a real `/tmp/.mount_XXXXXX/resources/app/` temporary folder.

**Change made in `package.json` under `"build"`:**
```json
"build": {
  "asar": false,
  ...
}
```
*(Note: As a side effect, the AppImage mount sequence will take slightly longer because the Linux kernel has to temporarily extract the raw `node_modules` file tree into `/tmp` rather than mounting a single `.asar` blob).*

### 10. Forcing True Persistence (Data Root Relocation)
SillyTavern hardcodes extremely aggressive assumptions about its local directory structure. When packaged as a read-only AppImage, it attempts to write files like `cookie-secret.txt` directly to `./data/` inside the mounted temporary `/tmp/.mount_XXXXXX/resources/app/` payload. This fails with `ENOENT` (Error NO ENTITY / permissions blocked) because the mounted payload is strictly read-only by the Linux kernel.

**The Fix:**
Fortunately, the developers of SillyTavern built in an environment variable flag specifically for Docker/remote environments called `SILLYTAVERN_DATAROOT` (as well as passing `--dataRoot` through args), which completely reroutes where the application expects to read and write all persistent files (`chats`, `characters`, `secrets`, `configs`, etc.).

We modified `main.cjs` to:
1. Detect if the environment is packaged inside an AppImage using Electron's `app.isPackaged` flag.
2. If true, resolve a persistent host directory, specifically `~/.local/share/SillyTavern` (the Linux standard for user state data).
3. Attempt to aggressively `mkdir -p` that directory structure on the host system to ensure it exists before the backend boots.
4. Pass `SILLYTAVERN_DATAROOT` as an explicit environment variable down to the child `fork()` process, instructing SillyTavern to use that external host folder for *everything*.

**Change in `main.cjs`:**
```javascript
const fs = require('fs');

const userDataPath = app.isPackaged 
    ? path.join(app.getPath('userData'), 'data') 
    : path.join(__dirname, 'data');

if (!fs.existsSync(userDataPath)) {
    console.log(`Creating persistent data directory at: ${userDataPath}`);
    fs.mkdirSync(userDataPath, { recursive: true });
}

serverProcess = fork(serverPath, [], {
    env: { 
        ...process.env, 
        PORT: port,
        SILLYTAVERN_DATAROOT: userDataPath
    },
    stdio: 'inherit'
});
```

### 11. Relocating Hardcoded `public/` Directories
SillyTavern successfully relocated its `dataRoot` for configurations and chats, but another hardcoded hurdle was hit. 
During server startup, SillyTavern calls `ensurePublicDirectoriesExist()` which blindly tries to execute `mkdir` on `./public/scripts/extensions`, `./public/img`, and `./public/sounds`. Since these folders live inside the read-only AppImage payload, the OS throws an `ENOENT` permission failure. 

We couldn't simply move `public/` out of the app because the Express backend explicitly statically serves the frontend (`.html`, `.css`) out of that directory (in `src/server-main.js`).

**The Fix:**
I patched the initialization loop in `src/users.js` itself. Before SillyTavern loops through the hardcoded `PUBLIC_DIRECTORIES` and runs `fs.mkdirSync(...)`, I injected a dynamic path resolver:
`dir = path.resolve(globalThis.DATA_ROOT || process.cwd(), dir);`

This forces the backend initialization sequence to securely map directory creation out into the host machine's persistent `DATA_ROOT` environment override, bypassing the read-only AppImage payload entirely, while still allowing the Express server to successfully host the read-only `.html` frontend files from the immutable payload!

**Modified `ensurePublicDirectoriesExist()` in `src/users.js`:**
```javascript
export async function ensurePublicDirectoriesExist() {
    for (let dir of Object.values(PUBLIC_DIRECTORIES)) {
        dir = path.resolve(globalThis.DATA_ROOT || process.cwd(), dir);
        if (!fs.existsSync(dir)) {
            fs.mkdirSync(dir, { recursive: true });
        }
    // ...
```

### 11.5 Fixing Syntax and Regex Mishaps (`users.js`)
During Step 11, a regex script used to undo an older iteration of the directory hack inadvertently stripped SillyTavern's native module imports (the line `import { PUBLIC_DIRECTORIES, USER_DIRECTORY_TEMPLATE ... } from './constants.js'`). This led to an immediate `ReferenceError: PUBLIC_DIRECTORIES is not defined` crash.

**The Fix:**
I ran `git checkout src/users.js` to completely restore the file to its original repository state, ensuring all native imports stayed exactly where they belonged. 
I then re-applied the specific `path.resolve` override loop patch using precise string matching instead of regex. The finalized, functional version of the loop inside `src/users.js` functions identically to what is documented in Step 11.

### 12. Suppressing Auto-Browser Launch
By default, SillyTavern spawns an external child process trying to open the system's default web browser when the server comes online. Because our AppImage intrinsically spawns its own native Electron Window, this resulted in the tavern opening twice: once in our custom desktop app, and once as a rogue tab in your host browser (like Chrome or Firefox).

**The Fix:**
I patched `src/server-main.js` inside the `postSetupTasks()` function. 
I altered the `if (cliArgs.browserLaunchEnabled)` condition to strictly evaluate as `if (false)`. This completely lobotomizes the auto-browser launch branch at initialization, forcing the backend loop to skip the `import('open')` statement that triggers external tabs, while still safely binding localhost:8000 for Electron to consume under the hood. 

This ensures only our standalone GUI displays upon launch.

### 13. Resolving the Extensions Endpoint `ENOENT` Exception
While testing the UI, launching the "Extensions" tab triggered an `ENOENT: no such file or directory` exception under the hood trying to recursively create `data/public/scripts/extensions/third-party`. This did not crash the server but signaled that some internal API endpoints were also blindly running `fs.mkdirSync()` on hardcoded global variables without properly resolving the external path to the actual persistent data root.

**The Fix:**
I patched `src/endpoints/extensions.js` inside the `router.get('/discover')` handler.
Instead of directly passing the relative `PUBLIC_DIRECTORIES.globalExtensions` path into `mkdirSync()`, I added a `path.resolve()` wrapper using `globalThis.DATA_ROOT`, natively pushing the folder creation safely outside the AppImage onto the host filesystem.

**Changes applied in `src/endpoints/extensions.js`:**
```javascript
// BEFORE:
if (!fs.existsSync(PUBLIC_DIRECTORIES.globalExtensions)) {
    fs.mkdirSync(PUBLIC_DIRECTORIES.globalExtensions);
}

// AFTER:
const globExt = path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.globalExtensions);
if (!fs.existsSync(globExt)) {
    fs.mkdirSync(globExt, { recursive: true });
}
```

### 14. Rewiring `readdirSync` and Plugin API Logic in `extensions.js`
In Step 13, we patched a `mkdirSync` crash inside the Extensions shop viewer, but right below it, SillyTavern immediately runs `fs.readdirSync()` against the exact same static un-resolved variables (`PUBLIC_DIRECTORIES.extensions` and `PUBLIC_DIRECTORIES.globalExtensions`). 

Further inspection revealed that `extensions.js` is riddled with direct references to these static strings in almost every single API endpoint (install, delete, update, list plugins). Since those constant routes point blindly inside the read-only AppImage environment, the entire Plugin system would throw silent `ENOENT` fails globally.

**The Fix:**
I ran a massive string replacement pass over `src/endpoints/extensions.js`. 
Every single time the code references `PUBLIC_DIRECTORIES.extensions` or `PUBLIC_DIRECTORIES.globalExtensions`, I wrapped it in:
`path.resolve(globalThis.DATA_ROOT || process.cwd(), <directory_string>)`

This forces the entire extension manager (Shop, Installer, Lister, Updater) to dynamically execute all read/write filesystem operations safely out into the host machine's persistent `~/.local/state/SillyTavern/...` plugins folder!

### 15. First-Run Built-In Extensions Sync
While testing the generated AppImage on a fresh PC, a new issue emerged: built-in extensions (living in `public/scripts/extensions`) are deliberately ignored by the `userDataPath` redirection mechanism because they are expected to be "factory installed". Since the AppImage is read-only, moving the `userDataPath` successfully saves new user-installed extensions, but the original UI completely lost access to the core extensions because they were left stranded inside the payload!

**The Fix:**
I had to write a hook into the main Election `main.cjs` wrapper. Now, when the AppImage first boots and realizes it is creating the host's persistent `.config` folder for the first time, it performs a deep copy of the `public/scripts/extensions` directory directly from the AppImage payload (`__dirname`) down onto the host machine. 

```javascript
    // Ensure built-in extensions from the AppImage are copied to the persistent data folder
    const builtinExtSource = path.join(__dirname, 'public', 'scripts', 'extensions');
    const persistentExtDest = path.join(userDataPath, 'public', 'scripts', 'extensions');

    if (fs.existsSync(builtinExtSource)) {
        if (!fs.existsSync(persistentExtDest)) {
            console.log("First run: Copying built-in extensions to persistent directory...");
            fs.mkdirSync(persistentExtDest, { recursive: true });
            fs.cpSync(builtinExtSource, persistentExtDest, { recursive: true });
        } else {
            console.log("Built-in extensions directory already exists in persistent storage.");
        }
    } else {
        console.log("Warning: Built-in extensions source not found in AppImage.");
    }
```

### 16. Fixing ES Module ReferenceError in Patches
SillyTavern's source files (`users.js`, `endpoints/extensions.js`) are ES Modules (`"type": "module"`). Injecting CommonJS `require('path')` into them via string replacement causes a `ReferenceError: require is not defined` crash at runtime. The patch script must use the `path` module directly since it is already natively imported at the top of those files (`import path from 'node:path';`). We updated the patch scripts to strictly use `path.resolve` and `path.join` without the require statement.

### 17. Fixing shake256 hash algorithm error in webpack.config.js
During the `npm run build:linux` phase, Webpack crashes with `Error: Digest method not supported` related to `crypto.createHash('shake256')`. This is due to OpenSSL 3 / Node hash algorithm availability conflicts. We patched `webpack.config.js` replacing `crypto.createHash('shake256', { outputLength: 8 })` with a standard `crypto.createHash('md5')` to safely generate the builder cache keys.
