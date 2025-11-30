# Implementation Tasks: SMARGE MVP

**Branch**: `main` | **Generated**: November 30, 2025  
**Spec**: [.specify/specification.md](../../.specify/specification.md)  
**Plan**: [plan.md](plan.md)

---

## Phase 1: Project Setup & Foundation

**Goal**: Initialize Flutter project with core dependencies and development environment.

### Tasks
- [ ] T001 [P] Create Flutter project with iOS minimum version 17.0, target iOS 26.1 using `flutter create smarge --platforms=ios`
- [ ] T002 [P] Configure pubspec.yaml dependencies: riverpod ^2.4.0, isar ^3.1.0, dio ^5.4.0, flutter_background_fetch ^1.2.0, flutter_local_notifications ^16.0.0, fl_chart ^0.65.0, flutter_secure_storage ^9.0.0
- [ ] T003 Install Isar Generator and build_runner for code generation: `flutter pub add -d isar_generator build_runner`
- [ ] T004 [P] Create initial project structure: `lib/models/`, `lib/services/`, `lib/providers/`, `lib/screens/`, `lib/widgets/`, `lib/utils/`
- [ ] T005 Configure iOS entitlements in `ios/Runner/Runner.entitlements` for background fetch and notifications
- [ ] T006 [P] Setup flutter_test package and create `test/` directory structure matching `lib/`
- [ ] T007 Setup integration_test package and create `integration_test/` directory
- [ ] T008 Create `.env` file template with placeholder API keys: TIBBER_TOKEN, MYSKODA_USERNAME, MYSKODA_PASSWORD

**Dependencies**: None (foundation tasks)

**Test Criteria**: 
- `flutter doctor` shows no errors for iOS
- `flutter run` launches blank app on iOS device/simulator
- `flutter test` executes successfully (even with no tests)

---

## Phase 2: Data Models & Local Storage

**Goal**: Implement CloudKit-ready Isar models for all domain entities.

**Depends On**: Phase 1 (T001-T008)

### Tasks
- [ ] T009 [P] Create `lib/models/bev_goal.dart` with BEVGoal model (defaultMinimumSoC, specificTargets with targetSoC/deadlineTime/isActive)
- [ ] T010 [P] Create `lib/models/charge_schedule.dart` with ChargeSchedule model (sessions with device/startTime/endTime/powerSource/estimatedKWh/estimatedCost/rationale)
- [ ] T011 [P] Create `lib/models/device_snapshot.dart` with DeviceSnapshot model (deviceType, timestamp, stateOfCharge, powerOutput, isCharging)
- [ ] T012 [P] Create `lib/models/energy_price.dart` with EnergyPrice model (timestamp, pricePerKWh, currency, source, isEstimated)
- [ ] T013 [P] Create `lib/models/solar_forecast.dart` with SolarForecast model (timestamp, expectedKWh, cloudCover, sunriseTime, sunsetTime)
- [ ] T014 [P] Create `lib/models/optimization_run.dart` with OptimizationRun model (timestamp, runType, durationMs, costEstimate, scheduleId, errorMessage)
- [ ] T015 [P] Create `lib/models/system_config.dart` with SystemConfig model (wallboxMaxPower, batteryCapacity, solarPanelCapacity, feedInRate)
- [ ] T016 [P] Create `lib/models/user_preferences.dart` with UserPreferences model (notificationsEnabled, morningReminderTime, afternoonReminderTime)
- [ ] T017 Run `flutter pub run build_runner build` to generate Isar schema files
- [ ] T018 Create `lib/services/database_service.dart` with IsarDB initialization, schema registration, and CRUD helpers
- [ ] T019 Write unit tests in `test/models/` for all 8 models validating serialization and CloudKit compatibility
- [ ] T020 Write unit tests in `test/services/database_service_test.dart` for CRUD operations

**Dependencies**: Phase 1

**Test Criteria**:
- All 8 models compile without errors
- `build_runner` generates Isar collection files successfully
- Database service initializes and performs CRUD on each model
- Unit tests achieve ≥80% coverage for models

---

## Phase 3: API Client Layer

**Goal**: Implement REST/GraphQL clients for Tibber, MySkoda, and Open-Meteo APIs.

**Depends On**: Phase 2 (T009-T020)

### Tasks
- [ ] T021 [P] Create `lib/services/api/tibber_api_client.dart` with GraphQL client using Dio, implement getCurrentDayPrices() query
- [ ] T022 [P] Create `lib/services/api/tibber_api_client.dart` method getTomorrowPrices() with availability check (returns null if not published)
- [ ] T023 Create `lib/services/api/tibber_api_client.dart` retry logic with exponential backoff (3 attempts, 1s/2s/4s delays)
- [ ] T024 [P] Create `lib/services/api/myskoda_api_client.dart` with OAuth 2.0 flow implementation using flutter_secure_storage
- [ ] T025 [P] Create `lib/services/api/myskoda_api_client.dart` method getVehicleStatus() returning SoC and charging state
- [ ] T026 [P] Create `lib/services/api/myskoda_api_client.dart` methods startCharging() and stopCharging() with command validation
- [ ] T027 Create `lib/services/api/myskoda_api_client.dart` credential refresh logic and token expiry handling
- [ ] T028 [P] Create `lib/services/api/openmeteo_api_client.dart` with getForecast(latitude, longitude) method
- [ ] T029 Create `lib/services/api/openmeteo_api_client.dart` solar radiation calculation from weather data (cloudCover → kWh estimation)
- [ ] T030 [P] Create `lib/utils/api_error_handler.dart` with user-friendly error messages for network failures, auth errors, rate limits
- [ ] T031 [P] Write unit tests in `test/services/api/` for all API clients using mock HTTP responses
- [ ] T032 Create integration tests in `integration_test/api_test.dart` testing real API calls with test credentials (skipped in CI)

**Dependencies**: Phase 2

**Test Criteria**:
- Tibber client fetches real price data with valid API token
- MySkoda client authenticates and retrieves vehicle status with test account
- Open-Meteo client returns valid forecast data
- Error handling gracefully degrades on network failures
- Unit tests mock all HTTP calls and achieve ≥75% coverage

---

## Phase 4: Background Services & Notifications

**Goal**: Implement iOS background fetch and local notification system.

**Depends On**: Phase 2 (T009-T020)

### Tasks
- [ ] T033 Create `lib/services/background_service.dart` with flutter_background_fetch initialization and task registration
- [ ] T034 Configure `ios/Runner/Info.plist` with BGTaskSchedulerPermittedIdentifiers for background modes
- [ ] T035 Create background task handler `_handlePriceFetch()` running hourly to check for Tibber price updates
- [ ] T036 Create background task handler `_handleAfternoonCheck()` running at 1:00 PM to detect tomorrow's prices
- [ ] T037 [P] Create `lib/services/notification_service.dart` with flutter_local_notifications initialization
- [ ] T038 [P] Create `lib/services/notification_service.dart` method showMorningReminder() scheduled for 6:30 AM
- [ ] T039 [P] Create `lib/services/notification_service.dart` method showAfternoonReminder() triggered when tomorrow prices available
- [ ] T040 [P] Create `lib/services/notification_service.dart` method showChargingAlert(device, action) for start/stop/error events
- [ ] T041 Create `lib/services/notification_service.dart` method showPriceAlert() when prices drop 30% below 24h average
- [ ] T042 Implement notification tap handling to open specific app screens (dashboard for reminders, schedule for charging alerts)
- [ ] T043 [P] Write unit tests in `test/services/background_service_test.dart` mocking background fetch callbacks
- [ ] T044 Write integration test in `integration_test/background_test.dart` simulating background task execution on physical iPhone

**Dependencies**: Phase 2

**Test Criteria**:
- Background fetch executes on iOS device with ≥1 execution within 24h
- Morning reminder notification appears at configured time
- Afternoon reminder triggers when tomorrow's prices detected
- Tapping notification opens correct app screen
- Unit tests verify task scheduling and notification payload generation

---

## Phase 5: US1 - Morning Solar Optimization

**User Story**: Morning Solar Optimization (Priority P1)  
**Goal**: User receives 6:30 AM notification, views current status, and runs daytime solar optimization.

**Depends On**: Phase 3 (T021-T032), Phase 4 (T033-T044)

**Independent Test**: User sets BEV goal, views price forecast, sees charging schedule without other features.

### Tasks
- [ ] T045 [US1] Create `lib/screens/dashboard_screen.dart` with AppBar, current BEV SoC widget, battery SoC widget, price chart, "Optimize Now" button
- [ ] T046 [US1] Create `lib/widgets/soc_display.dart` reusable widget showing device icon, percentage, last updated timestamp
- [ ] T047 [US1] Create `lib/widgets/price_chart.dart` using fl_chart to display 24h price graph with current hour highlighted
- [ ] T048 [US1] Create `lib/providers/device_state_provider.dart` using Riverpod to fetch and cache BEV/battery SoC from APIs
- [ ] T049 [US1] Create `lib/providers/price_provider.dart` using Riverpod to fetch Tibber prices and store in Isar
- [ ] T050 [US1] Create `lib/services/optimization_service.dart` with runMorningOptimization() method implementing greedy algorithm
- [ ] T051 [US1] Implement greedy solar optimization logic: prioritize daytime hours with high solar forecast, add note "Evening prices available after 1 PM"
- [ ] T052 [US1] Create `lib/services/optimization_service.dart` method saveSchedule(ChargeSchedule) persisting to Isar and scheduling background tasks
- [ ] T053 [US1] Add loading indicator and error handling to dashboard when optimization running or fails
- [ ] T054 [US1] Display optimization results: schedule preview with estimated solar kWh and grid kWh, cost breakdown
- [ ] T055 [US1] Add confirmation dialog after optimization showing "Schedule saved. Background tasks scheduled." message
- [ ] T056 [US1] Implement offline fallback: use cached prices if API unavailable, show staleness indicator
- [ ] T057 [US1] Write widget tests in `test/screens/dashboard_screen_test.dart` with mock providers
- [ ] T058 [US1] Write unit tests in `test/services/optimization_service_test.dart` validating greedy algorithm correctness
- [ ] T059 [US1] Write integration test in `integration_test/morning_optimization_test.dart` end-to-end from notification tap to schedule save

**Dependencies**: Phase 3, Phase 4

**Test Criteria**:
- User receives 6:30 AM notification on iOS device
- Dashboard displays real BEV SoC from MySkoda API
- Price chart shows current day Tibber prices
- Tapping "Optimize Now" completes in <10 seconds
- Schedule prioritizes solar hours (10 AM - 4 PM)
- Error message appears with "Retry" if network fails
- Cached data used when offline with freshness indicator

---

## Phase 6: US2 - Set BEV Charging Goal

**User Story**: Set BEV Charging Goal (Priority P1)  
**Goal**: User configures default minimum SoC (40%) and optionally sets higher targets with deadlines.

**Depends On**: Phase 2 (T009-T020)

**Independent Test**: User sets defaults and specific targets, schedule updates automatically.

### Tasks
- [ ] T060 [US2] Create `lib/screens/bev_settings_screen.dart` with current SoC display, default minimum selector (40%/50%/60%), "Add Specific Target" button
- [ ] T061 [US2] Create `lib/providers/bev_goal_provider.dart` using Riverpod to manage BEVGoal model with Isar persistence
- [ ] T062 [US2] Implement default minimum SoC selector with radio buttons, save to BEVGoal.defaultMinimumSoC on change
- [ ] T063 [US2] Display confirmation snackbar "BEV will charge to minimum 40% every night" when default updated
- [ ] T064 [US2] Create `lib/widgets/add_target_sheet.dart` bottom sheet with target SoC selector (80%/100%), DateTimePicker, "Save Target" button
- [ ] T065 [US2] Implement target validation in bev_goal_provider: check if achievable given charging rate (7.4 kW wallbox) and time available
- [ ] T066 [US2] Show warning dialog if target unachievable: "Cannot reach 80% by 8 AM. Earliest possible: 10:30 AM" with adjust/cancel options
- [ ] T067 [US2] Save specific target to BEVGoal.specificTargets list, trigger re-optimization via optimization_service
- [ ] T068 [US2] Display target list showing "80% by Nov 24, 8:00 AM" with swipe-to-delete gesture
- [ ] T069 [US2] Implement auto-removal background task: remove targets where deadline passed or SoC reached
- [ ] T070 [US2] Update optimization algorithm to prioritize active specific targets over default minimum
- [ ] T071 [US2] Write widget tests in `test/screens/bev_settings_screen_test.dart` with mock goal provider
- [ ] T072 [US2] Write unit tests in `test/providers/bev_goal_provider_test.dart` testing validation logic and auto-removal
- [ ] T073 [US2] Write integration test in `integration_test/bev_goal_test.dart` setting target, running optimization, verifying schedule update

**Dependencies**: Phase 2

**Test Criteria**:
- Default minimum selector saves to database immediately
- Adding 80% target by 8 AM triggers re-optimization
- Validation prevents impossible targets (e.g., 100% in 1 hour)
- Warning shows earliest achievable time when target invalid
- Expired targets auto-remove after deadline passes
- Optimization prioritizes specific target over default minimum

---

## Phase 7: US3 - Afternoon Price Optimization

**User Story**: Afternoon Price Optimization (Priority P1)  
**Goal**: User receives 1:30 PM notification when tomorrow's prices available, re-optimizes with actual data.

**Depends On**: Phase 3 (T021-T032), Phase 4 (T033-T044), Phase 5 (T045-T059)

**Independent Test**: Afternoon optimization updates schedule with tomorrow's price data.

### Tasks
- [ ] T074 [US3] Enhance background_service afternoon check: poll Tibber getTomorrowPrices() every 30 min from 1:00 PM until available or 6:00 PM
- [ ] T075 [US3] Trigger notification_service.showAfternoonReminder() when tomorrow prices detected
- [ ] T076 [US3] Update dashboard_screen to show price update indicator when tomorrow prices available but not yet optimized
- [ ] T077 [US3] Add "Re-optimize Now" button to dashboard when tomorrow prices available
- [ ] T078 [US3] Create `lib/services/optimization_service.dart` method runAfternoonOptimization() implementing overnight grid charging optimization
- [ ] T079 [US3] Implement greedy overnight optimization: find cheapest consecutive hours between 6 PM and 8 AM to meet BEV goal
- [ ] T080 [US3] Update existing schedule: preserve confirmed daytime solar sessions, add/update overnight grid sessions
- [ ] T081 [US3] Show fallback message if prices not available by 5 PM: "Tomorrow's prices not yet available. Using price estimates. Check back later."
- [ ] T082 [US3] Implement schedule comparison: calculate cost difference between morning plan and afternoon plan
- [ ] T083 [US3] Highlight changes in schedule view: color-code new sessions (green), modified sessions (orange), removed sessions (strikethrough)
- [ ] T084 [US3] Display savings banner if cost difference > €0.50: "Updated schedule saves €0.85 more"
- [ ] T085 [US3] Write unit tests in `test/services/optimization_service_test.dart` for afternoon optimization algorithm
- [ ] T086 [US3] Write integration test in `integration_test/afternoon_optimization_test.dart` simulating 1 PM notification to schedule update

**Dependencies**: Phase 3, Phase 4, Phase 5

**Test Criteria**:
- Background task detects tomorrow prices within 30 min of publication
- Notification appears at 1:30 PM with "Tomorrow's prices available"
- Dashboard shows price update indicator before re-optimization
- Re-optimization completes in <10 seconds
- Schedule includes optimal overnight grid charging (cheapest hours)
- Cost comparison shows savings vs. morning plan
- Fallback message appears if prices delayed past 5 PM

---

## Phase 8: US4 - View Charging Schedule

**User Story**: View Charging Schedule (Priority P1)  
**Goal**: User views timeline of planned charging sessions with rationale and real-time progress.

**Depends On**: Phase 5 (T045-T059)

**Independent Test**: Schedule view displays all sessions with mock data, shows detail sheets.

### Tasks
- [ ] T087 [US4] Create `lib/screens/schedule_screen.dart` with timeline view showing all planned ChargeSchedule sessions
- [ ] T088 [US4] Create `lib/widgets/schedule_timeline.dart` custom painter drawing vertical timeline with session blocks
- [ ] T089 [US4] Create `lib/widgets/session_card.dart` displaying device icon, time range, power source badge, kWh estimate
- [ ] T090 [US4] Implement session tap handler opening `lib/widgets/session_detail_sheet.dart` bottom sheet
- [ ] T091 [US4] Design session detail sheet showing: device name, start/end time, power source, kWh, cost, rationale text
- [ ] T092 [US4] Add rationale explanations: "Low price period (€0.08/kWh)", "Solar surplus (4.2 kWh available)", "Meeting BEV goal deadline"
- [ ] T093 [US4] Implement "Starting soon" indicator (orange highlight) for sessions within 15 minutes
- [ ] T094 [US4] Create `lib/providers/charging_progress_provider.dart` polling device status every 30s when session active
- [ ] T095 [US4] Show real-time progress for active sessions: current SoC, kWh added, time remaining, cost so far
- [ ] T096 [US4] Display empty state message: "No charging needed - BEV goal already met" or "Set a BEV goal to see schedule"
- [ ] T097 [US4] Add color coding: solar sessions (green), grid low-price (blue), manual override (yellow), error (red)
- [ ] T098 [US4] Write widget tests in `test/screens/schedule_screen_test.dart` with mock schedule data
- [ ] T099 [US4] Write widget tests in `test/widgets/session_detail_sheet_test.dart` verifying all fields render correctly
- [ ] T100 [US4] Write integration test in `integration_test/schedule_view_test.dart` creating schedule, viewing timeline, tapping session

**Dependencies**: Phase 5

**Test Criteria**:
- Timeline displays all sessions in chronological order
- Session cards show device, time, source, kWh, cost
- Tapping session opens detail sheet with complete information
- "Starting soon" appears 15 min before session start
- Active session shows real-time progress (SoC updating)
- Empty state message appears when no sessions scheduled
- Color coding correctly identifies solar vs. grid sessions

---

## Phase 9: US5 - Quick Charge Override

**User Story**: Quick Charge Override (Priority P2)  
**Goal**: User initiates immediate BEV charging, overriding planned schedule.

**Depends On**: Phase 3 (T021-T032)

**Independent Test**: Quick charge button sends command, shows progress, allows stop.

### Tasks
- [ ] T101 [US5] Add "Quick Charge" FloatingActionButton to dashboard_screen
- [ ] T102 [US5] Create `lib/widgets/quick_charge_dialog.dart` confirmation dialog: "Start charging BEV now? This will override your schedule."
- [ ] T103 [US5] Create `lib/services/wallbox_control_service.dart` with startImmediateCharging() method calling MySkoda API
- [ ] T104 [US5] Implement loading indicator during API call with 5-second timeout
- [ ] T105 [US5] Show success snackbar: "BEV charging started" with "Stop Charging" action button
- [ ] T106 [US5] Update dashboard to show "Manual charging in progress" banner with current power draw, SoC, "Stop Charging" button
- [ ] T107 [US5] Implement stopCharging() method in wallbox_control_service calling MySkoda stop command
- [ ] T108 [US5] Handle errors: "Cannot reach wallbox. Check connection." with "Retry" button
- [ ] T109 [US5] Log manual charge session in DeviceSnapshot for analytics (mark as manual override)
- [ ] T110 [US5] Write unit tests in `test/services/wallbox_control_service_test.dart` mocking MySkoda API
- [ ] T111 [US5] Write integration test in `integration_test/quick_charge_test.dart` starting/stopping charge on real vehicle (requires test setup)

**Dependencies**: Phase 3

**Test Criteria**:
- Quick Charge button appears on dashboard
- Confirmation dialog prevents accidental taps
- Charging starts within 5 seconds of confirmation
- Dashboard shows manual charging banner with real-time data
- Stop Charging button terminates session immediately
- Error message with Retry appears if API call fails

---

## Phase 10: US6 - Price Alert Notification

**User Story**: Price Alert Notification (Priority P2)  
**Goal**: Background task detects prices 30% below average, sends notification.

**Depends On**: Phase 4 (T033-T044), Phase 5 (T045-T059)

**Independent Test**: Price monitoring runs in background, sends alert when threshold met.

### Tasks
- [ ] T112 [US6] Create background task `_handlePriceMonitoring()` running every hour
- [ ] T113 [US6] Implement 24h rolling average calculation in price_provider
- [ ] T114 [US6] Add threshold detection: trigger when current price < (average * 0.7)
- [ ] T115 [US6] Call notification_service.showPriceAlert() when threshold crossed
- [ ] T116 [US6] Customize notification: "Energy prices are very low right now. Optimize to take advantage?"
- [ ] T117 [US6] Handle notification tap: open dashboard with price chart, highlight current low period
- [ ] T118 [US6] Add visual indicator on price chart showing threshold line and current price below it
- [ ] T119 [US6] Modify "Optimize Now" to prioritize current window when triggered from price alert
- [ ] T120 [US6] Implement notification throttling: max 1 price alert per 6 hours to avoid spam
- [ ] T121 [US6] Write unit tests in `test/providers/price_provider_test.dart` for average calculation and threshold detection
- [ ] T122 [US6] Write integration test in `integration_test/price_alert_test.dart` simulating price drop, verifying notification

**Dependencies**: Phase 4, Phase 5

**Test Criteria**:
- Background task calculates 24h average correctly
- Notification appears when price drops 30% below average
- Tapping notification opens dashboard with highlighted low period
- Optimization prioritizes current low-price window
- No more than 1 alert per 6 hours (throttling works)

---

## Phase 11: US7 - View Cost Analytics

**User Story**: View Cost Analytics (Priority P3)  
**Goal**: User reviews historical costs, savings, and trends over time.

**Depends On**: Phase 2 (T009-T020), Phase 5 (T045-T059)

**Independent Test**: Analytics tab displays historical data with savings calculation.

### Tasks
- [ ] T123 [US7] Create `lib/screens/analytics_screen.dart` with tabs: Today, Week, Month
- [ ] T124 [US7] Create `lib/providers/analytics_provider.dart` querying DeviceSnapshot and ChargeSchedule from Isar
- [ ] T125 [US7] Display summary cards: today's cost, week total, month total, estimated savings
- [ ] T126 [US7] Create `lib/widgets/cost_chart.dart` using fl_chart bar chart showing daily costs
- [ ] T127 [US7] Implement color coding in chart: solar (green), grid (blue), feed-in credit (orange)
- [ ] T128 [US7] Add trend line overlay showing 7-day or 30-day moving average
- [ ] T129 [US7] Create `lib/widgets/savings_detail_sheet.dart` explaining calculation
- [ ] T130 [US7] Implement baseline calculation: "Without SMARGE: €X (charged at avg. price)"
- [ ] T131 [US7] Implement actual calculation: "With SMARGE: €Y (optimized)"
- [ ] T132 [US7] Display savings: "Savings: €Z (Z%)" with breakdown by solar vs. price optimization
- [ ] T133 [US7] Add export button: generate CSV with daily cost/savings data
- [ ] T134 [US7] Write widget tests in `test/screens/analytics_screen_test.dart` with mock analytics data
- [ ] T135 [US7] Write unit tests in `test/providers/analytics_provider_test.dart` validating calculations
- [ ] T136 [US7] Write integration test in `integration_test/analytics_test.dart` verifying end-to-end data flow

**Dependencies**: Phase 2, Phase 5

**Test Criteria**:
- Analytics tab displays today/week/month costs correctly
- Bar chart renders with solar/grid color coding
- Trend line shows 7-day moving average
- Savings calculation explains baseline vs. optimized cost
- CSV export contains accurate daily data
- Unit tests verify savings calculation matches manual verification

---

## Phase 12: Q.HOME API Investigation (Parallel Track)

**Goal**: Determine feasible integration approach for Q.HOME+ ESS battery/wallbox control.

**Depends On**: None (runs in parallel with Phases 1-11)

**Timeline**: 3 weeks

### Tasks
- [ ] T137 [P] [QHOME] Check Q.HOME+ ESS web interface for Modbus TCP or API settings (access inverter at local IP)
- [ ] T138 [P] [QHOME] Email Q CELLS support requesting official API documentation or developer program access
- [ ] T139 [P] [QHOME] Test Modbus TCP connection using client tool (e.g., modpoll) if Modbus discovered in web interface
- [ ] T140 [QHOME] Identify Modbus register addresses for battery SoC, charge/discharge power, mode control if connection successful
- [ ] T141 [QHOME] Research community forums (Home Assistant, Reddit r/solar) for Q.HOME integration methods
- [ ] T142 [QHOME] Document findings in `specs/main/contracts/qhome-api.md` with integration feasibility assessment
- [ ] T143 [QHOME] If API unavailable: Design manual mode UX in `specs/main/research.md` (user enters battery SoC manually)
- [ ] T144 [QHOME] If API available: Update data-model.md with battery control commands and implement in Phase 13
- [ ] T145 [QHOME] Create decision document: "Proceed with automated integration" or "Ship MVP with manual mode, add API in v1.1"

**Dependencies**: None (parallel investigation)

**Test Criteria**:
- Week 1: Web interface checked, support email sent
- Week 2: Modbus tested OR manual mode designed
- Week 3: Decision documented with implementation plan

---

## Phase 13: Polish & Cross-Cutting Concerns

**Goal**: Production readiness - error handling, logging, onboarding, settings.

**Depends On**: Phases 5-11 (all user stories complete)

### Tasks
- [ ] T146 Create `lib/screens/onboarding_screen.dart` with 3-step wizard: API credentials, default BEV goal, notification permissions
- [ ] T147 Create `lib/screens/settings_screen.dart` with sections: Account, Devices, Notifications, Data & Privacy
- [ ] T148 Implement API credential management: edit Tibber token, MySkoda login, Q.HOME config (if available)
- [ ] T149 Add notification settings: enable/disable, change morning/afternoon times, toggle price alerts
- [ ] T150 Implement data export: "Download all data (JSON)" button for privacy compliance
- [ ] T151 Implement data deletion: "Delete all local data" with confirmation dialog
- [ ] T152 [P] Create `lib/utils/logger.dart` with log levels (debug, info, warn, error), file persistence
- [ ] T153 [P] Add logging to all API calls, optimization runs, background tasks, errors
- [ ] T154 Create `lib/screens/debug_screen.dart` (dev mode only) showing logs, last API responses, background task history
- [ ] T155 Implement global error boundary catching unhandled exceptions and logging to file
- [ ] T156 Add retry mechanism to all API calls with user-visible retry count
- [ ] T157 Implement stale data indicators: show "Last updated: 2h ago" on dashboard widgets
- [ ] T158 Create `lib/utils/analytics.dart` with opt-in telemetry (usage stats, crash reports) using Firebase Analytics
- [ ] T159 Add app version, build number, last git commit hash to Settings screen footer
- [ ] T160 Write UI/UX tests in `integration_test/onboarding_test.dart` for first-time user flow
- [ ] T161 Write UI/UX tests in `integration_test/settings_test.dart` for all settings changes
- [ ] T162 Perform manual testing checklist (20 scenarios) on physical iPhone with real APIs

**Dependencies**: Phases 5-11

**Test Criteria**:
- Onboarding wizard completes successfully for new user
- All settings persist and apply correctly
- Logs capture errors and can be exported for debugging
- Stale data indicators appear after 1 hour without refresh
- Retry mechanism recovers from transient failures
- Manual testing passes ≥18/20 scenarios

---

## Phase 14: Testing & Quality Assurance

**Goal**: Achieve ≥70% unit test coverage, validate all acceptance scenarios.

**Depends On**: Phases 1-13 (all implementation complete)

### Tasks
- [ ] T163 Run `flutter test --coverage` and generate coverage report
- [ ] T164 Identify untested code paths and write additional unit tests to reach 70% threshold
- [ ] T165 [P] Write acceptance test in `integration_test/us1_morning_optimization_test.dart` for all US1 scenarios
- [ ] T166 [P] Write acceptance test in `integration_test/us2_bev_goal_test.dart` for all US2 scenarios
- [ ] T167 [P] Write acceptance test in `integration_test/us3_afternoon_optimization_test.dart` for all US3 scenarios
- [ ] T168 [P] Write acceptance test in `integration_test/us4_schedule_view_test.dart` for all US4 scenarios
- [ ] T169 [P] Write acceptance test in `integration_test/us5_quick_charge_test.dart` for all US5 scenarios
- [ ] T170 [P] Write acceptance test in `integration_test/us6_price_alert_test.dart` for all US6 scenarios
- [ ] T171 [P] Write acceptance test in `integration_test/us7_analytics_test.dart` for all US7 scenarios
- [ ] T172 Create manual testing checklist in `specs/main/testing-checklist.md` covering edge cases
- [ ] T173 Execute manual tests on physical iPhone with real APIs, document results
- [ ] T174 Fix all critical bugs found during testing (P0 severity)
- [ ] T175 Triage and defer non-critical bugs to backlog (P1/P2 severity)
- [ ] T176 Run performance profiling: measure app launch (<2s), dashboard load (<1s), optimization time (<10s)
- [ ] T177 Optimize performance bottlenecks if targets not met
- [ ] T178 Test battery drain over 24h period (target <1% from background tasks)
- [ ] T179 Test memory usage under load (target <150 MB active, <50 MB background)
- [ ] T180 Create bug report template in `docs/bug-report-template.md`

**Dependencies**: Phases 1-13

**Test Criteria**:
- Unit test coverage ≥70% across all modules
- All 7 user story acceptance tests pass on physical iPhone
- Manual testing checklist: ≥18/20 scenarios pass
- Performance targets met: launch <2s, dashboard <1s, optimization <10s
- Battery drain <1% per day
- Memory usage within limits

---

## Phase 15: MVP Release Preparation

**Goal**: Package iOS build, prepare App Store submission, create user documentation.

**Depends On**: Phase 14 (T163-T180)

### Tasks
- [ ] T181 Create app icon in `ios/Runner/Assets.xcassets/AppIcon.appiconset/` (1024x1024 + required sizes)
- [ ] T182 Create launch screen in `ios/Runner/Base.lproj/LaunchScreen.storyboard`
- [ ] T183 Configure `ios/Runner/Info.plist` with privacy descriptions: NSLocationWhenInUseUsageDescription, NSUserNotificationsUsageDescription
- [ ] T184 Update `pubspec.yaml` with final version number (1.0.0) and build number (1)
- [ ] T185 Run `flutter build ios --release` and resolve any build errors
- [ ] T186 Archive build in Xcode: Product > Archive
- [ ] T187 Create App Store Connect listing: app name, description, screenshots, keywords
- [ ] T188 Generate screenshots (6.5" iPhone, 5.5" iPhone) for App Store: dashboard, schedule, settings
- [ ] T189 Write App Store description highlighting: solar optimization, cost savings, iOS 17+ compatibility
- [ ] T190 Create privacy policy document at `docs/privacy-policy.md` (required for App Store)
- [ ] T191 Create user guide at `docs/user-guide.md` with setup instructions and FAQ
- [ ] T192 Upload build to App Store Connect via Xcode Organizer
- [ ] T193 Submit for App Review with notes: "Requires Tibber account (free), MySkoda account (BEV owners)"
- [ ] T194 Create GitHub release tag `v1.0.0-mvp` with release notes
- [ ] T195 Update README.md with installation instructions, feature list, screenshots
- [ ] T196 Monitor App Review status and respond to feedback within 24h
- [ ] T197 Celebrate MVP launch! 🎉

**Dependencies**: Phase 14

**Test Criteria**:
- iOS build succeeds without errors
- App launches on TestFlight without crashes
- App Store listing includes all required metadata
- Privacy policy addresses all data collection
- User guide covers setup and common issues
- App approved by App Review (may take 1-2 iterations)

---

## Dependency Graph

```
Phase 1 (Setup)
└─> Phase 2 (Models) ──┬─> Phase 3 (APIs) ───┬─> Phase 5 (US1) ──┬─> Phase 7 (US3) ──┬─> Phase 13 (Polish)
    └─────────────────┬─> Phase 4 (Background)┘  └─────────────────┤                  │
                      │                                            ├─> Phase 8 (US4)  │
                      └─> Phase 6 (US2) ──────────────────────────┤                   │
                                                                    ├─> Phase 9 (US5)  │
                                                                    ├─> Phase 10 (US6) │
                                                                    └─> Phase 11 (US7) ┘
                                                                           │
Phase 12 (Q.HOME Investigation - Parallel) ─────────────────────────────────┘
                                                                           │
Phase 13 (Polish) ──> Phase 14 (Testing) ──> Phase 15 (Release)
```

**Critical Path**: Phase 1 → 2 → 3 → 5 → 7 → 13 → 14 → 15 (Core MVP flow)

**Parallel Work Opportunities**:
- Phase 12 (Q.HOME) runs independently throughout
- Phase 6 (US2) can start after Phase 2
- Phases 8-11 (US4-US7) can run in parallel after Phase 5

**MVP Scope** (Must Complete):
- Phases 1-8 (Setup through US4)
- Phase 13 (Polish)
- Phase 14 (Testing)
- Phase 15 (Release)

**Post-MVP** (Can Defer):
- Phase 9 (US5 - Quick Charge): Nice to have, not blocking
- Phase 10 (US6 - Price Alerts): Opportunistic optimization
- Phase 11 (US7 - Analytics): Motivational, not operational

---

## Task Summary

**Total Tasks**: 197  
**Foundation**: 44 tasks (Phases 1-4)  
**User Stories**: 121 tasks (Phases 5-11)  
**Quality**: 18 tasks (Phase 14)  
**Release**: 17 tasks (Phase 15)  
**Investigation**: 9 tasks (Phase 12)

**Estimated Effort** (rough):
- Phase 1: 1 day
- Phase 2: 2 days
- Phase 3: 3 days
- Phase 4: 2 days
- Phase 5 (US1): 4 days
- Phase 6 (US2): 2 days
- Phase 7 (US3): 3 days
- Phase 8 (US4): 3 days
- Phase 9 (US5): 1 day
- Phase 10 (US6): 2 days
- Phase 11 (US7): 2 days
- Phase 12 (Q.HOME): 3 weeks (parallel)
- Phase 13: 3 days
- Phase 14: 4 days
- Phase 15: 2 days

**Total Sequential**: ~32 days (excluding Q.HOME parallel track)  
**With Parallelization**: ~25 days (US4-US7 parallel, Q.HOME parallel)

---

## Next Steps

1. **Review this task breakdown** with stakeholders
2. **Begin Phase 1** (Project Setup) - 8 tasks, ~1 day
3. **Start Phase 12** (Q.HOME Investigation) in parallel - decision needed by Week 3
4. **Checkpoint after Phase 4**: Validate foundation (models, APIs, background services) before user story implementation
5. **MVP Checkpoint after Phase 8**: Validate US1-US4 work independently before polish
6. **Final Review Phase 14**: Quality gate before release

**Ready to proceed with implementation!** 🚀
