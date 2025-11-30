# MySkoda API Contract

**Base URL**: `https://mysmob.api.connect.skoda-auto.cz`  
**Protocol**: REST (JSON)  
**Authentication**: OAuth 2.0 (VW Group Identity)  
**Status**: ⚠️ Unofficial (reverse-engineered from MySkoda app)  
**Reference**: [myskoda Python library](https://github.com/skodaconnect/myskoda)

---

## Authentication

### OAuth 2.0 Flow

**Provider**: VW Group Identity Server  
**Auth URL**: `https://identity.vwgroup.io/oidc/v1/authorize`  
**Token URL**: `https://identity.vwgroup.io/oidc/v1/token`

**Step 1: Authorization Request**
```http
GET https://identity.vwgroup.io/oidc/v1/authorize
  ?client_id=7f045eee-7003-4379-9968-9355ed2adb06@apps_vw-dilab_com
  &redirect_uri=skodaconnect://oidc.login/
  &response_type=code
  &scope=openid%20profile%20email%20mbb%20cars%20dealers
  &state={random_state}
  &nonce={random_nonce}
```

**Step 2: User Login** (handled in WebView or browser)
- User enters email + password
- Receives authorization code

**Step 3: Exchange Code for Token**
```http
POST https://identity.vwgroup.io/oidc/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code={authorization_code}
&redirect_uri=skodaconnect://oidc.login/
&client_id=7f045eee-7003-4379-9968-9355ed2adb06@apps_vw-dilab_com
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Step 4: Refresh Token (when access token expires)**
```http
POST https://identity.vwgroup.io/oidc/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token={refresh_token}
&client_id=7f045eee-7003-4379-9968-9355ed2adb06@apps_vw-dilab_com
```

---

## Endpoints

### 1. List Vehicles

**Purpose**: Get user's vehicles and VINs

```http
GET /api/v1/garage
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "vehicles": [
    {
      "vin": "TMBJR1NV9N1234567",
      "name": "Enyaq",
      "model": "ENYAQ iV 80",
      "registrationDate": "2023-05-15",
      "specification": {
        "battery": {
          "capacityInKWh": 82
        }
      }
    }
  ]
}
```

---

### 2. Get Vehicle Status

**Purpose**: Fetch current BEV state (SoC, range, charging status)

```http
GET /api/v2/vehicle-status/{vin}
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "vin": "TMBJR1NV9N1234567",
  "state": "ACTIVE",
  "battery": {
    "stateOfChargeInPercent": 65,
    "remainingRangeInKm": 280,
    "stateOfChargeInKWh": 53.3
  },
  "charging": {
    "state": "CONNECT_CABLE",
    "chargingType": "AC",
    "chargePowerInKw": 0.0,
    "chargeRateInKmPerHour": 0,
    "remainingTimeToFullyChargedInMinutes": null,
    "targetStateOfChargeInPercent": 80,
    "chargeMode": "MANUAL"
  },
  "plug": {
    "connectionState": "CONNECTED",
    "lockState": "LOCKED"
  },
  "dataTimestamp": "2025-11-30T14:35:12Z"
}
```

**Charging States:**
- `READY_FOR_CHARGING`: Plugged in, not charging
- `CONNECT_CABLE`: Not plugged in
- `CHARGING`: Actively charging
- `CONSERVING`: Charging paused (conservation mode)
- `ERROR`: Charging error

**Field Descriptions:**
- `stateOfChargeInPercent`: Current battery % (0-100)
- `remainingRangeInKm`: Estimated range (WLTP)
- `chargePowerInKw`: Current charge rate (0 if not charging)
- `remainingTimeToFullyChargedInMinutes`: null if not charging
- `targetStateOfChargeInPercent`: User-set charge limit in MySkoda app
- `dataTimestamp`: When vehicle last reported (may be stale if vehicle asleep)

**Data Freshness:**
- Vehicle reports every 15-30 minutes when awake
- May be hours old if vehicle in deep sleep
- Check `dataTimestamp` to determine freshness

---

### 3. Start Charging

**Purpose**: Initiate BEV charging session

**⚠️ Requires S-PIN** (4-digit security code from MySkoda app settings)

```http
POST /api/v1/charging/{vin}/start
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "targetSOCInPercent": 80
}
```

**Response:**
```json
{
  "status": "IN_PROGRESS",
  "requestId": "abc123"
}
```

**Notes:**
- `targetSOCInPercent` is optional (uses current MySkoda app setting if omitted)
- Command is queued, not instant (vehicle must wake up)
- Check status with polling (see section 6)

---

### 4. Stop Charging

**Purpose**: Stop ongoing charging session

**⚠️ Requires S-PIN**

```http
POST /api/v1/charging/{vin}/stop
Authorization: Bearer {access_token}
Content-Type: application/json

{}
```

**Response:**
```json
{
  "status": "IN_PROGRESS",
  "requestId": "def456"
}
```

---

### 5. Set Charge Limit

**Purpose**: Change target SoC for charging

**⚠️ Requires S-PIN**

```http
PUT /api/v1/charging/{vin}/set-charge-limit
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "targetSOCInPercent": 80
}
```

**Response:**
```json
{
  "status": "IN_PROGRESS",
  "requestId": "ghi789"
}
```

**Valid Values**: 50, 60, 70, 80, 90, 100 (MySkoda app only allows these increments)

---

### 6. Check Operation Status

**Purpose**: Verify if command (start/stop/set-limit) succeeded

```http
GET /api/v1/operations/{requestId}
Authorization: Bearer {access_token}
```

**Response (In Progress):**
```json
{
  "id": "abc123",
  "operation": "START_CHARGING",
  "status": "IN_PROGRESS",
  "vehicleVin": "TMBJR1NV9N1234567"
}
```

**Response (Completed):**
```json
{
  "id": "abc123",
  "operation": "START_CHARGING",
  "status": "COMPLETED_SUCCESS",
  "vehicleVin": "TMBJR1NV9N1234567",
  "completedAt": "2025-11-30T14:37:45Z"
}
```

**Response (Failed):**
```json
{
  "id": "abc123",
  "operation": "START_CHARGING",
  "status": "COMPLETED_ERROR",
  "vehicleVin": "TMBJR1NV9N1234567",
  "error": {
    "code": "VEHICLE_NOT_REACHABLE",
    "message": "Vehicle is in deep sleep"
  }
}
```

**Status Values:**
- `IN_PROGRESS`: Command sent to vehicle, waiting for confirmation
- `COMPLETED_SUCCESS`: Command executed successfully
- `COMPLETED_ERROR`: Command failed
- `TIMEOUT`: Vehicle did not respond in time

**Polling Strategy:**
- Poll every 5 seconds for first minute
- Then every 30 seconds for up to 5 minutes
- Timeout after 5 minutes → Assume failure

---

## Error Handling

### HTTP Errors

| Status Code | Meaning | Action |
|-------------|---------|--------|
| 200 | Success | Parse response |
| 401 | Unauthorized | Refresh access token |
| 403 | Forbidden | S-PIN invalid or missing → Prompt user |
| 404 | Not found | VIN invalid → Check configuration |
| 429 | Rate limited | Back off, retry after 60s |
| 500 | Server error | Retry with exponential backoff |
| 502/503 | Service unavailable | Use cached data, retry later |

### API-Specific Errors

```json
{
  "error": {
    "code": "VEHICLE_NOT_REACHABLE",
    "message": "The vehicle is currently not reachable"
  }
}
```

**Common Error Codes:**
- `VEHICLE_NOT_REACHABLE`: Vehicle in deep sleep or no cellular connection
- `INVALID_S_PIN`: S-PIN incorrect or missing
- `OPERATION_NOT_SUPPORTED`: Vehicle doesn't support this feature
- `CHARGING_NOT_POSSIBLE`: Vehicle not plugged in
- `BATTERY_FULL`: Already at target SoC

---

## S-PIN Management

### What is S-PIN?

- 4-digit security code (like ATM PIN)
- Set by user in MySkoda mobile app (Settings → Security → S-PIN)
- Required for control operations (start/stop charging, lock/unlock doors, etc.)
- NOT required for read-only operations (get status)

### How to Use

**Option 1: Prompt user each time** (More secure)
```dart
final sPin = await showDialog<String>(
  context: context,
  builder: (context) => SPinDialog(),
);

await mySkodaClient.startCharging(vin, sPin: sPin);
```

**Option 2: Store encrypted** (More convenient, less secure)
```dart
// Store in Flutter Secure Storage
await secureStorage.write(key: 'myskoda_spin', value: userEnteredSPin);

// Retrieve when needed
final sPin = await secureStorage.read(key: 'myskoda_spin');
await mySkodaClient.startCharging(vin, sPin: sPin);
```

**Recommendation**: Option 2 (store encrypted) for MVP
- User enters S-PIN once during onboarding
- Encrypted in Flutter Secure Storage
- Used automatically for charging commands
- Show warning: "S-PIN is stored securely on this device only"

---

## Polling Strategy

### Status Updates

**Scenario 1: User opens app**
- Fetch vehicle status immediately
- Cache for 15 minutes
- Show last update timestamp

**Scenario 2: Optimization runs**
- Fetch vehicle status for current SoC
- If data stale (>30 min) → Wake vehicle (POST /api/v1/wakeup/{vin})
- Wait up to 2 minutes for fresh data
- Fallback to cached data if wake fails

**Scenario 3: After sending command**
- Poll operation status every 5 seconds
- Update UI when status changes
- Timeout after 5 minutes

### Rate Limiting

**Conservative Approach** (avoid API abuse):
- Max 1 request per 15 minutes for status updates
- Max 1 wake command per hour
- Cache aggressively
- Show "last updated" timestamp to user

---

## Flutter Implementation

```dart
import 'package:dio/dio.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class MySkodaClient {
  final Dio _dio;
  final FlutterSecureStorage _secureStorage;
  
  String? _accessToken;
  DateTime? _tokenExpiry;
  
  MySkodaClient({
    required FlutterSecureStorage secureStorage,
  }) : _secureStorage = secureStorage,
       _dio = Dio(BaseOptions(
         baseUrl: 'https://mysmob.api.connect.skoda-auto.cz',
       ));
  
  Future<void> authenticate(String email, String password) async {
    // Step 1: Get authorization code (simplified - actual flow uses WebView)
    // ... (OAuth flow implementation)
    
    // Step 2: Exchange for token
    final response = await _dio.post(
      'https://identity.vwgroup.io/oidc/v1/token',
      data: {
        'grant_type': 'authorization_code',
        'code': authCode,
        'redirect_uri': 'skodaconnect://oidc.login/',
        'client_id': '7f045eee-7003-4379-9968-9355ed2adb06@apps_vw-dilab_com',
      },
      options: Options(contentType: Headers.formUrlEncodedContentType),
    );
    
    _accessToken = response.data['access_token'];
    _tokenExpiry = DateTime.now().add(Duration(seconds: response.data['expires_in']));
    
    // Store refresh token
    await _secureStorage.write(
      key: 'myskoda_refresh_token',
      value: response.data['refresh_token'],
    );
  }
  
  Future<void> _ensureValidToken() async {
    if (_accessToken == null || DateTime.now().isAfter(_tokenExpiry!)) {
      final refreshToken = await _secureStorage.read(key: 'myskoda_refresh_token');
      if (refreshToken == null) throw Exception('Not authenticated');
      
      // Refresh token
      final response = await _dio.post(
        'https://identity.vwgroup.io/oidc/v1/token',
        data: {
          'grant_type': 'refresh_token',
          'refresh_token': refreshToken,
          'client_id': '7f045eee-7003-4379-9968-9355ed2adb06@apps_vw-dilab_com',
        },
        options: Options(contentType: Headers.formUrlEncodedContentType),
      );
      
      _accessToken = response.data['access_token'];
      _tokenExpiry = DateTime.now().add(Duration(seconds: response.data['expires_in']));
    }
  }
  
  Future<VehicleStatus> getVehicleStatus(String vin) async {
    await _ensureValidToken();
    
    final response = await _dio.get(
      '/api/v2/vehicle-status/$vin',
      options: Options(headers: {'Authorization': 'Bearer $_accessToken'}),
    );
    
    return VehicleStatus.fromJson(response.data);
  }
  
  Future<String> startCharging(String vin, {int? targetSoC}) async {
    await _ensureValidToken();
    
    final sPin = await _secureStorage.read(key: 'myskoda_spin');
    if (sPin == null) throw Exception('S-PIN not configured');
    
    final response = await _dio.post(
      '/api/v1/charging/$vin/start',
      data: targetSoC != null ? {'targetSOCInPercent': targetSoC} : {},
      options: Options(headers: {
        'Authorization': 'Bearer $_accessToken',
        'X-S-PIN': sPin,
      }),
    );
    
    return response.data['requestId'];
  }
  
  Future<OperationStatus> checkOperationStatus(String requestId) async {
    await _ensureValidToken();
    
    final response = await _dio.get(
      '/api/v1/operations/$requestId',
      options: Options(headers: {'Authorization': 'Bearer $_accessToken'}),
    );
    
    return OperationStatus.fromJson(response.data);
  }
}

class VehicleStatus {
  final String vin;
  final int socPercent;
  final double? socKWh;
  final int rangeKm;
  final String chargingState;
  final double? chargePowerKW;
  final int? minutesToFull;
  final int? targetSoC;
  final DateTime dataTimestamp;
  
  VehicleStatus({
    required this.vin,
    required this.socPercent,
    this.socKWh,
    required this.rangeKm,
    required this.chargingState,
    this.chargePowerKW,
    this.minutesToFull,
    this.targetSoC,
    required this.dataTimestamp,
  });
  
  factory VehicleStatus.fromJson(Map<String, dynamic> json) {
    return VehicleStatus(
      vin: json['vin'],
      socPercent: json['battery']['stateOfChargeInPercent'],
      socKWh: json['battery']['stateOfChargeInKWh'],
      rangeKm: json['battery']['remainingRangeInKm'],
      chargingState: json['charging']['state'],
      chargePowerKW: json['charging']['chargePowerInKw'],
      minutesToFull: json['charging']['remainingTimeToFullyChargedInMinutes'],
      targetSoC: json['charging']['targetStateOfChargeInPercent'],
      dataTimestamp: DateTime.parse(json['dataTimestamp']),
    );
  }
  
  bool get isCharging => chargingState == 'CHARGING';
  bool get isPluggedIn => chargingState != 'CONNECT_CABLE';
  bool get isDataFresh => DateTime.now().difference(dataTimestamp).inMinutes < 30;
}
```

---

## Testing

### Mock Responses

```dart
final mockVehicleStatus = {
  'vin': 'TMBJR1NV9N1234567',
  'state': 'ACTIVE',
  'battery': {
    'stateOfChargeInPercent': 65,
    'remainingRangeInKm': 280,
    'stateOfChargeInKWh': 53.3,
  },
  'charging': {
    'state': 'READY_FOR_CHARGING',
    'chargingType': 'AC',
    'chargePowerInKw': 0.0,
    'chargeRateInKmPerHour': 0,
    'targetStateOfChargeInPercent': 80,
    'chargeMode': 'MANUAL',
  },
  'plug': {
    'connectionState': 'CONNECTED',
    'lockState': 'LOCKED',
  },
  'dataTimestamp': '2025-11-30T14:35:12Z',
};
```

---

## References

- **myskoda Python Library**: https://github.com/skodaconnect/myskoda
- **Home Assistant Integration**: https://github.com/skodaconnect/homeassistant-myskoda
- **Reverse Engineering Notes**: https://github.com/tomansill/skoda-api-docs (community)

**⚠️ Disclaimer**: This is an unofficial API reverse-engineered from the MySkoda mobile app. Skoda/VW Group may change the API at any time without notice.
