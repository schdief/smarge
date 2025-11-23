# Pre-MVP Investigation Checklist

**Purpose:** Determine Q.HOME integration feasibility before starting iOS app development

**Timeline:** 1-3 weeks (parallel to initial setup)

---

## Week 1: Discovery

### Task 1: Access Inverter Web Interface
- [ ] Find inverter IP address (check router DHCP table or Q.HOME app settings)
- [ ] Access web interface: `http://[inverter-ip]`
- [ ] Check for login credentials (may be on inverter label or in documentation)
- [ ] Screenshot all settings pages, especially:
  - [ ] Communication settings
  - [ ] Network settings
  - [ ] API/Integration settings
  - [ ] Modbus settings (if available)

**Expected Findings:**
- Modbus TCP enabled/disabled toggle
- Port configuration (default: 502)
- Register map or documentation links

### Task 2: Reverse Engineer Q.HOME Web UI (NEW - PRIMARY FOR MONITORING)
- [ ] Access Q.HOME web interface: https://qhome-ess-g3.q-cells.eu/#/login
- [ ] Login with Q.HOME credentials
- [ ] Open browser Developer Tools (F12) → Network tab
- [ ] Navigate through web UI and capture all API calls:
  - [ ] Dashboard/overview page (battery SoC, solar, power flows)
  - [ ] Battery settings/status page
  - [ ] Wallbox/EDRIVE page (if available)
  - [ ] **Test if control actions exist** (start/stop charging buttons)
- [ ] Document all endpoints:
  - [ ] Base URL (likely `https://qhome-ess-g3.q-cells.eu/api/...` or similar)
  - [ ] Authentication method (Bearer token? Session cookie?)
  - [ ] Request headers (Authorization, Content-Type, etc.)
  - [ ] Request/response payloads (JSON structure)
- [ ] Export HAR file (HTTP Archive) for offline analysis
- [ ] Test API calls with curl/Postman to verify they work independently

**Expected Findings:**
- ✅ REST API endpoints for battery/solar/wallbox **monitoring**
- ❓ Control endpoints may NOT exist (web UI might be read-only for remote access)
- ✅ Authentication token mechanism
- ✅ Polling frequency used by web UI

**Important Note:**
- Web UI likely designed for **monitoring only** (view data remotely)
- **Control** (start/stop charging, battery config) may require:
  - Direct local inverter access (same network as device)
  - Official Q.HOME mobile app (uses different protocol)
  - MySkoda API (for BEV charging control)

**Success Criteria (Monitoring Only):**
- Can retrieve battery SoC via API call ✓
- Can retrieve solar production ✓
- Can get wallbox status (connected, charging, power) ✓
- Understand authentication flow ✓
- **Bonus:** Discover control endpoints (unlikely for remote web UI)

### Task 3: Contact Q CELLS Official Support (LOWER PRIORITY)
- [ ] Email: `sales@q-cells.com` or find developer contact
- [ ] Subject: "Developer API Access for Q.HOME+ ESS HYB-G3 Integration"
- [ ] Body template:

```
Hello Q CELLS Support Team,

I am developing a personal energy optimization application for iOS 
to maximize solar self-consumption in my household equipped with:
- Q.HOME+ ESS HYB-G3 inverter (13 kW solar, 12 kWh battery)
- Q.HOME EDRIVE wallbox (11 kW)

I would like to integrate my app with these devices to:
1. Monitor battery and BEV state of charge
2. Optimize charging schedules based on dynamic electricity pricing
3. Maximize solar power utilization

Could you please provide:
- Developer API documentation (REST, MQTT, Modbus, or similar)
- Authentication requirements
- Rate limits or usage restrictions
- Partner program information if applicable

The official Q.HOME app demonstrates this capability, so I believe 
programmatic access is possible.

Thank you for your assistance.

Best regards,
[Your Name]
[Your Q.HOME customer ID if available]
```

- [ ] Set reminder to follow up in 2 weeks if no response
- [ ] **Expectation:** Low probability of response - prioritize reverse engineering

### Task 4: MySkoda API Integration (PRIMARY FOR BEV DATA)
- [ ] Verify MySkoda app shows BEV SoC for Skoda Enyaq
- [ ] Confirm MySkoda account credentials (email + password)
- [ ] Review MySkoda open-source library: https://github.com/skodaconnect/myskoda
- [ ] Check Python library for Enyaq-specific data endpoints
- [ ] Decide: Python subprocess OR native Swift implementation

**Expected Findings:**
- BEV SoC directly from vehicle (via cellular, not wallbox)
- Charging state, power, time remaining
- Optional control: start/stop charging, set charge limit (requires S-PIN)

### Task 5: Check Inverter Local Access (FOR CONTROL CAPABILITIES)

**Purpose:** Web UI may only provide monitoring - local access needed for control

- [ ] Find inverter IP address (check router DHCP table or Q.HOME mobile app)
- [ ] Access local web interface: `http://[inverter-ip]`
- [ ] Check for login credentials (may be on inverter label, manual, or in mobile app settings)
- [ ] Compare local UI vs. cloud UI:
  - [ ] Does local UI have control buttons (start/stop charging)?
  - [ ] Can you configure battery charging schedule?
  - [ ] Can you set wallbox charging start/stop?
- [ ] Open DevTools on **local** web interface:
  - [ ] Capture control commands (if available)
  - [ ] Document local API endpoints
  - [ ] Check if authentication differs from cloud API

**Expected Findings:**
- **Local web UI** likely has full control capabilities
- **Cloud web UI** (qhome-ess-g3.q-cells.eu) likely monitoring only
- Local API may use different authentication or direct HTTP
- **Challenge:** Local API only works on same WiFi network as inverter

**Implications for SMARGE App:**
- ✅ Monitoring: Use cloud Web UI API (works from anywhere)
- ❓ Control: May require:
  - Option A: User's iPhone on same WiFi → use local inverter API
  - Option B: MySkoda API for BEV charging (works from anywhere)
  - Option C: Manual mode (user controls via Q.HOME mobile app)

### Task 6: Check Vehicle DetailsCK) (UPDATED)
- [ ] Verify MySkoda app on your iPhone shows Enyaq SoC  
- [✅] Confirmed: Q.HOME wallbox does NOT report BEV SoC
- [✅] Confirmed: BEV SoC comes from **MySkoda API** instead
- [ ] Note vehicle VIN (needed for MySkoda API calls)

---

## Week 2: Technical Testing

### Priority 1: Q.HOME Web UI Reverse Engineering (NEW - PRIMARY)

#### Analyze Captured API Calls
1. [ ] Review all endpoints captured from browser DevTools
2. [ ] Identify authentication mechanism:
   ```bash
   # Example: Test authentication
   curl -X POST https://qhome-ess-g3.q-cells.eu/api/auth/login \
     -H "Content-Type: application/json" \
     -d '{"username": "your-email", "password": "your-password"}'
   ```
3. [ ] Test data retrieval endpoints:
   ```bash
   # Example: Get battery status
   curl https://qhome-ess-g3.q-cells.eu/api/device/status \
     -H "Authorization: Bearer YOUR_TOKEN"
   ```
4. [ ] Document response structures (battery SoC, solar kW, wallbox state)
5. [ ] Test control endpoints (if available):
   - Start/stop battery charging
   - Start/stop wallbox charging
   - Set charge rate

#### Swift Implementation POC
```swift
class QHomeWebAPI {
    private let baseURL = "https://qhome-ess-g3.q-cells.eu"
    private var authToken: String?
    
    func login(email: String, password: String) async throws {
        // Replicate login flow from web UI
        let url = URL(string: "\(baseURL)/api/auth/login")!
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
        // Replicate data fetch from web UI
    }
}
```

**Success Criteria:**
- Can authenticate programmatically ✓
- Can retrieve battery SoC ✓
- Can retrieve solar production ✓
- Can control wallbox (optional for MVP) ?

---

### Priority 2: MySkoda API Testing

#### Test MySkoda Library
1. [ ] Install Python myskoda library: `pip install myskoda`
2. [ ] Test authentication:
   ```python
   from myskoda import MySkoda
   import asyncio
   
   async def test():
       api = MySkoda()
       await api.login("your-email@example.com", "your-password")
       vehicles = await api.get_vehicles()
       print(f"Found vehicles: {vehicles}")
       
       vehicle = vehicles[0]  # Your Enyaq
       status = await vehicle.get_status()
       print(f"BEV SoC: {status.battery.stateOfChargeInPercent}%")
       print(f"Charging: {status.charging.state}")
   
   asyncio.run(test())
   ```
3. [ ] Document successful response structure
4. [ ] Test polling frequency (try 15 min intervals)
5. [ ] Check for rate limiting

**Success Criteria:**
- Can authenticate with MySkoda credentials ✓
- Can retrieve BEV SoC % ✓  
- Can get charging state ✓
- Response time < 5 seconds ✓

#### Swift Implementation Decision
- [ ] Option A: Call Python script from Swift (quick, works immediately)
- [ ] Option B: Port OAuth2 flow to Swift (better long-term, more work)
- [ ] Create proof-of-concept in Swift

---

### Priority 3: Modbus Testing (FALLBACK - Only if Web UI fails)

### Option A: If Modbus TCP Available

#### Test with Modbus Client Tool
1. [ ] Download QModMaster (cross-platform, free): https://github.com/zhaoshijian/qmodmaster
   - Or ModbusPal, ModScan, etc.
2. [ ] Configure connection:
   - IP: [inverter IP from Week 1]
   - Port: 502 (default Modbus TCP)
   - Unit ID: 1 (typical default)
3. [ ] Test read operations:
   ```
   Function Code: 03 (Read Holding Registers)
   Start Address: 0
   Quantity: 10
   ```
4. [ ] Document any responses (screenshot)
5. [ ] Search for register map:
   - [ ] Check inverter manual/documentation
   - [ ] Search online: "Q.HOME+ ESS Modbus register map"
   - [ ] Try common SunSpec register addresses (40000-40xxx range)

#### Swift Modbus Library Research
- [ ] Test SwiftModbus: https://github.com/ABTech/SwiftModbus
- [ ] Or alternative: Create simple TCP socket connection test
- [ ] Create proof-of-concept: Read single register value

**Success Criteria:**
- Can establish TCP connection to port 502
- Can read at least one register value
- Found or discovered register addresses for battery/BEV data

### Option B: If No Modbus (Fallback Plan)

#### Design Manual Mode MVP
- [ ] Sketch UI for manual SoC entry
- [ ] Define notification timing: "⏰ Please enter current BEV SoC from Q.HOME app"
- [ ] Design schedule presentation without automated control
- [ ] Plan user workflow:
  1. App calculates optimal schedule
  2. App shows: "Start charging BEV at 11:30 PM (22¢/kWh)"
  3. User receives notification at 11:15 PM
  4. User opens official Q.HOME app and starts charging manually

**Success Criteria:**
- Clear user flow documented
- Still provides optimization value
- User accepts manual interaction requirement

---

## Week 3: Decision & Documentation

### Integration Strategy Decision

Create decision document answering:

1. **What integration is available?**
   - [✅] **MySkoda API** for BEV SoC (confirmed available)
   - [ ] **Q.HOME Web UI API** (cloud) for battery/solar **monitoring** (reverse engineering)
   - [ ] **Q.HOME Local API** (inverter) for battery/wallbox **control** (requires same WiFi)
   - [ ] Modbus TCP for battery/solar (fallback)
   - [ ] Official Q.HOME API (unlikely, low priority)
   - [ ] Manual mode for control (last resort)

2. **What data can be accessed?**
   - [✅] BEV SoC (automated via MySkoda API)
   - [ ] Battery SoC (automated via Cloud/Local Web UI OR Modbus OR manual)
   - [ ] Solar production (automated via Cloud/Local Web UI OR Modbus OR manual)
   - [ ] Wallbox status - connected/charging (automated via Cloud/Local Web UI OR manual)
   - [ ] Current power flows (automated via Cloud/Local Web UI OR Modbus OR manual)

3. **What control is available?**
   - [ ] **BEV charging** (MySkoda API with S-PIN OR manual via MySkoda app)
   - [ ] **Wallbox control** (Local inverter API OR manual via Q.HOME app)
   - [ ] **Battery schedule** (Local inverter API OR manual via Q.HOME app)
   - ❌ **Cloud Web UI control** (likely not available - monitoring only)

4. **MVP Approach:**

   **Monitoring (Data Collection):**
   - [ ] **Option 1**: Cloud Web UI API (BEST - works from anywhere)
   - [ ] **Option 2**: Local inverter API (GOOD - requires home WiFi)
   - [ ] **Option 3**: Modbus (ACCEPTABLE - requires home WiFi, complex)
   - [ ] **Option 4**: Manual entry (FALLBACK)

   **Control (Charging Commands):**
   - [ ] **Option A**: MySkoda API for BEV + Manual for battery/wallbox (REALISTIC MVP)
   - [ ] **Option B**: MySkoda + Local inverter API (GOOD - requires home WiFi)
   - [ ] **Option C**: Full manual mode (FALLBACK - notifications only)

**Recommended MVP Strategy:**

**Phase 1 (Working from anywhere):**
- ✅ MySkoda API: BEV SoC + charging control (works remotely)
- ✅ Cloud Web UI API: Battery SoC + solar monitoring (works remotely)
- 📱 Manual control: User uses Q.HOME app for battery (acceptable)
- 🔔 Notifications: "Start BEV charging now" (app does it via MySkoda)
- 🔔 Notifications: "Charge battery now" (user does it via Q.HOME app)

**Phase 2 (Home WiFi optimization):**
- ⚡ Local inverter API: Battery + wallbox control (when at home)
- 🏠 Detect home WiFi → enable local control
- 🌍 Away from home → monitoring only + MySkoda control

**Value Proposition Still Strong:**
- App calculates **optimal schedule** (always works)
- App shows **when to charge** (always works)
- App sends **timely notifications** (always works)
- MySkoda controls BEV automatically (works remotely)
- Battery/wallbox via app or manual (phase 2: automated at home)

### Update Specification

Based on findings, update `specification.md`:
- [ ] Replace "conceptual" API sections with actual endpoints/registers
- [ ] Document exact integration method
- [ ] Update requirements to match available functionality
- [ ] Adjust user stories for manual steps if needed
- [ ] Update architecture diagram

### Risk Assessment

Document risks and mitigation:

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| No API available | Medium | High | Manual mode still provides value |
| Modbus read-only | Medium | Medium | User controls via Q.HOME app |
| Official API delayed | High | Low | Start with other features |
| Register map unknown | Medium | Medium | Trial-and-error discovery |
| Write commands unsafe | High | High | Start read-only, validate thoroughly |

---

## Go/No-Go Decision

**Proceed with MVP if:**
- ✅ Can optimize schedule (always yes - just needs prices + solar forecast)
- ✅ Can get BEV/battery SoC (automated OR manual entry acceptable)
- ✅ User willing to interact twice daily (already confirmed)

**MVP Value Proposition Holds Even in Manual Mode:**
- App calculates optimal charging times → saves money
- User gets notifications at right times → reduces mental load
- Solar forecast integration → maximizes self-consumption
- Price visibility → better decisions

**MVP is viable with ANY of the integration outcomes.**

---

## Parallel Work (While Investigating)

Don't wait - start these in parallel:

1. ✅ Set up Xcode project
2. ✅ Implement Tibber API integration (fully documented)
3. ✅ Implement Open-Meteo weather API (free, no key)
4. ✅ Build basic UI (dashboard, settings)
5. ✅ Create SwiftData models (already defined)
6. ✅ Implement optimization algorithm with mock device data
7. ✅ Set up background tasks
8. ✅ Implement notifications

**Integration can be swapped in later - core value is optimization algorithm.**

---

## Appendix: Resources

### Modbus Resources
- Modbus TCP Specification: https://modbus.org/docs/Modbus_Messaging_Implementation_Guide_V1_0b.pdf
- SunSpec Alliance (solar standard): https://sunspec.org/
- SwiftModbus Library: https://github.com/ABTech/SwiftModbus

### Q CELLS Resources
- Q.HOME App: https://apps.apple.com/de/app/q-home/id1491103343
- Q CELLS Support: https://www.q-cells.com/support
- Q CELLS Germany: https://www.q-cells.de/

### Alternative Investigation
- Home Assistant Q CELLS integration search
- GitHub search: "q.home" OR "qcells" language:Swift
- Reddit: r/homeassistant, r/solar

---

**Next Step:** Begin Week 1 tasks immediately while setting up development environment.
