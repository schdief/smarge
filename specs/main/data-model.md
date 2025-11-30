# Data Models: SMARGE MVP

**Date**: November 30, 2025  
**Phase**: Phase 1 (Design)  
**Storage**: Isar (local persistence)  
**CloudKit Compatibility**: Designed for Phase 2 sync

This document defines all data models for the SMARGE application. Models use standard Dart types and flat structures to ensure CloudKit compatibility in Phase 2.

---

## Design Principles

1. **CloudKit-Ready**: Use String, int, double, DateTime, List<primitive> only
2. **Flat Structures**: Avoid deep nesting (max 1 level)
3. **Immutable**: All models are immutable (final fields)
4. **Indexed**: Critical query fields marked for Isar indexing
5. **Serializable**: All models support JSON serialization

---

## Core Data Models

### 1. BEVGoal

Represents a user-defined BEV charging target.

```dart
import 'package:isar/isar.dart';

@collection
class BEVGoal {
  Id id = Isar.autoIncrement;
  
  /// Unique identifier (for CloudKit CKRecord.ID in Phase 2)
  @Index(unique: true)
  late String goalId;
  
  /// Target state of charge (0.0 - 1.0)
  /// e.g., 0.80 = 80%
  late double targetSoC;
  
  /// Deadline to reach target SoC
  @Index()
  late DateTime targetTime;
  
  /// When this goal was created
  late DateTime createdAt;
  
  /// User who created (for multi-user Phase 2)
  /// Format: "user-{uuid}" or "default"
  late String createdBy;
  
  /// Whether this is the default minimum (vs. specific target)
  late bool isDefaultMinimum;
  
  /// Auto-remove after completion or deadline passed
  late bool isActive;
  
  /// Human-readable reason (optional)
  String? reason;
  
  // Validation
  bool get isValid {
    return targetSoC >= 0.0 && 
           targetSoC <= 1.0 && 
           targetTime.isAfter(DateTime.now());
  }
  
  // Computed kWh based on BEV capacity
  double toKWh(double bevCapacityKWh) {
    return targetSoC * bevCapacityKWh;
  }
}
```

**Use Cases:**
- Default minimum: `BEVGoal(targetSoC: 0.40, isDefaultMinimum: true, targetTime: tomorrow6AM)`
- Specific target: `BEVGoal(targetSoC: 0.80, targetTime: tomorrowAt8AM, reason: "Road trip")`

**Lifecycle:**
- Created when user sets goal
- Marked inactive after deadline or when target reached
- Auto-cleaned after 7 days of inactivity

---

### 2. ChargeSchedule

Represents a planned charging session.

```dart
@collection
class ChargeSchedule {
  Id id = Isar.autoIncrement;
  
  @Index(unique: true)
  late String scheduleId;
  
  /// Which device to charge
  @Enumerated(EnumType.name)
  late DeviceType device;
  
  /// Start time for charging
  @Index()
  late DateTime startTime;
  
  /// End time for charging
  late DateTime endTime;
  
  /// Energy source (solar or grid)
  @Enumerated(EnumType.name)
  late EnergySource source;
  
  /// Target energy to charge (kWh)
  late double targetKWh;
  
  /// Estimated cost (€)
  late double estimatedCost;
  
  /// Current schedule status
  @Enumerated(EnumType.name)
  late ScheduleStatus status;
  
  /// Rationale for this session (shown to user)
  late String rationale;
  
  /// Actual kWh charged (updated during execution)
  double? actualKWh;
  
  /// Actual cost (updated during execution)
  double? actualCost;
  
  /// When this schedule was created (optimization run)
  late DateTime createdAt;
  
  /// Which optimization run created this
  late String optimizationRunId;
  
  // Computed duration
  Duration get duration {
    return endTime.difference(startTime);
  }
  
  // Is this schedule starting soon?
  bool get isUpcoming {
    final now = DateTime.now();
    final diff = startTime.difference(now);
    return diff.inMinutes > 0 && diff.inMinutes <= 15;
  }
  
  // Is this schedule currently active?
  bool get isActive {
    final now = DateTime.now();
    return now.isAfter(startTime) && now.isBefore(endTime);
  }
}

enum DeviceType {
  bev,
  battery,
}

enum EnergySource {
  solar,
  grid,
}

enum ScheduleStatus {
  pending,    // Not started yet
  active,     // Currently executing
  completed,  // Finished successfully
  failed,     // Execution failed
  cancelled,  // User cancelled
}
```

**Use Cases:**
- Solar charging: `ChargeSchedule(device: bev, source: solar, startTime: 11AM, targetKWh: 8.0, rationale: "Solar surplus")`
- Grid charging: `ChargeSchedule(device: bev, source: grid, startTime: 2AM, targetKWh: 10.0, estimatedCost: 1.80, rationale: "Low price (€0.18/kWh)")`

**Lifecycle:**
- Created by optimization runs
- Status updates: pending → active → completed/failed
- Archived after 30 days

---

### 3. DeviceSnapshot

Captures point-in-time state of all devices.

```dart
@collection
class DeviceSnapshot {
  Id id = Isar.autoIncrement;
  
  @Index(unique: true)
  late String snapshotId;
  
  /// When this snapshot was taken
  @Index()
  late DateTime timestamp;
  
  // BEV State
  /// BEV state of charge (0.0 - 1.0), null if unavailable
  double? bevSoC;
  
  /// BEV SoC in absolute kWh (0-55 for Enyaq)
  double? bevSoCKWh;
  
  /// Is BEV currently charging?
  late bool bevIsCharging;
  
  /// Is BEV connected to wallbox?
  late bool bevIsConnected;
  
  // Home Battery State
  /// Battery SoC in kWh (0-12)
  double? batterySoCKWh;
  
  /// Battery power (+ charging, - discharging)
  double? batteryPowerKW;
  
  // Solar State
  /// Current solar production (kW)
  double? solarPowerKW;
  
  // Grid State
  /// Current grid power (+ import, - export)
  double? gridPowerKW;
  
  /// Data source (for debugging)
  /// e.g., "myskoda-api", "qhome-cloud", "manual-entry"
  late String dataSource;
  
  // Computed: Is household self-sufficient?
  bool get isSelfSufficient {
    if (solarPowerKW == null || gridPowerKW == null) return false;
    return gridPowerKW! <= 0; // Not importing from grid
  }
}
```

**Use Cases:**
- Periodic snapshots: Every 15 minutes when app active
- Before optimization: Capture current state for calculations
- After schedule execution: Verify results

**Lifecycle:**
- Created every 15-30 minutes (configurable)
- Kept for 1 year (365 days × 96 snapshots/day = ~35K records)
- Older records auto-deleted

---

### 4. EnergyPrice

Tibber spot price data (current + forecast).

```dart
@collection
class EnergyPrice {
  Id id = Isar.autoIncrement;
  
  @Index(unique: true)
  late String priceId;
  
  /// Start of the hour this price applies to
  @Index()
  late DateTime timestamp;
  
  /// Spot price before taxes (€/kWh)
  late double spotPrice;
  
  /// Total price including taxes & fees (€/kWh)
  late double totalPrice;
  
  /// Feed-in compensation rate (€/kWh)
  /// Typically ~0.09 €/kWh
  late double feedInRate;
  
  /// Is this a forecast vs. actual price?
  late bool isForecast;
  
  /// Price level indicator from Tibber
  /// "VERY_CHEAP", "CHEAP", "NORMAL", "EXPENSIVE", "VERY_EXPENSIVE"
  String? priceLevel;
  
  /// When this price data was fetched
  late DateTime fetchedAt;
  
  /// Tibber home ID (for multi-home future support)
  late String tibberHomeId;
  
  // Computed: Opportunity cost of using grid vs. selling solar
  double get opportunityCost {
    return totalPrice - feedInRate;
  }
  
  // Computed: Is this a good price?
  bool isGoodPrice(double averagePrice) {
    return totalPrice < averagePrice * 0.85; // 15% below average
  }
}
```

**Use Cases:**
- Current day prices: Fetched at app startup
- Next day prices: Fetched after 1 PM daily
- Historical prices: Kept for analytics

**Lifecycle:**
- Fetched twice daily (or more if prices change)
- Current + next day: Always kept (48-96 hours)
- Historical: Kept for 1 year
- Auto-delete after 365 days

---

### 5. SolarForecast

Weather-based solar production forecast.

```dart
@collection
class SolarForecast {
  Id id = Isar.autoIncrement;
  
  @Index(unique: true)
  late String forecastId;
  
  /// Hour this forecast applies to
  @Index()
  late DateTime timestamp;
  
  /// Predicted solar generation (kW)
  late double forecastKW;
  
  /// Forecast confidence (0.0 - 1.0)
  /// Based on cloud cover, time of day, historical accuracy
  late double confidence;
  
  /// Weather condition description
  /// e.g., "sunny", "partly_cloudy", "cloudy", "rainy"
  late String weatherCondition;
  
  /// Cloud cover percentage (0-100)
  late int cloudCoverPercent;
  
  /// Direct solar radiation (W/m²)
  late double directRadiation;
  
  /// Diffuse solar radiation (W/m²)
  late double diffuseRadiation;
  
  /// When this forecast was generated
  late DateTime forecastedAt;
  
  /// Data source (e.g., "open-meteo")
  late String dataSource;
  
  // Computed: Is this a good solar period?
  bool get isGoodSolarPeriod {
    return forecastKW >= 5.0 && confidence >= 0.7;
  }
  
  // Computed: Expected kWh for this hour
  double get expectedKWh {
    return forecastKW * confidence; // De-rated by confidence
  }
}
```

**Use Cases:**
- Generated from Open-Meteo API data
- Used by optimization algorithm to prioritize solar charging
- Updated every 1-3 hours (or when forecast changes significantly)

**Lifecycle:**
- Fetched when optimization runs
- Kept for 48 hours ahead
- Historical forecasts kept for 7 days (for ML improvement later)

---

### 6. OptimizationRun

Record of each optimization execution (for debugging and analytics).

```dart
@collection
class OptimizationRun {
  Id id = Isar.autoIncrement;
  
  @Index(unique: true)
  late String runId;
  
  /// When optimization was run
  @Index()
  late DateTime timestamp;
  
  /// Trigger type
  /// "manual", "morning-scheduled", "afternoon-scheduled", "price-alert"
  late String triggerType;
  
  // Input State
  late double inputBEVSoC;
  late double inputBatterySoC;
  late double inputBEVGoal;
  late DateTime inputBEVDeadline;
  
  // Input Data Counts (for debugging)
  late int priceDataPoints;
  late int solarForecastDataPoints;
  
  // Output Results
  /// IDs of generated schedules
  late List<String> outputScheduleIds;
  
  late double estimatedTotalCost;
  late double estimatedSolarUsageKWh;
  late double estimatedGridUsageKWh;
  
  /// Computation time (milliseconds)
  late int computationTimeMs;
  
  /// Algorithm used
  /// "greedy-v1", "linear-programming-v2", etc.
  late String algorithmVersion;
  
  /// Was optimization successful?
  late bool success;
  
  /// Error message if failed
  String? errorMessage;
  
  // Computed: Efficiency metrics
  double get solarPercentage {
    final total = estimatedSolarUsageKWh + estimatedGridUsageKWh;
    return total > 0 ? (estimatedSolarUsageKWh / total) : 0.0;
  }
}
```

**Use Cases:**
- Debugging optimization failures
- Performance monitoring (computation time trends)
- Analytics (cost savings over time)
- A/B testing different algorithms

**Lifecycle:**
- Created every time optimization runs (2x daily minimum)
- Kept for 90 days
- Archived for long-term analytics (optional)

---

## Configuration Models

### 7. SystemConfig

Overall system configuration (stored as singleton in Isar).

```dart
@collection
class SystemConfig {
  Id id = Isar.autoIncrement;
  
  /// Singleton pattern: Always id = 1
  @Index(unique: true)
  late int configVersion;
  
  /// Last updated timestamp
  late DateTime lastUpdated;
  
  // Solar Configuration
  late double solarCapacityKW;
  late String solarOrientation;
  late double solarTiltDegrees;
  late double latitude;
  late double longitude;
  
  // Battery Configuration
  late double batteryCapacityKWh;
  late double batteryMaxChargeRateKW;
  late double batteryMaxDischargeRateKW;
  late double batteryMinSoCKWh;
  late double batteryMaxSoCKWh;
  
  // BEV Configuration
  late double bevCapacityKWh;
  late double bevMaxChargeRateKW;
  late double bevDefaultMinimumSoC;
  late String bevVIN;
  late String bevModel;
  
  // Household Configuration
  late double annualConsumptionKWh;
  late double typicalDailyConsumptionKWh;
  
  // Notification Preferences
  late bool notificationsMorningOptimization;
  late bool notificationsAfternoonOptimization;
  late bool notificationsPriceAlerts;
  late bool notificationsChargingEvents;
  late String notificationsMorningTime;  // "06:30"
  late String notificationsAfternoonTime; // "13:30"
  
  // API Credentials (reference to secure storage)
  late bool hasTibberCredentials;
  late bool hasMySkodaCredentials;
  late bool hasQHomeCredentials;
  
  /// User has completed onboarding
  late bool onboardingCompleted;
  
  /// User's preferred units
  late String preferredCurrency; // "EUR", "USD"
  late String preferredLanguage; // "en", "de"
}
```

**Use Cases:**
- Loaded at app startup
- Referenced by optimization algorithm
- Updated in settings screen

**Lifecycle:**
- Created during onboarding
- Updated when user changes settings
- Single instance (singleton)

---

### 8. UserPreferences (Phase 2 - Multi-User)

Per-user preferences for multi-user households.

```dart
@collection
class UserPreferences {
  Id id = Isar.autoIncrement;
  
  @Index(unique: true)
  late String userId;
  
  late String displayName;
  
  /// Notification preferences (can differ per user)
  late bool enableMorningNotifications;
  late bool enableAfternoonNotifications;
  late bool enablePriceAlerts;
  late bool enableChargingNotifications;
  
  /// UI preferences
  late String theme; // "light", "dark", "system"
  late bool showAdvancedSettings;
  
  /// Last app version used (for migration)
  late String lastAppVersion;
  
  late DateTime createdAt;
  late DateTime lastActiveAt;
}
```

**Note**: Phase 2 only (multi-user with CloudKit).

---

## Relationships

```
BEVGoal ──(1:N)──> ChargeSchedule
   │                    │
   └──(via optimizationRunId)─> OptimizationRun
   
OptimizationRun ──(1:N)──> ChargeSchedule

DeviceSnapshot ──(independent)

EnergyPrice ──(used by)──> OptimizationRun

SolarForecast ──(used by)──> OptimizationRun

SystemConfig ──(singleton)
```

---

## Isar Queries (Examples)

### Get Active BEV Goal
```dart
final goal = await isar.bEVGoals
  .filter()
  .isActiveEqualTo(true)
  .and()
  .targetTimeGreaterThan(DateTime.now())
  .sortByTargetTime()
  .findFirst();
```

### Get Today's Prices
```dart
final today = DateTime.now();
final startOfDay = DateTime(today.year, today.month, today.day);
final endOfDay = startOfDay.add(Duration(days: 1));

final prices = await isar.energyPrices
  .where()
  .timestampBetween(startOfDay, endOfDay)
  .sortByTimestamp()
  .findAll();
```

### Get Upcoming Schedules
```dart
final upcoming = await isar.chargeSchedules
  .filter()
  .statusEqualTo(ScheduleStatus.pending)
  .and()
  .startTimeGreaterThan(DateTime.now())
  .sortByStartTime()
  .findAll();
```

### Calculate Monthly Cost
```dart
final thisMonth = DateTime.now();
final startOfMonth = DateTime(thisMonth.year, thisMonth.month, 1);

final schedules = await isar.chargeSchedules
  .filter()
  .statusEqualTo(ScheduleStatus.completed)
  .and()
  .startTimeBetween(startOfMonth, DateTime.now())
  .findAll();

final totalCost = schedules.fold(0.0, (sum, s) => sum + (s.actualCost ?? s.estimatedCost));
```

---

## CloudKit Compatibility Notes (Phase 2)

All models designed for CloudKit sync:

1. **Primary Keys**: 
   - Isar: `Id id` (auto-increment)
   - CloudKit: `String xxxId` (UUID, mapped to CKRecord.recordID)

2. **Timestamps**: 
   - Dart `DateTime` → CloudKit `Date`

3. **Enums**: 
   - Dart enums → CloudKit String (store enum name)

4. **Lists**: 
   - Primitive lists only (List<String>)
   - No List<CustomObject> (flatten to separate records)

5. **Nullability**: 
   - CloudKit supports optional fields
   - Use `late` for required, `Type?` for optional

6. **References**:
   - `String xxxId` → CloudKit CKRecord.Reference
   - Example: `optimizationRunId` → CKReference to OptimizationRun record

**Migration Strategy**:
- Phase 1: Isar local storage
- Phase 2: Add CloudKit sync layer
- Data models: No changes needed
- Sync logic: Map Isar ↔ CKRecord

---

## Summary

**Total Models**: 8 core + 1 Phase 2
**Storage**: Isar (local), CloudKit (Phase 2 multi-user)
**Data Retention**: 
- Prices: 1 year
- Snapshots: 1 year
- Schedules: 30 days
- Optimization runs: 90 days
- Goals: 7 days after completion
- Config: Indefinite

**Estimated Storage**: ~50 MB for 1 year of data
