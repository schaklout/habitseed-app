# Quickstart: Local Development Setup

**Created**: 2026-03-09  
**Purpose**: Get developers up and running locally in <15 minutes

---

## Prerequisites

- **Node.js**: 18.x or newer
- **npm** or **yarn**: Latest
- **Git**: Cloned repository
- **Xcode** (macOS): For iOS development/testing
- **Android Studio**: For Android emulator/testing
- **Expo Go app**: Installed on physical iOS/Android device (optional, for testing without build)

---

## Quick Setup (Frontend Only)

If you just want to run the React Native app locally without backend API:

```bash
# 1. Navigate to mobile directory
cd mobile

# 2. Install dependencies
npm install
# or
yarn install

# 3. Start Expo dev server
npm start
# or
expo start

# 4. Run on iOS emulator
Press 'i'

# 5. Run on Android emulator
Press 'a'

# 6. Run on physical device
Scan QR code with Expo Go app (iOS Camera or Android)
```

The app will run in **offline mode** using local AsyncStorage/SQLite.

---

## Full Setup (Frontend + Backend + Database)

### 1. Setup Backend (Node.js + Express)

```bash
# Navigate to API directory
cd api

# Install dependencies
npm install

# Create .env file from template
cp .env.example .env

# Edit .env with your MySQL credentials
nano .env

# Expected .env:
# DB_HOST=localhost
# DB_USER=root
# DB_PASS=your_secure_password
# DB_NAME=habitapp_dev
# JWT_SECRET=your_random_secret_key_at_least_32_characters_long
# PORT=3000
# NODE_ENV=development
```

### 2. Setup MySQL Database

**Option A: Using Docker** (recommended)

```bash
# From api/ directory, start MySQL via docker-compose
docker-compose up -d

# Verify MySQL is running
docker-compose logs mysql

# Database created: habitapp_dev
# User: root
# Password: root (change in docker-compose.yml for production!)
```

**Option B: Local MySQL Installation**

```bash
# macOS with Homebrew
brew install mysql
brew services start mysql

# Log in
mysql -u root

# Create database and user
CREATE DATABASE habitapp_dev;
CREATE USER 'habitapp'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON habitapp_dev.* TO 'habitapp'@'localhost';
FLUSH PRIVILEGES;
```

### 3. Run Database Migrations

```bash
# From api/ directory
npm run migrate

# Expected output: "All migrations completed successfully"
```

If migrations fail, manually check `api/src/migrations/` for error details.

### 4. Start Backend Server

```bash
# From api/ directory
npm run dev

# Expected output:
# Server running on http://localhost:3000
# Database connected to habitapp_dev
```

The backend is now ready at `https://localhost:3000` (note: development uses HTTP, HTTPS in production).

### 5. Configure Frontend to Use Backend

Edit `mobile/src/constants/api.ts`:

```ts
export const API_BASE_URL = __DEV__ 
  ? 'http://localhost:3000/v1'  // Local backend
  : 'https://api.habitseed.app/v1'; // Production backend
```

### 6. Start Frontend App (with Backend)

```bash
# From mobile/ directory
npm start

# Run on simulator/device
Press 'i' (iOS) or 'a' (Android)
```

The app will now sync data with local backend. Check browser console or Android Logcat for API calls.

---

## Testing the API Manually

Use **curl** or **Postman** to test endpoints:

### 1. Create a Habit

```bash
curl -X POST http://localhost:3000/v1/habits \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "name": "Morning jog",
    "description": "Run 30 minutes",
    "frequency": "DAILY"
  }'
```

For initial testing without JWT, you can remove auth middleware temporarily in `api/src/middleware/auth.ts`.

### 2. Get All Habits

```bash
curl http://localhost:3000/v1/habits \
  -H "Authorization: Bearer <TOKEN>"
```

### 3. Mark Completion

```bash
curl -X POST http://localhost:3000/v1/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
    "habitId": "550e8400-e29b-41d4-a716-446655440000",
    "completedAt": "2026-03-09",
    "notes": "Ran 5 km"
  }'
```

---

## Troubleshooting

### ❌ Expo won't start

```bash
# Clear cache and reinstall
rm -rf node_modules package-lock.json
npm install
npm start -- --no-cache
```

### ❌ MySQL won't connect

```bash
# Check if MySQL is running
mysql -u root -p

# If using Docker, verify container is running
docker-compose ps

# If container crashed, check logs
docker-compose logs mysql
```

### ❌ "Database habitapp_dev not found"

```bash
# Login and create manually
mysql -u root
CREATE DATABASE habitapp_dev;
USE habitapp_dev;

# Then run migrations
npm run migrate
```

### ❌ CORS errors from frontend

Make sure backend has proper CORS headers. Check `api/src/middleware/cors.ts`:

```ts
app.use(cors({
  origin: ['http://localhost:19000', 'http://localhost:19001'],
  credentials: true,
}));
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│         React Native (Expo)                 │
│         ├── AsyncStorage (local)            │
│         ├── SQLite (encrypted)              │
│         └── React Query (API client)        │
└──────────────────┬──────────────────────────┘
                   │ HTTPS/HTTP
                   ↓
┌─────────────────────────────────────────────┐
│       Express.js API (Node.js)              │
│       ├── Routes (habits, completions)      │
│       ├── Middleware (auth, validation)     │
│       └── Services (business logic)         │
└──────────────────┬──────────────────────────┘
                   │
                   ↓
         ┌─────────────────┐
         │  MySQL Database │
         │  (habits, compl) │
         └─────────────────┘
```

---

## Next Steps

1. **Run the app**: `npm start` from `mobile/`
2. **Create a habit**: Add one manually in the app
3. **Check sync**: Look at network tab in browser DevTools or Logcat
4. **Read core files**:
   - `mobile/src/services/habitService.ts` - Habit business logic
   - `api/src/controllers/habitController.ts` - API controllers
   - `api/src/models/Habit.ts` - Data model

---

## Common Development Tips

### Hot Reload / Fast Refresh

The app auto-reloads on code changes. For persistent issues, do a full rebuild:

```bash
# Clear Expo cache
npm start -- --clear

# Or from the Expo menu, press 'c' to clear cache
```

### Debugging Frontend

```bash
# Open React DevTools plugin (in Expo)
npm start
Press 'j' to open debugger

# Or use Chrome DevTools
http://localhost:19000/debugger-ui/
```

### Debugging Backend

```bash
# Enable debug logs
DEBUG=app:* npm run dev

# Or attach debugger
node --inspect-brk api/src/server.ts
# Then open chrome://inspect
```

### Database Inspection

```bash
# Using MySQL CLI
mysql -u root habitapp_dev

# View all habits
SELECT * FROM habits;

# View completions
SELECT * FROM completions;
```

---

## Environment Configuration

### Frontend (mobile/.env)

Not typically needed for local dev, but can add:

```env
EXPO_PUBLIC_API_URL=http://localhost:3000/v1
EXPO_PUBLIC_ENV=development
```

### Backend (api/.env)

```env
# Database
DB_HOST=localhost
DB_USER=root
DB_PASS=root
DB_NAME=habitapp_dev
DB_PORT=3306

# JWT
JWT_SECRET=dev-secret-key-change-in-production
JWT_EXPIRY=15m

# Server
PORT=3000
NODE_ENV=development
```

---

## CI/CD & Testing

### Frontend Tests

```bash
cd mobile
npm test                    # Jest unit tests
npm run test:integration   # Integration tests
npm run test:e2e           # Detox E2E tests
```

### Backend Tests

```bash
cd api
npm test           # Jest unit + integration tests
npm run test:cov   # Coverage report
```

---

## Next Time You Develop

```bash
# 1. Start backend
cd api && npm run dev

# 2. In another terminal, start frontend
cd mobile && npm start

# 3. Run app on emulator/device
# Press 'i' or 'a' or scan QR code
```

That's it! Your local dev environment is ready.
