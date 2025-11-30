# Open-Meteo Weather API Contract

**Base URL**: `https://api.open-meteo.com/v1`  
**Protocol**: REST (JSON)  
**Authentication**: None required (free, open-source)  
**Rate Limits**: None for personal use

---

## Endpoint: Forecast

**Purpose**: Get weather forecast including solar radiation data

```http
GET /v1/forecast
  ?latitude={lat}
  &longitude={lon}
  &hourly=direct_radiation,diffuse_radiation,cloud_cover,temperature_2m
  &forecast_days=2
  &timezone=auto
```

**Parameters:**
- `latitude`: Location latitude (e.g., 52.5200 for Berlin)
- `longitude`: Location longitude (e.g., 13.4050)
- `hourly`: Comma-separated variables to fetch
- `forecast_days`: Number of days (1-16, use 2 for 48h optimization window)
- `timezone`: `auto` or specific (e.g., `Europe/Berlin`)

**Response:**
```json
{
  "latitude": 52.52,
  "longitude": 13.41,
  "generationtime_ms": 0.123,
  "utc_offset_seconds": 3600,
  "timezone": "Europe/Berlin",
  "hourly_units": {
    "time": "iso8601",
    "direct_radiation": "W/m²",
    "diffuse_radiation": "W/m²",
    "cloud_cover": "%",
    "temperature_2m": "°C"
  },
  "hourly": {
    "time": [
      "2025-11-30T00:00",
      "2025-11-30T01:00",
      ...
    ],
    "direct_radiation": [0, 0, 0, 145, 328, 412, 456, ...],
    "diffuse_radiation": [0, 0, 0, 56, 98, 134, 167, ...],
    "cloud_cover": [75, 80, 65, 40, 20, 10, 5, ...],
    "temperature_2m": [4.2, 3.8, 3.5, 5.1, 7.8, ...]
  }
}
```

**Solar Production Calculation:**
```dart
double estimateSolarProduction({
  required double directRadiation,  // W/m²
  required double diffuseRadiation, // W/m²
  required double systemCapacityKW, // 13.0 kW
}) {
  const pvEfficiency = 0.85;  // 85% system efficiency
  const derateFactor = 0.80;  // 20% conservative buffer
  
  final totalRadiation = directRadiation + diffuseRadiation;
  final theoreticalPower = (totalRadiation / 1000) * systemCapacityKW;
  
  return theoreticalPower * pvEfficiency * derateFactor;
}
```

**Polling Strategy:**
- Fetch when optimization runs (2x daily)
- Cache for 1 hour
- Re-fetch if forecast changes >20% (check generationtime_ms)

---

## Flutter Implementation

```dart
import 'package:dio/dio.dart';

class OpenMeteoClient {
  final Dio _dio;
  final double latitude;
  final double longitude;
  
  OpenMeteoClient({
    required this.latitude,
    required this.longitude,
  }) : _dio = Dio(BaseOptions(
         baseUrl: 'https://api.open-meteo.com/v1',
       ));
  
  Future<List<SolarForecast>> getSolarForecast() async {
    final response = await _dio.get('/forecast', queryParameters: {
      'latitude': latitude,
      'longitude': longitude,
      'hourly': 'direct_radiation,diffuse_radiation,cloud_cover',
      'forecast_days': 2,
      'timezone': 'auto',
    });
    
    return _parseForecast(response.data);
  }
  
  List<SolarForecast> _parseForecast(Map<String, dynamic> data) {
    final times = data['hourly']['time'] as List;
    final direct = data['hourly']['direct_radiation'] as List;
    final diffuse = data['hourly']['diffuse_radiation'] as List;
    final cloudCover = data['hourly']['cloud_cover'] as List;
    
    final forecasts = <SolarForecast>[];
    for (int i = 0; i < times.length; i++) {
      final directRad = direct[i] as num;
      final diffuseRad = diffuse[i] as num;
      
      forecasts.add(SolarForecast(
        timestamp: DateTime.parse(times[i]),
        forecastKW: _estimateProduction(
          directRad.toDouble(),
          diffuseRad.toDouble(),
        ),
        confidence: _calculateConfidence(cloudCover[i] as int),
        weatherCondition: _getCondition(cloudCover[i] as int),
        cloudCoverPercent: cloudCover[i] as int,
        directRadiation: directRad.toDouble(),
        diffuseRadiation: diffuseRad.toDouble(),
      ));
    }
    
    return forecasts;
  }
  
  double _estimateProduction(double direct, double diffuse) {
    const systemCapacity = 13.0; // kW
    const pvEfficiency = 0.85;
    const derateFactor = 0.80;
    
    final totalRadiation = direct + diffuse;
    final theoretical = (totalRadiation / 1000) * systemCapacity;
    
    return theoretical * pvEfficiency * derateFactor;
  }
  
  double _calculateConfidence(int cloudCover) {
    if (cloudCover <= 20) return 0.95;
    if (cloudCover <= 50) return 0.80;
    if (cloudCover <= 75) return 0.60;
    return 0.40;
  }
  
  String _getCondition(int cloudCover) {
    if (cloudCover <= 20) return 'sunny';
    if (cloudCover <= 50) return 'partly_cloudy';
    if (cloudCover <= 75) return 'cloudy';
    return 'overcast';
  }
}
```

---

## References

- **Documentation**: https://open-meteo.com/en/docs
- **Source Code**: https://github.com/open-meteo/open-meteo
- **License**: Open-source, free for personal and commercial use
