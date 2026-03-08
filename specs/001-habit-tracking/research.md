# Research: Habit Tracking App - Phase 0

**Created**: 2026-03-09  
**Purpose**: Resolve technical unknowns and establish best practices for React Native + Node.js backend

---

## 1. React Native + Expo Framework Decision

**Decision**: Use **Expo managed workflow** with optional bare workflow escape hatch.

**Rationale**:
- Expo provides out-of-the-box tooling, OTA updates, and simplified iOS/Android deployment
- Managed workflow reduces complexity and dependency management
- EAS Build handles cloud compilation (no local iOS build farm needed)
- Supports push notifications, AsyncStorage, secure keychain access
- Can eject to bare React Native if native modules required later

**Alternatives Considered**:
- **Bare React Native**: More control, more complexity; start with Managed to validate, eject later if needed
- **Native (Swift/Kotlin)**: Separate codebases, higher maintenance; rejected in favor of cross-platform single codebase

**Implementation**:
- Use `npx create-expo-app` or `expo init`
- Configure `eas.json` for CI/CD builds
- Target iOS 15+, Android 12+ minimum versions

---

## 2. Local Encryption Strategy (AsyncStorage + Keychain)

**Decision**: Use **react-native-keychain** for sensitive data + **SQLite with encryption** via `react-native-sqlite-storage` or **WatermelonDB** for structured data.

**Rationale**:
- `react-native-keychain` leverages native secure storage (Keychain on iOS, SecureSharedPreferences on Android)
- Encrypts JWT tokens, auth credentials without custom crypto code
- SQLite with encryption (SQLCipher+ pattern) protects habit/completion records at rest
- AsyncStorage alone is not encrypted; must use Keychain for auth tokens

**Alternatives Considered**:
- **react-native-encrypted-storage**: Simpler API but fewer platform-specific features; can use this alongside Keychain
- **File-based encryption**: Manual key management adds complexity; rejected
- **Plain AsyncStorage**: Violates security requirements; rejected

**Implementation**:
```ts
// Store sensitive data (JWT tokens, API keys)
import * as Keychain from 'react-native-keychain';

await Keychain.setGenericPassword('habitapp_jwt', token, {
  accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
});

// Store structured data (habits, completions) in encrypted SQLite or WatermelonDB
```

---

## 3. API Authentication & Security (JWT + HTTPS)

**Decision**: **JWT (JSON Web Tokens)** for stateless auth + **HTTPS/TLS 1.3+** for all traffic.

**Rationale**:
- JWT is stateless: API servers don't need session storage
- Tokens stored securely in device Keychain
- HTTPS ensures man-in-the-middle protection
- JWT refresh token pattern handles expiration gracefully

**Alternatives Considered**:
- **Session cookies**: Requires server-side session store; more complex for mobile
- **OAuth/SSO**: Introduces external dependency; overkill for single-user app
- **HTTP + custom encryption**: Deprecated; HTTPS is standard

**Implementation**:
- API generates short-lived JWT (15-30 min) + long-lived refresh token (7 days)
- Mobile stores JWT in Keychain, refresh token in encrypted secure storage
- Axios/Fetch interceptor refreshes token automatically before expiry
- All API calls use `https://` only; enforce in CSP/CORS headers

---

## 4. Node.js/Express Backend Security Patterns

**Decision**: **Express.js** with **helmet** middleware, **joi** input validation, **bcryptjs** for password hashing, **dotenv** for credentials.

**Rationale**:
- Helmet provides HTTP security headers (HSTS, CSP, X-Frame-Options, etc.)
- Joi validates all inputs server-side (never trust client validation)
- bcryptjs prevents plaintext password storage
- dotenv loads credentials from `.env` file (never hardcoded)

**Implementation**:
```js
// .env example (NEVER commit this)
DB_HOST=localhost
DB_USER=root
DB_PASS=secure_password
DB_NAME=habitapp
JWT_SECRET=very_long_random_string

// server.ts
import helmet from 'helmet';
import joi from 'joi';

app.use(helmet());

app.post('/api/habits', authenticate, async (req, res) => {
  // Validate input server-side
  const schema = joi.object({
    name: joi.string().min(1).max(100).required(),
  });
  const { error, value } = schema.validate(req.body);
  if (error) return res.status(400).json({ error: error.details });
  
  // Continue with sanitized input
});
```

---

## 5. MySQL Credential Management

**Decision**: Use **environment variables** stored in `.env` file (development) and **Docker Secrets** or **AWS Secrets Manager** (production).

**Rationale**:
- `.env` never committed to repo (enforce with `.gitignore`)
- CI/CD pipeline injects secrets at deploy time
- Separate development, staging, and production credentials
- Source of truth is secure environment, not code

**Implementation**:
```sh
# .gitignore
.env
.env.local

# .env.example (safe to commit)
DB_HOST=localhost
DB_USER=root
DB_PASS=change_me
```

---

## 6. Offline-First Sync Pattern

**Decision**: Local-first architecture with **conflict-free replicated data type (CRDT) principles** or eventual consistency model.

**Rationale**:
- App works entirely offline using local AsyncStorage + SQLite
- Background sync uploads changes when online
- Conflicts (user edits locally, server also changes) resolved via last-write-wins or user prompts
- Reduces API calls, improves perceived performance

**Alternatives Considered**:
- **Always-online**: Breaks on poor connectivity; rejected
- **Full CRDT library (Automerge, Yjs)**: Over-complex for simple habits; rejected

**Implementation**:
```ts
// Local create/update happens immediately
await habitService.createHabit({ name: 'Morning jog' });

// Background sync queues and uploads
useEffect(() => {
  const syncInterval = setInterval(async () => {
    if (isOnline) {
      await syncService.uploadPendingChanges();
    }
  }, 30000); // sync every 30 seconds if online
  
  return () => clearInterval(syncInterval);
}, [isOnline]);
```

---

## 7. Input Validation & Sanitization

**Decision**: **Multi-layer validation**: client-side (UX), server-side (security), database constraints.

**Rationale**:
- Client validation provides immediate feedback to users
- Server validation is security-critical (never trust client)
- Database constraints prevent invalid data at persistence layer

**Implementation**:
```ts
// Mobile validation (UX feedback)
const validateHabitName = (name: string): string | null => {
  if (!name || name.trim().length === 0) return 'Name required';
  if (name.length > 100) return 'Name too long (max 100 chars)';
  return null;
};

// Server validation (security)
const habitSchema = joi.object({
  name: joi.string().min(1).max(100).required().trim(),
  userId: joi.string().required(), // from JWT
});

// Database constraints
CREATE TABLE habits (
  id UUID PRIMARY KEY,
  userId UUID NOT NULL,
  name VARCHAR(100) NOT NULL CHECK (name != ''),
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (userId) REFERENCES users(id)
);
```

---

## 8. Database Schema & Timezone Handling

**Decision**: Store all timestamps in **UTC** in MySQL; client converts to local time for display.

**Rationale**:
- UTC avoids timezone ambiguities (DST transitions, user traveling)
- Client device always knows true local time
- Completion marks for "today" determined by device's local date

**Implementation**:
```sql
CREATE TABLE completions (
  id UUID PRIMARY KEY,
  habitId UUID NOT NULL,
  completedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP -- stored UTC
);

-- Client logic
const today = new Date().toLocaleDateString('en-CA'); // YYYY-MM-DD local
const completion = completions.find(c => 
  new Date(c.completedAt).toLocaleDateString('en-CA') === today
);
```

---

## 9. Testing Strategy

**Decision**: 
- **Unit tests**: Jest for services, validation, utils (80%+ coverage goal)
- **Integration tests**: Jest + supertest for API endpoints
- **E2E tests**: Detox for mobile UI flows (critical user journeys)

**Rationale**:
- Unit tests catch logic bugs early
- Integration tests verify API contracts
- E2E tests validate full user flows on real devices/emulators

**Implementation**:
```ts
// Jest unit test
describe('validateHabitName', () => {
  it('rejects empty names', () => {
    expect(validateHabitName('')).toBe('Name required');
  });
  it('accepts valid names', () => {
    expect(validateHabitName('Morning jog')).toBeNull();
  });
});

// Detox E2E test
describe('Create Habit Flow', () => {
  beforeAll(async () => {
    await device.launchApp();
  });
  
  it('should create a habit and display it in the list', async () => {
    await element(by.id('addButton')).tap();
    await element(by.id('habitNameInput')).typeText('Morning jog');
    await element(by.id('submitButton')).tap();
    await expect(element(by.text('Morning jog'))).toBeVisible();
  });
});
```

---

## Summary of Decisions

| Area | Choice | Key Dependencies |
|------|--------|------------------|
| Mobile Framework | Expo (React Native) | `expo`, `react-native` |
| Local Storage | Keychain + SQLite/WatermelonDB | `react-native-keychain`, `react-native-sqlite-storage` |
| Auth | JWT + HTTPS | `jsonwebtoken`, `axios` with interceptors |
| Backend | Express.js | `express`, `helmet`, `joi` for validation |
| DB Credentials | Environment variables | `dotenv` |
| Validation | Multi-layer (client + server + DB) | `joi`, database constraints |
| Sync Strategy | Eventual consistency, local-first | Background sync service |
| Testing | Jest + Detox | `jest`, `detox` |

All decisions prioritize **security, simplicity, and mobile-first design** as per project constitution.
