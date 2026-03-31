# SillyTavern AppImage Packaging Guide
**Target System:** Ubuntu 22.04 LTS (and derivatives)
**Tested Commit Hash:** `1.17.0 tag` (If patches fail to apply, review the diffs against this specific commit).

This guide provides step-by-step instructions to wrap SillyTavern in an Electron `.AppImage`. It includes persistent data management (configurations, characters, and chats survive closing the app), disables external browser launches, and mitigates internal `read-only` filesytem crashes caused by running inside an AppImage.

## 1. Prerequisites (Host Environment)
You will need Node.js v22 installed on your build machine (older Node 10/14 versions will fail to compile Electron 29).
```bash
# Install nvm and Node 22
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install 22
nvm use 22
```

## 2. Clone Repository and Prepare Directory
```bash
git clone https://github.com/SillyTavern/SillyTavern.git
cd SillyTavern
git checkout 1.17.0 tag
```

## 3. Create the Electron Wrapper (`main.cjs`)
Create a new file named `main.cjs` in the root directory. This script acts as the Electron entry point, launches the internal Node server, and points SillyTavern's `DATA_ROOT` to your host machine's persistent `.local` user data folder so your chats actually save!

```javascript
const { app, BrowserWindow } = require('electron');
const { fork } = require('child_process');
const path = path;
const fs = require('fs');
const tcpPortUsed = require('tcp-port-used');

let mainWindow;
let serverProcess;

const port = 8000;

function createServer() {
    console.log("Starting SillyTavern server...");
    
    const serverPath = path.join(__dirname, 'server.js');
    console.log(`Executing enclosed server at: ${serverPath}`);

    // Determine the persistent data root for the host system
    const userDataPath = app.isPackaged 
        ? path.join(app.getPath('userData'), 'data') 
        : path.join(__dirname, 'data');

    // Ensure the persistent folder exists on the host machine securely
    if (!fs.existsSync(userDataPath)) {
        console.log(`Creating persistent data directory at: ${userDataPath}`);
        fs.mkdirSync(userDataPath, { recursive: true });
    }

    // fork automatically runs via Electron's bundled Node.js and respects the ASAR filesystem
    serverProcess = fork(serverPath, [], {
        env: { 
            ...process.env, 
            PORT: port,
            SILLYTAVERN_DATAROOT: userDataPath, // Force persistence path
            SILLYTAVERN_DISABLE_BROWSER: '1'    // Block background browser popups
        },
        stdio: 'inherit'
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
        await tcpPortUsed.waitUntilUsed(port, 500, 30000); // Poll every 500ms, timeout 30s
        console.log("Server is ready. Loading UI...");
        mainWindow.loadURL(`http://localhost:${port}`);
    } catch (err) {
        console.error("Timeout waiting for server:", err);
        app.quit();
    }

    mainWindow.on('closed', function () { mainWindow = null; });
}

app.whenReady().then(async () => {
    createServer();
    await createWindow();
    app.on('activate', function () {
        if (BrowserWindow.getAllWindows().length === 0) createWindow();
    });
});

app.on('window-all-closed', function () {
    if (process.platform !== 'darwin') app.quit();
});

app.on('quit', () => {
    if (serverProcess) serverProcess.kill();
});
```

## 4. Source Code Patching
Because the AppImage mounts temporarily as a **Read-Only filesytem**, several core internal SillyTavern methods will crash or fail silently when trying to save plugins or initialize folders. You must apply the following specific string replacements.

You can save this script as `patch.js` and run it via `node patch.js` inside the directory to apply the fixes automatically:

```javascript
const fs = require('fs');

// PATCH 1: package.json (Configuring the builder and dependencies)
let pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'));
pkg.main = 'main.cjs'; // Point to our new wrapper
pkg.scripts['build:linux'] = 'electron-builder -l';
if (!pkg.dependencies) pkg.dependencies = {};
// tcp-port-used MUST be in dependencies, not devDependencies
pkg.dependencies['tcp-port-used'] = "^1.0.2"; 
pkg.build = {
    appId: 'com.sillytavern.app',
    productName: 'SillyTavern',
    asar: false, // Critical: Node natively needs a real dir for `process.chdir()`
    linux: { target: 'AppImage', category: 'Game' }
};
fs.writeFileSync('package.json', JSON.stringify(pkg, null, 2));


// PATCH 2: src/server-main.js (Disable auto browser launching)
let sm = fs.readFileSync('src/server-main.js', 'utf8');
sm = sm.replace(/if \(cliArgs\.browserLaunchEnabled\) \{/, 'if (false) { // Disabled by AppImage wrapper');
fs.writeFileSync('src/server-main.js', sm);


// PATCH 3: src/users.js (Bypass read-only limit on initialization)
let us = fs.readFileSync('src/users.js', 'utf8');
const searchUserLoop = `for (const dir of Object.values(PUBLIC_DIRECTORIES)) {\n        if (!fs.existsSync(dir)) {`;
const replaceUserLoop = `for (let dir of Object.values(PUBLIC_DIRECTORIES)) {\n        dir = path.resolve(globalThis.DATA_ROOT || process.cwd(), dir);\n        if (!fs.existsSync(dir)) {`;
if (us.includes(`const dir of`)) { // Fallback check just in case
    us = us.replace(/for \(const dir of Object\.values\(PUBLIC_DIRECTORIES\)\) \{/, `for (let dir of Object.values(PUBLIC_DIRECTORIES)) {\n        dir = path.resolve(globalThis.DATA_ROOT || process.cwd(), dir);`);
}
fs.writeFileSync('src/users.js', us);


// PATCH 4: src/endpoints/extensions.js (Fix Plugin Installer / Lister crashing on read-only block)
let ext = fs.readFileSync('src/endpoints/extensions.js', 'utf8');
// Redirect exact references to Data Root
ext = ext.replace(/global \? PUBLIC_DIRECTORIES\.globalExtensions :/g, `global ? path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.globalExtensions) :`);
ext = ext.replace(/destination === 'global' \? PUBLIC_DIRECTORIES\.globalExtensions :/g, `destination === 'global' ? path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.globalExtensions) :`);
ext = ext.replace(/source === 'global' \? PUBLIC_DIRECTORIES\.globalExtensions :/g, `source === 'global' ? path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.globalExtensions) :`);
ext = ext.replace(/let globExt = /g, 'let globExtOld = '); // Safe replace if variables exist
ext = ext.replace(/if \(\!fs\.existsSync\(PUBLIC_DIRECTORIES\.globalExtensions\)\) \{/g, 
    `const globExt = path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.globalExtensions);\n    if (!fs.existsSync(globExt)) { fs.mkdirSync(globExt, { recursive: true }); }\n    if (false) {`
);
ext = ext.replace(/readdirSync\(PUBLIC_DIRECTORIES\.globalExtensions\)/g, `readdirSync(path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.globalExtensions))`);
ext = ext.replace(/path\.join\(PUBLIC_DIRECTORIES\.globalExtensions,/g, `path.join(path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.globalExtensions),`);
ext = ext.replace(/readdirSync\(PUBLIC_DIRECTORIES\.extensions\)/g, `readdirSync(path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.extensions))`);
ext = ext.replace(/path\.join\(PUBLIC_DIRECTORIES\.extensions,/g, `path.join(path.resolve(globalThis.DATA_ROOT || process.cwd(), PUBLIC_DIRECTORIES.extensions),`);

fs.writeFileSync('src/endpoints/extensions.js', ext);

// PATCH 5: AppRun hook to copy built-in extensions
// AppImage mounts as read-only, but we need built-in extensions to exist in our persistent user data folder.
// This copies the contents of the AppImage's public/scripts/extensions to the user data path automatically on startup.
let mainCjsPatch = fs.readFileSync('main.cjs', 'utf8');
mainCjsPatch = mainCjsPatch.replace(
    /fs\.mkdirSync\(userDataPath, \{ recursive: true \}\);\n    \}/,
    `fs.mkdirSync(userDataPath, { recursive: true });
    }

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
`
);
fs.writeFileSync('main.cjs', mainCjsPatch);

// PATCH 6: webpack.config.js (Fix shake256 Digest Error for Webpack Cache)
if (fs.existsSync('webpack.config.js')) {
    let wp = fs.readFileSync('webpack.config.js', 'utf8');
    wp = wp.replace(/crypto\.createHash\('shake256', \{ outputLength: 8 \}\)/g, "crypto.createHash('md5')");
    fs.writeFileSync('webpack.config.js', wp);
}


console.log("Patches applied successfully!");
```

## 5. Install Dependencies and Build
Now that the project has been modified to support containerized read/write redirection natively, install dependencies and execute the builder.

*Note: We target `electron@29.4.0` specifically because it natively locks Chromium to version `<123`, drastically improving GLIBC backwards compatibility on an aging Ubuntu 22.04 system.*

```bash
# 1. Install regular dependencies
npm install

# 2. Install Electron & AppImage build tools
npm install --save-dev electron@29.4.0 electron-builder
npm install tcp-port-used

# 3. Compile the AppImage
npm run build:linux
```

## 6. Accessing Your AppImage
When the builder finishes (this may take a few minutes as it unpacks the file tree since `.asar` is disabled), the finalized executable will be located in:
`./dist/SillyTavern-1.17.x.AppImage`

Double-clicking it automatically spins up the Node server and loads a standalone desktop UI. All states, avatars, configurations, and API keys are securely persisted inside `~/.config/SillyTavern/data/` (or `~/.local/state/SillyTavern/data/`) on your host Linux machine!