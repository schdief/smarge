# Tibber API Contract

**Base URL**: `https://api.tibber.com/v1-beta/gql`  
**Protocol**: GraphQL  
**Authentication**: Bearer token (Personal Access Token)  
**Rate Limits**: Not documented (assumed generous for personal use)

---

## Authentication

### Personal Access Token

**Obtaining Token:**
1. Login to https://developer.tibber.com/
2. Navigate to "Access Token" section
3. Generate new token
4. Store in Flutter Secure Storage

**Usage:**
```http
POST https://api.tibber.com/v1-beta/gql
Authorization: Bearer {access_token}
Content-Type: application/json
```

---

## Queries

### 1. Get Viewer and Homes

**Purpose**: Fetch user info and home IDs (required for subsequent queries)

**GraphQL Query:**
```graphql
query {
  viewer {
    name
    homes {
      id
      appNickname
      address {
        address1
        postalCode
        city
        country
      }
      meteringPointData {
        consumptionEan
      }
    }
  }
}
```

**Response:**
```json
{
  "data": {
    "viewer": {
      "name": "John Doe",
      "homes": [
        {
          "id": "96a14971-525a-4420-aae9-e5aedaa129ff",
          "appNickname": "Home",
          "address": {
            "address1": "Hauptstraße 123",
            "postalCode": "12345",
            "city": "Berlin",
            "country": "DE"
          },
          "meteringPointData": {
            "consumptionEan": "123456789012345678"
          }
        }
      ]
    }
  }
}
```

**Caching**: Cache home ID indefinitely (until user changes account)

---

### 2. Get Current and Future Prices

**Purpose**: Fetch spot prices for today and tomorrow (if available)

**GraphQL Query:**
```graphql
query Prices($homeId: ID!) {
  viewer {
    home(id: $homeId) {
      currentSubscription {
        priceInfo {
          current {
            total
            energy
            tax
            startsAt
            level
          }
          today {
            total
            energy
            tax
            startsAt
            level
          }
          tomorrow {
            total
            energy
            tax
            startsAt
            level
          }
        }
      }
    }
  }
}
```

**Variables:**
```json
{
  "homeId": "96a14971-525a-4420-aae9-e5aedaa129ff"
}
```

**Response:**
```json
{
  "data": {
    "viewer": {
      "home": {
        "currentSubscription": {
          "priceInfo": {
            "current": {
              "total": 0.2845,
              "energy": 0.1823,
              "tax": 0.1022,
              "startsAt": "2025-11-30T14:00:00.000+01:00",
              "level": "NORMAL"
            },
            "today": [
              {
                "total": 0.3156,
                "energy": 0.2134,
                "tax": 0.1022,
                "startsAt": "2025-11-30T00:00:00.000+01:00",
                "level": "EXPENSIVE"
              },
              {
                "total": 0.2623,
                "energy": 0.1601,
                "tax": 0.1022,
                "startsAt": "2025-11-30T01:00:00.000+01:00",
                "level": "NORMAL"
              }
              // ... 24 hours total
            ],
            "tomorrow": [
              {
                "total": 0.1987,
                "energy": 0.0965,
                "tax": 0.1022,
                "startsAt": "2025-12-01T00:00:00.000+01:00",
                "level": "CHEAP"
              }
              // ... 24 hours (or empty if not published yet)
            ]
          }
        }
      }
    }
  }
}
```

**Field Descriptions:**
- `total`: Total price including taxes (€/kWh)
- `energy`: Energy component (spot price + markup) (€/kWh)
- `tax`: Tax component (€/kWh)
- `startsAt`: ISO 8601 timestamp (hour start)
- `level`: Price level enum (see below)

**Price Levels:**
- `VERY_CHEAP`: < 60% of average
- `CHEAP`: 60-90% of average
- `NORMAL`: 90-115% of average
- `EXPENSIVE`: 115-140% of average
- `VERY_EXPENSIVE`: > 140% of average

**Polling Strategy:**
- Today's prices: Fetch once at app startup, cache for 24h
- Tomorrow's prices: 
  - Start polling at 1:00 PM
  - Poll every 30 minutes if `tomorrow` array is empty
  - Stop polling once prices received or after 6:00 PM
  - Tibber publishes ~1:00 PM, sometimes delayed until 5:00 PM

---

### 3. Get Real-Time Consumption (Optional - Future)

**Purpose**: Live power consumption data (requires Pulse device)

**GraphQL Query:**
```graphql
query RealTimeConsumption($homeId: ID!) {
  viewer {
    home(id: $homeId) {
      currentSubscription {
        priceInfo {
          current {
            total
          }
        }
      }
      features {
        realTimeConsumptionEnabled
      }
    }
  }
}
```

**Note**: Requires Tibber Pulse hardware (not in MVP scope)

---

## WebSocket Subscription (Future Enhancement)

**Purpose**: Real-time price updates instead of polling

**Endpoint**: `wss://api.tibber.com/v1-beta/gql/subscriptions`

**Subscription:**
```graphql
subscription {
  priceUpdate {
    total
    startsAt
  }
}
```

**Not in MVP**: Use polling instead (simpler, sufficient for twice-daily optimization)

---

## Error Handling

### HTTP Errors

| Status Code | Meaning | Action |
|-------------|---------|--------|
| 200 | Success | Parse data |
| 401 | Unauthorized | Token invalid/expired → Re-authenticate |
| 403 | Forbidden | Token lacks permissions → Show error to user |
| 429 | Rate limited | Exponential backoff (wait 60s, 120s, 300s) |
| 500 | Server error | Retry with backoff |
| 503 | Service unavailable | Use cached data, retry later |

### GraphQL Errors

```json
{
  "errors": [
    {
      "message": "Field 'tomorrow' doesn't exist on type 'PriceInfo'",
      "locations": [{"line": 10, "column": 5}]
    }
  ]
}
```

**Handling:**
- Check `errors` array in response
- If `errors` exists → Log error, show user-friendly message
- If `tomorrow` query fails → Assume prices not published yet

---

## Flutter Implementation

### Dart Client Example

```dart
import 'package:graphql_flutter/graphql_flutter.dart';

class TibberClient {
  late final GraphQLClient _client;
  final String _homeId;
  
  TibberClient({
    required String accessToken,
    required String homeId,
  }) : _homeId = homeId {
    _client = GraphQLClient(
      link: AuthLink(getToken: () => 'Bearer $accessToken')
        .concat(HttpLink('https://api.tibber.com/v1-beta/gql')),
      cache: GraphQLCache(),
    );
  }
  
  Future<List<EnergyPrice>> getPrices({bool includeTomorrow = true}) async {
    const query = r'''
      query Prices($homeId: ID!) {
        viewer {
          home(id: $homeId) {
            currentSubscription {
              priceInfo {
                today {
                  total
                  energy
                  tax
                  startsAt
                  level
                }
                tomorrow {
                  total
                  energy
                  tax
                  startsAt
                  level
                }
              }
            }
          }
        }
      }
    ''';
    
    final result = await _client.query(QueryOptions(
      document: gql(query),
      variables: {'homeId': _homeId},
    ));
    
    if (result.hasException) {
      if (result.exception!.graphqlErrors.isNotEmpty) {
        throw TibberApiException(result.exception!.graphqlErrors.first.message);
      }
      throw TibberApiException('Network error: ${result.exception}');
    }
    
    return _parsePrices(result.data!);
  }
  
  List<EnergyPrice> _parsePrices(Map<String, dynamic> data) {
    final priceInfo = data['viewer']['home']['currentSubscription']['priceInfo'];
    final prices = <EnergyPrice>[];
    
    // Today's prices
    for (var price in priceInfo['today']) {
      prices.add(_parsePrice(price, isForecast: false));
    }
    
    // Tomorrow's prices (may be empty)
    if (priceInfo['tomorrow'] != null) {
      for (var price in priceInfo['tomorrow']) {
        prices.add(_parsePrice(price, isForecast: true));
      }
    }
    
    return prices;
  }
  
  EnergyPrice _parsePrice(Map<String, dynamic> json, {required bool isForecast}) {
    return EnergyPrice(
      timestamp: DateTime.parse(json['startsAt']),
      spotPrice: json['energy'],
      totalPrice: json['total'],
      feedInRate: 0.09, // Germany feed-in tariff (configure in settings)
      isForecast: isForecast,
      priceLevel: json['level'],
    );
  }
}

class TibberApiException implements Exception {
  final String message;
  TibberApiException(this.message);
  
  @override
  String toString() => 'TibberApiException: $message';
}
```

---

## Testing

### Mock Response (for unit tests)

```dart
final mockTibberResponse = {
  'data': {
    'viewer': {
      'home': {
        'currentSubscription': {
          'priceInfo': {
            'today': [
              {
                'total': 0.2845,
                'energy': 0.1823,
                'tax': 0.1022,
                'startsAt': '2025-11-30T00:00:00.000+01:00',
                'level': 'NORMAL'
              },
              // ... more hours
            ],
            'tomorrow': [] // Empty until published
          }
        }
      }
    }
  }
};
```

---

## References

- **Official Documentation**: https://developer.tibber.com/docs/overview
- **GraphQL Explorer**: https://developer.tibber.com/explorer
- **API Status**: https://status.tibber.com/
