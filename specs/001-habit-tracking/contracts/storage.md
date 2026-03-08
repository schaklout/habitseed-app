# Storage Contract: Local Data Schema

**Created**: 2026-03-09  
**Purpose**: Define local storage schema for AsyncStorage and SQLite persistence

---

## Overview

Mobile app uses two storage mechanisms:

1. **AsyncStorage**: Key-value store for app metadata, user settings, and temporary state
2. **SQLite**: Structured database for habits and completions (encrypted with password)

---

## AsyncStorage Keys

All values stored as JSON strings and re-parsed on retrieval.

### User & Session

| Key | Value Type | Encryption | Retention | Purpose |
|-----|-----------|-----------|-----------|---------|
| `@habitseed/userId` | string (UUID) | ❌ | Permanent | Device-specific user ID |
| `@habitseed/email` | string\|null | ✅ Keychain | Permanent | User email (if registered) |
| `@habitseed/jwt` | string | ✅ Keychain | Temporary | JWT access token (15-30 min) |
| `@habitseed/refreshToken` | string | ✅ Keychain | Temporary | JWT refresh token (7 days) |
| `@habitseed/lastSyncAt` | timestamp | ❌ | Permanent | Last successful API sync |

### Settings & Preferences

| Key | Value Type | Encryption | Default | Purpose |
|-----|-----------|-----------|---------|---------|
| `@habitseed/appVersion` | string | ❌ | "1.0.0" | Installed app version |
| `@habitseed/theme` | 'light'\|'dark' | ❌ | 'light' | UI theme preference |
| `@habitseed/notificationsEnabled` | boolean | ❌ | true | Push notification opt-in |
| `@habitseed/syncInterval` | number | ❌ | 300000 | Sync frequency in ms (5 min) |

### Sync Status

| Key | Value Type | Encryption | Purpose |
|-----|-----------|-----------|---------|
| `@habitseed/pendingChanges` | JSON array | ❌ | Queue of offline changes (habits/completions) |
| `@habitseed/failedSyncs` | JSON array | ❌ | Log of failed sync attempts (for debugging) |

---

### AsyncStorage Example

```ts
// Storing user ID (on first launch)
const userId = generateUUID();
await AsyncStorage.setItem('@habitseed/userId', userId);

// Storing JWT in Keychain (secure)
await Keychain.setGenericPassword('habitapp_jwt', token, {
  accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
});

// Storing settings
await AsyncStorage.setItem('@habitseed/theme', JSON.stringify('dark'));

// Retrieving
const userId = await AsyncStorage.getItem('@habitseed/userId');
const theme = await AsyncStorage.getItem('@habitseed/theme'); // "dark"
```

---

## SQLite Schema

Encrypted local database using SQLCipher or WatermelonDB.

### Tables

#### `users`

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE,
  createdAt DATETIME DEFAULT CURRENT_TIMESTAMP,
  lastSyncAt DATETIME,
  isLocalOnly BOOLEAN DEFAULT true
);
```

**Notes**:
- Single row in typical usage (single-user app)
- `id` matches `@habitseed/userId`
- `isLocalOnly` = true means user hasn't registered/synced to server

---

#### `habits`

```sql
CREATE TABLE habits (
  id TEXT PRIMARY KEY,
  userId TEXT NOT NULL,
  name TEXT NOT NULL,
  description TEXT,
  frequency TEXT NOT NULL,
  createdAt DATETIME NOT NULL,
  updatedAt DATETIME NOT NULL,
  isDeleted BOOLEAN DEFAULT false,
  deletedAt DATETIME,
  isSyncPending BOOLEAN DEFAULT true,
  FOREIGN KEY (userId) REFERENCES users(id),
  CHECK (length(name) > 0 AND length(name) <= 100),
  CHECK (frequency IN ('DAILY', 'WEEKLY', 'CUSTOM'))
);

CREATE INDEX idx_habits_userId ON habits(userId);
CREATE INDEX idx_habits_isDeleted ON habits(isDeleted);
```

**Columns**:
- `id`: UUID generated on create
- `userId`: Foreign key to users table
- `name`: 1-100 characters, trimmed
- `frequency`: DAILY, WEEKLY, or CUSTOM
- `createdAt`, `updatedAt`: ISO timestamp strings
- `isDeleted`: Soft delete flag
- `isSyncPending`: true if not yet synced to server

---

#### `completions`

```sql
CREATE TABLE completions (
  id TEXT PRIMARY KEY,
  habitId TEXT NOT NULL,
  userId TEXT NOT NULL,
  completedAt DATETIME NOT NULL,
  recordedAt DATETIME NOT NULL,
  notes TEXT,
  isSyncPending BOOLEAN DEFAULT true,
  FOREIGN KEY (habitId) REFERENCES habits(id),
  FOREIGN KEY (userId) REFERENCES users(id),
  UNIQUE(habitId, date(completedAt)),
  CHECK (length(notes) <= 255),
  CHECK (completedAt <= datetime('now'))
);

CREATE INDEX idx_completions_habitId ON completions(habitId);
CREATE INDEX idx_completions_userId ON completions(userId);
CREATE INDEX idx_completions_completedAt ON completions(completedAt);
```

**Columns**:
- `id`: UUID generated on create
- `habitId`: Foreign key to habits table
- `userId`: Denormalized from habit (for query efficiency)
- `completedAt`: ISO date/time string (the day completed)
- `recordedAt`: When record was created locally
- `notes`: Optional user notes (≤255 chars)
- `isSyncPending`: true if not yet synced to server
- **UNIQUE constraint**: Only one completion per habit per calendar day

---

## Sync Pending Tracking

Both `habits` and `completions` have an `isSyncPending` boolean flag:

- **true**: Created/modified locally, not yet synced to server
- **false**: Synced to server successfully

### Sync Algorithm

```ts
async function syncPendingChanges() {
  // 1. Get all pending changes
  const pendingHabits = await db.query(
    'SELECT * FROM habits WHERE isSyncPending = true'
  );
  const pendingCompletions = await db.query(
    'SELECT * FROM completions WHERE isSyncPending = true'
  );
  
  // 2. Batch upload to API
  const response = await api.post('/sync', {
    habits: pendingHabits,
    completions: pendingCompletions,
  });
  
  // 3. Mark synced locally
  if (response.ok) {
    await db.execute('UPDATE habits SET isSyncPending = false WHERE id IN (...)', ids);
    await db.execute('UPDATE completions SET isSyncPending = false WHERE id IN (...)', ids);
  }
}
```

---

## Encryption Strategy

### Local Encryption

**AsyncStorage sensitive data** (JWT, refresh token, email):
```ts
import * as Keychain from 'react-native-keychain';

// Use React Native Keychain (not AsyncStorage) for sensitive values
await Keychain.setGenericPassword('habitapp_jwt', jwtToken, {
  accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
});

const { password } = await Keychain.getGenericPassword();
// password = jwtToken
```

**SQLite database**:
```ts
// WatermelonDB with SQLCipher encryption
import SQLite from 'react-native-sqlite-storage';

const db = new Database(
  'habitseed.db',
  password: 'device-specific-passphrase' // Derived from device ID + user ID
);

// Or use WatermelonDB which abstracts this
import { Database } from '@nozbe/watermelondb';
const database = new Database({
  dbName: 'habitseed',
  schema: appSchema, // Your defined schema
  encryption: true, // Enable SQLCipher
  password: derivePassword(userId, deviceId),
});
```

---

## Migration Strategy

When app version updates, migrations are applied:

```ts
// migrations/001_initial_schema.ts
export const migration = {
  toVersion: 1,
  steps: [
    {
      sql: `CREATE TABLE users (
        id TEXT PRIMARY KEY,
        email TEXT UNIQUE,
        createdAt DATETIME DEFAULT CURRENT_TIMESTAMP
      );`,
    },
    // ... more tables
  ],
};

// migrations/002_add_notes_to_completions.ts
export const migration = {
  toVersion: 2,
  steps: [
    {
      sql: `ALTER TABLE completions ADD COLUMN notes TEXT DEFAULT '';`,
    },
  ],
};
```

---

## Data Retention & Cleanup

- **Habits**: Kept indefinitely (soft deleted, never purged)
- **Completions**: Kept indefinitely (provides history)
- **Failed sync logs**: Purged after 7 days (in `@habitseed/failedSyncs`)
- **JWT tokens**: Refreshed or cleared on logout

---

## Examples: Reading & Writing

### Create Habit

```ts
async function createHabit(name: string, frequency: string): Promise<Habit> {
  const habitId = generateUUID();
  const now = new Date().toISOString();
  
  await db.execute(
    `INSERT INTO habits (id, userId, name, frequency, createdAt, updatedAt, isSyncPending)
     VALUES (?, ?, ?, ?, ?, ?, ?)`,
    [habitId, userId, name, frequency, now, now, true]
  );
  
  return { id: habitId, userId, name, frequency, createdAt: now, updatedAt: now, isSyncPending: true };
}
```

### Mark Completion

```ts
async function markCompletion(habitId: string, date: Date): Promise<Completion> {
  const completionId = generateUUID();
  const completedAt = date.toISOString();
  const recordedAt = new Date().toISOString();
  
  const userId = await AsyncStorage.getItem('@habitseed/userId');
  
  try {
    await db.execute(
      `INSERT INTO completions (id, habitId, userId, completedAt, recordedAt, isSyncPending)
       VALUES (?, ?, ?, ?, ?, ?)`,
      [completionId, habitId, userId, completedAt, recordedAt, true]
    );
  } catch (e) {
    if (e.message.includes('UNIQUE')) {
      // Already marked today, return existing
      return db.queryOne('SELECT * FROM completions WHERE habitId = ? AND date(completedAt) = ?', [habitId, date]);
    }
    throw e;
  }
  
  return { id: completionId, habitId, userId, completedAt, recordedAt, isSyncPending: true };
}
```

### Calculate Stats

```ts
async function getHabitStats(habitId: string): Promise<Stats> {
  const completedCount = await db.queryOne(
    'SELECT COUNT(*) as count FROM completions WHERE habitId = ? AND isSyncPending = false',
    [habitId]
  );
  
  const currentStreak = calculateStreak(await db.query(
    'SELECT completedAt FROM completions WHERE habitId = ? ORDER BY completedAt DESC LIMIT 365',
    [habitId]
  ));
  
  return {
    completionCount: completedCount.count.toNumber(),
    currentStreak,
  };
}
```

---

## Notes for Developers

1. **Always use parameterized queries** to prevent SQL injection
2. **Encrypt sensitive fields** (JWT, email) via Keychain, not AsyncStorage
3. **Track sync status** with `isSyncPending` flag for reliable offline operation
4. **Use transactions** for multi-table operations to ensure consistency
5. **Test encryption** with real device (emulator may have limitations)
6. **Migration testing** required when modifying schema
