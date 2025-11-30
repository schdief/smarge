# Implementation Plan: SMARGE MVP

**Branch**: `main` | **Date**: November 30, 2025 | **Spec**: [.specify/specification.md](../../.specify/specification.md)
**Input**: Feature specification from `.specify/specification.md`

## Summary

SMARGE is a Flutter mobile application for iOS (17+ minimum, targeting 26.1) that optimizes household energy costs by intelligently scheduling BEV and home battery charging. The MVP focuses on:

- **BEV charging optimization** using MySkoda API for state of charge and control
- **Dynamic pricing integration** via Tibber API (handling morning/afternoon optimization cycle)
- **Solar forecasting** using Open-Meteo free API
- **Two-phase daily optimization**: Morning (6:30 AM) for solar usage, Afternoon (1:30 PM) for overnight grid charging
- **Local-first architecture** with Hive/Isar storage, optional CloudKit sync in Phase 2
- **Notification-driven UX** to work within mobile background task limitations
- **Q.HOME integration** starting with monitoring (cloud API), with investigation needed for control capabilities

## Technical Context

**Language/Version**: Dart 3.5+, Flutter 3.24+  
**Primary Dependencies**: 
- State management: Riverpod or Provider
- Storage: Hive or Isar (local persistence)
- Networking: Dio or http package
- Background tasks: flutter_background_fetch, workmanager
- Notifications: flutter_local_notifications
- Charts: fl_chart
- Security: flutter_secure_storage (credential storage)

**Storage**: Hive/Isar for local persistence (~50MB for 1 year historical data), Flutter Secure Storage for API credentials  
**Testing**: flutter_test (unit/widget), integration_test (E2E), XCUITest equivalent via integration_test  
**Target Platform**: iOS 17+ (primary), targeting iOS 26.1, Android 6.0+ (Phase 2 future)  
**Project Type**: Mobile (Flutter single codebase, iOS deployment first)  
**Performance Goals**: 
- Optimization calculation: <10 seconds for 48-hour window
- App launch: <2 seconds
- Dashboard load: <1 second
- API response handling: <5 seconds per call

**Constraints**: 
- Background task execution: <30 seconds (iOS BGTaskScheduler limit)
- Memory: <150MB active, <50MB background
- Network: <5MB daily typical usage
- Battery: <1% drain per day from background tasks
- Offline capability required with graceful degradation

**Scale/Scope**: 
- Single household, single BEV initially (multi-vehicle Phase 3)
- 1 year historical data storage (~35K price records, ~35K device snapshots)
- 2 optimizations per day (morning + afternoon)
- 3-4 external API integrations (Tibber, MySkoda, Q.HOME, Open-Meteo)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### ✅ User Experience Principles

- **Notification-Driven Interaction**: ✅ COMPLIANT
  - Morning (6:30 AM) and afternoon (1:30 PM) optimization reminders designed
  - Notifications for charging events, errors, and price alerts specified
  - Two-phase approach works within platform background limitations

- **Simplicity First**: ✅ COMPLIANT
  - Default minimum BEV SoC (40%) with optional higher targets
  - One-tap "Optimize Now" and "Quick Charge" actions
  - Manual fallback for Q.HOME if API unavailable (Phase 1 acceptable)

- **Transparency & Trust**: ✅ COMPLIANT
  - Schedule displays rationale ("Low price period", "Solar surplus")
  - Cost estimates shown before/after optimization
  - Analytics tab for historical tracking (Phase 3)

### ✅ Architecture Principles

- **Mobile-First, Cross-Platform**: ✅ COMPLIANT
  - Flutter for iOS primary, Android-ready architecture
  - No required backend servers
  - All optimization runs locally

- **Smart Background Execution**: ✅ COMPLIANT
  - flutter_background_fetch + workmanager for periodic tasks
  - Commands scheduled via direct API calls (MySkoda, Q.HOME)
  - Notification reminders ensure user-triggered optimization

- **Privacy & Data Ownership**: ✅ COMPLIANT
  - Hive/Isar local storage (Phase 1)
  - CloudKit household sharing (Phase 2)
  - Flutter Secure Storage for credentials
  - No telemetry unless user consents

- **Resilience & Offline Capability**: ✅ COMPLIANT
  - Cached data for offline viewing
  - Queue commands when APIs offline
  - Last-known device states displayed
  - Data freshness indicators

### ✅ Optimization Principles

- **Solar Power Priority**: ✅ COMPLIANT
  - Open-Meteo API for weather forecasting
  - Algorithm prioritizes solar over grid
  - Opportunistic charging during surplus

- **Cost Minimization Secondary**: ✅ COMPLIANT
  - Tibber API integration for dynamic pricing
  - Re-optimization when prices change >20%
  - Feed-in rate (9¢/kWh) considered as opportunity cost

- **BEV Requirements Are Mandatory**: ✅ COMPLIANT
  - BEV goals have absolute priority
  - Home battery optimization is optional
  - Validation prevents unachievable goals
  - Early warnings if target cannot be met

- **Conservative Forecasting**: ✅ COMPLIANT
  - 10-15% buffer on solar predictions
  - Hourly re-optimization planned
  - Prefer excess charge over shortage

### ⚠️ Technical Principles

- **API-First Integration**: ⚠️ PARTIAL - **REQUIRES INVESTIGATION**
  - ✅ Tibber API: Documented, REST/GraphQL
  - ✅ MySkoda API: Community-documented, stable
  - ✅ Open-Meteo: Free, documented
  - ⚠️ Q.HOME API: **Needs reverse engineering** (REQ-INV-001 to REQ-INV-006)
  - **Gate Decision**: Proceed with manual fallback for MVP, investigate Q.HOME in parallel

- **Incremental Development**: ✅ COMPLIANT
  - Phase 1 MVP: Greedy algorithm, MySkoda only
  - Phase 2: Home battery + CloudKit
  - Phase 3: ML forecasting improvements
  - Phase 4: Polish and analytics

- **Testing & Validation**: ✅ COMPLIANT
  - ≥70% unit test coverage target
  - Integration tests with mock APIs
  - E2E testing via integration_test
  - Manual testing checklist defined

- **Extensibility**: ✅ COMPLIANT
  - Data models designed for CloudKit compatibility (Phase 2)
  - Pluggable architecture for pricing providers
  - Multi-vehicle support designed but deferred (Phase 3)

### 🎯 Overall Gate Status: **PROCEED WITH CONDITIONS**

**Conditions for Phase 0:**
1. Create Q.HOME API investigation task (3-week parallel track)
2. Design manual mode fallback for battery/wallbox control
3. Ensure MySkoda integration is priority path for MVP

**Re-evaluation Required After Phase 1:**
- If Q.HOME API documented → update contracts, add to MVP
- If Q.HOME API blocked → confirm manual mode UX acceptable
- If CloudKit design changes data models → verify no Phase 1 rework needed

## Project Structure

### Documentation (this feature)

```text
specs/main/
├── plan.md              # This file (implementation plan)
├── research.md          # Phase 0: Q.HOME API investigation, Flutter patterns, optimization algorithms
├── data-model.md        # Phase 1: All Hive/Isar data models with CloudKit compatibility
├── quickstart.md        # Phase 1: Developer setup, first build, running tests
├── contracts/           # Phase 1: External API contracts
│   ├── tibber-api.md         # Pricing API contract
│   ├── myskoda-api.md        # BEV state and control API
│   ├── qhome-api.md          # Battery/wallbox monitoring and control (TBD from investigation)
│   └── openmeteo-api.md      # Weather/solar forecast API
└── tasks.md             # Phase 2: Created by /speckit.tasks (NOT by this command)
```

### Source Code (repository root)

```text
smarge/
├── lib/
│   ├── main.dart                           # App entry point
│   ├── app.dart                            # MaterialApp configuration
│   │
│   ├── core/
│   │   ├── constants.dart                  # App-wide constants
│   │   ├── theme.dart                      # Theme configuration
│   │   └── routes.dart                     # Navigation routes
│   │
│   ├── models/                             # Data models (Hive/Isar)
│   │   ├── bev_goal.dart
│   │   ├── charge_schedule.dart
│   │   ├── device_snapshot.dart
│   │   ├── energy_price.dart
│   │   ├── solar_forecast.dart
│   │   ├── optimization_run.dart
│   │   └── config/                         # Configuration models
│   │       ├── system_config.dart
│   │       ├── solar_config.dart
│   │       ├── battery_config.dart
│   │       └── bev_config.dart
│   │
│   ├── services/                           # Business logic layer
│   │   ├── optimization/
│   │   │   ├── optimization_engine.dart    # Core optimization algorithm
│   │   │   ├── schedule_generator.dart     # Convert optimization to schedules
│   │   │   └── cost_calculator.dart        # Cost estimation
│   │   ├── forecasting/
│   │   │   ├── solar_forecaster.dart       # Solar production prediction
│   │   │   └── consumption_forecaster.dart # Household usage prediction
│   │   ├── api/                            # External API clients
│   │   │   ├── tibber_client.dart
│   │   │   ├── myskoda_client.dart
│   │   │   ├── qhome_client.dart
│   │   │   └── weather_client.dart
│   │   ├── background/
│   │   │   ├── background_task_service.dart
│   │   │   └── notification_manager.dart
│   │   └── storage/
│   │       ├── repository.dart             # Generic repository pattern
│   │       ├── hive_repository.dart        # Hive implementation
│   │       └── cache_manager.dart
│   │
│   ├── ui/                                 # Presentation layer
│   │   ├── dashboard/
│   │   │   ├── dashboard_screen.dart
│   │   │   └── widgets/
│   │   │       ├── energy_flow_card.dart
│   │   │       ├── bev_status_card.dart
│   │   │       ├── battery_status_card.dart
│   │   │       └── todays_cost_card.dart
│   │   ├── bev/
│   │   │   ├── bev_goal_screen.dart
│   │   │   ├── charge_schedule_screen.dart
│   │   │   └── quick_charge_sheet.dart
│   │   ├── analytics/
│   │   │   ├── analytics_screen.dart
│   │   │   └── widgets/
│   │   │       ├── price_chart.dart
│   │   │       └── cost_breakdown.dart
│   │   ├── settings/
│   │   │   ├── settings_screen.dart
│   │   │   ├── api_credentials_screen.dart
│   │   │   └── system_config_screen.dart
│   │   ├── onboarding/
│   │   │   └── onboarding_flow.dart
│   │   └── common/
│   │       └── widgets/                    # Shared widgets
│   │           ├── loading_indicator.dart
│   │           ├── error_display.dart
│   │           └── refresh_button.dart
│   │
│   └── utils/
│       ├── date_utils.dart
│       ├── formatters.dart
│       └── validators.dart
│
├── test/                                   # Tests
│   ├── unit/
│   │   ├── models/
│   │   ├── services/
│   │   │   ├── optimization_engine_test.dart
│   │   │   ├── forecasting_test.dart
│   │   │   └── api_clients_test.dart
│   │   └── utils/
│   ├── widget/                             # Widget tests
│   │   ├── dashboard_test.dart
│   │   └── bev_goal_test.dart
│   └── integration/                        # E2E tests
│       ├── app_test.dart
│       └── optimization_flow_test.dart
│
├── ios/                                    # iOS platform code
│   └── Runner/
│       └── AppDelegate.swift               # Background task registration
│
├── android/                                # Android platform code (Phase 2)
│
├── pubspec.yaml                            # Dependencies
├── analysis_options.yaml                   # Linting rules
└── README.md                               # Project overview
```

**Structure Decision**: **Mobile (Option 3)** - Flutter single codebase with platform-specific folders. The `lib/` structure follows Flutter best practices with clear separation of concerns: models (data), services (business logic), ui (presentation), and utils (helpers). The feature-based organization in `ui/` (dashboard, bev, analytics, settings) aligns with the app's main navigation tabs.

## Complexity Tracking

> **No constitutional violations requiring justification**

All architecture and technical decisions comply with constitution principles. The only "complexity" is the Q.HOME API investigation, which is acknowledged as a research task with a manual fallback strategy.

---

## Phase 0: Research & Investigation

**Status**: ✅ COMPLETE

**Output**: `research.md`

**Key Decisions:**
1. **Q.HOME API**: 3-week investigation timeline, manual mode fallback for MVP
2. **Background Tasks**: flutter_background_fetch for iOS BGTaskScheduler
3. **Optimization Algorithm**: Greedy for MVP → Linear programming in Phase 2
4. **Tibber**: GraphQL API with twice-daily polling
5. **MySkoda**: Reverse-engineered OAuth flow based on Python library
6. **Weather**: Open-Meteo free API (no key required)
7. **Storage**: Isar (better queries than Hive, CloudKit-ready)
8. **State Management**: Riverpod (type-safe, testable)

**Parallel Track**: Q.HOME API investigation (Weeks 1-3, non-blocking for MVP start)

---

## Phase 1: Design & Contracts

**Status**: ✅ COMPLETE

**Outputs:**
- `data-model.md` - All Isar models with CloudKit compatibility
- `contracts/tibber-api.md` - Tibber GraphQL API specification
- `contracts/myskoda-api.md` - MySkoda OAuth + REST API specification
- `contracts/openmeteo-api.md` - Open-Meteo weather API specification
- `contracts/qhome-api.md` - Q.HOME investigation plan + fallback strategy
- `quickstart.md` - Developer setup guide

**Data Models Defined:**
1. `BEVGoal` - User charging targets
2. `ChargeSchedule` - Planned charging sessions
3. `DeviceSnapshot` - Point-in-time device states
4. `EnergyPrice` - Tibber spot prices
5. `SolarForecast` - Weather-based production forecasts
6. `OptimizationRun` - Optimization execution records
7. `SystemConfig` - System configuration (singleton)
8. `UserPreferences` - Per-user settings (Phase 2)

**API Contracts Documented:**
- ✅ Tibber: GraphQL, fully documented
- ✅ MySkoda: REST, reverse-engineered, proven stable
- ✅ Open-Meteo: REST, free, well-documented
- ⏳ Q.HOME: Investigation in progress, manual fallback ready

---

## Phase 2: Implementation Planning

**Next Step**: Run `/speckit.tasks` to break down implementation into actionable tasks

**Not included in this plan** (separate command):
- Task breakdown by feature
- Sprint planning
- Issue creation
- Timeline estimation

---

## Re-evaluation: Constitution Check

*Final validation after Phase 1 design*

### Updated Assessment

All Phase 1 design decisions maintain constitutional compliance:

✅ **Data models**: Flat structures, CloudKit-ready (Phase 2 extensibility)  
✅ **API integrations**: 3 of 4 documented, 1 with investigation + fallback  
✅ **Testing strategy**: Unit/widget/integration test patterns defined  
✅ **Incremental development**: Greedy algorithm → LP evolution path clear  
✅ **User transparency**: All models include rationale/explanation fields  
✅ **Privacy**: Local storage (Isar), secure credentials (Flutter Secure Storage)  
✅ **Offline capability**: Caching strategy defined in each API contract  

### Outstanding Items

1. **Q.HOME API Investigation** (In Progress)
   - Week 1: Cloud API (monitoring)
   - Week 2: Local API (control) + Modbus
   - Week 3: Decision + documentation
   - **Fallback**: Manual mode UX designed and ready

2. **CloudKit Multi-User** (Phase 2)
   - Data models already compatible
   - No rework needed when adding sync
   - Platform-specific implementation deferred per constitution

### Final Gate Status: ✅ **APPROVED FOR IMPLEMENTATION**

**Recommendation**: Proceed with MVP development while Q.HOME investigation runs in parallel.

---

## Next Steps

1. ✅ **Planning Complete** - This document
2. 📋 **Create Tasks** - Run `/speckit.tasks` to generate task breakdown
3. 🚀 **Begin Implementation** - Start with core data models and services
4. 🔍 **Q.HOME Investigation** - Complete in parallel (non-blocking)
5. ✅ **Review Checkpoint** - After MVP core features, before UI polish

---

## Appendix: File Summary

| File | Purpose | Status |
|------|---------|--------|
| `plan.md` | This implementation plan | ✅ Complete |
| `research.md` | Technology decisions and investigations | ✅ Complete |
| `data-model.md` | All Isar data models | ✅ Complete |
| `contracts/tibber-api.md` | Tibber API specification | ✅ Complete |
| `contracts/myskoda-api.md` | MySkoda API specification | ✅ Complete |
| `contracts/openmeteo-api.md` | Weather API specification | ✅ Complete |
| `contracts/qhome-api.md` | Q.HOME investigation plan | ⏳ In Progress |
| `quickstart.md` | Developer setup guide | ✅ Complete |
| `tasks.md` | Implementation tasks | ⏳ Next: `/speckit.tasks` |

---

**Plan Version**: 1.0  
**Last Updated**: November 30, 2025  
**Next Review**: After task creation and before implementation starts

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
