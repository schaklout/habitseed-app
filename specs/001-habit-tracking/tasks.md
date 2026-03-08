---
description: "Implementation task list for mobile habit tracking application"
---

# Tasks: Habit Tracking App

**Input**: Design documents from `specs/001-habit-tracking/`  
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/

**Tests**: Optional - only include if explicitly requested in spec  
**Organization**: Tasks grouped by user story to enable independent implementation and testing

## Format: [ID] [P?] [Story?] Description

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Mobile**: `mobile/src/`, `mobile/tests/`
- **Backend**: `api/src/`, `api/tests/`
- Shared: `.env.example`, `docker-compose.yml`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization for both mobile and backend

- [ ] T001 Create mobile and API project directories: `mobile/`, `api/`
- [ ] T002 [P] Initialize React Native/Expo project: `mobile/package.json`, `mobile/app.json`, `mobile/eas.json`
- [ ] T003 [P] Initialize Node.js/Express backend: `api/package.json`, `api/src/server.ts`
- [ ] T004 [P] Configure TypeScript for mobile: `mobile/tsconfig.json`, `mobile/.eslintrc.js`, `mobile/.prettierrc`
- [ ] T005 [P] Configure TypeScript for backend: `api/tsconfig.json`, `api/.eslintrc.js`, `api/.prettierrc`
- [ ] T006 [P] Setup Docker Compose for local MySQL: `api/docker-compose.yml`
- [ ] T007 [P] Create environment configuration templates: `api/.env.example`, `mobile/.env.example`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**CRITICAL**: No user story work can begin until this phase is complete

### Mobile Foundation

- [ ] T008 Setup local storage infrastructure: `mobile/src/utils/storage.ts`, `mobile/src/utils/keychain.ts`
- [ ] T009 [P] Implement encryption utilities: `mobile/src/utils/encryption.ts`
- [ ] T010 [P] Create type definitions for domain models: `mobile/src/types/index.ts`
- [ ] T011 [P] Setup state management (Redux/Zustand): `mobile/src/state/store.ts`
- [ ] T012 [P] Setup API client with interceptors: `mobile/src/services/apiClient.ts`
- [ ] T013 [P] Implement JWT token management: `mobile/src/services/authService.ts`
- [ ] T014 [P] Implement input validation utilities: `mobile/src/utils/validation.ts`
- [ ] T015 [P] Implement date/timezone utilities: `mobile/src/utils/dateUtils.ts`
- [ ] T016 [P] Setup ESLint, Prettier, and pre-commit hooks for mobile

### Backend Foundation

- [ ] T017 Setup Express.js server structure: `api/src/app.ts`, `api/src/server.ts`
- [ ] T018 [P] Configure middleware stack: `api/src/middleware/cors.ts`, `api/src/middleware/helmet.ts`, `api/src/middleware/errorHandler.ts`
- [ ] T019 [P] Setup MySQL database connection: `api/src/config/database.ts`
- [ ] T020 [P] Create migration framework: `api/src/migrations/`, `api/scripts/migrate.ts`
- [ ] T021 [P] Setup JWT authentication middleware: `api/src/middleware/auth.ts`
- [ ] T022 [P] Create input validation middleware: `api/src/middleware/inputValidation.ts`
- [ ] T023 [P] Create type definitions for domain models: `api/src/types/index.ts`
- [ ] T024 [P] Implement logging utility: `api/src/utils/logger.ts`
- [ ] T025 [P] Setup ESLint, Prettier, and pre-commit hooks for backend

### Database Schema

- [ ] T026 Create migration for User table: `api/src/migrations/001_create_users_table.ts`
- [ ] T027 Create migration for Habit table: `api/src/migrations/002_create_habits_table.ts`
- [ ] T028 Create migration for CompletionRecord table: `api/src/migrations/003_create_completions_table.ts`
- [ ] T029 [P] Create indexes for performance: `api/src/migrations/004_create_indexes.ts`

Checkpoint: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Create and View Habits (Priority: P1) MVP

**Goal**: Users can create new habits and see a list of all their habits

**Independent Test**: Launch app in fresh state, create habit "Morning jog", verify it appears in list

### Backend: Habit CRUD Endpoints

- [ ] T030 Create Habit data model: `api/src/models/Habit.ts`
- [ ] T031 [P] Create Habit repository: `api/src/repositories/habitRepository.ts`
- [ ] T032 [P] Create Habit service: `api/src/services/habitService.ts`
- [ ] T033 [P] Create Habit controller: `api/src/controllers/habitController.ts`
- [ ] T034 [US1] Setup Habit routes: `api/src/routes/habits.ts`
- [ ] T035 [P] [US1] Create Habit validation schema: `api/src/schemas/habitSchema.ts`
- [ ] T036 [US1] Implement error handling for habit endpoints
- [ ] T037 [P] [US1] Write unit tests for habitService: `api/tests/unit/services/habitService.test.ts`
- [ ] T038 [P] [US1] Write contract tests for habit endpoints: `api/tests/contract/endpoints/habits.test.ts`

### Mobile: Habit UI and Local Storage

- [ ] T039 [P] [US1] Create Habit types: `mobile/src/types/index.ts`
- [ ] T040 [P] [US1] Implement local habit storage: `mobile/src/services/habitService.ts`
- [ ] T041 [P] [US1] Create Redux habit slices: `mobile/src/state/habitSlice.ts`
- [ ] T042 [P] [US1] Create HabitItem component: `mobile/src/components/HabitItem.tsx`
- [ ] T043 [P] [US1] Create HabitsList component: `mobile/src/components/HabitsList.tsx`
- [ ] T044 [US1] Create HabitsListScreen: `mobile/src/screens/HabitsListScreen.tsx`
- [ ] T045 [P] [US1] Create CreateHabitScreen: `mobile/src/screens/CreateHabitScreen.tsx`
- [ ] T046 [P] [US1] Setup navigation: `mobile/src/navigation/RootNavigator.tsx`
- [ ] T047 [US1] Connect habit creation to local storage and state
- [ ] T048 [P] [US1] Write unit tests for habitService: `mobile/tests/unit/services/habitService.test.ts`
- [ ] T049 [P] [US1] Write integration tests for habit creation: `mobile/tests/integration/habitCreation.test.ts`

### API Integration and Sync

- [ ] T050 [US1] Implement sync service for habits: `mobile/src/services/syncService.ts`
- [ ] T051 [P] [US1] Add sync pending tracking to state: `mobile/src/state/habitSlice.ts`
- [ ] T052 [US1] Connect habits to API endpoints via sync service
- [ ] T053 [P] [US1] Write E2E test for habit creation: `mobile/tests/e2e/habitCreation.e2e.ts`

Checkpoint: User Story 1 fully functional and testable

---

## Phase 4: User Story 2 - Mark Habits as Completed (Priority: P2)

**Goal**: Users can mark habits as completed for today with UI feedback

**Independent Test**: With existing habit, tap completion button and verify UI updates

### Backend: Completion Endpoints

- [ ] T054 Create CompletionRecord model: `api/src/models/Completion.ts`
- [ ] T055 [P] Create Completion repository: `api/src/repositories/completionRepository.ts`
- [ ] T056 [P] Create Completion service: `api/src/services/completionService.ts`
- [ ] T057 [P] Create Completion controller: `api/src/controllers/completionController.ts`
- [ ] T058 [US2] Setup Completion routes: `api/src/routes/completions.ts`
- [ ] T059 [P] [US2] Create Completion validation schema: `api/src/schemas/completionSchema.ts`
- [ ] T060 [P] [US2] Write unit tests for completionService: `api/tests/unit/services/completionService.test.ts`
- [ ] T061 [P] [US2] Write contract tests for completion endpoints: `api/tests/contract/endpoints/completions.test.ts`

### Mobile: Completion UI and Recording

- [ ] T062 [P] [US2] Create CompletionRecord types: `mobile/src/types/index.ts`
- [ ] T063 [P] [US2] Implement local completion storage: `mobile/src/services/completionService.ts`
- [ ] T064 [P] [US2] Create Redux completion slices: `mobile/src/state/completionSlice.ts`
- [ ] T065 [P] [US2] Add completion button to HabitItem: `mobile/src/components/HabitItem.tsx`
- [ ] T066 [US2] Implement visual feedback for completed habits
- [ ] T067 [P] [US2] Implement completion date handling: `mobile/src/utils/dateUtils.ts`
- [ ] T068 [US2] Connect completion button to storage and state
- [ ] T069 [P] [US2] Write unit tests for completionService: `mobile/tests/unit/services/completionService.test.ts`
- [ ] T070 [P] [US2] Write integration tests for completion marking: `mobile/tests/integration/completionMarking.test.ts`
- [ ] T071 [P] [US2] Write E2E test for completion workflow: `mobile/tests/e2e/markCompletion.e2e.ts`

### Sync Integration

- [ ] T072 [US2] Update syncService for completions: `mobile/src/services/syncService.ts`
- [ ] T073 [P] [US2] Implement completion conflict resolution

Checkpoint: User Stories 1 and 2 fully functional

---

## Phase 5: User Story 3 - View Progress Tracking (Priority: P3)

**Goal**: Users can see completion count and streaks for each habit

**Independent Test**: After marking completions, view habit detail and verify stats displayed

### Backend: Statistics Endpoints

- [ ] T074 [US3] Add streak/count query methods: `api/src/services/completionService.ts`
- [ ] T075 [P] [US3] Create progress stats endpoint: `api/src/routes/completions.ts`
- [ ] T076 [P] [US3] Write tests for streak calculations: `api/tests/unit/services/completionService.test.ts`

### Mobile: Progress UI

- [ ] T077 [P] [US3] Create ProgressCard component: `mobile/src/components/ProgressCard.tsx`
- [ ] T078 [P] [US3] Create HabitDetailScreen: `mobile/src/screens/HabitDetailScreen.tsx`
- [ ] T079 [US3] Implement streak calculation: `mobile/src/utils/progressUtils.ts`
- [ ] T080 [P] [US3] Add navigation to habit detail
- [ ] T081 [P] [US3] Display progress stats in HabitItem
- [ ] T082 [US3] Connect progress display to completion records
- [ ] T083 [P] [US3] Write unit tests for streak logic: `mobile/tests/unit/utils/progressUtils.test.ts`
- [ ] T084 [P] [US3] Write integration tests for progress display: `mobile/tests/integration/progressTracking.test.ts`
- [ ] T085 [P] [US3] Write E2E test for viewing progress: `mobile/tests/e2e/viewProgress.e2e.ts`

Checkpoint: All user stories fully functional and tested

---

## Phase 6: Polish and Cross-Cutting Concerns

**Purpose**: Improvements affecting multiple user stories

### Security and Data Protection

- [ ] T086 [P] Encrypt AsyncStorage: `mobile/src/utils/encryptedStorage.ts`
- [ ] T087 [P] Integrate react-native-keychain: `mobile/src/services/authService.ts`
- [ ] T088 [P] Setup SQLite encryption: `api/src/config/database.ts`
- [ ] T089 [P] Add rate limiting middleware: `api/src/middleware/rateLimit.ts`
- [ ] T090 [P] Add HTTPS validation: `api/src/middleware/https.ts`

### Error Handling and Logging

- [ ] T091 [P] Enhance error handling across services
- [ ] T092 [P] Implement comprehensive logging: `mobile/src/utils/logger.ts`, `api/src/utils/logger.ts`
- [ ] T093 [P] Add error boundaries: `mobile/src/components/ErrorBoundary.tsx`
- [ ] T094 [P] Setup structured backend logging

### Performance and UX

- [ ] T095 [P] Implement loading states for async operations
- [ ] T096 [P] Add pull-to-refresh: `mobile/src/screens/HabitsListScreen.tsx`
- [ ] T097 [P] Optimize database queries
- [ ] T098 [P] Add offline indicator: `mobile/src/components/`
- [ ] T099 [P] Add success/error notifications

### Testing Coverage

- [ ] T100 [P] Reach 80% code coverage (mobile): `mobile/tests/unit/`
- [ ] T101 [P] Reach 80% code coverage (backend): `api/tests/unit/`
- [ ] T102 [P] Add full lifecycle integration test
- [ ] T103 [P] Setup CI/CD pipeline: GitHub Actions

### Documentation

- [ ] T104 [P] Finalize API documentation: `specs/001-habit-tracking/contracts/api.md`
- [ ] T105 [P] Finalize storage documentation: `specs/001-habit-tracking/contracts/storage.md`
- [ ] T106 [P] Create developer guide: `docs/DEVELOPMENT.md`
- [ ] T107 [P] Create deployment guide: `docs/DEPLOYMENT.md`
- [ ] T108 [P] Add JSDoc comments to all exports

### Deployment

- [ ] T109 [P] Configure Expo build: `mobile/eas.json`
- [ ] T110 [P] Setup backend deployment
- [ ] T111 [P] Configure environment separation
- [ ] T112 [P] Setup database backup procedures
- [ ] T113 [P] Test on real devices (iOS/Android)
- [ ] T114 Validate quickstart.md: `specs/001-habit-tracking/quickstart.md`

---

## Dependencies and Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational completion
  - US1 (Phase 3): Can start after Foundational
  - US2 (Phase 4): Depends on US1 backend models, can parallel after Foundational
  - US3 (Phase 5): Depends on US2 data, can parallel after US2 backend
- **Polish (Phase 6)**: Depends on desired user stories completion

### User Story Dependencies

- **US1 (P1)**: Start after Foundational (T008-T029)
- **US2 (P2)**: Blocks on US1 backend models (T030-T035)
- **US3 (P3)**: Blocks on US2 completion records (T054-T058)

### Parallel Opportunities

- Phase 1: All [P] tasks (T002-T007)
- Phase 2: Mobile tasks parallel with backend tasks (T008-T016 vs T017-T025)
- Phase 3: Backend API (T030-T038) parallel with mobile UI (T039-T053)
- Phase 4: Backend (T054-T061) parallel with mobile (T062-T073)
- Phase 5: Backend less critical, mobile can parallel (T077-T085)
- Phase 6: All [P] tasks fully parallel

---

## Success Criteria

- Phase 1: All projects initialized, dependencies installed
- Phase 2: All foundation services working, database connected
- Phase 3: US1 fully functional - create and view habits works
- Phase 4: US2 fully functional - mark completions works
- Phase 5: US3 fully functional - progress stats displays correctly
- Phase 6: All tests passing, documentation complete, app ready for release

---

## Estimated Effort

Phase 1: 1 day  
Phase 2: 2 days  
Phase 3: 3 days  
Phase 4: 2 days  
Phase 5: 2 days  
Phase 6: 2 days  
Total: 12 days with 4-person team
