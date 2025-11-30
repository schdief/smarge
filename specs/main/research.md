# Research & Investigation: SMARGE MVP

**Date**: November 30, 2025  
**Phase**: Phase 0 (Pre-Development Research)  
**Status**: Investigation Required

This document captures research findings and decisions needed before implementation begins.

---

## 1. Q.HOME+ ESS API Integration

### Investigation Status: 🔍 **IN PROGRESS** (3-week timeline)

**Goal**: Determine best approach for Q.HOME+ ESS battery and Q.HOME EDRIVE wallbox integration.

### 1.1 Investigation Tasks

**Week 1: Initial Discovery**
- [ ] Access Q.HOME+ ESS web interface at https://qhome-ess-g3.q-cells.eu
- [ ] Login with credentials and open browser DevTools (Network tab)
- [ ] Capture all API calls during:
  - Login/authentication flow
  - Dashboard loading (battery status, solar production)
  - Wallbox status viewing
  - Any control operations visible in web UI
- [ ] Document endpoints, headers, request/response formats
- [ ] Test if control endpoints (battery scheduling, wallbox start/stop) exist in cloud API
- [ ] Contact Q CELLS support requesting official API documentation

**Week 2: Local Inverter Investigation**
- [ ] Find inverter IP address (check router DHCP table or Q.HOME mobile app settings)
- [ ] Access local web interface at `http://[inverter-ip]`
- [ ] Explore control settings pages with DevTools
- [ ] Capture local API calls for:
  - Battery charge mode changes (auto/manual/force charge)
  - Battery charging schedule configuration
  - Wallbox start/stop commands (if available)
- [ ] Test Modbus TCP support (default port 502):
  - Use tool like `modpoll` or Python `pymodbus` to scan for Modbus registers
  - Check inverter documentation or web UI for "Communication Settings" or "Modbus Configuration"
  - If Modbus supported, identify register addresses for:
    - Battery SoC (read-only)
    - Solar production (read-only)
    - Wallbox status (read-only)
    - Charging control registers (read/write)

**Week 3: Decision & Fallback Design**
- [ ] Consolidate findings from cloud vs. local APIs
- [ ] If API available: Document authentication, rate limits, available operations
- [ ] If no API: Design manual mode UX (notification-based user actions)
- [ ] Update data model and service layer designs based on findings
- [ ] Finalize MVP integration strategy

### 1.2 Known Constraints

**BEV SoC is NOT from Q.HOME:**
- Q.HOME wallbox does NOT report BEV state of charge to inverter
- BEV SoC obtained from MySkoda API instead (vehicle reports via cellular)
- Q.HOME needed for: Battery SoC, solar production, wallbox status (connected/charging/power)

**Expected Capabilities by API Type:**

| Capability | Cloud API (Remote) | Local API (Home WiFi) | Modbus TCP |
|------------|-------------------|----------------------|------------|
| Battery SoC | ✅ Likely | ✅ Likely | ✅ Likely |
| Solar Production | ✅ Likely | ✅ Likely | ✅ Likely |
| Wallbox Status | ✅ Likely | ✅ Likely | ✅ Likely |
| Battery Charge Control | ❌ Unlikely (safety) | ✅ Possible | ✅ Possible |
| Wallbox Start/Stop | ❌ Unlikely (safety) | ✅ Possible | ✅ Possible |
| Real-time Updates | Polling only | Polling or WS | Polling only |

### 1.3 Fallback Strategy (If No Control API)

**MVP Approach: Monitoring + Manual Control**

**What works automatically:**
- ✅ Fetch battery SoC from Q.HOME (monitoring API)
- ✅ Fetch solar production from Q.HOME
- ✅ Fetch wallbox status (connected, charging, power)
- ✅ BEV SoC and control via MySkoda API (fully automated)
- ✅ Optimization calculations run locally
- ✅ Schedule generation and display

**What requires manual user action:**
- User receives notification: "Start battery charging now (low price period)"
- User opens Q.HOME mobile app → enables charging mode
- SMARGE app shows: "Waiting for battery charging to start..." with retry button
- User confirms action in SMARGE when complete

**UX Flow for Manual Mode:**
```
6:30 AM → SMARGE optimization runs
          ↓
       Schedule shows: "Battery charge 2:00 AM - 4:00 AM (€0.18/kWh)"
          ↓
1:45 AM → Notification: "Battery charge starting in 15 min - Enable in Q.HOME app"
          ↓
       User opens Q.HOME app → Sets to charge mode
          ↓
       User returns to SMARGE → Taps "I've enabled charging"
          ↓
       SMARGE polls status → Confirms charging started → Shows green checkmark
```

**Acceptable for MVP because:**
- BEV optimization (primary value) works fully automated via MySkoda
- Battery optimization is secondary/optional (constitution principle)
- Manual mode proves optimization value before investing in reverse engineering
- Can add automation in Phase 2 after investigation completes

---

## 2. Flutter Background Tasks on iOS

### Decision: **Use flutter_background_fetch**

**Rationale:**
- iOS BGTaskScheduler wrapper designed for Flutter
- Handles iOS background execution limits (~30 seconds)
- Proven in production apps
- Better iOS integration than workmanager alone

**Implementation Pattern:**
```dart
import 'package:background_fetch/background_fetch.dart';

// In main.dart
void backgroundFetchHeadlessTask(HeadlessTask task) async {
  String taskId = task.taskId;
  
  // Run optimization
  await OptimizationService.runScheduledOptimization();
  
  BackgroundFetch.finish(taskId);
}

void main() {
  runApp(SmargeApp());
  BackgroundFetch.registerHeadlessTask(backgroundFetchHeadlessTask);
}

// In app initialization
BackgroundFetch.configure(
  BackgroundFetchConfig(
    minimumFetchInterval: 15, // iOS allows 15-30 min
    stopOnTerminate: false,
    enableHeadless: true,
    requiresBatteryNotLow: false,
    requiresCharging: false,
    requiresStorageNotLow: false,
    requiresDeviceIdle: false,
  ),
  (String taskId) async {
    // Runs when app in foreground/background
    await OptimizationService.runScheduledOptimization();
    BackgroundFetch.finish(taskId);
  },
);
```

**Background Task Constraints:**
- Execution window: ~30 seconds max
- Frequency: iOS decides (request 15 min, might get 30-60 min)
- No guaranteed timing (system discretion)
- Must complete quickly or task will be killed

**Strategy to Work Within Limits:**
1. **Quick operations only**: Fetch prices, check if optimization needed
2. **Defer heavy work**: If full optimization needed, send notification instead
3. **Notification reminders**: Ensure user opens app at 6:30 AM and 1:30 PM
4. **Cache aggressively**: Minimize network calls in background

**Alternative for Future:**
- iOS 17+ supports background URLSession for longer downloads
- Consider for large dataset fetches if needed
- Not required for MVP (small API responses)

---

## 3. Optimization Algorithm Approach

### Decision: **Start with Greedy Algorithm, Evolve to Linear Programming**

**Phase 1 MVP: Greedy Algorithm**

**Why Greedy First:**
- Simple to implement and test
- Achieves 80% of optimal results
- Fast computation (<1 second vs. <10 second requirement)
- Proves core value before complexity investment
- Easier to explain to users ("charge during cheapest hours")

**Greedy Algorithm Pseudocode:**
```dart
// Simplified greedy approach
class GreedyOptimizer {
  List<ChargeSchedule> optimize({
    required BEVGoal goal,
    required double currentBEVSoC,
    required List<EnergyPrice> prices,
    required List<SolarForecast> solarForecast,
  }) {
    final schedules = <ChargeSchedule>[];
    double remainingKWhNeeded = calculateKWhNeeded(goal, currentBEVSoC);
    
    // 1. Prioritize solar periods
    for (var hour in solarForecast.where((f) => f.forecastKW > 5.0)) {
      if (remainingKWhNeeded <= 0) break;
      
      final kWhFromSolar = min(hour.forecastKW, 11.0); // 11kW wallbox max
      schedules.add(ChargeSchedule(
        startTime: hour.timestamp,
        targetKWh: kWhFromSolar,
        source: EnergySource.solar,
        estimatedCost: 0.0, // Solar is "free" (opportunity cost handled separately)
      ));
      
      remainingKWhNeeded -= kWhFromSolar;
    }
    
    // 2. If solar insufficient, charge during cheapest grid hours
    if (remainingKWhNeeded > 0) {
      final sortedPrices = prices
        .where((p) => p.timestamp.isBefore(goal.targetTime))
        .sortedBy((p) => p.totalPrice); // Cheapest first
      
      for (var priceSlot in sortedPrices) {
        if (remainingKWhNeeded <= 0) break;
        
        final kWhThisHour = min(remainingKWhNeeded, 11.0);
        schedules.add(ChargeSchedule(
          startTime: priceSlot.timestamp,
          targetKWh: kWhThisHour,
          source: EnergySource.grid,
          estimatedCost: kWhThisHour * priceSlot.totalPrice,
        ));
        
        remainingKWhNeeded -= kWhThisHour;
      }
    }
    
    return schedules.sortedBy((s) => s.startTime);
  }
}
```

**Known Limitations:**
- Doesn't account for battery charging/discharging optimization
- Doesn't consider ramping (starting charge before solar peak)
- Doesn't optimize for partial-hour charging
- No constraint solving (assumes simple scenarios)

**Acceptable for MVP because:**
- BEV charging is primary use case (simpler than battery arbitrage)
- Solar forecasts are hourly (matches granularity)
- Constitution principle: "Start simple, evolve complexity"

**Phase 2: Linear Programming (Future)**

**When to upgrade:**
- After MVP validates user value
- When adding home battery optimization (charge/discharge arbitrage)
- When users report suboptimal schedules

**Approach:**
- Use Dart package `dart_lp` or call Python SciPy via process
- Model as constraint satisfaction problem:
  - Objective: Minimize cost
  - Constraints: BEV target SoC, battery limits, wallbox power, time deadlines
  - Variables: Charge/discharge power per hour
- Expected computation time: 3-5 seconds for 48-hour window

---

## 4. Tibber API Integration

### Decision: **Use Tibber GraphQL API**

**Authentication:**
- Personal access token (API key)
- User generates in Tibber developer portal
- Store in Flutter Secure Storage

**Key Endpoints:**
```graphql
# Get viewer info and home ID
query {
  viewer {
    homes {
      id
      address {
        address1
      }
    }
  }
}

# Get current and future prices
query Prices($homeId: ID!) {
  viewer {
    home(id: $homeId) {
      currentSubscription {
        priceInfo {
          current {
            total
            startsAt
          }
          today {
            total
            startsAt
            level # "VERY_CHEAP", "CHEAP", "NORMAL", "EXPENSIVE"
          }
          tomorrow {
            total
            startsAt
            level
          }
        }
      }
    }
  }
}
```

**Polling Strategy:**
- Current day prices: Once at app startup, cache for 24h
- Next day prices: Poll starting at 1:00 PM until available (every 30 min until 6:00 PM)
- Background task: Check for price updates hourly

**Price Availability Timeline:**
- Current day (0:00 - 23:59): Available 24/7
- Next day: Published ~1:00 PM (sometimes delayed until 5:00 PM)
- Two days ahead: Not available (must wait until next afternoon)

**Error Handling:**
- If next-day prices unavailable at 1:30 PM → Use historical average as estimate
- Show indicator: "Using estimated prices - check back after 5 PM"
- Re-optimize automatically when real prices arrive

**Rate Limits:**
- Tibber API: Not documented, assumed generous for personal use
- Implement exponential backoff on errors
- Cache responses aggressively

**Flutter Implementation:**
```dart
import 'package:graphql_flutter/graphql_flutter.dart';

class TibberClient {
  final GraphQLClient _client;
  
  TibberClient(String accessToken) 
    : _client = GraphQLClient(
        link: AuthLink(getToken: () => 'Bearer $accessToken')
          .concat(HttpLink('https://api.tibber.com/v1-beta/gql')),
        cache: GraphQLCache(),
      );
  
  Future<List<EnergyPrice>> getPrices({bool includeTomorrow = true}) async {
    final result = await _client.query(QueryOptions(
      document: gql(pricesQuery),
      variables: {'homeId': await _getHomeId()},
    ));
    
    if (result.hasException) throw result.exception!;
    
    return _parsePrices(result.data!);
  }
}
```

---

## 5. MySkoda API Integration

### Decision: **Reverse-engineer from `myskoda` Python library**

**Why Python Library as Reference:**
- Active open-source project: https://github.com/skodaconnect/myskoda
- Proven OAuth2 flow implementation
- Documented API endpoints and response formats
- Home Assistant integration uses same library (thousands of users)

**Authentication Flow (from library analysis):**
```dart
// 1. Initial login to get authorization code
// URL: https://identity.vwgroup.io/oidc/v1/authorize
// Params: client_id, redirect_uri, scope, response_type=code

// 2. Exchange code for access token
// URL: https://identity.vwgroup.io/oidc/v1/token
// Body: code, grant_type=authorization_code

// 3. Use access token for API calls
// Header: Authorization: Bearer {access_token}
// Refresh token when expired (typical: 1 hour)
```

**API Endpoints (from myskoda library):**
```
Base URL: https://mysmob.api.connect.skoda-auto.cz

GET  /api/v2/vehicle-status/{vin}
Response:
{
  "battery": {
    "stateOfChargeInPercent": 65,
    "remainingRangeInKm": 280
  },
  "charging": {
    "state": "CONNECT_CABLE", // or "CHARGING", "READY_FOR_CHARGING"
    "chargePowerInKw": 11.0,
    "remainingTimeToFullyChargedInMinutes": 120,
    "targetStateOfChargeInPercent": 80
  }
}

POST /api/v1/charging/{vin}/start
POST /api/v1/charging/{vin}/stop
PUT  /api/v1/charging/{vin}/set-charge-limit
Body: { "targetSOCInPercent": 80 }
```

**Control Operations (Requires S-PIN):**
- User must provide S-PIN (4-digit code from MySkoda app settings)
- Used to authorize control commands (start/stop charging, set limit)
- NOT required for read-only operations (get status)

**Polling Strategy:**
- Update every 15 minutes when vehicle connected to wallbox
- Update every 30 minutes otherwise
- Cache last-known value for 1 hour
- Manual refresh button in UI

**Fallback:**
- If API unavailable: Manual SoC entry
- Show last update timestamp
- Prompt user to check MySkoda app and enter current %

**Flutter Implementation:**
```dart
class MySkodaClient {
  final Dio _dio;
  String? _accessToken;
  DateTime? _tokenExpiry;
  
  Future<void> authenticate(String email, String password) async {
    // Implement OAuth2 flow based on myskoda library
    final authUrl = 'https://identity.vwgroup.io/oidc/v1/authorize';
    // ... (full implementation based on library analysis)
  }
  
  Future<VehicleStatus> getVehicleStatus(String vin) async {
    await _ensureValidToken();
    
    final response = await _dio.get(
      '/api/v2/vehicle-status/$vin',
      options: Options(headers: {'Authorization': 'Bearer $_accessToken'}),
    );
    
    return VehicleStatus.fromJson(response.data);
  }
  
  Future<void> startCharging(String vin, String sPin) async {
    await _dio.post(
      '/api/v1/charging/$vin/start',
      data: {'pin': sPin},
      options: Options(headers: {'Authorization': 'Bearer $_accessToken'}),
    );
  }
}
```

---

## 6. Open-Meteo Weather API

### Decision: **Use Open-Meteo Free API (No Key Required)**

**Why Open-Meteo:**
- ✅ Free, no API key required
- ✅ Accurate weather forecasts (comparable to paid services)
- ✅ Solar radiation data for PV production forecasting
- ✅ 7-day forecast (sufficient for 48-hour optimization window)
- ✅ Well-documented REST API

**Key Endpoint:**
```
GET https://api.open-meteo.com/v1/forecast

Required params:
- latitude (from SolarConfig)
- longitude (from SolarConfig)
- hourly=direct_radiation,diffuse_radiation,cloud_cover
- forecast_days=2
- timezone=auto

Example:
https://api.open-meteo.com/v1/forecast?latitude=52.52&longitude=13.41&hourly=direct_radiation,diffuse_radiation&forecast_days=2
```

**Response Format:**
```json
{
  "hourly": {
    "time": ["2025-11-30T00:00", "2025-11-30T01:00", ...],
    "direct_radiation": [0, 0, 0, 145, 328, 412, ...], // W/m²
    "diffuse_radiation": [0, 0, 0, 56, 98, 134, ...],  // W/m²
    "cloud_cover": [75, 80, 65, 40, 20, 10, ...]       // %
  }
}
```

**Solar Production Estimation:**
```dart
class SolarForecaster {
  // Convert radiation to kW production
  double estimateProduction({
    required double directRadiation,  // W/m²
    required double diffuseRadiation, // W/m²
    required SolarConfig config,
  }) {
    const pvEfficiency = 0.85;  // 85% system efficiency
    const derateFactor = 0.80;   // 20% conservative buffer
    
    final totalRadiation = directRadiation + diffuseRadiation;
    final theoreticalPower = (totalRadiation / 1000) * config.capacityKW;
    
    return theoreticalPower * pvEfficiency * derateFactor;
  }
}
```

**Polling Strategy:**
- Fetch forecast when user runs optimization
- Cache for 1 hour
- Re-fetch if forecast changes significantly (>20% for any hour)

**No API Key Required:**
- Open-Meteo is free and open-source
- Commercial use allowed
- No rate limits for reasonable personal use
- Consider donation if heavy usage

---

## 7. Data Storage: Hive vs. Isar

### Decision: **Use Isar for MVP**

**Comparison:**

| Feature | Hive | Isar |
|---------|------|------|
| Performance | Fast | Faster (native indexing) |
| Queries | Manual filtering | SQL-like queries |
| Indexes | No | Yes |
| Type safety | TypeAdapter required | Code generation |
| File size | Smaller | Slightly larger |
| Maturity | Stable | Newer but proven |
| CloudKit compatibility | Manual sync | Manual sync (both similar) |

**Rationale for Isar:**
- Better query performance for analytics (filtering by date, aggregations)
- Native indexing on timestamp fields (faster lookups)
- Cleaner query syntax vs. manual iteration
- Still pure Dart (no native dependencies in Flutter context)
- CloudKit sync: Both require manual implementation, no advantage to Hive

**Example Isar Model:**
```dart
import 'package:isar/isar.dart';

@collection
class EnergyPrice {
  Id id = Isar.autoIncrement;
  
  @Index()
  late DateTime timestamp;
  
  late double spotPrice;
  late double totalPrice;
  late double feedInRate;
  late bool isForecast;
}

// Query example
final todaysPrices = await isar.energyPrices
  .where()
  .timestampBetween(startOfDay, endOfDay)
  .sortByTimestamp()
  .findAll();
```

**Migration Path:**
- If Isar becomes unmaintained → Can export to JSON and migrate to Hive
- Data models stay the same (just change storage backend)

---

## 8. State Management: Riverpod vs. Provider

### Decision: **Use Riverpod**

**Rationale:**
- Compile-time safety (fewer runtime errors)
- Better testing support (providers are mockable)
- No BuildContext required for accessing state
- Growing community adoption
- Created by same author as Provider (Remi Rousselet)

**Key Patterns for SMARGE:**

```dart
// Service providers
@riverpod
OptimizationEngine optimizationEngine(OptimizationEngineRef ref) {
  return OptimizationEngine(
    tibberClient: ref.watch(tibberClientProvider),
    myskodalient: ref.watch(myskodaClientProvider),
    repository: ref.watch(repositoryProvider),
  );
}

// State providers
@riverpod
class BevGoalNotifier extends _$BevGoalNotifier {
  @override
  Future<BEVGoal?> build() async {
    final repo = ref.read(repositoryProvider);
    return repo.getCurrentBEVGoal();
  }
  
  Future<void> setGoal(BEVGoal goal) async {
    state = AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final repo = ref.read(repositoryProvider);
      await repo.saveBEVGoal(goal);
      return goal;
    });
  }
}

// UI consumption
class DashboardScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final bevGoal = ref.watch(bevGoalNotifierProvider);
    
    return bevGoal.when(
      data: (goal) => Text('Target: ${goal?.targetSoC}%'),
      loading: () => CircularProgressIndicator(),
      error: (e, st) => ErrorDisplay(error: e),
    );
  }
}
```

**Alternatives Considered:**
- **Provider**: Simpler but less type-safe, requires BuildContext
- **Bloc**: More boilerplate, overkill for this app size
- **GetX**: Not recommended (global state, anti-pattern concerns)

---

## 9. CloudKit Data Model Design (Phase 2)

### Decision: **Design for CloudKit from Start, Implement in Phase 2**

**Why Design Now:**
- Avoid data model rework later
- Ensure MVP models are CloudKit-compatible
- Use standard Dart types (String, int, double, DateTime, List)
- Flat structures (no deep nesting)

**CloudKit-Compatible Model Patterns:**
```dart
// ✅ Good: Flat structure, standard types
class BEVGoal {
  final String id;              // CKRecord.ID
  final DateTime targetTime;    // CKRecord Date
  final double targetSoC;       // CKRecord Double
  final DateTime createdAt;
  final String createdBy;       // CKRecord Reference to user
}

// ❌ Avoid: Nested objects, custom classes
class BadExample {
  final CustomClass nested;     // Won't sync to CloudKit easily
  final Map<String, dynamic> untyped;  // CloudKit needs typed fields
}
```

**Phase 2 CloudKit Implementation:**
1. Add CloudKit capability in Xcode
2. Create CKShare for household sharing
3. Map Dart models to CKRecord types
4. Implement CKSubscription for remote changes
5. Handle conflict resolution (last-write-wins)

**Phase 1 MVP:**
- Use same data models
- Store in Isar locally
- No sync logic yet
- Models ready for Phase 2 migration

---

## 10. Error Handling Strategy

### Decision: **Result Pattern + User-Friendly Messages**

**Pattern:**
```dart
// Result type for operations that can fail
sealed class Result<T> {
  const Result();
}

class Success<T> extends Result<T> {
  final T value;
  const Success(this.value);
}

class Failure<T> extends Result<T> {
  final String message;
  final Exception? exception;
  const Failure(this.message, [this.exception]);
}

// Usage in services
Future<Result<List<ChargeSchedule>>> optimize() async {
  try {
    final prices = await _tibberClient.getPrices();
    if (prices.isEmpty) {
      return Failure('No price data available');
    }
    
    final schedules = _engine.calculate(prices);
    return Success(schedules);
  } on NetworkException catch (e) {
    return Failure('Network error - check connection', e);
  } catch (e) {
    return Failure('Optimization failed', e as Exception);
  }
}

// UI handling
final result = await ref.read(optimizationEngineProvider).optimize();
switch (result) {
  case Success(:final value):
    // Show schedules
  case Failure(:final message):
    // Show user-friendly error
}
```

**User-Friendly Error Messages:**
- "Cannot reach Tibber - check internet connection"
- "BEV data unavailable - enter manually?"
- "Optimization failed - try again later"
- Avoid: "Exception: SocketException: Connection refused"

**Logging Strategy:**
- Use `logger` package
- Log all errors with stack traces (for debugging)
- Separate user messages from technical logs
- No PII in logs (mask API tokens, personal data)

---

## Summary of Decisions

| Area | Decision | Rationale |
|------|----------|-----------|
| Q.HOME API | ⏳ Investigation required | 3-week timeline, manual fallback for MVP |
| Background Tasks | flutter_background_fetch | Best iOS BGTaskScheduler wrapper |
| Optimization Algorithm | Greedy → LP | Start simple, evolve when needed |
| Tibber Integration | GraphQL API | Official, well-documented |
| MySkoda Integration | Reverse-engineered | Based on proven Python library |
| Weather Data | Open-Meteo | Free, accurate, no key required |
| Local Storage | Isar | Better queries, CloudKit-ready |
| State Management | Riverpod | Type-safe, testable |
| Error Handling | Result pattern | Explicit, user-friendly |

**Next Steps:**
1. Complete Q.HOME investigation (parallel track)
2. Proceed to Phase 1: Data models and contracts
3. Generate quickstart.md for developer setup
4. Begin implementation after plan approval
