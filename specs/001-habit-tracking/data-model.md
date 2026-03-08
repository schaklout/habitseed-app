# Data Model: Habit Tracking App

**Created**: 2026-03-09  
**Purpose**: Define entities, relationships, validation rules, and state transitions

---

## Entity Definitions

### 1. User

Represents a device user (single user per app instance).

| Field | Type | Required | Validation | Notes |
|-------|------|----------|-----------|-------|
| `id` | UUID | ✅ | Must be unique | Primary key; generated on first app launch |
| `email` | string | ❌ | Valid email format, max 255 chars | Optional for offline-only users |
| `createdAt` | timestamp | ✅ | Stored in UTC | Device creation timestamp |
| `lastSyncAt` | timestamp | ❌ | Stored in UTC | Last successful API sync |

**Constraints**:
- One user per device (local-first)
- Email optional (can use app without registration for offline use)
- If email provided, must be unique across users in MySQL (when syncing)

---

### 2. Habit

Represents a habit the user wants to track.

| Field | Type | Required | Validation | Notes |
|-------|------|----------|-----------|-------|
| `id` | UUID | ✅ | Must be unique | Primary key; generated client-side |
| `userId` | UUID | ✅ | Foreign key to User | Ties habit to user |
| `name` | string | ✅ | 1-100 chars, no leading/trailing spaces | Habit description (e.g., "Morning jog") |
| `description` | string | ❌ | Max 500 chars | Optional details/notes |
| `frequency` | enum | ✅ | DAILY, WEEKLY, CUSTOM | How often habit should repeat |
| `createdAt` | timestamp | ✅ | Stored in UTC | Created date |
| `updatedAt` | timestamp | ✅ | Stored in UTC | Last modified date |
| `isDeleted` | boolean | ✅ | false by default | Soft delete flag (for sync conflicts) |
| `deletedAt` | timestamp | ❌ | Stored in UTC | When habit was deleted |

**Validation Rules**:
- `name` must be non-empty after trimming whitespace
- `name` max 100 characters
- `frequency` must be one of: `DAILY`, `WEEKLY`, `CUSTOM`
- No habit can be created without a valid `userId`

**State Transitions**:
```
[Created] → [Active] → [Deleted]
```

---

### 3. CompletionRecord

Represents marking a habit as completed on a specific date.

| Field | Type | Required | Validation | Notes |
|-------|------|----------|-----------|-------|
| `id` | UUID | ✅ | Must be unique | Primary key |
| `habitId` | UUID | ✅ | Foreign key to Habit | Ties completion to habit |
| `userId` | UUID | ✅ | Foreign key to User | Denormalized for sync/query efficiency |
| `completedAt` | timestamp | ✅ | Stored in UTC, end of day | When completion was recorded |
| `recordedAt` | timestamp | ✅ | Stored in UTC | When record was created locally |
| `notes` | string | ❌ | Max 255 chars | Optional completion notes (e.g., "ran 5km") |
| `isSyncPending` | boolean | ✅ | true initially | Sync status for offline-first |

**Validation Rules**:
- `completedAt` must be a valid date
- Only one completion per habit per calendar day (natural key: habitId + date)
- `completedAt` must not be in the future
- `recordedAt` must be >= `completedAt` and <= current time

**Business Logic**:
- User can mark completion for today or past dates
- Completion is immutable once synced to server
- Soft delete via `isSyncPending` = false if user undoes (mobile UX)

---

## Relationships

```
User (1) ─→ (many) Habit
User (1) ─→ (many) CompletionRecord
Habit (1) ─→ (many) CompletionRecord
```

### Integrity Rules

1. **Habit depends on User**: If user deleted, habits soft-deleted
2. **CompletionRecord depends on Habit**: If habit deleted, completions soft-deleted
3. **Unique constraint**: Only one completion per habit per day
4. **Referential integrity**: All foreign keys must exist

---

## Data Validation Rules Summary

### Habit Creation
```ts
interface CreateHabitInput {
  name: string;        // 1-100 non-empty chars
  description?: string; // 0-500 chars
  frequency: 'DAILY' | 'WEEKLY' | 'CUSTOM';
}

// Validation
function validateHabit(input: CreateHabitInput): ValidationError[] {
  const errors = [];
  
  if (!input.name || input.name.trim().length === 0) {
    errors.push('Name required');
  }
  if (input.name.length > 100) {
    errors.push('Name exceeds 100 characters');
  }
  if (!['DAILY', 'WEEKLY', 'CUSTOM'].includes(input.frequency)) {
    errors.push('Invalid frequency');
  }
  
  return errors;
}
```

### Completion Marking
```ts
interface MarkCompletionInput {
  habitId: string;
  date: Date;          // Past or today, not future
  notes?: string;      // 0-255 chars
}

// Validation
function validateCompletion(input: MarkCompletionInput): ValidationError[] {
  const errors = [];
  
  if (!isValidUUID(input.habitId)) {
    errors.push('Invalid habit ID');
  }
  if (input.date > new Date()) {
    errors.push('Cannot mark completion for future date');
  }
  if (input.notes && input.notes.length > 255) {
    errors.push('Notes exceed 255 characters');
  }
  
  return errors;
}
```

---

## Calculated Fields (Not Stored)

These are computed in-memory for display, not persisted.

| Name | Computes | Logic |
|------|----------|-------|
| `currentStreak` | Days in a row completed | Count consecutive calendars days with >= 1 completion |
| `longestStreak` | All-time best streak | Max consecutive days completed |
| `completionCount` | Total completions | COUNT(*) group by habitId |
| `completionRate` | Percentage completed | completionCount / days_tracked |
| `lastCompletedDate` | Most recent completion | MAX(completedAt) group by habitId |

### Example: Current Streak Calculation
```ts
function calcCurrentStreak(habit: Habit, completions: CompletionRecord[]): number {
  const habitCompletions = completions
    .filter(c => c.habitId === habit.id)
    .sort((a, b) => new Date(b.completedAt).getTime() - new Date(a.completedAt).getTime());
  
  if (!habitCompletions.length) return 0;
  
  let streak = 1;
  let expectedDate = new Date(habitCompletions[0].completedAt);
  expectedDate.setDate(expectedDate.getDate() - 1);
  
  for (let i = 1; i < habitCompletions.length; i++) {
    const completionDate = new Date(habitCompletions[i].completedAt);
    if (completionDate.toDateString() === expectedDate.toDateString()) {
      streak++;
      expectedDate.setDate(expectedDate.getDate() - 1);
    } else {
      break;
    }
  }
  
  return streak;
}
```

---

## Constraints Summary

| Type | Details |
|------|---------|
| **NOT NULL** | id, userId, habitId, name, frequency, createdAt, updatedAt |
| **UNIQUE** | User.email (if not null), Habit.id, CompletionRecord.id, (habitId, completedAt) composite |
| **CHECK** | Habit.name != '' (after trim), CompletionRecord.completedAt <= now |
| **FOREIGN KEY** | Habit.userId → User.id, CompletionRecord.habitId → Habit.id, CompletionRecord.userId → User.id |
| **DEFAULT** | isDeleted=false, isSyncPending=true, createdAt=CURRENT_TIMESTAMP |

---

## Implementation Notes

### Client (React Native / AsyncStorage)
- Store as JSON in encrypted AsyncStorage or SQLite
- Sync status tracked locally (`isSyncPending` flag)
- User ID generated on first launch via UUID v4

### Server (MySQL)
- Tables use DATETIME(6) for millisecond precision
- All timestamps stored in UTC
- Indexes on userId, habitId, completedAt for query performance
- Soft deletes via `isDeleted` flag (preserves audit trail)
