# smarge
SMARt chARGE orchestrator for households with dynamic tariff, solar power with battery and a BEV

# mission
Smarge aims at optimizing the energy bill, by orchestrating the charging of your home battery and BEV, by trying to use as much solar power as possible.
When solar power is or will not be sufficiently available for the charging demands it places the charging periods strategically to achieve lowest energy spending as possible.

# functionality
## BEV
Based on the knowledge of the future desired charging state of the BEV (40%, 80% or 100% tomorrow at Xam) it plans the required charging phase(s) prioritizing on solar power and if needed times with lowest energy prices.
It can also be used to override any planning and perform a quick charge.

## home battery
As the home battery has no desired charge state, but just acts as a buffer to allow usage of solar power or power of cheaper day times at times were no solar power is available and/or energy is expensive, smarge plans the required charging phase(s) to utilize the buffer as much as possible - therefore it avoids charging the battery from grid although the weather forecast shows that with a high probability solar power will be fed back to grid.

# usage scenario
Our household consumes 17.000 kW/h p. a. of electrical energy to heat water and floor, to charge our BEV and to provide all electrical devices.
We have a 13 kW solar power system (Q.HOME⁺ ESS HYB-G3) on our roof with a south-west orientation at about 45 degrees.
The solar power system also consists of a home battery with 12 kWh capacity.
Our BEV (Skoda Enyaq) has a 55 kWh battery and is used every day, mostly in the morning and afternoon.
It is charged via our Q.HOME EDRIVE Wallbox at a max rate of 11 kw/h.
We use a dynamic electrical tariff from Tibber where the cost per energy unit depend on the spot market (15 minute windows), which leads to costs on average of about 28 Cents/kWh, but could also be more than 1€ or in theory even be negative - but the lowest prices we have seen were around 20 Cents due to taxes and net utilization fees. 
For feeding back energy to the grid we are compensated with about 9 Cents/kWh, so that is the price for consuming our own solar energy in terms of opportunity costs. 
