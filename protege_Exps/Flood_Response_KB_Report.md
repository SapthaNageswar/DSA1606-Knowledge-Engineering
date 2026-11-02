# Flood-Response Knowledge Base for a Coastal City

Files: `flood_response.owl` (open in Protégé), `flood_response.ttl` (same ontology in Turtle), `competency_queries.rq`, `semantic_network.svg/.png`.
Namespace: `http://example.org/floodresponse#`

Verified: the file loads, round-trips losslessly between Turtle and RDF/XML, is consistent under HermiT, and the three SPARQL queries below were run on it.

## 1. Domain concepts (25) and class hierarchy

```
owl:Thing
├── GeographicArea
│   ├── Ward
│   │   ├── LowLyingWard        (elevation < 3.0 m)
│   │   └── HighGroundWard      (elevation >= 3.0 m)
│   └── WaterBody
├── Infrastructure
│   ├── PumpStation
│   ├── StormDrain
│   └── RainGauge
├── Facility
│   ├── ReliefCamp
│   └── Hospital
├── HazardEvent
│   ├── RainfallEvent
│   ├── HighTide
│   └── FloodEvent
├── Alert
│   ├── RainfallAlertLevel      (Green, Yellow, Orange, Red)
│   └── EvacuationOrder
├── Resource
│   ├── Pump                    (portable pumps: isPortable = true)
│   └── RescueBoat
└── Actor
    ├── Agency
    └── RescueTeam
```

The seven top-level groups are disjoint, as are the sibling event and alert classes, and LowLyingWard and HighGroundWard. Both ward classes are defined by an elevation restriction, so a reasoner can classify a ward from its elevation alone.

## 2. Properties

### Object properties (17)

| Property | Domain | Range | Notes |
|---|---|---|---|
| locatedIn | Facility ⊔ Infrastructure | Ward | |
| adjacentTo | Ward | Ward | symmetric |
| dischargesTo | PumpStation | WaterBody | |
| isPartOf / hasPart | Pump / PumpStation | PumpStation / Pump | inverses |
| causes | HazardEvent | HazardEvent | transitive |
| triggersAlert | RainfallEvent | RainfallAlertLevel | |
| affects / isAffectedBy | FloodEvent / Ward | Ward / FloodEvent | inverses |
| hasAlertLevel | Ward | RainfallAlertLevel | functional |
| deployedTo | Resource | Ward | |
| operatesIn | RescueTeam | Ward | |
| issuedFor | EvacuationOrder | Ward | |
| evacuatesTo / shelters | Ward / ReliefCamp | ReliefCamp / Ward | inverses |
| managedBy | PumpStation ⊔ ReliefCamp ⊔ Hospital | Agency | |
| monitors | RainGauge | Ward | |

### Data properties (9, all functional)

| Property | Domain | Range |
|---|---|---|
| hasElevation (m above MSL) | Ward | xsd:decimal |
| hasPopulation | Ward | xsd:integer |
| pumpCapacity (m³/min) | Pump | xsd:decimal |
| isPortable | Pump | xsd:boolean |
| isOperational | PumpStation | xsd:boolean |
| campCapacity | ReliefCamp | xsd:integer |
| rainfallMm24h | RainfallEvent | xsd:decimal |
| floodDepthCm | FloodEvent | xsd:decimal |
| alertLevelCode (1 Green … 4 Red) | RainfallAlertLevel | xsd:integer |

## 3. Individuals (55)

8 wards (5 low-lying, 3 high-ground), 3 water bodies, 4 pump stations, 8 pumps (3 portable), 2 storm drains, 3 rain gauges, 4 relief camps, 3 hospitals, 4 alert levels, 4 agencies, 2 rainfall events, 1 high tide, 2 flood events, 2 evacuation orders, 2 rescue boats, 3 rescue teams.

Scenario: Rain_Oct2026 (240.5 mm) causes Flood_Marina_Oct2026 (85 cm), which causes Flood_Harbour_Oct2026 (35 cm, also caused by Tide_Oct2026). W_Marina is Red; W_Harbour and W_CoveEnd are Orange; PS_Harbour is out of service.

## 4. Semantic network

See `semantic_network.svg` (26 nodes). Blue edges are is-a/instance-of, green are part-of, red are causes. Two nodes (Drainage System, Coastal City) are dashed grouping nodes that exist only in the network drawing, not in the OWL file.

- **Inheritance:** W_Marina is an instance of LowLyingWard, which is-a Ward, which is-a GeographicArea. So W_Marina is also a Ward and a GeographicArea.
- **Property inheritance:** Ward declares hasElevation, hasPopulation and hasAlertLevel. LowLyingWard inherits all three, and W_Marina fills them (1.2 m, 52000, Alert_Red).

## 5. Competency questions and SPARQL

Full text is in `competency_queries.rq`. Results below come from running the queries on the ontology.

**CQ1. Which low-lying wards are at Orange or Red alert and have no operational pump station?**

```sparql
PREFIX : <http://example.org/floodresponse#>
SELECT ?ward ?alertLevel
WHERE {
  ?ward a :LowLyingWard ; :hasAlertLevel ?alert .
  ?alert :alertLevelCode ?code .
  BIND(STRAFTER(STR(?alert), "#") AS ?alertLevel)
  FILTER(?code >= 3)
  FILTER NOT EXISTS { ?ps a :PumpStation ; :locatedIn ?ward ; :isOperational true . }
}
ORDER BY DESC(?code) ?ward
```
Result: W_CoveEnd (no station at all) and W_Harbour (station out of service). W_Marina is excluded because PS_Marina works.

**CQ2. For wards at Orange or Red, is the designated relief-camp capacity enough for the population?**

```sparql
PREFIX : <http://example.org/floodresponse#>
SELECT ?ward ?population (SUM(?cap) AS ?campCapacity)
       (IF(SUM(?cap) >= ?population, "adequate", "SHORTFALL") AS ?status)
WHERE {
  ?ward :hasAlertLevel ?alert ; :hasPopulation ?population ; :evacuatesTo ?camp .
  ?alert :alertLevelCode ?code . FILTER(?code >= 3)
  ?camp :campCapacity ?cap .
}
GROUP BY ?ward ?population ORDER BY ?ward
```
Result: W_CoveEnd 21000 vs 25000, adequate. W_Harbour 38000 vs 24000, SHORTFALL. W_Marina 52000 vs 26000, SHORTFALL.

**CQ3. Which hospitals are in wards flooded by an event that a rainfall event causes, directly or through a chain, and who manages them?**

```sparql
PREFIX : <http://example.org/floodresponse#>
SELECT DISTINCT ?rain ?hospital ?ward ?agency
WHERE {
  ?rain a :RainfallEvent ; :causes+ ?flood .
  ?flood a :FloodEvent ; :affects ?ward .
  ?hospital a :Hospital ; :locatedIn ?ward ; :managedBy ?agency .
}
ORDER BY ?rain ?hospital
```
Result: Rain_Oct2026 reaches H_Marina (Ag_Health) and H_PortHealth (Ag_Municipal). H_PortHealth is only found through the two-step causal chain.

## 6. Production rules (12)

Alert thresholds follow IMD 24-hour rainfall categories; replace them with your city's SOP values.

| # | Group | Rule |
|---|---|---|
| R1 | Alert | IF rainfall24h ≥ 204.5 mm THEN set every ward's alert to Red |
| R2 | Alert | IF 115.6 ≤ rainfall24h ≤ 204.4 mm THEN set every ward's alert to Orange |
| R3 | Alert | IF 64.5 ≤ rainfall24h ≤ 115.5 mm THEN set every ward's alert to Yellow |
| R4 | Alert | IF ward is LowLyingWard AND alert = Yellow AND a HighTide is active THEN raise alert to Orange |
| R5 | Alert | IF ward alert = Orange AND an affecting FloodEvent has floodDepthCm ≥ 30 THEN raise alert to Red |
| R6 | Evacuation | IF ward is LowLyingWard AND alert = Red THEN issue a mandatory EvacuationOrder and open every camp the ward evacuatesTo |
| R7 | Evacuation | IF a Hospital is in a ward with floodDepthCm ≥ 20 THEN start patient transfer to a hospital in a HighGroundWard |
| R8 | Evacuation | IF total capacity of the ward's designated camps < ward population THEN activate camps in adjacent HighGroundWards |
| R9 | Evacuation | IF an EvacuationOrder is issued for a ward THEN deploy an available RescueBoat and RescueTeam to that ward |
| R10 | Pumping | IF a PumpStation is not operational AND its ward alert ≥ Orange THEN deploy an available portable Pump to that ward |
| R11 | Pumping | IF ward alert = Red AND no portable pump is available THEN reassign pumps from stations in Green HighGroundWards |
| R12 | De-escalation | IF rainfall < 64.5 mm for 6 consecutive hours AND floodDepthCm < 10 THEN lower alert by one level and withdraw advisories |

## 7. Priority ranking and conflict resolution

| Rank | Rule | Why |
|---|---|---|
| 1 | R7 | Patients in hospitals are the most time-critical lives |
| 2 | R6 | Mandatory evacuation of Red low-lying wards |
| 3 | R9 | Rescue assets must follow the evacuation order |
| 4 | R1 | Extreme rainfall sets the Red baseline |
| 5 | R5 | Observed flood depth overrides forecast-based level |
| 6 | R4 | Tide-compounded escalation |
| 7 | R2 | Orange baseline |
| 8 | R10 | Backup pump for a failed station |
| 9 | R8 | Extra camp capacity |
| 10 | R11 | Pump reassignment (draws on other wards) |
| 11 | R3 | Yellow baseline |
| 12 | R12 | De-escalation is always last |

Rank decides which rule wins when firing rules conflict. It is not the execution order: forward chaining still fires every applicable rule, and the alert rules run before the evacuation and pumping rules that read the alert level.

**Conflict-resolution strategy**

1. **Priority first.** Of two conflicting actions, the higher-ranked rule wins.
2. **Monotonic escalation.** Alert levels only move up while any escalating rule (R1, R2, R4, R5) is satisfied, so the highest level always wins (for example R3 and R5 firing together gives Red). This is the safety tie-break.
3. **De-escalation needs consensus.** R12 fires only when no escalation rule is currently satisfied, moves one level at a time, and needs operator confirmation before an evacuation order is withdrawn.
4. **Specificity.** If two rules have equal rank, the one with more conditions wins (R5 over R2 for the same ward).
5. **Resource contention (R9, R10, R11 competing for the same boats or pumps).** Allocate by alert level, then ward population, then presence of a hospital. R11 never strips a ward that is itself at Yellow or above.
6. **Recency.** If still tied, the rule whose triggering fact is newest wins, because it reflects current conditions.

## 8. Three rules in a second formalism

These are also in the ontology as defined classes, so HermiT classifies the individuals automatically: W_Marina is an EvacuationRequiredWard, PS_Harbour is a PumpStationNeedingBackup, and W_Harbour and W_CoveEnd are RedEscalationWards.

**R6 (mandatory evacuation)**

Description logic:
`EvacuationRequiredWard ≡ LowLyingWard ⊓ ∃hasAlertLevel.{Alert_Red}`

Frame:
```
Frame: Mandatory-Evacuation   priority: 2
  if:   ward  instance-of LowLyingWard
        ward.hasAlertLevel = Alert_Red
  then: create EvacuationOrder (issuedFor = ward)
        for each camp in ward.evacuatesTo: camp.status = open
```

**R10 (backup pump)**

Description logic:
`PumpStationNeedingBackup ≡ PumpStation ⊓ ∃isOperational.{false} ⊓ ∃locatedIn.(∃hasAlertLevel.{Alert_Orange, Alert_Red})`

Frame:
```
Frame: Backup-Pump   priority: 8
  if:   station instance-of PumpStation
        station.isOperational = false
        station.locatedIn.hasAlertLevel in {Alert_Orange, Alert_Red}
  then: pick Pump p with p.isPortable = true and no deployedTo
        p.deployedTo = station.locatedIn
```

**R5 (escalate to Red)**

Description logic:
`RedEscalationWard ≡ Ward ⊓ ∃hasAlertLevel.{Alert_Orange} ⊓ ∃isAffectedBy.(FloodEvent ⊓ ∃floodDepthCm.[≥ 30])`

Frame:
```
Frame: Flood-Escalation   priority: 5
  if:   ward instance-of Ward
        ward.hasAlertLevel = Alert_Orange
        some f in ward.isAffectedBy with f.floodDepthCm >= 30
  then: ward.hasAlertLevel = Alert_Red
```

## 9. Opening it in Protégé

1. File → Open → `flood_response.owl`.
2. Reasoner → HermiT → Start reasoner. The three rule classes then show their inferred members under Individuals.
3. For SPARQL, use the SPARQL Query tab and paste a query from `competency_queries.rq`.

The ontology has no SWRL rules. The 12 production rules live in this document, and the three DL encodings above are the executable part.
