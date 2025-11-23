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

### Task 2: Contact Q CELLS Official Support
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

- [ ] Set reminder to follow up in 1 week if no response

### Task 3: Check Vehicle Integration
- [ ] Verify Skoda Enyaq provides SoC in its own app (Skoda Connect/MySkoda)
- [ ] Check if vehicle reports SoC to wallbox (test by checking Q.HOME app)
- [ ] Document current manual process for checking BEV charge level

---

## Week 2: Technical Testing

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
   - [ ] Modbus TCP (full/read-only)
   - [ ] Official API (approved/pending/rejected)
   - [ ] Manual mode only

2. **What data can be accessed?**
   - [ ] Battery SoC (automated/manual)
   - [ ] BEV SoC (automated/manual)
   - [ ] Solar production (automated/manual)
   - [ ] Current power flows (automated/manual)

3. **What control is available?**
   - [ ] Start/stop charging (automated/manual)
   - [ ] Set charge rate (automated/manual)
   - [ ] Schedule commands (automated/manual)

4. **MVP Approach:**
   - [ ] Option 1: Full Modbus integration (best case)
   - [ ] Option 2: Read-only Modbus + manual control (good case)
   - [ ] Option 3: Manual mode only (acceptable case)

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
