# Q.HOME+ ESS HYB-G3 & Q.HOME EDRIVE Integration Research

**Date**: November 23, 2025  
**Purpose**: Determine integration approach for Swift iOS app  
**Devices**:
- Q.HOME+ ESS HYB-G3 (13 kW solar inverter with battery)
- Q.HOME EDRIVE (11 kW wallbox)

---

## Executive Summary

**Current Status**: ⚠️ **No official public API available**

Q.HOME devices (manufactured by Hanwha Q CELLS) do **not** provide publicly documented APIs for third-party developers. The official Q.HOME app (iOS/Android) uses proprietary communication protocols that are not published in developer documentation.

**Recommended Approach**: Local network reverse engineering or waiting for official API release.

---

## Research Findings

### 1. Official API Status

**Finding**: ❌ **No public developer API**

- **Q CELLS Website**: No developer portal or API documentation found
  - Visited: https://www.q-cells.eu/, https://www.q-cells.de/, https://www.q-cells.com/
  - Product pages exist but no developer/API section
- **Official App**: Q.HOME app (App Store ID: 1491103343)
  - App can monitor and control devices
  - Uses proprietary communication (likely cloud-based + local network)
- **Developer Documentation**: None found on Hanwha Q CELLS or Q.HOME websites
- **Contact Attempts**: Would need to contact Q CELLS technical support or sales to inquire about partner API access

**Conclusion**: Hanwha Q CELLS has not made a public API available for third-party developers.

---

### 2. Communication Protocols

**Known Information**:

The Q.HOME+ ESS HYB-G3 inverter likely supports one or more of these standard protocols:

#### Common Solar Inverter Protocols:
1. **Modbus TCP** (most common for commercial inverters)
   - Standard protocol for industrial devices
   - Typically accessible via local network on port 502
   - Requires knowledge of register mappings
   
2. **SunSpec Protocol**
   - Industry standard for solar equipment interoperability
   - Built on top of Modbus
   - Standardized register layout (if supported)
   - Many Hanwha Q CELLS inverters support SunSpec

3. **REST API** (local or cloud)
   - Some modern inverters provide local HTTP/HTTPS API
   - Q.HOME app likely uses this
   
4. **MQTT** (less common for inverters, more for IoT)
   - Unlikely primary protocol but possible for cloud communication

**Most Likely**: The inverter supports **Modbus TCP** and possibly **SunSpec** for local network access. The Q.HOME app probably uses a proprietary REST API (either cloud-based or local network) that is not documented.

---

### 3. Community & Open-Source Projects

**Search Results**: Limited community activity

#### Home Assistant Integration:
- **Status**: ❌ No official Home Assistant integration found for Q.HOME+ ESS
- **Generic Solar Integrations**: Home Assistant has Modbus and SunSpec integrations that *might* work if protocol is supported
- **Alternative**: Some users may have created custom YAML configurations (would need to search Home Assistant forums)

#### GitHub Projects:
- **Direct Q.HOME Projects**: ⚠️ No significant GitHub repositories found specifically for Q.HOME integration
- **Similar Projects**: Some reverse engineering projects exist for other solar inverters (SolarEdge, Fronius, Growatt)
- **Hanwha/Q-CELLS**: No official Hanwha Q CELLS GitHub organization with public APIs

#### Forums & Communities:
- **photovoltaikforum.com**: German solar forum - might have users discussing Q.HOME integration
- **reddit.com/r/homeassistant**: Some discussions about integrating solar inverters
- **energyhub.org**: Technical discussions about solar systems

**Conclusion**: Limited community reverse engineering work. Would need to pioneer this integration.

---

### 4. Wallbox (Q.HOME EDRIVE) Integration

**Separate or Unified?**: Likely **both**

The Q.HOME EDRIVE wallbox is designed to work within the Q.HOME ecosystem:
- **Through Inverter**: Wallbox likely reports to inverter, which aggregates data
- **Direct API**: Wallbox may also have its own API/endpoint
- **Q.HOME App**: Controls both devices, suggesting unified backend

**Wallbox Protocols**:
- Most EV wallboxes support: OCPP (Open Charge Point Protocol), Modbus TCP, or proprietary REST APIs
- Q.HOME EDRIVE specification unclear (not publicly documented)

---

### 5. Typical Solar Inverter Integration Approaches

Based on industry practices for similar devices:

#### Approach A: Official Partnership
- Contact Q CELLS for partner/developer API access
- Pros: Official support, stable API, documentation
- Cons: May require business agreement, NDA, limited to Q CELLS roadmap
- Timeline: Weeks to months for approval

#### Approach B: Local Network Protocol (Modbus/SunSpec)
- Connect to inverter on local network
- Use Modbus TCP library (port 502)
- Read standard SunSpec registers if supported
- Pros: No cloud dependency, real-time data, no auth needed (local only)
- Cons: Requires register mapping knowledge, read-only (control may not work), user must have inverter on network
- Timeline: 1-2 weeks to implement and test

#### Approach C: Reverse Engineering Q.HOME App
- Intercept network traffic from official Q.HOME app
- Identify API endpoints and authentication
- Replicate in Swift app
- Pros: Full feature parity with official app
- Cons: Against ToS, API may change, authentication challenges, legal/ethical concerns
- Timeline: 1-4 weeks, high maintenance risk

#### Approach D: Cloud API Discovery
- Check if Q.HOME has undocumented cloud API
- Use browser dev tools when accessing Q.HOME web portal (if exists)
- Pros: May discover official endpoints
- Cons: Still unofficial, may be blocked
- Timeline: Days to discover, ongoing risk

---

## Recommended Integration Strategy

### Phase 1: Verification & Discovery (1-2 weeks)

1. **Check Modbus/SunSpec Support**
   - Access inverter web interface (usually at http://[inverter-ip])
   - Check settings for Modbus TCP enable/disable
   - Test connection using free Modbus tools (ModScan, QModMaster)
   - If SunSpec: Use standard register layout
   - **Swift Libraries**: 
     - SwiftModbus (https://github.com/carabina/SwiftModbus)
     - Or use C library bindings (libmodbus)

2. **Contact Q CELLS Support**
   - Email: sales@q-cells.com, technical support
   - Ask: "Do you provide API access for third-party app developers?"
   - Ask: "Does Q.HOME+ ESS HYB-G3 support Modbus TCP or SunSpec?"
   - Request: Developer documentation or partner program information

3. **Check for Web Interface**
   - Most inverters have local web UI
   - Access at `http://192.168.1.X` (scan local network)
   - Inspect browser network tab for API calls
   - May reveal REST API endpoints

### Phase 2: Implementation Options

#### Option 1: Modbus TCP Integration (RECOMMENDED)
**If inverter supports Modbus/SunSpec**:
- Use Swift Modbus library
- Read real-time data: SoC, power, voltage, current
- Monitor only (no control) initially
- Control functions may require authentication

**Implementation**:
```swift
// Example using Modbus TCP
import SwiftModbus

class QHomeModbusClient {
    let client = ModbusClient(host: "192.168.1.100", port: 502)
    
    func readBatterySOC() async throws -> Double {
        // Read SunSpec register 0x9C58 (example)
        let registers = try await client.readHoldingRegisters(address: 0x9C58, count: 1)
        return Double(registers[0]) / 100.0
    }
}
```

#### Option 2: Wait for Official API
- Continue with mock data in app
- Use dummy/simulated Q.HOME responses
- Replace with real API when available

#### Option 3: Hybrid Approach
- Implement Modbus for read-only monitoring
- Use manual controls (buttons in app) that user performs in official app
- Document that control features require official API

---

## Swift iOS Implementation Considerations

### Network Requirements
- **Local Network**: App needs local network permission (iOS 14+)
  - Add `NSLocalNetworkUsageDescription` to Info.plist
  - Request permissions before connecting to inverter
  
- **Background Access**: Limited
  - Background URLSession for cloud APIs
  - Local network access restricted in background

### Libraries for Swift
1. **Modbus TCP**: 
   - SwiftModbus (may need updates)
   - Custom TCP socket implementation
   
2. **REST API** (if discovered):
   - URLSession (built-in)
   - Alamofire (optional, popular)

3. **Network Discovery**:
   - Bonjour/mDNS for finding devices
   - Network framework for scanning

### Security Considerations
- Store API credentials in Keychain (if cloud API)
- HTTPS only for cloud endpoints
- Local network: verify device identity (certificate pinning if available)

---

## Known Limitations & Challenges

### Integration Challenges
1. **No Official Documentation**: Blind development without specs
2. **Authentication**: Cloud API likely requires credentials from Q.HOME app
3. **Rate Limiting**: Unknown limits on requests
4. **API Stability**: Unofficial APIs may change without notice
5. **Control Commands**: Write operations may be restricted/protected

### Device Limitations
- **Inverter Location**: Must be on same network or accessible remotely
- **Firmware Updates**: May break custom integrations
- **Multiple Devices**: Q.HOME system may have multiple endpoints

### iOS App Challenges
- **Background Execution**: 30-minute limit for background tasks
- **Network Access**: Background network access restricted
- **User Configuration**: User must provide IP address or cloud credentials

---

## Next Steps: Concrete Actions

### Immediate (This Week)
1. ✅ **Access Inverter Web Interface**
   - Find inverter IP on network (check router, or use network scanner)
   - Visit `http://[inverter-ip]` in browser
   - Document available settings, especially "Communication" or "API" sections
   - Screenshot any relevant configuration pages

2. ✅ **Email Q CELLS Support**
   - Draft email requesting developer API access
   - CC: sales@q-cells.com and check website for technical support email
   - Ask specific questions about Modbus, SunSpec, and third-party integration

3. ✅ **Test Modbus Connection** (if enabled)
   - Download QModMaster (free Modbus testing tool)
   - Try connecting to inverter on port 502
   - Read registers in range 40000-49999 (typical holding registers)
   - Document successful reads

### Short Term (Next 2-4 Weeks)
4. **Implement Mock API in Swift App**
   - Create `QHomeBatteryAPI` and `QHomeWallboxAPI` protocols
   - Build mock implementations returning dummy data
   - App functions fully with simulated data
   - Easy to swap with real implementation later

5. **Research Home Assistant Community**
   - Search: "Q.HOME Home Assistant integration"
   - Check forums: community.home-assistant.io
   - Look for custom component YAML files

6. **Network Traffic Analysis** (if desperate)
   - Set up man-in-the-middle proxy (Charles Proxy, mitmproxy)
   - Route Q.HOME app through proxy
   - Capture API calls
   - ⚠️ Only for personal research, respect ToS

### Medium Term (1-3 Months)
7. **Prototype Modbus Integration** (if supported)
   - Integrate SwiftModbus or write custom TCP client
   - Test read operations for battery SoC, power flow
   - Validate data accuracy against Q.HOME app

8. **Build User Configuration UI**
   - Settings screen for entering inverter IP or API endpoint
   - Credential storage in Keychain
   - Connection testing/validation

9. **Fallback Strategy**
   - If no API available: manual data entry mode
   - User inputs battery SoC manually
   - App provides optimization recommendations
   - User manually controls devices via Q.HOME app

---

## Alternative: Competitive Analysis

If Q.HOME integration proves impossible, consider:

**Other Battery Systems with APIs**:
- **Tesla Powerwall**: Has unofficial API (well documented by community)
- **sonnenBatterie**: Official API available
- **LG Chem RESU**: Limited API support
- **Enphase Envoy**: Official API via Enlighten platform

**Wallbox Alternatives with APIs**:
- **Wallbox Pulsar Plus**: Official API
- **Zappi by myenergi**: Open API
- **OpenWB**: Open-source wallbox with full API

⚠️ **Note**: Switching hardware is not recommended, but understanding competitors helps assess Q.HOME's market position.

---

## Resources & References

### Documentation to Request from Q CELLS:
- [ ] Q.HOME+ ESS HYB-G3 Installation Manual (communication section)
- [ ] Q.HOME EDRIVE Technical Specification
- [ ] Developer API documentation (if exists)
- [ ] Modbus register map
- [ ] SunSpec implementation guide

### Community Resources:
- Home Assistant Forum: https://community.home-assistant.io/
- Photovoltaikforum (German): https://www.photovoltaikforum.com/
- Reddit r/homeassistant: https://reddit.com/r/homeassistant
- Reddit r/solar: https://reddit.com/r/solar

### Swift Libraries:
- SwiftModbus: https://github.com/carabina/SwiftModbus
- BlueSocket: https://github.com/Kitura/BlueSocket (for raw TCP)
- Network.framework: Apple's modern networking (iOS 12+)

### Testing Tools:
- QModMaster: https://sourceforge.net/projects/qmodmaster/
- ModScan: https://www.modbusdriver.com/modscan.html
- Charles Proxy: https://www.charlesproxy.com/
- Wireshark: https://www.wireshark.org/

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| No official API available | **HIGH** | HIGH | Use Modbus or mock implementation |
| Modbus not supported | MEDIUM | MEDIUM | Request API from Q CELLS |
| Cloud API authentication blocked | MEDIUM | HIGH | Focus on local network only |
| Firmware update breaks integration | MEDIUM | MEDIUM | Monitor Q CELLS updates, graceful degradation |
| Legal/ToS issues with reverse engineering | LOW | HIGH | Only use official or standard protocols |
| User configuration complexity | MEDIUM | MEDIUM | Provide clear setup wizard |

---

## Conclusion

**Most Viable Approach for Swift iOS App**:

1. **Primary**: **Modbus TCP integration** (if supported by Q.HOME+ ESS HYB-G3)
   - Standard protocol, no authentication needed
   - Local network access
   - Real-time monitoring
   - Potentially read-only (control may require additional auth)

2. **Secondary**: **Request official API** from Q CELLS
   - Long-term solution
   - May require business relationship
   - Timeline uncertain

3. **Fallback**: **Manual mode** with app recommendations
   - App calculates optimal schedule
   - User manually executes via Q.HOME app
   - No API dependency

**Immediate Action**: Access inverter settings and test Modbus connectivity this week. Contact Q CELLS support simultaneously. Build app with mock API to maintain development momentum.

---

## Open Questions

- [ ] Does Q.HOME+ ESS HYB-G3 have built-in web interface?
- [ ] Is Modbus TCP enabled by default or requires activation?
- [ ] What is the inverter's local IP address on your network?
- [ ] Does Q.HOME app work when internet is down? (indicates local API)
- [ ] Is there a Q.HOME cloud portal for web access?
- [ ] Has Q CELLS announced any developer program or API roadmap?

---

**Document Status**: Research complete, awaiting device verification and Q CELLS response.
