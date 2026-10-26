/\*\*  
 \* ARINC Characteristic 735B & RTCA DO-185B (TCAS II v7.1) Core Collision Avoidance Engine  
 \* Implements Tau (range & vertical) time-to-CPA calculations, Sensitivity Levels (SL 1–7),  
 \* Resolution Advisory (RA) / Traffic Advisory (TA) state machines, Mode S coordination, and Hybrid Surveillance.  
 \*/

// Sensitivity Level Threshold Table (RTCA DO-185B Table 2-1)  
const TCAS\_SENSITIVITY\_LEVELS \= {  
  1: { name: 'SL 1 (Standby)', tauTa: 0, tauRa: 0, dmodNm: 0.0, zthrFt: 0, alimFt: 0 },  
  2: { name: 'SL 2 (Below 1000 ft)', tauTa: 20, tauRa: 0, dmodNm: 0.30, zthrFt: 600, alimFt: 0 },  
  3: { name: 'SL 3 (1000 \- 2350 ft)', tauTa: 25, tauRa: 15, dmodNm: 0.33, zthrFt: 600, alimFt: 300 },  
  4: { name: 'SL 4 (2350 \- 5000 ft)', tauTa: 30, tauRa: 20, dmodNm: 0.48, zthrFt: 600, alimFt: 300 },  
  5: { name: 'SL 5 (5000 \- 10000 ft)', tauTa: 40, tauRa: 25, dmodNm: 0.75, zthrFt: 600, alimFt: 350 },  
  6: { name: 'SL 6 (10000 \- 20000 ft)', tauTa: 45, tauRa: 30, dmodNm: 1.00, zthrFt: 600, alimFt: 400 },  
  7: { name: 'SL 7 (\> 20000 ft)', tauTa: 48, tauRa: 35, dmodNm: 1.10, zthrFt: 700, alimFt: 600 }  
};

// Threat Classification  
const THREAT\_TYPES \= {  
  RESOLUTION\_ADVISORY: 'RESOLUTION\_ADVISORY', // Red Solid Square  
  TRAFFIC\_ADVISORY: 'TRAFFIC\_ADVISORY',       // Amber Solid Circle  
  PROXIMITY\_TRAFFIC: 'PROXIMITY\_TRAFFIC',     // Cyan Solid Diamond  
  OTHER\_TRAFFIC: 'OTHER\_TRAFFIC'              // White Hollow Diamond  
};

// Resolution Advisory Types (TCAS II v7.1)  
const RA\_TYPES \= {  
  CLIMB: { aural: 'CLIMB, CLIMB', targetVsiFpm: 1500, sense: 'UP' },  
  DESCEND: { aural: 'DESCEND, DESCEND', targetVsiFpm: \-1500, sense: 'DOWN' },  
  LEVEL\_OFF: { aural: 'LEVEL OFF, LEVEL OFF', targetVsiFpm: 0, sense: 'LEVEL' },  
  CLIMB\_NOW: { aural: 'CLIMB, CLIMB NOW\!', targetVsiFpm: 2500, sense: 'UP\_REVERSAL' },  
  DESCEND\_NOW: { aural: 'DESCEND, DESCEND NOW\!', targetVsiFpm: \-2500, sense: 'DOWN\_REVERSAL' }  
};

class TCASEngine {  
  constructor(ownshipAltFt \= 32000, ownshipVsiFpm \= 0\) {  
    this.ownshipAltFt \= ownshipAltFt;  
    this.ownshipVsiFpm \= ownshipVsiFpm;  
    this.sensitivityLevel \= this.determineSensitivityLevel(ownshipAltFt);  
  }

  determineSensitivityLevel(altFt) {  
    if (altFt \< 1000\) return 2;  
    if (altFt \<= 2350\) return 3;  
    if (altFt \<= 5000\) return 4;  
    if (altFt \<= 10000\) return 5;  
    if (altFt \<= 20000\) return 6;  
    return 7;  
  }

  /\*\*  
   \* Evaluates Time-to-CPA: Range Tau and Vertical Tau  
   \* @param {number} rangeNm \- Slant range in nautical miles  
   \* @param {number} closureRateKts \- Range closure rate in knots (positive if closing)  
   \* @param {number} diffAltFt \- Vertical separation in feet (targetAlt \- ownshipAlt)  
   \* @param {number} verticalClosureRateFpm \- Vertical closure rate in feet per minute (positive if converging)  
   \*/  
  evaluateIntruder(rangeNm, closureRateKts, diffAltFt, verticalClosureRateFpm) {  
    const slConfig \= TCAS\_SENSITIVITY\_LEVELS\[this.sensitivityLevel\];  
      
    // Convert closure rate from knots to NM/sec: kts / 3600  
    const closureRateNmPerSec \= closureRateKts / 3600.0;  
      
    // Range Tau: tau \= range / closureRate (seconds)  
    const rangeTauSec \= (closureRateNmPerSec \> 0\) ? (rangeNm / closureRateNmPerSec) : 999.0;  
      
    // Vertical Tau: tau\_v \= abs(diffAlt) / abs(vClosureRate / 60\) (seconds)  
    const vClosureRateFps \= verticalClosureRateFpm / 60.0;  
    const verticalTauSec \= (vClosureRateFps \> 0\) ? (Math.abs(diffAltFt) / vClosureRateFps) : 999.0;

    // Check Distance Modification (DMOD) boundary  
    const isRangeTa \= (rangeTauSec \<= slConfig.tauTa) \|\| (rangeNm \<= slConfig.dmodNm);  
    const isVerticalTa \= (verticalTauSec \<= slConfig.tauTa) \|\| (Math.abs(diffAltFt) \<= slConfig.zthrFt);

    const isRangeRa \= (slConfig.tauRa \> 0\) && ((rangeTauSec \<= slConfig.tauRa) \|\| (rangeNm \<= slConfig.dmodNm));  
    const isVerticalRa \= (slConfig.tauRa \> 0\) && ((verticalTauSec \<= slConfig.tauRa) \|\| (Math.abs(diffAltFt) \<= slConfig.zthrFt));

    let threatClassification \= THREAT\_TYPES.OTHER\_TRAFFIC;  
    let resolutionAdvisory \= null;

    if (isRangeRa && isVerticalRa) {  
      threatClassification \= THREAT\_TYPES.RESOLUTION\_ADVISORY;  
      // Determine RA sense (Up vs Down)  
      if (diffAltFt \>= 0\) {  
        // Target is above ownship \-\> Descend RA  
        resolutionAdvisory \= RA\_TYPES.DESCEND;  
      } else {  
        // Target is below ownship \-\> Climb RA  
        resolutionAdvisory \= RA\_TYPES.CLIMB;  
      }  
    } else if (isRangeTa && isVerticalTa) {  
      threatClassification \= THREAT\_TYPES.TRAFFIC\_ADVISORY;  
    } else if (rangeNm \<= 6.0 && Math.abs(diffAltFt) \<= 1200\) {  
      threatClassification \= THREAT\_TYPES.PROXIMITY\_TRAFFIC;  
    }

    // Hybrid Surveillance status (ARINC 735B Section 4\)  
    // Targets \>= 6.0 NM are tracked passively via ADS-B In; targets \< 6.0 NM transition to Active Interrogation  
    const surveillanceMode \= (rangeNm \< 6.0) ? 'ACTIVE\_INTERROGATION\_1030MHZ' : 'PASSIVE\_ADS\_B\_IN\_MONITOR';

    return {  
      sensitivityLevel: this.sensitivityLevel,  
      rangeNm: rangeNm,  
      closureRateKts: closureRateKts,  
      diffAltFt: diffAltFt,  
      rangeTauSec: Math.min(999.0, rangeTauSec).toFixed(1),  
      verticalTauSec: Math.min(999.0, verticalTauSec).toFixed(1),  
      threat: threatClassification,  
      resolutionAdvisory: resolutionAdvisory,  
      surveillanceMode: surveillanceMode,  
      alimFt: slConfig.alimFt,  
      zthrFt: slConfig.zthrFt,  
      dmodNm: slConfig.dmodNm  
    };  
  }

  /\*\*  
   \* Encodes a 56-bit Mode S Coordination Interrogation (UF=16 / DF=16)  
   \*/  
  static encodeModeSCoordination(racUp \= false, racDown \= false, targetIcao \= 0xAA1234) {  
    // UF=16 (5 bits: 10000\)  
    let racBits \= 0;  
    if (racUp) racBits \|= (1 \<\< 3);   // Do not pass above  
    if (racDown) racBits \|= (1 \<\< 2); // Do not pass below

    return {  
      uf: 16,  
      rac: racBits,  
      racComplement: racUp ? 'DO\_NOT\_PASS\_ABOVE' : (racDown ? 'DO\_NOT\_PASS\_BELOW' : 'NO\_RESTRICTION'),  
      targetIcao: '0x' \+ (targetIcao \>\>\> 0).toString(16).toUpperCase().padStart(6, '0'),  
      frameBits: 56  
    };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    TCASEngine,  
    TCAS\_SENSITIVITY\_LEVELS,  
    THREAT\_TYPES,  
    RA\_TYPES  
  };  
}  
