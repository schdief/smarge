# SMARGE Project Constitution

**Version:** 1.0  
**Date:** November 23, 2025  
**Project:** SMARt chARGE orchestrator for households

---

## 1. Project Mission & Vision

### Mission
Optimize household energy costs by intelligently orchestrating the charging of home batteries and battery electric vehicles (BEVs), maximizing solar power utilization and strategically leveraging dynamic electricity pricing.

### Vision
Enable households with solar power systems, home batteries, and electric vehicles to achieve maximum energy independence and minimum energy costs through intelligent, automated charging orchestration.

### Success Metrics
- **Cost Reduction**: Reduce annual electricity costs by ≥20% compared to unoptimized charging
- **Solar Utilization**: Maximize self-consumption of solar power (target: ≥70% of generated solar energy)
- **User Satisfaction**: Daily interaction time ≤2 minutes, with high user satisfaction ratings
- **Reliability**: ≥99% uptime for optimization and charging execution

---

## 2. Core Principles

### 2.1 User Experience Principles

**Notification-Driven Interaction**
- System proactively notifies users when interaction is needed
- User opens app twice daily for optimization:
  - **Morning (6:30 AM)**: Optimize for solar power usage during the day
  - **Afternoon (1:30 PM)**: Re-optimize once tomorrow's prices are available from Tibber
- Critical notifications for charging events, price anomalies, and system issues
- No silent failures - always inform user of significant changes

**Simplicity First**
- Default behavior should "just work" for 80% of use cases
- Complex settings are available but not required
- Clear visual feedback on all decisions and schedules
- One-tap quick actions for common tasks (e.g., "Quick Charge BEV Now")

**Transparency & Trust**
- Always show WHY decisions were made (solar availability, price levels)
- Display cost estimates before and after optimization
- Historical tracking to demonstrate savings over time
- Explain trade-offs when multiple strategies are available

### 2.2 Architecture Principles

**iOS-First, Standalone Architecture**
- Native iOS app as primary and sole interface
- No mandatory external servers or cloud dependencies
- All optimization logic runs locally on device
- Optional companion device (Raspberry Pi) for advanced scenarios only

**Smart Background Execution**
- Leverage iOS background tasks for periodic updates
- Schedule device commands through direct API calls
- Graceful degradation when background tasks are limited
- Notification reminders ensure user can trigger optimization

**Privacy & Data Ownership**
- All data stored locally on device (SwiftData for MVP, CloudKit sync in Phase 2)
- No telemetry or analytics without explicit user consent
- User owns and controls all historical data
- Shared household data via CloudKit (Phase 2: multi-user support)
- Secure credential storage for API tokens (Keychain)

**Resilience & Offline Capability**
- App functions with cached data when network unavailable
- Show last-known states when devices unreachable
- Queue commands when APIs temporarily offline
- Clear indicators of data freshness and connectivity status

### 2.3 Optimization Principles

**Solar Power Priority**
- Always prefer solar power over grid power when available
- Predict solar production based on weather forecasts
- Opportunistic charging during solar surplus periods
- Avoid grid charging when solar is forecasted to be sufficient

**Cost Minimization Secondary**
- When solar insufficient, charge during cheapest price periods
- Consider feed-in compensation rate (≈9 cents/kWh) as opportunity cost
- Dynamic re-optimization when prices change significantly (>20%)
- Balance immediate cost vs. future availability needs

**BEV Requirements Are Mandatory**
- BEV charging goals have absolute priority
- Home battery charging is opportunistic/optional
- Ensure BEV target SoC reached before deadline
- Alert user early if goal cannot be met within constraints

**Conservative Forecasting**
- Err on side of caution for solar predictions (10-15% buffer)
- Account for forecast uncertainty in optimization
- Prefer having excess charge over running short
- Re-optimize frequently (hourly) to adjust for actuals vs. forecast

### 2.4 Technical Principles

**API-First Integration**
- Direct integration with device APIs (no screen scraping)
- Graceful error handling for API failures
- Rate limiting respect and backoff strategies
- Cache API responses to minimize redundant calls

**Incremental Development**
- Start with simple rule-based algorithms (greedy approach)
- Evolve to linear programming for optimization
- Add machine learning for forecasting only when needed
- Each phase must be fully functional before adding complexity

**Testing & Validation**
- Unit tests for optimization algorithms
- Integration tests with mock APIs
- Validate cost calculations against actual bills
- Historical backtesting against past data

**Extensibility**
- Design for future expansion (multiple vehicles, devices)
- Pluggable architecture for new pricing providers
- Support for future technologies (V2G, additional batteries)
- Open to new optimization strategies
- Data models designed for CloudKit compatibility (Phase 2 multi-user)

---

## 3. System Context & Constraints

### 3.1 Use Case Environment

**Household Profile**
- Annual consumption: 17,000 kWh
- Solar system: 13 kW (Q.HOME+ ESS HYB-G3), SW orientation, 45° tilt
- Home battery: 12 kWh capacity
- BEV: 55 kWh battery (Skoda Enyaq)
- Wallbox: Q.HOME EDRIVE
- Tariff: Dynamic (Tibber), ≈28 cents/kWh average, range 20 cents - €1+
- Feed-in: ≈9 cents/kWh

**Usage Patterns**
- BEV used daily, primarily morning and afternoon commutes
- BEV typically plugged in overnight at home
- Desired BEV charge levels: 40%, 80%, or 100% by morning
- Household consumption patterns vary by season and day type

### 3.2 Technical Constraints

**iOS Platform Limitations**
- Background tasks limited to ~30 minutes of execution
- Background refresh occurs at system's discretion (not guaranteed timing)
- Network requests must complete within background task window
- Push notifications require user permission

**Device API Limitations**
- Q.HOME devices: API rate limits, authentication requirements
- Tibber: Next-day prices published around 1:00 PM (sometimes delayed until 5:00 PM)
- Tibber: Current day prices available, but next-day unavailable until afternoon
- Weather services: Forecast accuracy degrades beyond 48 hours
- All APIs subject to occasional downtime

**Optimization Constraints**
- BEV max charge rate: 11 kW (wallbox limit)
- Battery max charge/discharge: ~5 kW
- Charging BEV from home battery is impractical (12 kWh battery vs. 55 kWh BEV = only ~20% charge)
- Grid connection limits household draw

### 3.3 Non-Functional Requirements

**Performance**
- Optimization calculation: <10 seconds for 48-hour window
- App launch time: <2 seconds on modern iPhone
- Background task completion: within iOS 30-minute limit
- API response handling: <5 seconds per call

**Reliability**
- Graceful degradation when APIs unavailable
- Data integrity maintained across app crashes
- No data loss during iOS updates or device restarts
- Automatic recovery from transient failures

**Security**
- API credentials stored in iOS Keychain
- Network communications over HTTPS only
- No credentials in logs or crash reports
- Secure handling of pricing and usage data

**Maintainability**
- Clear code documentation and architecture diagrams
- Separation of concerns (UI, business logic, data)
- Comprehensive logging for troubleshooting
- Version compatibility with iOS releases

---

## 4. Out of Scope (For Initial Version)

The following are explicitly NOT included in the initial release:

- **Multi-vehicle support**: Only one BEV supported initially
- **Vehicle-to-Grid (V2G)**: BEV can only charge, not discharge to home
- **Smart home integration**: No control of other household devices
- **Energy community features**: No peer-to-peer energy sharing
- **Advanced ML models**: Start with simpler statistical forecasting
- **Export to other platforms**: iOS only (no Android, web)
- **Automatic tariff switching**: User must manually change provider settings

**Note**: Multi-user support via CloudKit is planned for Phase 2 (see Development Phases)

---

## 5. Key Decisions & Rationale

### Decision 1: iOS-Only, No Required Backend
**Rationale**: Eliminates hosting costs, reduces complexity, ensures user privacy, enables offline functionality. User is willing to open app daily when notified.

### Decision 2: Notification-Driven Daily Interaction
**Rationale**: Works within iOS background task limitations, ensures optimization runs regularly, gives user control while minimizing manual effort.

### Decision 3: Solar Priority Over Cost Optimization
**Rationale**: Self-consumed solar has ~9 cent opportunity cost (feed-in rate), often better than grid prices. Maximizes energy independence and sustainability goals.

### Decision 4: BEV Goals Are Mandatory Constraints
**Rationale**: Running out of BEV charge is unacceptable user experience. Home battery optimization is "nice to have," BEV charging is "must have."

### Decision 5: Start Simple, Evolve Complexity
**Rationale**: Rule-based greedy algorithm can achieve 80% of optimal results with 20% of complexity. Proves value before investing in sophisticated optimization.

### Decision 6: Multi-User in Phase 2 (Not MVP)
**Rationale**: Start with single-device MVP to prove core value quickly. Add CloudKit-based household sharing in Phase 2 once optimization is validated. Design data models from start to be CloudKit-compatible (use standard Swift types, avoid complex relationships). Apple Family Sharing provides natural household grouping.

---

## 6. Technology Stack

### Development Environment
- **Hardware**: M1 MacBook Air (Apple Silicon)
- **OS**: macOS Sequoia 15.x (or latest)
- **IDE**: Xcode 16+
- **Version Control**: Git + GitHub
- **Specification**: GitHub Spec Kit

### Primary Technologies
- **Platform**: iOS 18+ (minimum iOS 18.0, targeting iOS 26.1)
- **Language**: Swift 5.10+
- **UI Framework**: SwiftUI 5.0+
- **Async**: Swift Concurrency (async/await, actors)
- **Reactive**: Combine framework
- **Storage**: SwiftData (iOS 17+) for local persistence
- **Sync**: CloudKit (Phase 2) for multi-user household sharing
- **Networking**: URLSession with async/await
- **Background**: BGTaskScheduler (BGAppRefreshTask, BGProcessingTask)
- **Notifications**: UserNotifications framework
- **Charts**: Swift Charts (iOS 16+)
- **Security**: Keychain Services for credential storage

### External APIs
- **Tibber API**: Dynamic electricity pricing and consumption data
  - REST API with GraphQL endpoints
  - Real-time price updates via WebSocket (optional)
- **Weather API**: Solar production forecasting
  - Primary: Open-Meteo (free, no API key required)
  - Fallback: OpenWeatherMap or similar
- **Q.HOME+ ESS API**: Home battery control and monitoring
  - Battery state of charge (SoC)
  - Charge/discharge commands
  - Power flow monitoring
- **Q.HOME EDRIVE API**: Wallbox and BEV control and monitoring
  - BEV state of charge
  - Charging start/stop commands
  - Charging rate control

### Development Tools
- **Testing**: XCTest (unit), XCUITest (UI automation)
- **Debugging**: Instruments, OSLog for structured logging
- **Package Manager**: Swift Package Manager (SPM)
- **CI/CD**: GitHub Actions (optional for automated testing)
- **Analytics**: None initially (privacy-first approach)

---

## 7. Development Phases

### Phase 1: MVP (Minimum Viable Product)
**Goal**: Prove core value - optimize BEV charging based on prices and solar
- Manual BEV goal entry
- Tibber price integration
- Basic solar forecast
- Simple greedy optimization
- Daily notification reminders
- Schedule display

### Phase 2: Home Battery Integration & Multi-User
**Goal**: Add home battery optimization and household member sharing

**Battery Features:**
- Q.HOME battery API integration
- Battery charge/discharge scheduling
- Combined BEV + battery optimization
- Real-time status monitoring

**Multi-User Features (CloudKit):**
- CloudKit shared database setup
- Household sharing via Apple Family Sharing
- Invitation/acceptance flow
- Real-time sync across household devices
- Conflict resolution (last-write-wins)
- Shared view of schedules, goals, and device states

### Phase 3: Intelligence & Automation
**Goal**: Improve forecasting and reduce user input
- Historical pattern learning
- Improved solar forecasting
- Consumption prediction
- Calendar integration for BEV planning
- Automatic goal suggestions

### Phase 4: Polish & Advanced Features
**Goal**: Production-ready quality and additional features
- Cost tracking and analytics
- Savings reports
- What-if scenarios
- Export data
- Widget support
- Apple Watch complication

---

## 8. Quality Gates

Each development phase must meet these criteria before proceeding:

**Functional Completeness**
- All planned features for phase implemented
- Core user workflows tested end-to-end
- API integrations verified with live services

**Quality Standards**
- Unit test coverage ≥70% for business logic
- No critical or high-severity bugs
- Performance meets defined thresholds
- User interface follows iOS Human Interface Guidelines

**User Validation**
- Developer (primary user) validates functionality
- At least 7 days of real-world usage
- Cost tracking shows expected savings
- No blocking issues identified

**Documentation**
- Code documented (inline comments + architecture docs)
- User guide for key features
- API integration notes
- Known limitations documented

---

## 9. Governance & Evolution

### Change Management
- Constitution can be updated as project evolves
- Major changes require explicit documentation of rationale
- Breaking changes to core principles require strong justification
- Version history maintained in git

### Decision Authority
- Project owner (developer/user) has final decision authority
- Technical decisions guided by principles in this document
- Trade-offs between principles resolved in favor of user value

### Review Cadence
- Constitution reviewed at end of each development phase
- Updated based on learnings and changing requirements
- Obsolete sections marked but retained for historical context

---

**Document Control**
- Initial version: 1.0
- Last updated: November 23, 2025
- Updates:
  - Corrected iOS version target to iOS 26.1
  - Added twice-daily optimization strategy based on Tibber price publication schedule
  - Added CloudKit multi-user planning for Phase 2
  - Updated data architecture to support CloudKit from start (CloudKit-compatible models)
- Next review: End of Phase 1 (MVP completion)
- Owner: SMARGE Project Team
