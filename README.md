 # Kalba

Meditation & workshop platform — FastAPI backend + React Native (Expo) frontend.

## Prerequisites

| Tool | Version | Check |
|------|---------|-------|
| **Node.js** | 20+ | `node -v` |
| **Python** | 3.13 | `python3 --version` |
| **uv** (Python package manager) | latest | `uv --version` ([install](https://docs.astral.sh/uv/getting-started/installation/)) |
| **Xcode** | 15+ | App Store or `xcode-select --install` for CLI tools |
| **CocoaPods** | latest | `sudo gem install cocoapods` |
| **Docker** | latest | For PostgreSQL (`docker --version`) |
| **Git** | any | `git --version` |

## Running on a physical iPhone — step by step

### Step 1: Clone and set up the backend

```bash
cd kalba/backend-kalba
```

**1a. Start PostgreSQL**

```bash
docker compose -f docker-compose.local.yml up -d
```

This starts PostgreSQL on `localhost:5432` (user: `postgres`, pass: `postgres`, db: `kalba`).

**1b. Create `.env.local`**

```bash
cp .env.local.example .env.local   # or create manually
```

Contents of `backend-kalba/.env.local`:

```env
APP_ENV=local
DEBUG=true
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/kalba
JWT_SECRET_KEY=your-random-secret-key-here
GOOGLE_CLIENT_ID=your-google-web-client-id.apps.googleusercontent.com
GOOGLE_IOS_CLIENT_ID=your-google-ios-client-id.apps.googleusercontent.com
DAILY_API_KEY=your-daily-api-key
DAILY_DOMAIN=your-domain.daily.co
CORS_ORIGINS=["*"]
```

- Get Google OAuth client IDs from [Google Cloud Console](https://console.cloud.google.com/apis/credentials) — you need both a **Web** client and an **iOS** client (with bundle ID `com.kalba.app`).
- Get Daily.co credentials from [Daily.co Dashboard](https://dashboard.daily.co).

**1c. Install dependencies and run migrations**

```bash
uv sync
uv run alembic upgrade head
```

**1d. Start the backend**

```bash
uv run uvicorn app.main:app --reload --host 0.0.0.0
```

The `--host 0.0.0.0` flag is critical — it makes the backend accessible from your iPhone on the same Wi-Fi network.

**1e. Find your Mac's local IP**

```bash
ipconfig getifaddr en0
```

Note this IP (e.g. `192.168.0.223`). Your iPhone will connect to the backend at `http://<this-ip>:8000`.

### Step 2: Set up the frontend

```bash
cd kalba/frontend-kalba
```

**2a. Install dependencies**

```bash
npm install
```

**2b. Create `.env.local`**

```env
EXPO_PUBLIC_API_URL_WEB=http://localhost:8000/api/v1
EXPO_PUBLIC_API_URL_NATIVE=http://<YOUR_MAC_IP>:8000/api/v1
EXPO_PUBLIC_GOOGLE_CLIENT_ID=your-google-web-client-id.apps.googleusercontent.com
EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID=your-google-ios-client-id.apps.googleusercontent.com
```

Replace `<YOUR_MAC_IP>` with the IP from step 1e.

### Step 3: Apple Developer setup

You need a free or paid Apple Developer account to run on a physical device.

**3a. Open Xcode and sign in**

1. Open **Xcode** > **Settings** (Cmd+,) > **Accounts** tab
2. Click **+** and sign in with your Apple ID
3. This registers you as a developer (free account works for personal device testing)

**3b. Configure code signing**

1. Open the Xcode workspace:
   ```bash
   open ios/Kalba.xcworkspace
   ```
2. In the left sidebar, click the **Kalba** project (blue icon at the top)
3. Select the **Kalba** target under "Targets"
4. Go to the **Signing & Capabilities** tab
5. Check **Automatically manage signing**
6. Select your **Team** (your Apple ID / personal team) from the dropdown
7. If the bundle identifier `com.kalba.app` conflicts, change it to something unique like `com.yourname.kalba`

### Step 4: Install CocoaPods dependencies

```bash
cd ios
pod install
cd ..
```

If you get errors, try:
```bash
cd ios
pod deintegrate
pod install --repo-update
cd ..
```

### Step 5: Build and run on your iPhone

**5a. Connect your iPhone to your Mac via USB cable**

**5b. Trust the computer**

On your iPhone, tap **Trust** when the "Trust This Computer?" dialog appears.

**5c. Enable Developer Mode on iPhone** (iOS 16+)

1. Go to **Settings** > **Privacy & Security** > **Developer Mode**
2. Toggle it **ON**
3. Your iPhone will restart — confirm when prompted

**5d. Build and deploy**

```bash
npx expo run:ios --device
```

This will:
- List connected devices — select your iPhone
- Build the native app (~5-10 min first time)
- Install and launch on your device

**5e. Trust the developer certificate** (first time only)

After the app installs, you may see "Untrusted Developer" when trying to open it:

1. Go to **Settings** > **General** > **VPN & Device Management**
2. Find your developer certificate under "Developer App"
3. Tap it and then tap **Trust**
4. Open the app again

### Step 6: Subsequent runs

After the first build, you don't need to rebuild every time. Just start the dev server:

```bash
# Terminal 1 — Backend
cd backend-kalba
uv run uvicorn app.main:app --reload --host 0.0.0.0

# Terminal 2 — Frontend dev server
cd frontend-kalba
npx expo start --dev-client
```

Then open the **Kalba** app on your iPhone — it connects to the Metro dev server automatically (shake to open dev menu if needed).

If the app can't find the dev server:
1. Make sure iPhone and Mac are on the **same Wi-Fi network**
2. Shake your iPhone to open the dev menu
3. Tap **Change server URL** and enter `http://<YOUR_MAC_IP>:8081`

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `node: command not found` during build | Set Node path in `ios/.xcode.env.local`: `export NODE_BINARY=$(which node)` |
| Pod install fails | Run `cd ios && pod deintegrate && pod install --repo-update && cd ..` |
| "Untrusted Developer" on iPhone | Settings > General > VPN & Device Management > Trust your certificate |
| App can't reach backend | Verify Mac IP with `ipconfig getifaddr en0`, check both devices on same Wi-Fi, check `--host 0.0.0.0` |
| Google Sign-In fails | Verify iOS client ID in `.env.local` matches Google Cloud Console, bundle ID must match Xcode signing |
| Camera/mic not working | Grant permissions when prompted. Must use dev client build, not Expo Go |
| Build fails with signing error | Open `ios/Kalba.xcworkspace` in Xcode, fix signing under Signing & Capabilities |
| Metro bundler not found from device | Shake phone > Change server URL > enter `http://<MAC_IP>:8081` |
