# TCASForge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your web browser.  
2. In the "Airborne Collision Avoidance Config" panel, set ownship barometric altitude (e.g. `32,000 ft / FL320`). Notice that Sensitivity Level automatically adapts to `SL 7`.  
3. Input the intruder encounter geometry:  
   - Slant range (e.g. `2.5 NM`).  
   - Range closure rate (e.g. `450 knots`).  
   - Vertical separation (e.g. `400 ft` above ownship).  
   - Vertical convergence rate (e.g. `1,200 fpm`).  
4. Click **Evaluate Intruder Threat** to execute the Range Tau (\$\\tau\$) and Vertical Tau (\$\\tau\_v\$) equations, determine threat status (Resolution Advisory), and generate the commanded maneuver (Descend, target VSI \-1500 fpm).  
5. Click **Simulate "Level Off" RA** to test TCAS 7.1 "Level Off, Level Off" aural command execution.  
6. Review the Mode S Coordination Interrogation (UF=16) frame in the hex dissector.  
7. Click **Run Normative Audit** to evaluate all 10 RTCA DO-185B / ARINC 735B compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 tcas-cli.py \--audit \--alt 32000 \--range 2.5 \--closure 450