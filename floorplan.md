# KNX Device Floor Plan

All devices are controlled via FHEM on `192.168.178.4` over KNX/TP1 (busware TPUART → knxd → FHEM TUL).
Symbols: `L` = Light  `S` = Shutter  `P` = Power socket  `★` = Central/Scene

KNX group address format: `main/middle/sub`
- Main `1/x/y` = Lights — `1/0/y` switch cmd, `1/4/y` status
- Main `2/x/y` = Shutters — `2/0/y` move, `2/1/y` stop, `2/2/y` position set, `2/6/y` position status
- Main `5/x/y` = Scenes

Full GA scheme (physical address space, middle group functions, sub-address tables): see [`knx_GA_map.md`](knx_GA_map.md)

---

## Central / Systemweit

| FHEM Device | Alias | Type | KNX Group Addresses |
|-------------|-------|------|---------------------|
| `SCN_Away` | Away | ★ Scene | `5/0/000` cmd · `5/4/000` status |

---

## F0 — UG (Untergeschoss / Basement)

```
 ┌──────────────────────────────────────────────────────┐
 │                                                      │
 │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
 │  │  Keller I   │  │  Keller II  │  │  Keller III │  │
 │  │  (Technik)  │  │             │  │   (Büro)    │  │
 │  │             │  │             │  │             │  │
 │  │  L  SHU_C1  │  │  L          │  │  L  SHU_C3  │  │
 │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
 │         │                │                │          │
 │    ─────┴────────────────┴────────────────┴─────     │
 │                    Kellertreppe  L                    │
 └──────────────────────────────────────────────────────┘
```

| FHEM Device | Alias | Type | KNX Group Addresses | Notes |
|-------------|-------|------|---------------------|-------|
| `SHU_F0_all` | Shutter UG | S Central | `2/0/200` move | All basement shutters |
| `LGT_F0_C1` | C1 | L | `1/0/201` cmd · `1/4/201` status | Keller I (Technik) |
| `SHU_F0_C1` | C1 | S | `2/2/201` pos · `2/0/201` move · `2/1/201` stop · `2/6/201` pos-status | Keller I |
| `LGT_F0_C2` | C2 | L | `1/0/202` cmd · `1/4/202` status | Keller II |
| `LGT_F0_C3` | C3 | L | `1/0/203` cmd · `1/4/203` status | Keller III (Büro) |
| `SHU_F0_C3` | C3 | S | `2/2/203` pos · `2/0/203` move · `2/1/203` stop · `2/6/203` pos-status | Keller III |
| `LGT_F0_stairs` | Stairs | L | `1/0/205` cmd · `1/4/205` status | Kellertreppe |

---

## F1 — EG (Erdgeschoss / Ground Floor)

```
 ┌──────────────────────────────────────────────────────────────┐
 │                                                              │
 │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
 │  │    Küche     │  │  Esszimmer   │  │  Wohnzimmer  │       │
 │  │  (Kitchen)   │  │ (Dining Rm)  │  │  (Living Rm) │       │
 │  │              │  │              │  │              │  ~~~   │
 │  │  L  L(LED)   │  │  L           │  │  L           │  L    │
 │  │  S(Win) S(D) │  │  S           │  │  S           │(Out.) │
 │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  ~~~  │
 │         │                 │                  │               │
 │  ┌──────┴─────────────────┘                  │               │
 │  │         Flur (Hall)                        │               │
 │  │         L(Door)  L(Stairs)                │               │
 │  └──────┬──────────────────────────────────┬─┘               │
 │         │                                  │                 │
 │  ┌──────┴───────┐              ┌───────────┴──────┐          │
 │  │    Büro      │              │       WC         │          │
 │  │   (Office)   │              │                  │          │
 │  │              │              │                  │          │
 │  │  L           │              │  L               │          │
 │  │  S           │              │  S               │          │
 │  └──────────────┘              └──────────────────┘          │
 └──────────────────────────────────────────────────────────────┘
```

| FHEM Device | Alias | Type | KNX Group Addresses | Notes |
|-------------|-------|------|---------------------|-------|
| `SHU_F1_all` | Shutter EG | S Central | `2/0/0` move · `2/1/0` stop | All ground floor shutters |
| `LGT_F1_hall_door` | Hall (Door) | L | `1/0/1` cmd · `1/4/1` status | Flur — door side |
| `LGT_F1_hall_stairs` | Hall (Stairs) | L | `1/0/8` cmd · `1/4/8` status | Flur — staircase side |
| `LGT_F1_kitchen` | Kitchen | L | `1/0/2` cmd · `1/4/2` status | Küche main light |
| `LGT_F1_kitchen_led` | Kitchen LED | L | `1/0/7` cmd · `1/4/7` status | Küche LED accent |
| `SHU_F1_kitchen_door` | Kitchen (Door) | S | `2/2/2` pos · `2/0/2` move · `2/1/2` stop · `2/6/2` pos-status | Kitchen door shutter |
| `SHU_F1_kitchen_window` | Kitchen (Window) | S | `2/2/1` pos · `2/0/1` move · `2/1/1` stop · `2/6/1` pos-status | Kitchen window shutter |
| `LGT_F1_diningroom` | Dining Room | L | `1/0/3` cmd · `1/4/3` status | Esszimmer |
| `SHU_F1_diningroom` | Dining Room | S | `2/2/3` pos · `2/0/3` move · `2/1/3` stop · `2/6/3` pos-status | Esszimmer |
| `LGT_F1_livingroom` | Living Room | L | `1/0/4` cmd · `1/4/4` status | Wohnzimmer |
| `SHU_F1_livingroom` | Living Room | S | `2/2/4` pos · `2/0/4` move · `2/1/4` stop · `2/6/4` pos-status | Wohnzimmer |
| `LGT_F1_office` | Office | L | `1/0/5` cmd · `1/4/5` status | Büro/Arbeiten |
| `SHU_F1_office` | Office | S | `2/2/5` pos · `2/0/5` move · `2/1/5` stop · `2/6/5` pos-status | Büro/Arbeiten |
| `LGT_F1_wc` | WC | L | `1/0/6` cmd · `1/4/6` status | WC |
| `SHU_F1_wc` | WC | S | `2/2/6` pos · `2/0/6` move · `2/1/6` stop · `2/6/6` pos-status | WC |
| `LGT_F1_outside` | Outside | L | `1/0/10` cmd · `1/4/10` status | Terrasse/exterior |

---

## F2 — OG (Obergeschoss / Upper Floor)

```
 ┌──────────────────────────────────────────────────────────────┐
 │                                                              │
 │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
 │  │     Bad      │  │    Hannah    │  │   Nicolas    │       │
 │  │  (Bathroom)  │  │  (Child 1)   │  │  (Child 2)   │       │
 │  │              │  │              │  │              │       │
 │  │  L(Sink)     │  │  L           │  │  L           │       │
 │  │  L(Shower)   │  │  S           │  │  S           │       │
 │  │  S           │  │              │  │              │       │
 │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
 │         │                 │                  │               │
 │  ┌──────┴─────────────────┴──────────────────┘               │
 │  │           Flur (Hall)                                      │
 │  │           L(Stairs)  L(Window)  S                         │
 │  └──────────────────────┬─────────────────────────           │
 │                         │                                    │
 │              ┌──────────┴───────┐                            │
 │              │    Schlafzimmer  │                            │
 │              │    (Bedroom)     │                            │
 │              │                  │                            │
 │              │  L   P   S       │                            │
 │              └──────────────────┘                            │
 └──────────────────────────────────────────────────────────────┘
```

| FHEM Device | Alias | Type | KNX Group Addresses | Notes |
|-------------|-------|------|---------------------|-------|
| `LGT_F2_hall_stairs` | Hall (Stairs) | L | `1/0/101` cmd · `1/4/101` status | Flur — staircase |
| `LGT_F2_hall_window` | Hall (Window) | L | `1/0/107` cmd · `1/4/107` status | Flur — window side |
| `SHU_F2_hall` | Hall | S | `2/2/101` pos · `2/0/101` move · `2/1/101` stop · `2/6/101` pos-status | Flur |
| `LGT_F2_bathroom` | Bath Room (Sink) | L | `1/0/102` cmd · `1/4/102` status | Bad — sink area |
| `LGT_F2_shower` | Bath Room (Shower) | L | `1/0/106` cmd · `1/4/106` status | Bad — shower area |
| `SHU_F2_bathroom` | Bath Room | S | `2/2/102` pos · `2/0/102` move · `2/1/102` stop · `2/6/102` pos-status | Bad |
| `LGT_F2_hannah` | Hannah | L | `1/0/103` cmd · `1/4/103` status | Hannah's room |
| `SHU_F2_hannah` | Hannah | S | `2/2/103` pos · `2/0/103` move · `2/1/103` stop · `2/6/103` pos-status | Hannah's room |
| `LGT_F2_nicolas` | Nicolas | L | `1/0/104` cmd · `1/4/104` status | Nicolas's room |
| `SHU_F2_nicolas` | Nicolas | S | `2/2/104` pos · `2/0/104` move · `2/1/104` stop · `2/6/104` pos-status | Nicolas's room |
| `LGT_F2_bedroom` | Bed Room | L | `1/0/105` cmd · `1/4/105` status | Schlafzimmer |
| `PWR_F2_bedroom` | Bed Room | P | `1/0/108` cmd · `1/4/108` status | Schlafzimmer power socket (stromfrei) |
| `SHU_F2_bedroom` | Bed Room | S | `2/2/105` pos · `2/0/105` move · `2/1/105` stop · `2/6/105` pos-status | Schlafzimmer |

---

## Device Summary

| Floor | Room | Lights | Shutters | Other |
|-------|------|--------|----------|-------|
| F0 UG | Keller I (Technik/C1) | 1 | 1 | |
| F0 UG | Keller II (C2) | 1 | — | |
| F0 UG | Keller III (Büro/C3) | 1 | 1 | |
| F0 UG | Kellertreppe | 1 | — | |
| **F0 total** | | **4** | **2** | |
| F1 EG | Flur | 2 | — | |
| F1 EG | Küche | 2 | 2 (door + window) | |
| F1 EG | Esszimmer | 1 | 1 | |
| F1 EG | Wohnzimmer | 1 | 1 | |
| F1 EG | Büro/Arbeiten | 1 | 1 | |
| F1 EG | WC | 1 | 1 | |
| F1 EG | Terrasse | 1 | — | |
| **F1 total** | | **9** | **6** | |
| F2 OG | Flur | 2 | 1 | |
| F2 OG | Bad | 2 | 1 | |
| F2 OG | Hannah | 1 | 1 | |
| F2 OG | Nicolas | 1 | 1 | |
| F2 OG | Schlafzimmer | 1 | 1 | 1× power socket |
| **F2 total** | | **7** | **5** | **1 socket** |
| **Grand total** | | **20 lights** | **13 shutters** | **1 socket + 1 scene** |

---

## KNX Group Address Map

### Main Group 1 — Lights (cmd: `1/0/y`, status: `1/4/y`)

| Sub (`y`) | Device | Room | Floor |
|-----------|--------|------|-------|
| 1 | `LGT_F1_hall_door` | Hall (Door) | EG |
| 2 | `LGT_F1_kitchen` | Kitchen | EG |
| 3 | `LGT_F1_diningroom` | Dining Room | EG |
| 4 | `LGT_F1_livingroom` | Living Room | EG |
| 5 | `LGT_F1_office` | Office | EG |
| 6 | `LGT_F1_wc` | WC | EG |
| 7 | `LGT_F1_kitchen_led` | Kitchen LED | EG |
| 8 | `LGT_F1_hall_stairs` | Hall (Stairs) | EG |
| 10 | `LGT_F1_outside` | Outside/Terrasse | EG |
| 101 | `LGT_F2_hall_stairs` | Hall (Stairs) | OG |
| 102 | `LGT_F2_bathroom` | Bathroom (Sink) | OG |
| 103 | `LGT_F2_hannah` | Hannah | OG |
| 104 | `LGT_F2_nicolas` | Nicolas | OG |
| 105 | `LGT_F2_bedroom` | Bedroom | OG |
| 106 | `LGT_F2_shower` | Bathroom (Shower) | OG |
| 107 | `LGT_F2_hall_window` | Hall (Window) | OG |
| 108 | `PWR_F2_bedroom` | Bedroom power socket | OG |
| 201 | `LGT_F0_C1` | Keller I | UG |
| 202 | `LGT_F0_C2` | Keller II | UG |
| 203 | `LGT_F0_C3` | Keller III | UG |
| 205 | `LGT_F0_stairs` | Kellertreppe | UG |

### Main Group 2 — Shutters (pos: `2/2/y`, move: `2/0/y`, stop: `2/1/y`, pos-status: `2/6/y`)

| Sub (`y`) | Device | Room | Floor |
|-----------|--------|------|-------|
| 0 | `SHU_F1_all` | All EG shutters | EG central |
| 1 | `SHU_F1_kitchen_window` | Kitchen (Window) | EG |
| 2 | `SHU_F1_kitchen_door` | Kitchen (Door) | EG |
| 3 | `SHU_F1_diningroom` | Dining Room | EG |
| 4 | `SHU_F1_livingroom` | Living Room | EG |
| 5 | `SHU_F1_office` | Office | EG |
| 6 | `SHU_F1_wc` | WC | EG |
| 101 | `SHU_F2_hall` | Hall | OG |
| 102 | `SHU_F2_bathroom` | Bathroom | OG |
| 103 | `SHU_F2_hannah` | Hannah | OG |
| 104 | `SHU_F2_nicolas` | Nicolas | OG |
| 105 | `SHU_F2_bedroom` | Bedroom | OG |
| 200 | `SHU_F0_all` | All UG shutters | UG central |
| 201 | `SHU_F0_C1` | Keller I | UG |
| 203 | `SHU_F0_C3` | Keller III | UG |

### Main Group 5 — Scenes

| Address | Device | Description |
|---------|--------|-------------|
| `5/0/000` | `SCN_Away` | Away mode command |
| `5/4/000` | `SCN_Away` | Away mode status |
