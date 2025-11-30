# SMARGE System Specification

**Version**: 1.4  
**Date**: November 23, 2025  
**Status**: Draft  
**References**: [Constitution](constitution.md)  
**Changelog**:
- v1.0: Initial specification
- v1.1: Added CloudKit multi-user architecture (Phase 2), updated data models for CloudKit compatibility
- v1.2: Major API clarification - BEV SoC from MySkoda API (not wallbox), confirmed Open-Meteo free, updated BEV goal model (default minimum + optional targets)
- v1.3: Added Q.HOME Web UI reverse engineering as primary approach for battery/wallbox integration
- v1.4: Separated Q.HOME monitoring (cloud API) vs. control (local API or manual), realistic MVP expectations
- v1.5: Switched from Swift/iOS native to Flutter cross-platform (iOS 17+ minimum, targeting iOS 26.1)

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Overview](#2-system-overview)
3. [User Scenarios & Workflows](#3-user-scenarios--workflows)
4. [Functional Requirements](#4-functional-requirements)
5. [System Architecture](#5-system-architecture)
6. [Data Models](#6-data-models)
7. [API Specifications](#7-api-specifications)
8. [User Interface Design](#8-user-interface-design)
9. [Background Processing](#9-background-processing)
10. [Optimization Algorithm](#10-optimization-algorithm)
11. [Error Handling & Edge Cases](#11-error-handling--edge-cases)
12. [Security & Privacy](#12-security--privacy)
13. [Performance Requirements](#13-performance-requirements)
14. [Testing Strategy](#14-testing-strategy)

---

## 1. Introduction

### 1.1 Purpose
This specification defines the detailed requirements and design for SMARGE (SMARt chARGE orchestrator), a Flutter mobile application for iOS (minimum iOS 17+, targeting iOS 26.1) that optimizes household energy costs by intelligently scheduling BEV (Battery Electric Vehicle) and home battery charging based on dynamic electricity pricing and solar power availability.

### 1.2 Scope
**In Scope:**
- Flutter mobile application for iPhone (iOS 17+ minimum, targeting iOS 26.1)
- BEV charging optimization and control
- Home battery charging/discharging optimization
- Integration with Tibber (pricing), weather services (solar forecast), and Q.HOME devices
- Local data storage and processing
- Background task scheduling
- Push notifications

**Out of Scope (v1.0):**
- Multi-vehicle support
- Vehicle-to-Grid (V2G)
- Smart home device integration beyond charging
- Web platform

**Planned for Phase 2:**
- Multi-user household sharing via platform-specific sync (iOS: CloudKit, Android: Firebase)
- Android platform support

### 1.3 Target Users
- Primary: Household owner with solar power, home battery, and BEV
- Technical literacy: Moderate (comfortable with smartphone apps, basic energy concepts)
- Usage pattern: Twice-daily interaction via notifications (morning & afternoon), 1-2 minutes per session

### 1.4 System Context
**Environment:**
- Annual consumption: 17,000 kWh
- Solar: 13 kW (SW orientation, 45° tilt)
- Battery: 12 kWh capacity
- BEV: 55 kWh battery
- Tariff: Dynamic (Tibber), avg. 28¢/kWh, range 20¢-€1+
- Feed-in: 9¢/kWh

**Critical Constraint - Tibber Price Publication Schedule:**
- **Current day prices**: Available 24/7
- **Next-day prices**: Published around 1:00 PM (sometimes delayed until 5:00 PM)
- **Implication**: Two-phase optimization required:
  - **Morning (6:30 AM)**: Optimize daytime solar usage with current prices only
  - **Afternoon (1:30 PM)**: Re-optimize overnight charging once next-day prices available

---

## 2. System Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────┐
│      iPhone (SMARGE Flutter App)            │
├─────────────────────────────────────────────┤
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ UI Layer │  │ Business  │  │Background │ │
│  │(Flutter) │→ │  Logic    │→ │ Tasks     │ │
│  └──────────┘  └──────────┘  └───────────┘ │
│                      ↓                       │
│              ┌──────────────┐               │
│              │ Hive/Isar    │               │
│              │ (Local DB)   │               │
│              └──────────────┘               │
│                      ↓ (Phase 2)            │
│              ┌──────────────┐               │
│              │ Platform Sync│               │
│              │ (Multi-User) │               │
│              └──────────────┘               │
└──────────────────┬──────────────────────────┘
                   │ HTTPS APIs
         ┌─────────┼──────────┬──────────┐
         ↓         ↓          ↓          ↓
    ┌────────┐ ┌───────┐ ┌────────┐ ┌────────┐
    │Tibber  │ │Weather│ │Q.HOME+ │ │Q.HOME  │
    │API     │ │API    │ │ESS     │ │EDRIVE  │
    └────────┘ └───────┘ └────────┘ └────────┘
```

### 2.2 Core Components

**Presentation Layer:**
- Flutter widgets for all user interfaces
- State management with Riverpod/Provider
- Charts for visualizations (fl_chart)

**Business Logic Layer:**
- Optimization engine (charging schedule calculation)
- Forecasting service (solar, consumption prediction)
- Device controller (API command execution)

**Data Layer:**
- Hive/Isar for persistent storage
- In-memory cache for real-time data
- Flutter Secure Storage for credentials

**Integration Layer:**
- API clients for external services
- Dio/http for networking with retry logic
- WebSocket support (future: real-time Tibber prices)

**Background Layer:**
- flutter_background_fetch / workmanager for periodic updates
- flutter_local_notifications for notification manager
- Background data sync

---

## 3. User Scenarios & Workflows

### 3.1 User Story 1 - Morning Solar Optimization (Priority: P1)

**Scenario:** User receives morning reminder and optimizes daytime solar power usage.

**Why this priority:** Core value proposition - daily cost optimization. Must work for MVP.

**Independent Test:** User can set BEV goal, view price forecast, and see charging schedule without other features.

**Acceptance Scenarios:**

1. **Given** app is installed and configured  
   **When** user receives 6:30 AM notification "Good morning! Optimize solar usage for today"  
   **Then** notification appears with app icon and tapping opens dashboard

2. **Given** user opens app from notification  
   **When** app loads  
   **Then** displays current BEV SoC, battery SoC, today's price graph, and "Optimize Now" button

3. **Given** user taps "Optimize Now"  
   **When** optimization completes (< 10 seconds)  
   **Then** shows schedule prioritizing solar usage, with note "Evening prices available after 1 PM"

4. **Given** optimization shows schedule  
   **When** user reviews and confirms  
   **Then** schedule is saved, background tasks are scheduled, and confirmation message appears

5. **Given** optimization failed due to network error  
   **When** error occurs  
   **Then** shows error message with "Retry" option and uses cached data if available

---

### 3.2 User Story 2 - Set BEV Charging Goal (Priority: P1)

**Scenario:** User configures default minimum charge (40%) and optionally sets higher target for specific time.

**Why this priority:** Essential for daily planning. BEV goal is primary constraint.

**Independent Test:** User can set default minimum and optional targets, schedule updates accordingly.

**Acceptance Scenarios:**

1. **Given** user opens "BEV Settings" screen  
   **When** screen loads  
   **Then** displays current SoC, default minimum selector (40%/50%/60%), and "Add Specific Target" button

2. **Given** user sets default minimum to 40%  
   **When** selection made  
   **Then** UI updates, shows "BEV will charge to minimum 40% every night" confirmation

3. **Given** user taps "Add Specific Target"  
   **When** button tapped  
   **Then** shows sheet with target SoC selector (80%/100%), time picker, and "Save Target" button

4. **Given** user selects 80% by tomorrow 8:00 AM  
   **When** target saved  
   **Then** shows in list: "80% by Nov 24, 8:00 AM", re-runs optimization, updates schedule

5. **Given** target time passes or is reached  
   **When** target completed or deadline passed  
   **Then** target auto-removed, system reverts to default minimum (40%)

6. **Given** goal cannot be met (insufficient time + charging capacity)  
   **When** validation fails  
   **Then** shows warning: "Cannot reach 80% by 8 AM. Earliest possible: 10:30 AM" with options to adjust or cancel

---

### 3.3 User Story 3 - Afternoon Price Optimization (Priority: P1)

**Scenario:** User receives afternoon reminder once tomorrow's prices are published by Tibber.

**Why this priority:** Essential for overnight/grid charging optimization. Tibber publishes next-day prices at 1-5 PM.

**Independent Test:** Afternoon optimization updates schedule with actual price data.

**Acceptance Scenarios:**

1. **Given** Tibber has published tomorrow's prices (after 1:00 PM)  
   **When** user receives 1:30 PM notification "Tomorrow's prices available - re-optimize?"  
   **Then** notification appears and tapping opens dashboard with price update indicator

2. **Given** user opens app from afternoon notification  
   **When** app loads  
   **Then** displays tomorrow's hourly prices in chart and "Re-optimize Now" button

3. **Given** user taps "Re-optimize Now"  
   **When** optimization runs with new price data  
   **Then** updates schedule to include optimal overnight/grid charging periods based on actual prices

4. **Given** Tibber prices delayed (not available by 5:00 PM)  
   **When** afternoon optimization attempted  
   **Then** shows message: "Tomorrow's prices not yet available. Using price estimates. Check back later."

5. **Given** re-optimization changes schedule significantly  
   **When** new schedule differs by > 2 hours from morning plan  
   **Then** highlights changes and shows cost comparison: "Updated schedule saves €0.85 more"

---

### 3.4 User Story 4 - View Charging Schedule (Priority: P1)

**Scenario:** User views upcoming charging sessions and understands rationale.

**Why this priority:** Transparency is core principle. User must trust the decisions.

**Independent Test:** Schedule view works with mock data, shows all relevant information.

**Acceptance Scenarios:**

1. **Given** optimization has created a schedule  
   **When** user navigates to "Schedule" tab  
   **Then** displays timeline view with all planned charging sessions

2. **Given** schedule shows charging session  
   **When** user taps session  
   **Then** shows detail sheet: device (BEV/Battery), start/end time, power source, kWh, cost, rationale ("Low price period" / "Solar surplus")

3. **Given** session is starting in < 15 minutes  
   **When** session upcoming  
   **Then** highlights session in orange with "Starting soon" indicator

4. **Given** session is currently active  
   **When** charging in progress  
   **Then** shows real-time progress: current SoC, kWh added, time remaining, cost so far

5. **Given** no sessions scheduled  
   **When** schedule is empty  
   **Then** shows message: "No charging needed - BEV goal already met" or "Set a BEV goal to see schedule"

---

### 3.5 User Story 5 - Quick Charge Override (Priority: P2)

**Scenario:** User needs immediate charging, overriding planned schedule.

**Why this priority:** Important for unexpected situations, but not needed for basic functionality.

**Independent Test:** Quick charge button works independently, sends immediate command.

**Acceptance Scenarios:**

1. **Given** user opens dashboard  
   **When** user taps "Quick Charge" button  
   **Then** shows confirmation sheet: "Start charging BEV now? This will override your schedule."

2. **Given** user confirms quick charge  
   **When** command sent to wallbox  
   **Then** shows loading indicator, then success: "BEV charging started" with option to stop

3. **Given** quick charge initiated  
   **When** charging starts  
   **Then** dashboard shows "Manual charging in progress" with current power, SoC, and "Stop Charging" button

4. **Given** quick charge fails (wallbox offline)  
   **When** API call fails  
   **Then** shows error: "Cannot reach wallbox. Check connection." with "Retry" option

---

### 3.6 User Story 6 - Price Alert Notification (Priority: P2)

**Scenario:** User receives notification when prices drop significantly below average.

**Why this priority:** Opportunistic optimization can increase savings, but not critical for core function.

**Independent Test:** Price monitoring runs in background, sends notification when threshold met.

**Acceptance Scenarios:**

1. **Given** background task detects prices 30% below 24h average  
   **When** threshold crossed  
   **Then** sends notification: "Energy prices are very low right now. Optimize to take advantage?"

2. **Given** user taps price alert notification  
   **When** app opens  
   **Then** opens price chart highlighting current low period with "Optimize Now" button

3. **Given** user optimizes from price alert  
   **When** optimization runs  
   **Then** prioritizes charging during current low-price window, updates schedule

---

### 3.7 User Story 7 - View Cost Analytics (Priority: P3)

**Scenario:** User reviews historical costs and savings over time.

**Why this priority:** Nice to have for motivation and validation, not essential for operation.

**Independent Test:** Analytics view displays historical data, calculates savings vs. baseline.

**Acceptance Scenarios:**

1. **Given** user navigates to "Analytics" tab  
   **When** tab loads  
   **Then** displays: today's cost, week total, month total, estimated savings vs. no optimization

2. **Given** user views monthly chart  
   **When** chart renders  
   **Then** shows bar chart: daily costs, color-coded by source (solar/grid), with trend line

3. **Given** user taps "Savings Calculation"  
   **When** detail sheet opens  
   **Then** explains: "Without SMARGE: €X (charged at avg. price). With SMARGE: €Y (optimized). Savings: €Z (Z%)"

---

## 4. Functional Requirements

### 4.1 Pre-Development Investigation

**REQ-INV-001**: MUST investigate Q.HOME+ ESS Modbus TCP support before MVP implementation  
**REQ-INV-002**: MUST test inverter web interface for communication settings  
**REQ-INV-003**: MUST contact Q CELLS support requesting official API documentation  
**REQ-INV-004**: SHOULD test Modbus connection with client tool if supported  
**REQ-INV-005**: MUST define fallback strategy (manual mode) if no API available  
**REQ-INV-006**: SHOULD identify Modbus register addresses for battery/BEV data  

**Investigation Timeline:**
- Week 1: Inverter web interface check + Q CELLS support email
- Week 2: Modbus testing (if available) or manual mode design
- Week 3: Decision on integration approach for MVP

### 4.2 Data Collection

**REQ-DC-001**: System SHALL fetch current day Tibber spot prices every hour  
**REQ-DC-002**: System SHALL check for next-day Tibber prices after 1:00 PM daily  
**REQ-DC-003**: System SHALL retry price fetch every 30 min if not available (until 6:00 PM)  
**REQ-DC-004**: System SHALL fetch weather forecast (Open-Meteo, free) for solar prediction when user opens app or runs optimization  
**REQ-DC-005**: System SHALL query BEV SoC from MySkoda API when user opens app or taps refresh  
**REQ-DC-006**: System SHALL query home battery SoC when user opens app or taps refresh (if API available)  
**REQ-DC-007**: System SHALL support manual SoC entry if automated queries unavailable  
**REQ-DC-008**: System SHALL cache all fetched data for offline viewing  
**REQ-DC-009**: System SHALL retry failed API calls up to 3 times with exponential backoff  
**REQ-DC-010**: System SHALL store MySkoda credentials securely in Flutter Secure Storage  

### 4.3 BEV Goal Management

**REQ-GOAL-001**: System SHALL maintain default minimum BEV SoC (default 40%, user-configurable)  
**REQ-GOAL-002**: System SHALL ensure BEV charges to minimum SoC every night  
**REQ-GOAL-003**: System SHALL allow user to set time-specific higher targets (e.g., \"80% by 8 AM tomorrow\")  
**REQ-GOAL-004**: System SHALL support multiple future targets with different deadlines  
**REQ-GOAL-005**: System SHALL auto-remove completed or expired targets  
**REQ-GOAL-006**: System SHALL validate target is achievable before acceptance  
**REQ-GOAL-007**: System SHALL warn user if target cannot be met with available time and charge rate  

### 4.4 Optimization

**REQ-OPT-001**: System SHALL calculate optimal charging schedule for available price window  
**REQ-OPT-002**: Morning optimization SHALL prioritize daytime solar power usage  
**REQ-OPT-003**: Afternoon optimization SHALL optimize overnight charging based on actual prices  
**REQ-OPT-004**: Optimization SHALL prioritize solar power over grid power when available  
**REQ-OPT-005**: Optimization SHALL ensure default minimum SoC is met every night  
**REQ-OPT-006**: Optimization SHALL ensure time-specific BEV goals are met before deadlines  
**REQ-OPT-007**: Optimization SHALL minimize total electricity cost  
**REQ-OPT-008**: System SHALL re-optimize automatically when next-day prices become available  
**REQ-OPT-009**: System SHALL re-optimize when weather forecast changes > 20%  
**REQ-OPT-010**: System SHALL re-optimize when user updates BEV goal or default minimum  
**REQ-OPT-011**: Optimization SHALL complete within 10 seconds  

### 4.5 Schedule Execution

**REQ-SCH-001**: System SHALL send start charging command 2 minutes before scheduled time (if API available)  
**REQ-SCH-002**: System SHALL send stop charging command at scheduled end time (if API available)  
**REQ-SCH-003**: System SHALL verify command execution via device status query (if API available)  
**REQ-SCH-004**: System SHALL retry failed commands up to 2 times  
**REQ-SCH-005**: System SHALL notify user if command fails after retries  
**REQ-SCH-006**: System SHALL notify user with schedule if manual control required (no API)  
**REQ-SCH-007**: System SHALL log all commands and responses for debugging  

### 4.6 Notifications

**REQ-NOT-001**: System SHALL send morning optimization reminder at 6:30 AM (solar planning)  
**REQ-NOT-002**: System SHALL send afternoon optimization reminder at 1:30 PM (price-based planning)  
**REQ-NOT-003**: System SHALL send notification at scheduled charge time if manual mode active  
**REQ-NOT-004**: System SHALL send charging start notification 15 minutes before session (automated mode)  
**REQ-NOT-005**: System SHALL send price alert when prices drop > 30% below average  
**REQ-NOT-006**: System SHALL send error notification when device becomes unreachable (if API mode)  
**REQ-NOT-007**: System SHALL send reminder if default minimum SoC not configured  
**REQ-NOT-008**: All notifications SHALL be actionable (deep-link to relevant screen)  

### 4.7 Background Processing

**REQ-BG-001**: System SHALL register background tasks using flutter_background_fetch or workmanager  
**REQ-BG-002**: System SHALL register background processing task for optimization  
**REQ-BG-003**: Background tasks SHALL execute at least every 30 minutes when app opened daily  
**REQ-BG-004**: System SHALL complete background work within platform limits (iOS: 25 sec, Android: varies)  
**REQ-BG-005**: System SHALL schedule next background task after each execution  

---

## 5. System Architecture

### 5.1 Layer Architecture

```
┌─────────────────────────────────────────────────┐
│            Presentation Layer                    │
│  ├─ Widgets (Flutter)                           │
│  ├─ State Management (Riverpod/Provider)        │
│  └─ Navigation & Routing                         │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│            Business Logic Layer                  │
│  ├─ OptimizationEngine                          │
│  ├─ ForecastingService                          │
│  ├─ DeviceController                            │
│  ├─ ScheduleGenerator                           │
│  └─ NotificationManager                          │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│            Data Layer                            │
│  ├─ Repository Pattern                          │
│  ├─ Hive/Isar Models                            │
│  ├─ Cache Manager                               │
│  └─ Flutter Secure Storage                      │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│            Integration Layer                     │
│  ├─ API Clients (Tibber, Weather, Q.HOME)      │
│  ├─ Dio/Http Manager                            │
│  └─ WebSocket Handler (future)                  │
└─────────────────────────────────────────────────┘
```

### 5.2 Module Structure

```
SMARGE/
├── lib/
│   ├── main.dart                       # App entry point
│   ├── app.dart                        # App configuration
│   │
│   ├── ui/
│   │   ├── dashboard/
│   │   │   ├── dashboard_screen.dart
│   │   │   ├── widgets/
│   │   │   │   ├── energy_flow_card.dart
│   │   │   │   ├── bev_status_card.dart
│   │   │   │   ├── battery_status_card.dart
│   │   │   │   └── todays_cost_card.dart
│   │   ├── bev/
│   │   │   ├── bev_goal_screen.dart
│   │   │   ├── charge_schedule_screen.dart
│   │   │   └── quick_charge_sheet.dart
│   │   ├── analytics/
│   │   │   ├── analytics_screen.dart
│   │   │   ├── price_chart.dart
│   │   │   ├── solar_forecast_chart.dart
│   │   │   └── cost_history.dart
│   │   └── settings/
│   │       ├── settings_screen.dart
│   │       ├── system_config_screen.dart
│   │       ├── api_credentials_screen.dart
│   │       └── notification_settings_screen.dart
│   │
│   ├── state/
│   │   ├── dashboard_provider.dart
│   │   ├── bev_provider.dart
│   │   ├── analytics_provider.dart
│   │   └── settings_provider.dart
│   │
│   ├── models/
│   │   ├── domain/
│   │   │   ├── bev_goal.dart
│   │   │   ├── charge_schedule.dart
│   │   │   ├── device_state.dart
│   │   │   ├── energy_price.dart
│   │   │   └── solar_forecast.dart
│   │   └── dtos/
│   │       ├── tibber_price_response.dart
│   │       ├── weather_response.dart
│   │       └── qhome_status_response.dart
│   │
│   ├── services/
│   │   ├── core/
│   │   │   ├── optimization_engine.dart
│   │   │   ├── forecasting_service.dart
│   │   │   ├── schedule_generator.dart
│   │   │   └── device_controller.dart
│   │   ├── api/
│   │   │   ├── tibber_api.dart
│   │   │   ├── weather_api.dart
│   │   │   ├── qhome_battery_api.dart
│   │   │   └── qhome_wallbox_api.dart
│   │   ├── background/
│   │   │   ├── background_task_manager.dart
│   │   │   ├── notification_manager.dart
│   │   │   └── data_sync_service.dart
│   │   └── storage/
│   │       ├── repository.dart
│   │       ├── cache_manager.dart
│   │       └── secure_storage_service.dart
│   │
│   └── utils/
│       ├── extensions/
│       │   ├── date_extensions.dart
│       │   ├── double_extensions.dart
│       │   └── string_extensions.dart
│       ├── helpers/
│       │   ├── logger.dart
│       │   └── formatters.dart
│       └── constants.dart
│
├── assets/
│   ├── images/
│   └── fonts/
│
├── test/
└── pubspec.yaml
```

---

## 6. Data Models

### 6.1 Platform Sync Design Considerations (Phase 2)

All data models are designed to be **sync-compatible** from the start:

**Design Principles:**
- Use standard Dart types (String, int, double, DateTime)
- Avoid complex nested relationships
- Use simple enums with string values
- Keep models flat where possible
- Plan for platform-specific sync (iOS: CloudKit, Android: Firebase)

**Sharing Strategy (Phase 2):**
- **Shared Data**: BEVGoal, ChargeSchedule, OptimizationRun
- **Private Data**: User preferences, notification settings
- **Conflict Resolution**: Last-write-wins for most entities, manual for BEVGoal conflicts
- **Permissions**: All household members read/write on shared data

### 6.2 Core Domain Models

```dart
// BEV Charging Configuration
@HiveType(typeId: 0)
class BEVConfig {
  @HiveField(0)
  final String id;
  
  @HiveField(1)
  double defaultMinimumSoC;  // Daily minimum (0.40 default)
  
  @HiveField(2)
  final DateTime createdAt;
  
  @HiveField(3)
  DateTime updatedAt;
  
  BEVConfig({
    required this.id,
    this.defaultMinimumSoC = 0.40,
    required this.createdAt,
    required this.updatedAt,
  });
}

// BEV Charging Goal (optional, time-specific override)
@HiveType(typeId: 1)
class BEVGoal {
  @HiveField(0)
  final String id;
  
  @HiveField(1)
  final double targetSoC;              // 0.0 - 1.0 (80% or 100%)
  
  @HiveField(2)
  final DateTime targetTime;           // Specific deadline (e.g., "tomorrow 8 AM")
  
  @HiveField(3)
  final DateTime createdAt;
  
  @HiveField(4)
  bool isCompleted;                    // Auto-set when target reached
  
  // Computed
  int get targetSoCPercentage => (targetSoC * 100).toInt();
  
  // Validation
  bool get isValid => targetSoC > 0.40 && targetTime.isAfter(DateTime.now());
  
  BEVGoal({
    required this.id,
    required this.targetSoC,
    required this.targetTime,
    required this.createdAt,
    this.isCompleted = false,
  });
}

// Charge Schedule Entry
@HiveType(typeId: 2)
class ChargeSchedule {
  @HiveField(0)
  final String id;
  
  @HiveField(1)
  final DeviceType device;             // DeviceType.bev or DeviceType.battery
  
  @HiveField(2)
  final DateTime startTime;
  
  @HiveField(3)
  final DateTime endTime;
  
  @HiveField(4)
  final double powerKW;
  
  @HiveField(5)
  final EnergySource source;           // EnergySource.solar or EnergySource.grid
  
  @HiveField(6)
  final double targetKWh;
  
  @HiveField(7)
  final double estimatedCost;
  
  @HiveField(8)
  final String rationale;              // "Low price period" / "Solar surplus"
  
  @HiveField(9)
  ScheduleStatus status;               // ScheduleStatus.pending, .active, etc.
  
  @HiveField(10)
  double? actualKWh;                   // Filled after execution
  
  @HiveField(11)
  double? actualCost;
  
  ChargeSchedule({
    required this.id,
    required this.device,
    required this.startTime,
    required this.endTime,
    required this.powerKW,
    required this.source,
    required this.targetKWh,
    required this.estimatedCost,
    required this.rationale,
    required this.status,
    this.actualKWh,
    this.actualCost,
  });
}

@HiveType(typeId: 10)
enum DeviceType {
  @HiveField(0)
  bev,
  
  @HiveField(1)
  battery,
}

@HiveType(typeId: 11)
enum EnergySource {
  @HiveField(0)
  solar,
  
  @HiveField(1)
  grid,
}

@HiveType(typeId: 12)
enum ScheduleStatus {
  @HiveField(0)
  pending,
  
  @HiveField(1)
  active,
  
  @HiveField(2)
  completed,
  
  @HiveField(3)
  failed,
  
  @HiveField(4)
  cancelled,
}

// Device State Snapshot
@HiveType(typeId: 3)
class DeviceSnapshot {
  @HiveField(0)
  final String id;
  
  @HiveField(1)
  final DateTime timestamp;
  
  @HiveField(2)
  final double? bevSoC;                // 0.0 - 1.0
  
  @HiveField(3)
  final double? bevSoCKWh;             // Absolute kWh (0-55)
  
  @HiveField(4)
  final bool bevIsCharging;
  
  @HiveField(5)
  final bool bevIsConnected;
  
  @HiveField(6)
  final double? batterySoCKWh;         // 0-12 kWh
  
  @HiveField(7)
  final double? batteryPowerKW;        // +/- (charging/discharging)
  
  @HiveField(8)
  final double? solarPowerKW;          // Current production
  
  @HiveField(9)
  final double? gridPowerKW;           // Current import/export
  
  DeviceSnapshot({
    required this.id,
    required this.timestamp,
    this.bevSoC,
    this.bevSoCKWh,
    required this.bevIsCharging,
    required this.bevIsConnected,
    this.batterySoCKWh,
    this.batteryPowerKW,
    this.solarPowerKW,
    this.gridPowerKW,
  });
}

// Energy Price
@HiveType(typeId: 4)
class EnergyPrice {
  @HiveField(0)
  final String id;
  
  @HiveField(1)
  final DateTime timestamp;            // Start of hour
  
  @HiveField(2)
  final double spotPrice;              // €/kWh (before taxes)
  
  @HiveField(3)
  final double totalPrice;             // €/kWh (including taxes & fees)
  
  @HiveField(4)
  final double feedInRate;             // €/kWh compensation for export
  
  @HiveField(5)
  final bool isForecast;               // true if future prediction
  
  EnergyPrice({
    required this.id,
    required this.timestamp,
    required this.spotPrice,
    required this.totalPrice,
    required this.feedInRate,
    required this.isForecast,
  });
}

// Solar Forecast
@HiveType(typeId: 5)
class SolarForecast {
  @HiveField(0)
  final String id;
  
  @HiveField(1)
  final DateTime timestamp;            // Hour start
  
  @HiveField(2)
  final double forecastKW;             // Predicted generation
  
  @HiveField(3)
  final double confidence;             // 0.0 - 1.0
  
  @HiveField(4)
  final String weatherCondition;       // "sunny", "cloudy", etc.
  
  SolarForecast({
    required this.id,
    required this.timestamp,
    required this.forecastKW,
    required this.confidence,
    required this.weatherCondition,
  });
}

// Optimization Run Record
@HiveType(typeId: 6)
class OptimizationRun {
  @HiveField(0)
  final String id;
  
  @HiveField(1)
  final DateTime timestamp;
  
  @HiveField(2)
  final double inputBEVSoC;
  
  @HiveField(3)
  final double inputBatterySoC;
  
  @HiveField(4)
  final double inputBEVGoal;
  
  @HiveField(5)
  final DateTime inputBEVDeadline;
  
  @HiveField(6)
  final List<ChargeSchedule> outputSchedules;
  
  @HiveField(7)
  final double estimatedTotalCost;
  
  @HiveField(8)
  final double estimatedSolarUsageKWh;
  
  @HiveField(9)
  final double estimatedGridUsageKWh;
  
  @HiveField(10)
  final double computationTimeMs;
  
  OptimizationRun({
    required this.id,
    required this.timestamp,
    required this.inputBEVSoC,
    required this.inputBatterySoC,
    required this.inputBEVGoal,
    required this.inputBEVDeadline,
    required this.outputSchedules,
    required this.estimatedTotalCost,
    required this.estimatedSolarUsageKWh,
    required this.estimatedGridUsageKWh,
    required this.computationTimeMs,
  });
}
```

### 6.3 Configuration Models

```dart
// System Configuration
class SystemConfig {
  final SolarConfig solar;
  final BatteryConfig battery;
  final BEVConfig bev;
  final HouseholdConfig household;
  
  SystemConfig({
    required this.solar,
    required this.battery,
    required this.bev,
    required this.household,
  });
  
  Map<String, dynamic> toJson() => {
    'solar': solar.toJson(),
    'battery': battery.toJson(),
    'bev': bev.toJson(),
    'household': household.toJson(),
  };
  
  factory SystemConfig.fromJson(Map<String, dynamic> json) => SystemConfig(
    solar: SolarConfig.fromJson(json['solar']),
    battery: BatteryConfig.fromJson(json['battery']),
    bev: BEVConfig.fromJson(json['bev']),
    household: HouseholdConfig.fromJson(json['household']),
  );
}

class SolarConfig {
  final double capacityKW;
  final String orientation;
  final double tiltDegrees;
  final Location location;
  
  SolarConfig({
    this.capacityKW = 13.0,
    this.orientation = "southwest",
    this.tiltDegrees = 45.0,
    required this.location,
  });
  
  Map<String, dynamic> toJson() => {
    'capacityKW': capacityKW,
    'orientation': orientation,
    'tiltDegrees': tiltDegrees,
    'location': location.toJson(),
  };
  
  factory SolarConfig.fromJson(Map<String, dynamic> json) => SolarConfig(
    capacityKW: json['capacityKW'],
    orientation: json['orientation'],
    tiltDegrees: json['tiltDegrees'],
    location: Location.fromJson(json['location']),
  );
}

class Location {
  final double latitude;
  final double longitude;
  
  Location({
    required this.latitude,
    required this.longitude,
  });
  
  Map<String, dynamic> toJson() => {
    'latitude': latitude,
    'longitude': longitude,
  };
  
  factory Location.fromJson(Map<String, dynamic> json) => Location(
    latitude: json['latitude'],
    longitude: json['longitude'],
  );
}

class BatteryConfig {
  final double capacityKWh;
  final double maxChargeRateKW;
  final double maxDischargeRateKW;
  final double minSoCKWh;        // Reserve capacity
  final double maxSoCKWh;        // Avoid 100% for longevity
  
  BatteryConfig({
    this.capacityKWh = 12.0,
    this.maxChargeRateKW = 5.0,
    this.maxDischargeRateKW = 5.0,
    this.minSoCKWh = 1.0,
    this.maxSoCKWh = 11.5,
  });
  
  Map<String, dynamic> toJson() => {
    'capacityKWh': capacityKWh,
    'maxChargeRateKW': maxChargeRateKW,
    'maxDischargeRateKW': maxDischargeRateKW,
    'minSoCKWh': minSoCKWh,
    'maxSoCKWh': maxSoCKWh,
  };
  
  factory BatteryConfig.fromJson(Map<String, dynamic> json) => BatteryConfig(
    capacityKWh: json['capacityKWh'],
    maxChargeRateKW: json['maxChargeRateKW'],
    maxDischargeRateKW: json['maxDischargeRateKW'],
    minSoCKWh: json['minSoCKWh'],
    maxSoCKWh: json['maxSoCKWh'],
  );
}

class BEVConfig {
  final double capacityKWh;
  final double maxChargeRateKW;
  final double minSoCKWh;        // Emergency reserve
  
  BEVConfig({
    this.capacityKWh = 55.0,
    this.maxChargeRateKW = 11.0,
    this.minSoCKWh = 5.0,
  });
  
  Map<String, dynamic> toJson() => {
    'capacityKWh': capacityKWh,
    'maxChargeRateKW': maxChargeRateKW,
    'minSoCKWh': minSoCKWh,
  };
  
  factory BEVConfig.fromJson(Map<String, dynamic> json) => BEVConfig(
    capacityKWh: json['capacityKWh'],
    maxChargeRateKW: json['maxChargeRateKW'],
    minSoCKWh: json['minSoCKWh'],
  );
}

class HouseholdConfig {
  final double annualConsumptionKWh;
  
  HouseholdConfig({
    this.annualConsumptionKWh = 17000.0,
  });
  
  double get averageHourlyConsumptionKW => annualConsumptionKWh / 8760.0;  // ~1.94 kW
  
  Map<String, dynamic> toJson() => {
    'annualConsumptionKWh': annualConsumptionKWh,
  };
  
  factory HouseholdConfig.fromJson(Map<String, dynamic> json) => HouseholdConfig(
    annualConsumptionKWh: json['annualConsumptionKWh'],
  );
}
```

---

## 7. API Specifications

### 7.1 Tibber API

**Base URL:** `https://api.tibber.com/v1-beta/gql`  
**Auth:** Bearer token in header  
**Protocol:** GraphQL over HTTPS

**Query: Get Current Price**
```graphql
query {
  viewer {
    homes {
      currentSubscription {
        priceInfo {
          current {
            total
            energy
            tax
            startsAt
          }
        }
      }
    }
  }
}
```

**Query: Get Price Forecast (24-48h)**
```graphql
query {
  viewer {
    homes {
      currentSubscription {
        priceInfo {
          today {
            total
            startsAt
          }
          tomorrow {
            total
            startsAt
          }
        }
      }
    }
  }
}
```

**Dart Implementation:**
```dart
import 'package:dio/dio.dart';

class TibberAPI {
  final String baseURL = "https://api.tibber.com/v1-beta/gql";
  final String apiKey;
  final Dio _dio;
  
  TibberAPI({required this.apiKey}) : _dio = Dio() {
    _dio.options.headers['Authorization'] = 'Bearer $apiKey';
    _dio.options.headers['Content-Type'] = 'application/json';
  }
  
  Future<List<EnergyPrice>> fetchPriceForecast() async {
    const query = """
    query {
      viewer {
        homes {
          currentSubscription {
            priceInfo {
              today { total startsAt }
              tomorrow { total startsAt }
            }
          }
        }
      }
    }
    """;
    
    try {
      final response = await _dio.post(
        baseURL,
        data: {'query': query},
      );
      
      final tibberResponse = TibberPriceResponse.fromJson(response.data);
      return tibberResponse.toPrices();
    } catch (e) {
      throw Exception('Failed to fetch price forecast: $e');
    }
  }
}
```

### 7.2 Weather API (Open-Meteo)

**Base URL:** `https://api.open-meteo.com/v1/forecast`  
**Auth:** None required  
**Cost:** **FREE** - No API key, no registration, no usage limits for non-commercial use  
**Protocol:** REST/JSON  
**License:** [CC BY 4.0](https://open-meteo.com/en/license) - Free for personal/research use

**Why Open-Meteo:**
- Zero cost (no subscription fees)
- No API key management
- High-quality solar radiation forecasts
- Reliable uptime (>99%)
- No usage limits for personal projects

**Endpoint: Solar Radiation Forecast**
```
GET /v1/forecast
  ?latitude=<lat>
  &longitude=<lon>
  &hourly=shortwave_radiation,cloudcover
  &forecast_days=2
```

**Response:**
```json
{
  "hourly": {
    "time": ["2025-11-23T00:00", "2025-11-23T01:00", ...],
    "shortwave_radiation": [0, 0, 45, 180, 350, ...],
    "cloudcover": [80, 75, 60, 40, 20, ...]
  }
}
```

**Dart Implementation:**
```dart
import 'package:dio/dio.dart';

class WeatherAPI {
  final Dio _dio = Dio();
  
  Future<List<SolarForecast>> fetchSolarForecast({
    required double lat,
    required double lon,
  }) async {
    final url = 'https://api.open-meteo.com/v1/forecast'
        '?latitude=$lat&longitude=$lon'
        '&hourly=shortwave_radiation,cloudcover'
        '&forecast_days=2';
    
    try {
      final response = await _dio.get(url);
      final weatherResponse = WeatherResponse.fromJson(response.data);
      return weatherResponse.toSolarForecasts(systemCapacity: 13.0);
    } catch (e) {
      throw Exception('Failed to fetch solar forecast: $e');
    }
  }
}
```

### 7.3 MySkoda API (BEV State of Charge)

**Integration Status:** ✅ **AVAILABLE** - Official MySkoda API (reverse-engineered but stable)

**Why MySkoda:**
- Q.HOME wallbox does NOT report BEV SoC to inverter
- Vehicle reports SoC directly to Skoda servers via cellular
- MySkoda app provides real-time battery data
- Active open-source library: [`myskoda`](https://github.com/skodaconnect/myskoda)

**Library:** Python `myskoda` (can be referenced for Swift implementation)

**Authentication:**
- Email + password (same as MySkoda app)
- OAuth2 flow (tokens managed by library)
- Session persistence

**Available Data for Enyaq:**
```python
# From myskoda library (Python reference)
charging_status = vehicle.charging
# Returns:
{
  "battery_soc_percent": 65,      # State of charge %
  "charging_state": "CONNECT_CABLE",  # or "CHARGING", "READY_FOR_CHARGING"
  "charging_power_kw": 11.0,      # Current charge rate
  "charging_rate_km_per_hour": 45,  # Estimated range added/hour
  "time_to_finish_min": 120,      # Minutes until full (if charging)
  "target_soc_percent": 80        # User-set target in MySkoda app
}
```

**Control Operations (if S-PIN provided):**
- Start charging: `vehicle.start_charging()`
- Stop charging: `vehicle.stop_charging()`
- Set charge limit: `vehicle.set_charge_limit(target_percent=80)`

**Swift Implementation Approach:**

**Option 1: Use Python Library via Process** (Quick MVP)
```swift
class MySkodaClient {
    func getBEVSoC() async throws -> Double {
        // Call Python script that uses myskoda library
        let process = Process()
        process.executableURL = URL(fileURLWithPath: "/usr/bin/python3")
        process.arguments = ["get_bev_soc.py"]
        // Parse output
        return socPercent
    }
}
```

**Option 2: Native Swift Implementation** (Better long-term)
```swift
import Foundation

class MySkodaAPI {
    private let baseURL = "https://mysmob.api.connect.skoda-auto.cz"
    private var accessToken: String?
    
    // Reverse-engineer from myskoda Python library
    // OAuth2 flow implementation
    func authenticate(email: String, password: String) async throws {
        // Implementation based on myskoda library OAuth flow
    }
    
    func getVehicleStatus(vin: String) async throws -> VehicleStatus {
        let url = URL(string: "\(baseURL)/api/v2/vehicle-status/\(vin)")!
        var request = URLRequest(url: url)
        request.setValue("Bearer \(accessToken!)", forHTTPHeaderField: "Authorization")
        
        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(VehicleStatus.self, from: data)
    }
}

struct VehicleStatus: Codable {
    let battery: BatteryStatus
    let charging: ChargingStatus?
}

struct BatteryStatus: Codable {
    let stateOfChargeInPercent: Int
    let remainingRangeInKm: Int
}

struct ChargingStatus: Codable {
    let state: String  // "CHARGING", "CONNECT_CABLE", "READY_FOR_CHARGING"
    let chargePowerInKw: Double?
    let remainingTimeToFullyChargedInMinutes: Int?
}
```

**Polling Strategy:**
- Update every 15 minutes when vehicle connected to wallbox
- Update every 30 minutes otherwise
- Use cached value if API unavailable

**Rate Limits:**
- MySkoda API rate limits unknown (based on unofficial usage)
- Recommend conservative polling (15-30 min intervals)
- Home Assistant integration reports no issues with 30-min polling

**Fallback:**
- If MySkoda API unavailable: Manual SoC entry by user
- User can check Skoda app and enter current %

---

### 7.4 Q.HOME+ ESS & EDRIVE Integration

**Purpose:** Home battery monitoring + wallbox charging control  
**NOT USED FOR:** BEV SoC (comes from MySkoda API instead)

**Integration Status:** 🔍 **REVERSE ENGINEERING REQUIRED**

**Two Distinct APIs Likely:**
1. **Cloud Web UI** (https://qhome-ess-g3.q-cells.eu) - Monitoring only, works remotely
2. **Local Inverter API** (http://[inverter-ip]) - Full control, requires home WiFi

**Why Two APIs:**
- ✅ **Cloud API** for remote monitoring (view data from anywhere)
- ✅ **Local API** for control (change settings, start/stop charging)
- ⚠️ Web UI may not expose control endpoints remotely (security/safety)
- ⚠️ Battery charging configuration likely only in local UI

**Important Clarification:**
- ❌ Q.HOME wallbox **does NOT know** BEV state of charge
- ❌ BEV SoC is **NOT reported** through wallbox to inverter  
- ✅ BEV SoC obtained from **MySkoda API** (vehicle reports via cellular)
- ✅ Q.HOME needed for: Battery SoC, solar production, wallbox status
- ✅ BEV charging control via **MySkoda API** (works remotely)

---

#### Option 1A: Q.HOME Cloud Web UI API (Monitoring - Works Remotely)

#### Option 1A: Q.HOME Cloud Web UI API (Monitoring - Works Remotely)

**URL:** https://qhome-ess-g3.q-cells.eu  
**Purpose:** Remote monitoring (view battery, solar, wallbox status from anywhere)  
**Control:** Likely **NOT available** remotely (monitoring only)

**Process:**
1. Login to web UI with browser DevTools open (Network tab)
2. Capture all API calls (authentication, data fetching)
3. Document endpoints, headers, request/response formats
4. Replicate in Swift using URLSession

**Expected Endpoints (to be verified during investigation):**
```
POST /api/auth/login              # Authentication
GET  /api/device/status           # Battery SoC, solar, power flows
GET  /api/battery/info            # Battery details
GET  /api/wallbox/status          # Wallbox state (connected, charging, kW)
GET  /api/realtime/data           # Current production/consumption
```

**Likely NOT available remotely:**
```
POST /api/wallbox/start           # ❌ Start charging (safety risk)
POST /api/wallbox/stop            # ❌ Stop charging
POST /api/battery/schedule        # ❌ Configure battery charging times
POST /api/battery/charge          # ❌ Force battery charging
```

**Swift Implementation:**
```swift
class QHomeCloudAPI {
    private let baseURL = "https://qhome-ess-g3.q-cells.eu"
    private var authToken: String?
    
    func login(email: String, password: String) async throws {
        // Replicate login flow captured from web UI
        let url = URL(string: \"\\(baseURL)/api/auth/login\")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        let credentials = ["username": email, "password": password]
        request.httpBody = try JSONEncoder().encode(credentials)
        
        let (data, _) = try await URLSession.shared.data(for: request)
        let response = try JSONDecoder().decode(AuthResponse.self, from: data)
        self.authToken = response.token
    }
    
    func getBatteryStatus() async throws -> BatteryStatus {
        guard let token = authToken else { throw APIError.notAuthenticated }
        
        let url = URL(string: \"\\(baseURL)/api/device/status\")!
        var request = URLRequest(url: url)
        request.setValue("Bearer \\(token)", forHTTPHeaderField: "Authorization")
        
        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(BatteryStatus.self, from: data)
    }
}

struct BatteryStatus: Codable {
    let socPercent: Double          // Battery state of charge
    let socKWh: Double              // Absolute kWh (0-12)
    let powerKW: Double             // Current charge/discharge rate (+/-)
    let solarProductionKW: Double   // Current solar generation
    let gridPowerKW: Double         // Grid import/export (+/-)
    let wallboxStatus: WallboxStatus?
}

struct WallboxStatus: Codable {
    let connected: Bool             // Vehicle plugged in
    let charging: Bool              // Currently charging
    let powerKW: Double             // Charge rate
    let sessionKWh: Double          // Energy delivered this session
}
```

**Advantages:**
- Works from anywhere (not just home WiFi)
- Standard REST API (HTTP + JSON)
- Easy to implement in Swift

**Limitations:**
- Monitoring only (no control)
- User must use Q.HOME mobile app for battery control
- BEV charging via MySkoda API instead (which is fine - works remotely)

---

#### Option 1B: Q.HOME Local Inverter API (Control - Requires Home WiFi)

**URL:** `http://[inverter-ip]` (local network only)  
**Purpose:** Full control capabilities (configure battery, start/stop wallbox)  
**Network:** Only works when iPhone on same WiFi as inverter

**Process:**
1. Find inverter IP address (router, Q.HOME app settings)
2. Access local web interface in browser
3. Open DevTools, navigate to control pages
4. Capture control commands (start charging, set schedule, etc.)
5. Test with curl on local network

**Expected Control Endpoints:**
```
POST http://192.168.1.XXX/api/wallbox/start     # Start wallbox charging
POST http://192.168.1.XXX/api/wallbox/stop      # Stop wallbox charging
POST http://192.168.1.XXX/api/battery/schedule  # Set battery charge times
POST http://192.168.1.XXX/api/battery/mode      # Auto/Manual/Force charge
```

**Swift Implementation:**
```swift
class QHomeLocalAPI {
    private var inverterIP: String  // User-configured
    
    func isOnHomeNetwork() -> Bool {
        // Detect if connected to home WiFi
        // Check SSID or try ping inverter IP
        return checkInverterReachable(ip: inverterIP)
    }
    
    func startWallboxCharging() async throws {
        guard isOnHomeNetwork() else {
            throw APIError.notOnHomeNetwork
        }
        
        let url = URL(string: "http://\\(inverterIP)/api/wallbox/start")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        // May require authentication or PIN
        
        let (_, response) = try await URLSession.shared.data(for: request)
        // Handle response
    }
}
```

**Advantages:**
- Full control capabilities
- Can automate battery and wallbox charging
- Direct communication (faster, no cloud dependency)

**Limitations:**
- Only works at home (same WiFi network)
- App must detect network and switch APIs
- Security concerns (direct device access)

---

#### Option 2: MySkoda API for BEV Control (Works Remotely)

**See Section 7.3 for full MySkoda integration details**

**BEV Charging Control:**
```swift
class MySkodaAPI {
    func startCharging() async throws {
        // Requires S-PIN for authorization
        try await vehicle.startCharging()
    }
    
    func stopCharging() async throws {
        try await vehicle.stopCharging()
    }
    
    func setChargeLimit(percent: Int) async throws {
        try await vehicle.setChargeLimit(targetPercent: percent)
    }
}
```

**Advantages:**
- ✅ Works from anywhere (cloud-based)
- ✅ Official Skoda backend (reliable)
- ✅ Proven integration (Home Assistant uses it)

**For MVP:**
- Use MySkoda for BEV control (automated, works remotely)
- Use Q.HOME Cloud API for battery monitoring (works remotely)
- Manual battery control via Q.HOME app (acceptable for MVP)

---

#### Option 3: Modbus TCP (FALLBACK)

---

#### Option 2: Modbus TCP (FALLBACK - Only if Web UI reverse engineering fails)

Most solar inverters support Modbus TCP protocol for local network monitoring.

**Pre-Implementation Tasks:**
1. Access inverter web interface (`http://[inverter-ip]`)
2. Check for Modbus TCP settings in communication/network section
3. Test connection using Modbus client tool (QModMaster, ModbusPal)
4. Identify register addresses for battery/BEV data

**If Modbus is supported:**

**Connection:**
- **Protocol:** Modbus TCP (standard port 502)
- **Auth:** May require inverter password/PIN
- **Network:** Local network only (same WiFi as inverter)

**Swift Implementation (using SwiftModbus or similar):**
```swift
import SwiftModbus

class QHomeModbusClient {
    let client: ModbusTCPClient
    let inverterIP = "192.168.1.XXX"  // User-configured
    
    func readBatteryStatus() async throws -> BatteryStatus {
        // Example register addresses (MUST BE VERIFIED)
        let socRegister: UInt16 = 1000    // Battery SoC %
        let powerRegister: UInt16 = 1001  // Battery power kW
        
        let socValue = try await client.readHoldingRegisters(
            address: socRegister, 
            count: 1
        )
        let powerValue = try await client.readHoldingRegisters(
            address: powerRegister,
            count: 1
        )
        
        return BatteryStatus(
            socPercent: Double(socValue[0]),
            powerKW: Double(powerValue[0]) / 10.0  // Scale factor
        )
    }
    
    // Control commands may not be available or require authentication
    func startCharging(rateKW: Double) async throws {
        // May require write access - MUST VERIFY SAFETY
        // Could potentially damage equipment if incorrect
        throw ModbusError.writeNotSupported
    }
}
```

**Modbus Limitations:**
- Read-only access likely (monitoring only)
- Write/control commands may be restricted or unsafe
- Register addresses not publicly documented (requires discovery)
- No official support - reverse engineering required

---

#### Option 3: Official API Request (LOW PRIORITY - Unlikely to succeed)

**Action Items:**
1. Email Q CELLS support: `sales@q-cells.com` or developer portal
2. Request:
   - Developer API documentation
   - Partner program for third-party integration
   - Technical specifications for programmatic access
3. Reference official Q.HOME app as proof of capability

**Timeline:** Likely 2-4 weeks for response, **unlikely to be approved**  
**Recommendation:** Don't wait for this - proceed with web UI reverse engineering

---

#### Option 4: Manual Mode (LAST RESORT - Only if all else fails)

If no API available for MVP:

**App provides:**
- Optimized charging schedule with times and power levels
- Push notifications at schedule times
- Manual data entry for battery/BEV SoC

**User manually:**
- Starts/stops charging via official Q.HOME app
- Enters current SoC values when prompted

**Implementation:**
```swift
struct ManualDeviceStatus {
    var bevSoC: Double      // User enters from official app
    var batterySoC: Double  // User enters from official app
    var lastUpdated: Date
    var isManualMode: Bool = true
}

// App shows:
// "⚠️ Manual Mode: Please check Q.HOME app and enter current values"
// "BEV SoC: [Text Field] %" 
// "Battery SoC: [Text Field] %"
// "Last updated: 5 minutes ago"
  "duration_minutes": 120
}
```

### 7.4 Q.HOME+ ESS & EDRIVE Integration

**Purpose:** Home battery monitoring + wallbox charging control  
**NOT USED FOR:** BEV SoC (comes from MySkoda API instead)

**CRITICAL RESEARCH FINDING:** Q.HOME devices do not provide publicly documented APIs.

**Official Q.HOME App:** Uses proprietary undocumented protocols ([App Store](https://apps.apple.com/de/app/q-home/id1491103343))

**Important Clarification:**
- ❌ Q.HOME wallbox **does NOT know** BEV state of charge
- ❌ BEV SoC is **NOT reported** through wallbox to inverter  
- ✅ BEV SoC obtained from **MySkoda API** (vehicle reports via cellular)
- ✅ Q.HOME needed for: Battery SoC, solar production, wallbox control

**Integration Status:** ⚠️ **UNCERTAIN - Requires Investigation**
**Data Available (if Modbus/API accessible):**
- BEV connection status (plugged in / not connected)  
- Charging state (charging / idle)
- Current charge rate (kW)
- Session energy delivered (kWh)
- ❌ BEV SoC % is **NOT available** (use MySkoda API)

**Expected Modbus Registers (MUST VERIFY):**
```swift
// Conceptual - actual addresses unknown
let wallboxRegisters = [
    "vehicle_connected": 2000,    // Bool
    "charging_active": 2002,      // Bool  
    "charge_rate_kw": 2003,       // Float * 10
    "session_kwh": 2004           // Float * 100
]
```

**Control Commands:**
- Start/stop charging (if write access available)
- Set charge rate limit (if supported)
- **Safety Warning:** Untested write commands could damage equipment or vehicle

**Fallback:**
- Wallbox control via official Q.HOME app
- User starts/stops charging manually when notified by SMARGE

---

## 8. User Interface Design

### 8.1 Navigation Structure

```
TabView
├─ Dashboard (Home icon)
├─ BEV (Car icon)
├─ Schedule (Calendar icon)
├─ Analytics (Chart icon)
└─ Settings (Gear icon)
```

### 8.2 Dashboard View

**Layout:**
```
┌─────────────────────────────────────┐
│ ☀️ SMARGE          🔔 (notifications)│
├─────────────────────────────────────┤
│                                      │
│  ┌────────────────────────────────┐ │
│  │   Energy Flow Visualization    │ │
│  │   Solar → Battery → House       │ │
│  │         ↘ BEV                   │ │
│  │   Current: 3.2 kW solar         │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌──────────────┐  ┌──────────────┐ │
│  │ BEV: 65%     │  │Battery: 71%  │ │
│  │ 35.8 kWh     │  │ 8.5 kWh      │ │
│  │ Next: 80% @  │  │ Charging     │ │
│  │ 7:00 AM      │  │ 2.3 kW       │ │
│  └──────────────┘  └──────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Today's Cost: €4.20            │ │
│  │ Savings: €1.80 (30%)           │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │    [Optimize Now]              │ │
│  └────────────────────────────────┘ │
│                                      │
│  Next Charging Session:             │
│  ⚡ BEV at 11:00 PM (Solar surplus) │
│                                      │
└─────────────────────────────────────┘
```

**Key Components:**
- **Energy Flow Diagram**: Animated Sankey-style visualization showing real-time power flow
- **Status Cards**: Glanceable current state of BEV and battery
- **Cost Summary**: Today's spend with comparison to baseline
- **Primary Action**: Large "Optimize Now" button
- **Next Session Preview**: Upcoming charging with rationale

### 8.3 BEV Goal View

**Layout:**
```
┌─────────────────────────────────────┐
│ ← BEV Charging Goal                 │
├─────────────────────────────────────┤
│                                      │
│  Current Charge                      │
│  ━━━━━━━━━━━━━━━━━━━━ 65%          │
│  35.8 kWh                            │
│                                      │
│  Target Charge Level                 │
│  ┌────┬────┬────┐                   │
│  │40% │80% │100%│  ← Segmented      │
│  └────┴────┴────┘     picker        │
│                                      │
│  Ready By                            │
│  ┌────────────────────────────────┐ │
│  │ Tomorrow at 7:00 AM            │ │
│  └────────────────────────────────┘ │
│                                      │
│  ℹ️ Charging needed: 8.2 kWh        │
│     Time required: ~45 min           │
│                                      │
│  ┌────────────────────────────────┐ │
│  │  Estimated Cost: €1.85          │ │
│  │  (mostly solar surplus)          │ │
│  └────────────────────────────────┘ │
│                                      │
│       [Update Goal]                  │
│                                      │
└─────────────────────────────────────┘
```

### 8.4 Schedule View

**Timeline Layout:**
```
┌─────────────────────────────────────┐
│ ← Charging Schedule                  │
├─────────────────────────────────────┤
│                                      │
│  Today • Nov 23                      │
│                                      │
│  11:00 PM ━━━━━━━━ 12:30 AM         │
│  🔋 BEV (Solar surplus)              │
│  8.2 kWh • €0.74                     │
│  ▸ Tap for details                   │
│                                      │
│  Tomorrow • Nov 24                   │
│                                      │
│  2:00 AM ━━━━━━━━ 3:30 AM           │
│  🏠 Battery (Low price)              │
│  5.0 kWh • €1.10                     │
│                                      │
│  11:00 AM ━━━━━━━━ 1:00 PM          │
│  🏠 Battery (Solar surplus)          │
│  7.0 kWh • €0.63                     │
│                                      │
│  ─────────────────────────────       │
│  Total: 20.2 kWh • €2.47             │
│                                      │
└─────────────────────────────────────┘
```

**Session Detail Sheet:**
```
┌─────────────────────────────────────┐
│         BEV Charging Session         │
├─────────────────────────────────────┤
│                                      │
│  When: Tonight 11:00 PM - 12:30 AM   │
│  Duration: 1.5 hours                 │
│  Power: 5.5 kW                       │
│  Energy: 8.2 kWh                     │
│  Source: Solar surplus               │
│  Cost: €0.74                         │
│                                      │
│  Why this time?                      │
│  Solar production forecasted to      │
│  exceed household consumption by     │
│  ~6 kW. Using surplus to charge BEV  │
│  with minimal grid import.           │
│                                      │
│       [Cancel Session]               │
│                                      │
└─────────────────────────────────────┘
```

---

## 9. Background Processing

### 9.1 Background Task Types

**BGAppRefreshTask:**
- **Identifier:** `com.smarge.refresh`
- **Purpose:** Quick data updates (prices, device states)
- **Frequency:** Every 15-30 minutes (iOS decides)
- **Duration:** < 30 seconds
- **Tasks:**
  - Fetch latest prices
  - Query device states
  - Check for schedule changes needed
  - Update notifications

**BGProcessingTask:**
- **Identifier:** `com.smarge.optimize`
- **Purpose:** Run optimization and schedule updates
- **Frequency:** Daily or when triggered
- **Duration:** < 5 minutes
- **Tasks:**
  - Fetch all forecast data
  - Run optimization algorithm
  - Update schedule
  - Send device commands
  - Schedule notifications

### 9.2 Background Task Implementation

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:workmanager/workmanager.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    switch (task) {
      case 'com.smarge.refresh':
        await handleAppRefresh();
        break;
      case 'com.smarge.optimize':
        await handleOptimization();
        break;
      default:
        break;
    }
    return Future.value(true);
  });
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize WorkManager for background tasks
  await Workmanager().initialize(
    callbackDispatcher,
    isInDebugMode: false,
  );
  
  // Register background tasks
  await registerBackgroundTasks();
  
  runApp(const SmargeApp());
}

Future<void> registerBackgroundTasks() async {
  // Register periodic refresh task (every 15 minutes)
  await Workmanager().registerPeriodicTask(
    'com.smarge.refresh',
    'com.smarge.refresh',
    frequency: const Duration(minutes: 15),
    constraints: Constraints(
      networkType: NetworkType.connected,
    ),
  );
  
  // Register optimization task (daily)
  await Workmanager().registerPeriodicTask(
    'com.smarge.optimize',
    'com.smarge.optimize',
    frequency: const Duration(hours: 1),
    constraints: Constraints(
      networkType: NetworkType.connected,
    ),
  );
}

Future<void> handleAppRefresh() async {
  final refreshService = RefreshDataService();
  await refreshService.execute();
}

Future<void> handleOptimization() async {
  final optimizationService = OptimizationService();
  await optimizationService.execute();
}
```

### 9.3 Notification Scheduling

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/timezone.dart' as tz;

class NotificationManager {
  static final NotificationManager _instance = NotificationManager._internal();
  factory NotificationManager() => _instance;
  NotificationManager._internal();
  
  final FlutterLocalNotificationsPlugin _notifications = 
      FlutterLocalNotificationsPlugin();
  
  Future<void> initialize() async {
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: true,
      requestBadgePermission: true,
      requestSoundPermission: true,
    );
    
    const settings = InitializationSettings(
      android: androidSettings,
      iOS: iosSettings,
    );
    
    await _notifications.initialize(settings);
  }
  
  Future<void> scheduleMorningOptimizationReminder() async {
    const androidDetails = AndroidNotificationDetails(
      'optimization',
      'Optimization Reminders',
      channelDescription: 'Daily optimization reminders',
      importance: Importance.high,
      priority: Priority.high,
    );
    
    const iosDetails = DarwinNotificationDetails(
      presentAlert: true,
      presentBadge: true,
      presentSound: true,
    );
    
    const details = NotificationDetails(
      android: androidDetails,
      iOS: iosDetails,
    );
    
    // Schedule for 6:30 AM daily
    await _notifications.zonedSchedule(
      0,  // Notification ID
      'Good morning!',
      'Optimize solar usage for today',
      _nextInstanceOf(hour: 6, minute: 30),
      details,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation: 
          UILocalNotificationDateInterpretation.absoluteTime,
      matchDateTimeComponents: DateTimeComponents.time,
    );
  }
  
  Future<void> scheduleAfternoonOptimizationReminder() async {
    const androidDetails = AndroidNotificationDetails(
      'optimization',
      'Optimization Reminders',
      channelDescription: 'Daily optimization reminders',
      importance: Importance.high,
      priority: Priority.high,
    );
    
    const iosDetails = DarwinNotificationDetails(
      presentAlert: true,
      presentBadge: true,
      presentSound: true,
    );
    
    const details = NotificationDetails(
      android: androidDetails,
      iOS: iosDetails,
    );
    
    // Schedule for 1:30 PM daily
    await _notifications.zonedSchedule(
      1,  // Notification ID
      'Prices updated!',
      'Tomorrow\\'s electricity prices are available. Re-optimize?',
      _nextInstanceOf(hour: 13, minute: 30),
      details,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation: 
          UILocalNotificationDateInterpretation.absoluteTime,
      matchDateTimeComponents: DateTimeComponents.time,
    );
  }
  
  Future<void> scheduleChargeSessionReminder(ChargeSchedule session) async {
    final reminderTime = session.startTime.subtract(const Duration(minutes: 15));
    
    if (reminderTime.isBefore(DateTime.now())) return;
    
    const androidDetails = AndroidNotificationDetails(
      'charging',
      'Charging Sessions',
      channelDescription: 'Upcoming charging session notifications',
      importance: Importance.high,
      priority: Priority.high,
    );
    
    const iosDetails = DarwinNotificationDetails(
      presentAlert: true,
      presentBadge: true,
      presentSound: true,
    );
    
    const details = NotificationDetails(
      android: androidDetails,
      iOS: iosDetails,
    );
    
    await _notifications.zonedSchedule(
      session.id.hashCode,  // Use session ID as notification ID
      '${session.device.name} charging soon',
      'Charging starts in 15 minutes',
      tz.TZDateTime.from(reminderTime, tz.local),
      details,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation: 
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }
  
  tz.TZDateTime _nextInstanceOf({required int hour, required int minute}) {
    final now = tz.TZDateTime.now(tz.local);
    var scheduledDate = tz.TZDateTime(
      tz.local,
      now.year,
      now.month,
      now.day,
      hour,
      minute,
    );
    
    if (scheduledDate.isBefore(now)) {
      scheduledDate = scheduledDate.add(const Duration(days: 1));
    }
    
    return scheduledDate;
  }
}
```
        content.categoryIdentifier = "CHARGE_STARTING"
        content.userInfo = ["scheduleId": session.id.uuidString]
        
        let trigger = UNCalendarNotificationTrigger(
            dateMatching: Calendar.current.dateComponents(
                [.year, .month, .day, .hour, .minute],
                from: reminderTime
            ),
            repeats: false
        )
        
        let request = UNNotificationRequest(
            identifier: "charge-reminder-\(session.id)",
            content: content,
            trigger: trigger
        )
        
        UNUserNotificationCenter.current().add(request)
    }
}
```

---

## 10. Optimization Algorithm

### 10.1 Algorithm Overview

**Approach:** Greedy algorithm for MVP (Phase 1), evolving to linear programming in Phase 2.

**Inputs:**
- BEV goal (target SoC, deadline)
- Current device states (BEV SoC, battery SoC)
- Price forecast (48 hours)
- Solar forecast (48 hours)
- System configuration (capacities, limits)

**Output:**
- Charging schedule for BEV and battery

**Constraints:**
- BEV must reach target SoC by deadline
- Battery SoC within limits (1-11.5 kWh)
- Charging rates within device limits
- Cannot exceed grid connection limits

**Objective:**
Minimize: Total cost = Σ(Grid_Import × Price) - Σ(Solar_Export × Feed_In_Rate)

### 10.2 Greedy Algorithm (MVP)

```swift
class OptimizationEngine {
    func optimize(
        bevGoal: BEVGoal,
        currentBEVSoC: Double,
        currentBatterySoC: Double,
        prices: [EnergyPrice],
        solarForecast: [SolarForecast],
        config: SystemConfig
    ) -> [ChargeSchedule] {
        
        var schedules: [ChargeSchedule] = []
        
        // 1. Calculate BEV charging need
        let bevKWhNeeded = (bevGoal.targetSoC - currentBEVSoC) * config.bev.capacityKWh
        
        guard bevKWhNeeded > 0 else {
            // Already at target
            return []
        }
        
        // 2. Identify time slots until deadline
        let timeSlots = generateTimeSlots(until: bevGoal.targetTime)
        
        // 3. Identify solar surplus periods
        let solarSurplusSlots = identifySolarSurplus(
            forecast: solarForecast,
            consumption: config.household.averageHourlyConsumptionKW,
            timeSlots: timeSlots
        )
        
        // 4. Allocate BEV charging to solar surplus first
        var remainingBEVCharge = bevKWhNeeded
        
        for slot in solarSurplusSlots {
            guard remainingBEVCharge > 0 else { break }
            
            let chargeableKWh = min(
                remainingBEVCharge,
                slot.surplusKW * slot.durationHours,
                config.bev.maxChargeRateKW * slot.durationHours
            )
            
            if chargeableKWh > 0.5 { // Minimum 0.5 kWh per session
                schedules.append(ChargeSchedule(
                    device: .bev,
                    startTime: slot.startTime,
                    endTime: slot.startTime.addingTimeInterval(slot.durationHours * 3600),
                    powerKW: chargeableKWh / slot.durationHours,
                    source: .solar,
                    targetKWh: chargeableKWh,
                    estimatedCost: chargeableKWh * 0.09, // Opportunity cost
                    rationale: "Solar surplus available"
                ))
                
                remainingBEVCharge -= chargeableKWh
            }
        }
        
        // 5. Fill remaining BEV charge with cheapest grid periods
        if remainingBEVCharge > 0 {
            let cheapestSlots = timeSlots
                .sorted { prices[$0].totalPrice < prices[$1].totalPrice }
                .prefix(Int(ceil(remainingBEVCharge / config.bev.maxChargeRateKW)))
            
            for slot in cheapestSlots {
                guard remainingBEVCharge > 0 else { break }
                
                let chargeableKWh = min(
                    remainingBEVCharge,
                    config.bev.maxChargeRateKW * slot.durationHours
                )
                
                schedules.append(ChargeSchedule(
                    device: .bev,
                    startTime: slot.startTime,
                    endTime: slot.startTime.addingTimeInterval(slot.durationHours * 3600),
                    powerKW: config.bev.maxChargeRateKW,
                    source: .grid,
                    targetKWh: chargeableKWh,
                    estimatedCost: chargeableKWh * prices[slot].totalPrice,
                    rationale: "Low price period (€\(String(format: "%.2f", prices[slot].totalPrice))/kWh)"
                ))
                
                remainingBEVCharge -= chargeableKWh
            }
        }
        
        // 6. Optimize battery charging (opportunistic)
        schedules.append(contentsOf: optimizeBatteryCharging(
            solarForecast: solarForecast,
            prices: prices,
            timeSlots: timeSlots,
            config: config,
            currentSoC: currentBatterySoC,
            bevSchedules: schedules
        ))
        
        return schedules.sorted { $0.startTime < $1.startTime }
    }
    
    private func identifySolarSurplus(
        forecast: [SolarForecast],
        consumption: Double,
        timeSlots: [TimeSlot]
    ) -> [SurplusSlot] {
        
        return timeSlots.compactMap { slot in
            guard let solarKW = forecast.first(where: { $0.timestamp == slot.startTime })?.forecastKW else {
                return nil
            }
            
            let surplusKW = solarKW - consumption
            
            guard surplusKW > 1.0 else { return nil } // Minimum 1 kW surplus
            
            return SurplusSlot(
                startTime: slot.startTime,
                durationHours: slot.durationHours,
                surplusKW: surplusKW
            )
        }
    }
    
    private func optimizeBatteryCharging(
        solarForecast: [SolarForecast],
        prices: [EnergyPrice],
        timeSlots: [TimeSlot],
        config: SystemConfig,
        currentSoC: Double,
        bevSchedules: [ChargeSchedule]
    ) -> [ChargeSchedule] {
        
        // Simplified: charge battery during cheapest periods
        // when BEV is not charging and there's capacity
        
        var batterySchedules: [ChargeSchedule] = []
        
        // ... implementation details
        
        return batterySchedules
    }
}

struct TimeSlot {
    let startTime: Date
    let durationHours: Double
}

struct SurplusSlot {
    let startTime: Date
    let durationHours: Double
    let surplusKW: Double
}
```

### 10.3 Algorithm Validation

**Test Cases:**
1. **Sufficient solar**: BEV charged entirely from solar surplus
2. **Insufficient solar**: Mixed solar + cheapest grid periods
3. **Tight deadline**: Charge immediately regardless of price
4. **Already at goal**: Return empty schedule
5. **Impossible goal**: Return error with earliest possible time

---

## 11. Error Handling & Edge Cases

### 11.1 API Failures

**Scenario:** Tibber API unavailable

**Handling:**
1. Retry up to 3 times with exponential backoff (1s, 2s, 4s)
2. Fall back to cached price data (warn user data is stale)
3. If cache > 24h old, show error: "Cannot fetch prices. Optimization may be suboptimal."
4. Allow user to proceed with cached data or cancel

**Scenario:** Tibber next-day prices not yet available (before 1 PM or delayed)

**Handling:**
1. Morning optimization proceeds with solar-only strategy
2. Show notice: "Overnight price optimization will be available this afternoon"
3. After 1:00 PM, check every 30 minutes for price availability
4. Send notification immediately when prices become available
5. If prices still unavailable by 6:00 PM, use price estimates based on historical averages

**Scenario:** Weather API unavailable

**Handling:**
1. Use historical average solar production for time of day/season
2. Warn user: "Solar forecast unavailable. Using average estimates."
3. Apply conservative buffer (reduce estimated production by 20%)

**Scenario:** Q.HOME device unreachable

**Handling:**
1. Show notification: "Cannot reach [device]. Check connection."
2. Display last-known state with timestamp
3. Disable charging commands, show "Device offline" status
4. Retry connection every 5 minutes in background

### 11.2 Optimization Failures

**Scenario:** Cannot meet BEV goal before deadline

**Handling:**
1. Calculate earliest possible time to reach goal
2. Show alert: "Cannot reach 80% by 7:00 AM. Earliest possible: 8:30 AM"
3. Options: "Adjust goal to 60%" / "Accept delay to 8:30 AM" / "Cancel"
4. If user accepts delay, update goal and re-optimize

**Scenario:** Optimization timeout (> 10 seconds)

**Handling:**
1. Cancel optimization
2. Fall back to simple rule: charge BEV at cheapest periods until goal met
3. Log error for debugging
4. Show notice: "Using simplified schedule. Full optimization unavailable."

### 11.3 Command Execution Failures

**Scenario:** Charging command fails

**Handling:**
1. Retry command after 30 seconds
2. If second attempt fails, send notification: "Failed to start BEV charging. Check wallbox."
3. Mark schedule entry as "Failed"
4. Reschedule for next available slot

**Scenario:** BEV disconnected during scheduled charge

**Handling:**
1. Detect via status query (vehicle_connected = false)
2. Cancel scheduled session
3. Send notification: "BEV disconnected. Charging cancelled."
4. Re-optimize when vehicle reconnected

### 11.4 Data Integrity

**Scenario:** Corrupted schedule data

**Handling:**
1. Detect via SwiftData validation
2. Delete corrupted records
3. Force re-optimization
4. Log incident for debugging

**Scenario:** Clock/timezone changes

**Handling:**
1. Detect time jump > 1 hour
2. Cancel all pending schedules
3. Force re-optimization with new time reference
4. Reschedule notifications

---

## 12. Security & Privacy

### 12.1 Credential Storage

**API Tokens:**
- Store in iOS Keychain (kSecAttrAccessibleWhenUnlocked)
- Never log tokens
- Never include in error messages or crash reports

```swift
class KeychainService {
    func save(token: String, for service: String) throws {
        let data = token.data(using: .utf8)!
        
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "api_token",
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlocked
        ]
        
        SecItemDelete(query as CFDictionary) // Delete existing
        let status = SecItemAdd(query as CFDictionary, nil)
        
        guard status == errSecSuccess else {
            throw KeychainError.unableToSave
        }
    }
    
    func retrieve(for service: String) throws -> String {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "api_token",
            kSecReturnData as String: true
        ]
        
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        
        guard status == errSecSuccess,
              let data = result as? Data,
              let token = String(data: data, encoding: .utf8) else {
            throw KeychainError.notFound
        }
        
        return token
    }
}
```

### 12.2 Network Security

**HTTPS Only:**
- All API calls use HTTPS
- Certificate pinning for production (future enhancement)
- TLS 1.2+ required

**Request Validation:**
- Validate all API responses against expected schema
- Reject malformed data
- Sanitize user inputs before API calls

### 12.3 Data Privacy

**Local Storage:**
- All user data stored locally on device
- No cloud backup of sensitive data (exclude from iCloud)
- User can delete all data via Settings

**Analytics:**
- No analytics in v1.0
- Future: opt-in only, anonymized, user-controlled

**Logging:**
- Development: verbose logging
- Production: minimal logging, no PII
- Crash reports: scrub credentials and personal data

---

## 12.5 Multi-User & CloudKit (Phase 2)

### 12.5.1 Architecture Overview

**CloudKit Integration:**
- **Container**: Private container with shared database support
- **Sharing Model**: CKShare for household data via Apple Family Sharing
- **Database Types**:
  - **Private Database**: User-specific preferences, notification settings
  - **Shared Database**: BEVGoal, ChargeSchedule, DeviceSnapshot, OptimizationRun
  - **Public Database**: Not used

### 12.5.2 Sharing Flow

**Setup (One-time):**
1. First household member creates household in app
2. App creates CKShare record for household data
3. User shares via native share sheet (Messages, Mail, AirDrop)
4. Invited members accept share via system notification
5. Shared data automatically syncs to all household devices

**Access Control:**
- **Owner**: Can add/remove members, delete household
- **Participants**: Full read/write on shared data
- **No hierarchy**: Any participant can set BEV goals, modify schedules

### 12.5.3 Data Sync Strategy

**Real-time Sync:**
- CKSubscription for shared database changes
- Silent push notifications trigger local updates
- UI updates automatically via SwiftData/CloudKit integration

**Conflict Resolution:**
- **BEVGoal**: Last-write-wins, notify all users of change
- **ChargeSchedule**: Last-write-wins (optimization runs are frequent)
- **DeviceSnapshot**: Most recent timestamp wins (read-only snapshots)
- **User Preferences**: Private per user, no conflicts

**Offline Support:**
- Changes queue locally when offline
- Automatic sync when connectivity restored
- Conflict resolution on reconnection

### 12.5.4 Privacy in Multi-User

**Shared Data:**
- BEV goals, schedules, device states, optimization results

**Private Data:**
- Notification preferences (e.g., wife wants notifications, user doesn't)
- UI preferences (theme, units)
- Tibber/device credentials (shared via Keychain, not CloudKit)

**Permission Model:**
- All household members see same optimization data
- Individual notification settings per device
- No "admin" role - fully collaborative

### 12.5.5 Implementation Phases

**Phase 2.1: Foundation (Week 1-2)**
- Enable CloudKit capability in Xcode
- Set up CKContainer and database references
- Add CKShare creation/management
- Implement sharing UI (share sheet integration)

**Phase 2.2: Data Layer (Week 2-3)**
- Migrate SwiftData models to CloudKit-backed store
- Implement CKSubscription for remote changes
- Add conflict resolution logic
- Test sync with two devices

**Phase 2.3: Multi-User UX (Week 3-4)**
- Add household member list view
- Show "who set goal" attribution
- Real-time update indicators ("Wife just set BEV goal to 80%")
- Leave household option

**Phase 2.4: Testing & Polish (Week 4)**
- Test offline sync scenarios
- Test concurrent edits
- Performance testing with sync
- Edge case handling (member leaves, device offline for days)

### 12.5.6 CloudKit Requirements

**Entitlements:**
- `iCloud` (CloudKit)
- `com.apple.developer.icloud-container-identifiers`
- Background Modes: Remote notifications

**Apple Developer Account:**
- Requires paid Apple Developer Program membership ($99/year)
- CloudKit usage within free tier (generous limits for personal use)

**Testing:**
- CloudKit Dashboard for data inspection
- Development vs. Production containers
- TestFlight for multi-device testing

---

## 13. Performance Requirements

### 13.1 Response Times

| Operation | Target | Maximum |
|-----------|--------|---------|
| App launch | < 1s | 2s |
| Dashboard load | < 0.5s | 1s |
| Optimization calculation | < 5s | 10s |
| API call (single) | < 2s | 5s |
| Background task execution | < 20s | 25s |
| View transition | < 0.3s | 0.5s |

### 13.2 Resource Usage

**Memory:**
- Target: < 100 MB active
- Maximum: < 150 MB active
- Background: < 50 MB

**Battery:**
- Background tasks: < 1% battery per day
- Foreground usage: < 5% battery per 10 min session
- No significant drain when idle

**Network:**
- API calls: < 1 MB per optimization cycle
- Daily total: < 5 MB (typical), < 20 MB (maximum with forecasts)

### 13.3 Scalability

**Data Volume:**
- Support 1 year of historical data (365 days)
- ~35,000 price records
- ~35,000 device snapshots
- ~2,000 charge schedules
- Total DB size: < 50 MB

**Performance Testing:**
- Test with 1 year of data
- Verify optimization time < 10s with full dataset
- Ensure UI remains responsive during background sync

---

## 14. Testing Strategy

### 14.1 Unit Tests

**Coverage Target:** ≥70% for business logic

**Key Test Suites:**
- `OptimizationEngineTests`: Algorithm correctness
- `ForecastingServiceTests`: Solar/consumption predictions
- `ScheduleGeneratorTests`: Schedule creation logic
- `APIClientTests`: Request/response parsing (mocked)
- `DataModelTests`: SwiftData model validation

**Example:**
```swift
class OptimizationEngineTests: XCTestCase {
    func testBEVChargingWithSufficientSolar() {
        // Given
        let goal = BEVGoal(targetSoC: 0.8, targetTime: Date().addingTimeInterval(86400))
        let currentBEVSoC = 0.5
        let prices = MockData.cheapNightPrices
        let solar = MockData.sunnDayForecast
        
        // When
        let schedules = engine.optimize(
            bevGoal: goal,
            currentBEVSoC: currentBEVSoC,
            currentBatterySoC: 0.5,
            prices: prices,
            solarForecast: solar,
            config: testConfig
        )
        
        // Then
        XCTAssertGreaterThan(schedules.count, 0)
        
        let solarCharging = schedules.filter { $0.source == .solar }
        let totalSolarKWh = solarCharging.reduce(0.0) { $0 + $1.targetKWh }
        
        XCTAssertGreaterThan(totalSolarKWh, 10.0, "Should use significant solar")
    }
}
```

### 14.2 Integration Tests

**Test Scenarios:**
1. **End-to-End Optimization:**
   - Fetch real price data (test API)
   - Run optimization
   - Verify schedule created
   - Validate schedule feasibility

2. **Background Task Execution:**
   - Trigger BGAppRefreshTask
   - Verify data fetched
   - Confirm notifications scheduled

3. **Device Control Flow:**
   - Send charging command
   - Verify API call made
   - Confirm status updated

### 14.3 UI Tests

**XCUITest Suites:**
- **Onboarding:** First-time setup flow
- **Dashboard:** View loading, interactions
- **BEV Goal:** Setting and updating goals
- **Schedule:** Viewing and interacting with timeline
- **Settings:** Configuration changes

**Example:**
```swift
class DashboardUITests: XCTestCase {
    func testOptimizeButtonTriggersOptimization() {
        let app = XCUIApplication()
        app.launch()
        
        let optimizeButton = app.buttons["Optimize Now"]
        XCTAssertTrue(optimizeButton.exists)
        
        optimizeButton.tap()
        
        // Wait for completion (loading indicator disappears)
        let loadingIndicator = app.activityIndicators.firstMatch
        XCTAssertTrue(loadingIndicator.waitForExistence(timeout: 1))
        XCTAssertFalse(loadingIndicator.waitForNonExistence(timeout: 15))
        
        // Verify schedule appears
        let scheduleView = app.otherElements["ChargeScheduleList"]
        XCTAssertTrue(scheduleView.exists)
    }
}
```

### 14.4 Manual Testing Checklist

**Pre-Release Validation:**
- [ ] Install on clean device, complete onboarding
- [ ] Set BEV goal, verify schedule generated
- [ ] Receive and interact with all notification types
- [ ] Test with airplane mode (offline behavior)
- [ ] Test with each API failing independently
- [ ] Verify background tasks execute (check logs after 30 min)
- [ ] Test quick charge override
- [ ] Review analytics/cost tracking accuracy
- [ ] Verify all settings persist after app restart
- [ ] Test on iOS 16, 17, and 18

---

## 15. Appendices

### Appendix A: Glossary

- **BEV**: Battery Electric Vehicle
- **SoC**: State of Charge (battery percentage)
- **kWh**: Kilowatt-hour (unit of energy)
- **kW**: Kilowatt (unit of power)
- **Feed-in Rate**: Compensation for exporting solar power to grid
- **Spot Price**: Real-time electricity price on energy market
- **Greedy Algorithm**: Optimization approach that makes locally optimal choices

### Appendix B: References

- [Tibber API Documentation](https://developer.tibber.com)
- [Open-Meteo API](https://open-meteo.com)
- [Apple BGTaskScheduler](https://developer.apple.com/documentation/backgroundtasks)
- [SwiftData Documentation](https://developer.apple.com/documentation/swiftdata)

### Appendix C: Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-23 | Initial specification draft |

---

**Document Status:** DRAFT - Ready for review and planning phase

**Next Steps:**
1. Review and approve specification
2. Create implementation plan (`/speckit.plan`)
3. Break down into tasks (`/speckit.tasks`)
4. Begin Phase 1 (MVP) development
