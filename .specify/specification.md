# SMARGE System Specification

**Version**: 1.1  
**Date**: November 23, 2025  
**Status**: Draft  
**References**: [Constitution](constitution.md)  
**Changelog**:
- v1.0: Initial specification
- v1.1: Added CloudKit multi-user architecture (Phase 2), updated data models for CloudKit compatibility

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
This specification defines the detailed requirements and design for SMARGE (SMARt chARGE orchestrator), a native iOS application that optimizes household energy costs by intelligently scheduling BEV (Battery Electric Vehicle) and home battery charging based on dynamic electricity pricing and solar power availability.

### 1.2 Scope
**In Scope:**
- iOS native application for iPhone (iOS 18+, targeting iOS 26.1)
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
- Android or web versions

**Planned for Phase 2:**
- Multi-user household sharing via CloudKit and Apple Family Sharing

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
│         iPhone (SMARGE iOS App)              │
├─────────────────────────────────────────────┤
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ UI Layer │  │ Business  │  │Background │ │
│  │(SwiftUI) │→ │  Logic    │→ │ Tasks     │ │
│  └──────────┘  └──────────┘  └───────────┘ │
│                      ↓                       │
│              ┌──────────────┐               │
│              │ SwiftData    │               │
│              │ (Local DB)   │               │
│              └──────────────┘               │
│                      ↓ (Phase 2)            │
│              ┌──────────────┐               │
│              │  CloudKit    │               │
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
- SwiftUI views for all user interfaces
- View models managing UI state
- Charts for visualizations

**Business Logic Layer:**
- Optimization engine (charging schedule calculation)
- Forecasting service (solar, consumption prediction)
- Device controller (API command execution)

**Data Layer:**
- SwiftData for persistent storage
- In-memory cache for real-time data
- Keychain for credentials

**Integration Layer:**
- API clients for external services
- Network manager with retry logic
- WebSocket support (future: real-time Tibber prices)

**Background Layer:**
- BGTaskScheduler for periodic updates
- Notification manager
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
**REQ-DC-004**: System SHALL fetch weather forecast (Open-Meteo, free) for solar prediction every 6 hours  
**REQ-DC-005**: System SHALL query BEV SoC every 15 minutes when connected (if API available)  
**REQ-DC-006**: System SHALL query home battery SoC every 15 minutes (if API available)  
**REQ-DC-007**: System SHALL support manual SoC entry if automated queries unavailable  
**REQ-DC-008**: System SHALL cache all fetched data for offline viewing  
**REQ-DC-009**: System SHALL retry failed API calls up to 3 times with exponential backoff  

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

**REQ-BG-001**: System SHALL register BGAppRefreshTask for periodic data updates  
**REQ-BG-002**: System SHALL register BGProcessingTask for optimization  
**REQ-BG-003**: Background tasks SHALL execute at least every 30 minutes when app opened daily  
**REQ-BG-004**: System SHALL complete background work within 25 seconds (iOS limit buffer)  
**REQ-BG-005**: System SHALL schedule next background task after each execution  

---

## 5. System Architecture

### 5.1 Layer Architecture

```
┌─────────────────────────────────────────────────┐
│            Presentation Layer                    │
│  ├─ Views (SwiftUI)                             │
│  ├─ ViewModels (ObservableObject)               │
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
│  ├─ SwiftData Models                            │
│  ├─ Cache Manager                               │
│  └─ Keychain Service                            │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│            Integration Layer                     │
│  ├─ API Clients (Tibber, Weather, Q.HOME)      │
│  ├─ NetworkManager                              │
│  └─ WebSocket Handler (future)                  │
└─────────────────────────────────────────────────┘
```

### 5.2 Module Structure

```
SMARGE/
├── App/
│   ├── SmargeApp.swift                  # App entry point
│   └── AppDelegate.swift                # Background task registration
│
├── Views/
│   ├── Dashboard/
│   │   ├── DashboardView.swift
│   │   ├── EnergyFlowCard.swift
│   │   ├── BEVStatusCard.swift
│   │   ├── BatteryStatusCard.swift
│   │   └── TodaysCostCard.swift
│   ├── BEV/
│   │   ├── BEVGoalView.swift
│   │   ├── ChargeScheduleView.swift
│   │   └── QuickChargeSheet.swift
│   ├── Analytics/
│   │   ├── AnalyticsView.swift
│   │   ├── PriceChartView.swift
│   │   ├── SolarForecastView.swift
│   │   └── CostHistoryView.swift
│   └── Settings/
│       ├── SettingsView.swift
│       ├── SystemConfigView.swift
│       ├── APICredentialsView.swift
│       └── NotificationSettingsView.swift
│
├── ViewModels/
│   ├── DashboardViewModel.swift
│   ├── BEVViewModel.swift
│   ├── AnalyticsViewModel.swift
│   └── SettingsViewModel.swift
│
├── Models/
│   ├── Domain/
│   │   ├── BEVGoal.swift
│   │   ├── ChargeSchedule.swift
│   │   ├── DeviceState.swift
│   │   ├── EnergyPrice.swift
│   │   └── SolarForecast.swift
│   └── DTOs/                            # Data Transfer Objects
│       ├── TibberPriceResponse.swift
│       ├── WeatherResponse.swift
│       └── QHomeStatusResponse.swift
│
├── Services/
│   ├── Core/
│   │   ├── OptimizationEngine.swift
│   │   ├── ForecastingService.swift
│   │   ├── ScheduleGenerator.swift
│   │   └── DeviceController.swift
│   ├── APIs/
│   │   ├── TibberAPI.swift
│   │   ├── WeatherAPI.swift
│   │   ├── QHomeBatteryAPI.swift
│   │   └── QHomeWallboxAPI.swift
│   ├── Background/
│   │   ├── BackgroundTaskManager.swift
│   │   ├── NotificationManager.swift
│   │   └── DataSyncService.swift
│   └── Storage/
│       ├── Repository.swift
│       ├── CacheManager.swift
│       └── KeychainService.swift
│
├── Utilities/
│   ├── Extensions/
│   │   ├── Date+Extensions.swift
│   │   ├── Double+Extensions.swift
│   │   └── View+Extensions.swift
│   ├── Helpers/
│   │   ├── Logger.swift
│   │   └── Formatters.swift
│   └── Constants.swift
│
└── Resources/
    ├── Assets.xcassets
    ├── Localizable.strings           # Future: localization
    └── SMARGE.xcdatamodeld            # SwiftData schema
```

---

## 6. Data Models

### 6.1 CloudKit Design Considerations (Phase 2)

All data models are designed to be **CloudKit-compatible** from the start:

**Design Principles:**
- Use standard Swift types (String, Int, Double, Date, UUID)
- Avoid complex relationships (minimize @Relationship complexity)
- Use simple enums with String raw values (CloudKit-compatible)
- Keep models flat where possible
- Plan for CKRecord representation

**Sharing Strategy (Phase 2):**
- **Shared Database (CKShare)**: BEVGoal, ChargeSchedule, OptimizationRun
- **Private Database**: User preferences, notification settings
- **Conflict Resolution**: Last-write-wins for most entities, manual for BEVGoal conflicts
- **Permissions**: All household members read/write on shared data

### 6.2 Core Domain Models

```swift
// BEV Charging Configuration
@Model
class BEVConfig {
    var id: UUID
    var defaultMinimumSoC: Double = 0.40  // Daily minimum (40% default)
    var createdAt: Date
    var updatedAt: Date
}

// BEV Charging Goal (optional, time-specific override)
@Model
class BEVGoal {
    var id: UUID
    var targetSoC: Double              // 0.0 - 1.0 (80% or 100%)
    var targetTime: Date               // Specific deadline (e.g., "tomorrow 8 AM")
    var createdAt: Date
    var isCompleted: Bool              // Auto-set when target reached
    
    // Computed
    var targetSoCPercentage: Int { Int(targetSoC * 100) }
    
    // Validation
    var isValid: Bool { targetSoC > 0.40 && targetTime > Date() }
}

// Charge Schedule Entry
@Model
class ChargeSchedule {
    var id: UUID
    var device: DeviceType             // .bev or .battery
    var startTime: Date
    var endTime: Date
    var powerKW: Double
    var source: EnergySource           // .solar or .grid
    var targetKWh: Double
    var estimatedCost: Double
    var rationale: String              // "Low price period" / "Solar surplus"
    var status: ScheduleStatus         // .pending, .active, .completed, .failed
    var actualKWh: Double?             // Filled after execution
    var actualCost: Double?
}

enum DeviceType: String, Codable {
    case bev = "BEV"
    case battery = "Battery"
}

enum EnergySource: String, Codable {
    case solar = "Solar"
    case grid = "Grid"
}

enum ScheduleStatus: String, Codable {
    case pending = "Pending"
    case active = "Active"
    case completed = "Completed"
    case failed = "Failed"
    case cancelled = "Cancelled"
}

// Device State Snapshot
@Model
class DeviceSnapshot {
    var id: UUID
    var timestamp: Date
    var bevSoC: Double?                // 0.0 - 1.0
    var bevSoCKWh: Double?             // Absolute kWh (0-55)
    var bevIsCharging: Bool
    var bevIsConnected: Bool
    var batterySoCKWh: Double?         // 0-12 kWh
    var batteryPowerKW: Double?        // +/- (charging/discharging)
    var solarPowerKW: Double?          // Current production
    var gridPowerKW: Double?           // Current import/export
}

// Energy Price
@Model
class EnergyPrice {
    var id: UUID
    var timestamp: Date                // Start of hour
    var spotPrice: Double              // €/kWh (before taxes)
    var totalPrice: Double             // €/kWh (including taxes & fees)
    var feedInRate: Double             // €/kWh compensation for export
    var isForecast: Bool               // true if future prediction
}

// Solar Forecast
@Model
class SolarForecast {
    var id: UUID
    var timestamp: Date                // Hour start
    var forecastKW: Double             // Predicted generation
    var confidence: Double             // 0.0 - 1.0
    var weatherCondition: String       // "sunny", "cloudy", etc.
}

// Optimization Run Record
@Model
class OptimizationRun {
    var id: UUID
    var timestamp: Date
    var inputBEVSoC: Double
    var inputBatterySoC: Double
    var inputBEVGoal: Double
    var inputBEVDeadline: Date
    var outputSchedules: [ChargeSchedule]
    var estimatedTotalCost: Double
    var estimatedSolarUsageKWh: Double
    var estimatedGridUsageKWh: Double
    var computationTimeMs: Double
}
```

### 6.3 Configuration Models

```swift
// System Configuration
struct SystemConfig: Codable {
    var solar: SolarConfig
    var battery: BatteryConfig
    var bev: BEVConfig
    var household: HouseholdConfig
}

struct SolarConfig: Codable {
    var capacityKW: Double = 13.0
    var orientation: String = "southwest"
    var tiltDegrees: Double = 45.0
    var location: Location
}

struct Location: Codable {
    var latitude: Double
    var longitude: Double
}

struct BatteryConfig: Codable {
    var capacityKWh: Double = 12.0
    var maxChargeRateKW: Double = 5.0
    var maxDischargeRateKW: Double = 5.0
    var minSoCKWh: Double = 1.0        // Reserve capacity
    var maxSoCKWh: Double = 11.5       // Avoid 100% for longevity
}

struct BEVConfig: Codable {
    var capacityKWh: Double = 55.0
    var maxChargeRateKW: Double = 11.0
    var minSoCKWh: Double = 5.0        // Emergency reserve
}

struct HouseholdConfig: Codable {
    var annualConsumptionKWh: Double = 17000.0
    var averageHourlyConsumptionKW: Double {
        annualConsumptionKWh / 8760.0  // ~1.94 kW
    }
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

**Swift Implementation:**
```swift
class TibberAPI {
    private let baseURL = "https://api.tibber.com/v1-beta/gql"
    private let apiKey: String
    
    func fetchPriceForecast() async throws -> [EnergyPrice] {
        let query = """
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
        """
        
        let request = GraphQLRequest(query: query)
        let response: TibberPriceResponse = try await networkManager.execute(request)
        return response.toPrices()
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

**Swift Implementation:**
```swift
class WeatherAPI {
    func fetchSolarForecast(lat: Double, lon: Double) async throws -> [SolarForecast] {
        let url = URL(string: """
            https://api.open-meteo.com/v1/forecast\
            ?latitude=\(lat)&longitude=\(lon)\
            &hourly=shortwave_radiation,cloudcover\
            &forecast_days=2
            """)!
        
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(WeatherResponse.self, from: data)
        return response.toSolarForecasts(systemCapacity: 13.0)
    }
}
```

### 7.3 Q.HOME+ ESS & EDRIVE Integration

**CRITICAL RESEARCH FINDING:** Q.HOME devices do not provide publicly documented APIs.

**Official Q.HOME App:** Uses proprietary undocumented protocols ([App Store](https://apps.apple.com/de/app/q-home/id1491103343))

**Integration Status:** ⚠️ **UNCERTAIN - Requires Investigation**

---

#### Option 1: Modbus TCP (RECOMMENDED FOR INVESTIGATION)

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

#### Option 2: Official API Request (PARALLEL EFFORT)

**Action Items:**
1. Email Q CELLS support: `sales@q-cells.com` or developer portal
2. Request:
   - Developer API documentation
   - Partner program for third-party integration
   - Technical specifications for programmatic access
3. Reference official Q.HOME app as proof of capability

**Timeline:** Likely 2-4 weeks for response, uncertain approval

---

#### Option 3: Manual Mode (MVP FALLBACK)

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

### 7.4 Q.HOME EDRIVE Integration (Wallbox)

**Integration Path:** Via Q.HOME+ ESS inverter (unified system)

**Research Findings:**
- Q.HOME EDRIVE wallbox is controlled **through** the inverter, not independently
- Official Q.HOME app shows "EDRIVE G2" access via platform login
- No separate wallbox API - all data/control via inverter

**Data Available (if Modbus/API accessible):**
- BEV connection status (plugged in / not connected)
- BEV current SoC % (if vehicle reports to wallbox)
- Charging state (charging / idle)
- Current charge rate (kW)
- Session energy delivered (kWh)

**Expected Modbus Registers (MUST VERIFY):**
```swift
// Conceptual - actual addresses unknown
let wallboxRegisters = [
    "vehicle_connected": 2000,    // Bool
    "vehicle_soc": 2001,          // Percent (0-100)
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
- User checks BEV SoC from vehicle's own app (Skoda Connect, etc.)
- Manual entry in SMARGE app
- Optimization still works with user-provided data

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

```swift
// AppDelegate.swift
class AppDelegate: NSObject, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil) -> Bool {
        
        // Register background tasks
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.smarge.refresh",
            using: nil
        ) { task in
            self.handleAppRefresh(task: task as! BGAppRefreshTask)
        }
        
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.smarge.optimize",
            using: nil
        ) { task in
            self.handleOptimization(task: task as! BGProcessingTask)
        }
        
        return true
    }
    
    func handleAppRefresh(task: BGAppRefreshTask) {
        // Schedule next refresh
        scheduleAppRefresh()
        
        let operation = RefreshDataOperation()
        
        task.expirationHandler = {
            operation.cancel()
        }
        
        operation.completionBlock = {
            task.setTaskCompleted(success: !operation.isCancelled)
        }
        
        OperationQueue().addOperation(operation)
    }
    
    func handleOptimization(task: BGProcessingTask) {
        scheduleOptimization()
        
        let operation = OptimizationOperation()
        
        task.expirationHandler = {
            operation.cancel()
        }
        
        operation.completionBlock = {
            task.setTaskCompleted(success: !operation.isCancelled)
        }
        
        OperationQueue().addOperation(operation)
    }
    
    func scheduleAppRefresh() {
        let request = BGAppRefreshTaskRequest(identifier: "com.smarge.refresh")
        request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)
        
        try? BGTaskScheduler.shared.submit(request)
    }
    
    func scheduleOptimization() {
        let request = BGProcessingTaskRequest(identifier: "com.smarge.optimize")
        request.requiresNetworkConnectivity = true
        request.earliestBeginDate = Date(timeIntervalSinceNow: 60 * 60)
        
        try? BGTaskScheduler.shared.submit(request)
    }
}
```

### 9.3 Notification Scheduling

```swift
class NotificationManager {
    static let shared = NotificationManager()
    
    func scheduleMorningOptimizationReminder() {
        let content = UNMutableNotificationContent()
        content.title = "Good morning!"
        content.body = "Optimize solar usage for today"
        content.sound = .default
        content.categoryIdentifier = "MORNING_OPTIMIZATION"
        content.userInfo = ["action": "optimize_solar"]
        
        var dateComponents = DateComponents()
        dateComponents.hour = 6
        dateComponents.minute = 30
        
        let trigger = UNCalendarNotificationTrigger(
            dateMatching: dateComponents,
            repeats: true
        )
        
        let request = UNNotificationRequest(
            identifier: "morning-optimization",
            content: content,
            trigger: trigger
        )
        
        UNUserNotificationCenter.current().add(request)
    }
    
    func scheduleAfternoonOptimizationReminder() {
        let content = UNMutableNotificationContent()
        content.title = "Prices updated!"
        content.body = "Tomorrow's electricity prices are available. Re-optimize?"
        content.sound = .default
        content.categoryIdentifier = "AFTERNOON_OPTIMIZATION"
        content.userInfo = ["action": "optimize_prices"]
        
        var dateComponents = DateComponents()
        dateComponents.hour = 13
        dateComponents.minute = 30
        
        let trigger = UNCalendarNotificationTrigger(
            dateMatching: dateComponents,
            repeats: true
        )
        
        let request = UNNotificationRequest(
            identifier: "afternoon-optimization",
            content: content,
            trigger: trigger
        )
        
        UNUserNotificationCenter.current().add(request)
    }
    
    func scheduleChargeSessionReminder(session: ChargeSchedule) {
        let reminderTime = session.startTime.addingTimeInterval(-15 * 60)
        
        guard reminderTime > Date() else { return }
        
        let content = UNMutableNotificationContent()
        content.title = "\(session.device.rawValue) charging soon"
        content.body = "Charging starts in 15 minutes"
        content.sound = .default
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
