# ARINC Characteristic 735B & RTCA DO-185B (TCAS II v7.1) Technical Architecture

## 1\. Executive Summary & Specification Scope

Airborne Collision Avoidance Systems (TCAS II and ACAS X) are mandated by the International Civil Aviation Organization (ICAO Annex 10\) and national civil aviation authorities (FAA, EASA) on commercial passenger and cargo aircraft. TCAS II Version 7.1 operates as an autonomous, airborne safety net protecting against mid-air collisions.

### Key Architectural Concepts:

1. **Collision Geometry & Time-to-CPA (\$\\tau\$) Modeling**:  
   - **Range Tau (\$\\tau\$)**: Ratio of range to closure rate: \$\$\\tau \= \-\\frac{r}{\\dot{r}}\$\$  
   - **Distance Modification (DMOD)**: Adds a protective range cushion when closure rates approach zero, protecting against low-speed slow-convergence encounters.  
   - **Vertical Tau (\$\\tau\_v\$)**: Ratio of vertical separation to vertical convergence rate: \$\$\\tau\_v \= \-\\frac{h}{\\dot{h}}\$\$  
   - **Vertical Threshold (ZTHR) & Altitude Limit (ALIM)**: ZTHR defines the vertical boundary triggering an advisory (\$600\\text{--}700\\text{ ft}\$), while ALIM defines the guaranteed minimum vertical separation commanded at CPA (\$300\\text{--}700\\text{ ft}\$).  
2. **Multi-Tier Sensitivity Levels (SL 1 to 7\)**:  
   - Dynamic threshold adaptation based on ownship barometric altitude, minimizing nuisance alerts in terminal airspace while maximizing early detection at high cruising speeds (\>FL200).  
3. **Four-Tier Threat Classification**:  
   - **Resolution Advisory (RA)**: Red Solid Square, severe threat projected within 15–35 seconds. Commands flight crew vertical maneuvers.  
   - **Traffic Advisory (TA)**: Amber Solid Circle, potential threat projected within 20–48 seconds. Directs visual acquisition.  
   - **Proximity Traffic**: Cyan Solid Diamond, non-threat traffic within 6 NM and \$\\pm 1200\\text{ ft}\$.  
   - **Other Traffic**: White Hollow Diamond, traffic beyond 6 NM or \$\>1200\\text{ ft}\$ separation.  
4. **Resolution Advisory Logic & TCAS 7.1 "Level Off"**:  
   - Implements TCAS II v7.1 voice aurals and VSI green fly-to arc guidance:  
     - "Climb, Climb" / "Descend, Descend" (\$1500\\text{ fpm}\$).  
     - "Level Off, Level Off" (\$0\\text{ fpm}\$) preventing vertical overshooting when the intruder levels off.  
     - Sense Reversals ("Climb, Climb NOW\!" / "Descend, Descend NOW\!").  
5. **Air-to-Air Mode S Coordination Protocol**:  
   - Transmits 56-bit coordination interrogations (UF=16 / DF=16) carrying Resolution Advisory Complement (RAC) bitfields over 1030 MHz to guarantee complementary maneuvers between converging TCAS aircraft.  
6. **ARINC 735B Hybrid Surveillance**:  
   - Tracks distant targets (\$\\ge 6\\text{ NM}\$) passively using 1090 MHz ADS-B In, transitioning to active 1030 MHz Mode C/S interrogations only when an intruder penetrates within 6 NM, reducing RF channel loading by over 70%.