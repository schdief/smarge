# Q.HOME API Contract

**Status**: 🔍 **INVESTIGATION REQUIRED** (3-week timeline)  
**Purpose**: Home battery and wallbox monitoring/control  
**Priority**: P2 (BEV control via MySkoda is P1)

---

## Investigation Plan

### Week 1: Cloud API Discovery

**Target**: https://qhome-ess-g3.q-cells.eu (remote access)

**Tasks:**
1. Login to Q.HOME web interface
2. Open browser DevTools → Network tab
3. Capture all API requests during:
   - Authentication
   - Dashboard viewing (battery SoC, solar production)
   - Wallbox status check
   - Any visible control operations
4. Document:
   - Base URL
   - Authentication mechanism
   - Endpoint patterns
   - Request/response formats

**Expected Capabilities** (Monitoring):
- ✅ Battery state of charge (kWh and %)
- ✅ Solar production (current kW)
- ✅ Grid power (import/export)
- ✅ Wallbox status (connected, charging, power)

**Likely NOT Available** (Control):
- ❌ Battery charge/discharge commands (safety concern)
- ❌ Wallbox start/stop (safety concern)

### Week 2: Local API Investigation

**Target**: `http://[inverter-ip]` (home WiFi only)

**Tasks:**
1. Find inverter IP address:
   - Check router DHCP table
   - Check Q.HOME mobile app settings
   - nmap scan local network for open ports
2. Access local web interface
3. Explore settings pages:
   - Battery management
   - Charging schedules
   - Communication settings
4. Capture API calls with DevTools
5. Test Modbus TCP:
   - Check if port 502 open (`nmap -p 502 [inverter-ip]`)
   - Use `modpoll` or `pymodbus` to scan registers
   - Identify read/write registers

**Expected Capabilities** (Control):
- ✅ Battery charging mode (auto/manual/force)
- ✅ Battery charge/discharge schedules
- ✅ Wallbox start/stop commands (maybe)
- ✅ Real-time monitoring

### Week 3: Decision & Documentation

**Outcomes:**

**Option A: API Available**
- Document authentication, endpoints, rate limits
- Implement Dart client
- Update data models if needed
- Add automated control to MVP

**Option B: No API (Manual Mode)**
- Design notification-based UX
- User manually controls via Q.HOME app
- SMARGE shows schedule + reminders
- Automated BEV control still works (MySkoda)

---

## Expected Data Models (Based on Typical Inverter APIs)

### Monitoring (Likely Available via Cloud)

```json
{
  "battery": {
    "socPercent": 65,
    "socKWh": 7.8,
    "powerKW": -2.5,
    "status": "discharging",
    "temperature": 23.5
  },
  "solar": {
    "currentPowerKW": 8.2,
    "dailyYieldKWh": 35.6
  },
  "grid": {
    "powerKW": 1.2,
    "dailyImportKWh": 5.3,
    "dailyExportKWh": 12.1
  },
  "wallbox": {
    "connected": true,
    "charging": false,
    "powerKW": 0.0,
    "sessionKWh": 0.0,
    "status": "ready"
  }
}
```

### Control (Likely Requires Local API)

**Set Battery Mode:**
```http
POST /api/battery/mode
{
  "mode": "force_charge"  // or "auto", "manual"
}
```

**Set Charging Schedule:**
```http
POST /api/battery/schedule
{
  "enabled": true,
  "periods": [
    {
      "start": "02:00",
      "end": "04:00",
      "action": "charge",
      "powerKW": 5.0
    }
  ]
}
```

**Wallbox Control:**
```http
POST /api/wallbox/start
{}

POST /api/wallbox/stop
{}
```

---

## Fallback: Manual Mode (If No API)

### UX Flow

**6:30 AM → Morning Optimization**
- ✅ App fetches battery SoC (monitoring API or manual entry)
- ✅ App calculates solar charging schedule
- ✅ User confirms schedule
- ℹ️ No automated battery control (manual via Q.HOME app if needed)

**1:30 PM → Afternoon Optimization**
- ✅ App fetches next-day prices (Tibber)
- ✅ App calculates optimal overnight charging
- ✅ Schedule includes battery + BEV charging times
- 🔔 Notification: "Optimize for tonight?"
- User taps "Optimize Now"

**Example Schedule Output:**
```
Tonight's Schedule:

BEV Charging (Automated via MySkoda):
  2:00 AM - 4:00 AM: 20 kWh @ €0.18/kWh = €3.60
  ✅ Will start automatically

Battery Charging (Manual):
  3:00 AM - 5:00 AM: 10 kWh @ €0.19/kWh = €1.90
  ⚠️ Set in Q.HOME app before bedtime
  [Open Q.HOME App] button
```

**Notification at 2:55 AM:**
```
Battery charging should start in 5 minutes
Open Q.HOME app to enable charging mode
[Open App] [Remind in 5 min] [Done]
```

### Why This is Acceptable for MVP

1. **BEV automation works** (via MySkoda) - Primary value delivered
2. **Battery optimization** is secondary (constitution principle)
3. **Proves optimization algorithm** without API dependency
4. **Can add automation** in Phase 2 after investigation
5. **User still saves money** following schedule manually

---

## Modbus TCP Reference

**If Supported:**

**Connection:**
```
Host: [inverter-ip]
Port: 502 (Modbus TCP default)
Unit ID: 1 (typical)
```

**Example Registers** (vendor-specific, must be discovered):
```
40001: Battery SoC (%)         [Read-Only]
40002: Battery Power (W)       [Read-Only]
40003: Solar Power (W)         [Read-Only]
40004: Grid Power (W)          [Read-Only]
40100: Charge Mode             [Read-Write]
       0 = Auto, 1 = Manual, 2 = Force Charge
40101: Charge Power Setpoint   [Read-Write] (W)
```

**Dart Implementation** (if Modbus available):
```dart
import 'package:modbus/modbus.dart';

class QHomeModbusClient {
  final ModbusClient _client;
  
  QHomeModbusClient(String inverterIP) 
    : _client = ModbusClient.tcp(inverterIP, port: 502);
  
  Future<double> getBatterySoC() async {
    final response = await _client.readHoldingRegisters(40001, 1);
    return response[0] / 100.0; // Convert to percentage
  }
  
  Future<void> setBatteryChargeMode(int mode) async {
    await _client.writeSingleRegister(40100, mode);
  }
}
```

---

## Next Steps

1. **Week 1**: Complete cloud API investigation
2. **Week 2**: Complete local API investigation  
3. **Week 3**: Make decision (API vs. manual mode)
4. **Update**: Revise this contract with actual findings
5. **Implement**: Based on decision, build client or manual UX

**Decision Deadline**: December 21, 2025 (before MVP implementation starts)
