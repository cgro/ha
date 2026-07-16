# KNX Group Address Map

Complete KNX addressing scheme for this installation — physical address space,
three-level GA structure, main/middle group assignments, and full sub-address tables.

---

## 1. Physical Address Space

KNX individual addresses use the format `area.line.device`:

| Area | Line | Device range | Location |
|------|------|--------------|----------|
| 1    | 1    | 0 – 19       | KG (Keller / Basement) |
| 1    | 1    | 20 – 99      | EG (Erdgeschoss / Ground floor) |
| 1    | 1    | 100 – 149    | OG (Obergeschoss / Upper floor) |
| 1    | 1    | 150 – 199    | reserved |
| 1    | 1    | 200 – 255    | HVT (Hauptverteilung / Distribution cabinet) |

---

## 2. Group Address Structure

Group addresses use a three-level hierarchy `main / middle / sub`:

```
  x  /  y  /  z
  │     │     └── Untergruppe  (sub,  0–255) :  target — room or device
  │     └──────── Mittelgruppe (mid,  0–7  ) :  function — command type
  └────────────── Hauptgruppe  (main, 0–31 ) :  system — trade (Gewerk)
```

### Bit encoding on the wire (16-bit address)

```
 15        11 10       8 7             0
 ┌──────────┬──────────┬───────────────┐
 │  main    │  middle  │      sub      │
 │  5 bits  │  3 bits  │    8 bits     │
 └──────────┴──────────┴───────────────┘
   0 – 31     0 – 7        0 – 255
```

---

## 3. Main Groups (Hauptgruppe)

| Value | System     | Description |
|-------|------------|-------------|
| 0     | Zentral    | Central / system-wide |
| 1     | Licht      | Lighting |
| 2     | Jalousie   | Shutters / blinds |
| 3     | Heizung    | Heating |
| 4     | Sicherheit | Security |
| 5     | Szenen     | Scenes |

---

## 4. Middle Groups (Mittelgruppe)

Function codes per main group:

| Value | Licht (1)          | Jalousie (2)              | Sicherheit (4)       |
|-------|--------------------|---------------------------|----------------------|
| 0     | schalten (switch)  | auf/ab (move up/down)     | Bewegung (motion)    |
| 1     | —                  | stop                      | —                    |
| 2     | —                  | Position (setpoint)       | —                    |
| 3     | —                  | —                         | —                    |
| 4     | Status             | Endlage oben (end up)     | —                    |
| 5     | —                  | Endlage unten (end down)  | —                    |
| 6     | —                  | Position (status)         | —                    |
| 7     | n/a                | —                         | —                    |

---

## 5. Sub-address Tables (Untergruppe)

### 5.1 Licht — main group 1

| GA function  | Format  |
|--------------|---------|
| Switch cmd   | `1/0/z` |
| Status       | `1/4/z` |

| z   | Floor | Room / Device |
|-----|-------|---------------|
| **EG (Ground floor) 0 – 99** |||
| 000 | EG    | komplett — all EG lights (central) |
| 001 | EG    | Flur (Hall) |
| 002 | EG    | Küche, Haupt (Kitchen main) |
| 003 | EG    | Essen (Dining room) |
| 004 | EG    | Wohnen (Living room) |
| 005 | EG    | Arbeiten (Office) |
| 006 | EG    | WC |
| 007 | EG    | Küche, Neben / Ambient (Kitchen LED) |
| 008 | EG    | Flur-Treppe (Hall stairs) |
| 010 | EG    | Terrasse, außen (Outside) |
| **OG (Upper floor) 100 – 199** |||
| 100 | OG    | komplett — all OG lights (central) |
| 101 | OG    | Flur (Hall) |
| 102 | OG    | Bad (Bathroom) |
| 103 | OG    | Hannah |
| 104 | OG    | Nicolas |
| 105 | OG    | Schlafen (Bedroom) |
| 106 | OG    | Bad-2, Dusche (Bathroom shower) |
| 107 | OG    | Flur-2, Fenster (Hall window) |
| 108 | OG    | stromfrei Schlafen (Bedroom switched socket) |
| 109 | OG    | stromfrei Nicolas |
| 110 | OG    | stromfrei Hannah |
| **KG (Basement) 200 – 255** |||
| 200 | KG    | komplett — all KG lights (central) |
| 201 | KG    | Keller I (Technik) |
| 202 | KG    | Keller II |
| 203 | KG    | Keller III (Büro) |
| 204 | KG    | außen (exterior) |
| 205 | KG    | Kellertreppe (Basement stairs) |

---

### 5.2 Jalousie — main group 2

| GA function      | Format  | DPT      |
|------------------|---------|----------|
| Move up/down     | `2/0/z` | dpt1.008 |
| Stop             | `2/1/z` | dpt1.008 |
| Position setpoint| `2/2/z` | dpt5.001 |
| End position up  | `2/4/z` | dpt1.011 |
| End position down| `2/5/z` | dpt1.011 |
| Position status  | `2/6/z` | dpt5.001 |

> **Correction (2026-07-16):** `2/4/z` and `2/5/z` were originally declared as `dpt1.008`
> (Up/Down) in `device_config.cfg`. That datapoint's label semantics (0→"up", 1→"down")
> don't match what these GAs actually carry — a boolean "reached end position"
> confirmation, not a direction. Since FHEM's `state` reading is set from whichever GA
> value arrived last, the mislabeled confirmation could silently overwrite a correct
> position reading with the literal text "down" (observed on `SHU_F0_C1`, `SHU_F0_C3`,
> `SHU_F1_office` after moving fully up). Fixed by retyping both GAs to `dpt1.011`
> (inactive/active) and pinning `state` to the position-status GA (`2/6/z`, 0–100 %) via
> a `stateRegex` attribute, applied to all 13 individually-addressable shutters across
> F0/F1/F2. The three floor-wide broadcast devices (`SHU_F0_all`, `SHU_F1_all`,
> `SHU_F2_all`) only define `2/0/z`/`2/1/z` and were never affected.

| z   | Floor | Room / Device |
|-----|-------|---------------|
| **EG (Ground floor) 0 – 99** |||
| 000 | EG    | komplett — all EG shutters (central) |
| 001 | EG    | Küchenfenster (Kitchen window) |
| 002 | EG    | Küche, Haupt (Kitchen door) |
| 003 | EG    | Essen (Dining room) |
| 004 | EG    | Wohnen (Living room) |
| 005 | EG    | Arbeiten (Office) |
| 006 | EG    | WC |
| **OG (Upper floor) 100 – 199** |||
| 100 | OG    | komplett — all OG shutters (central) |
| 101 | OG    | Flur (Hall) |
| 102 | OG    | Bad (Bathroom) |
| 103 | OG    | Hannah |
| 104 | OG    | Nicolas |
| 105 | OG    | Schlafen (Bedroom) |
| **KG (Basement) 200 – 255** |||
| 200 | KG    | komplett — all KG shutters (central) |
| 201 | KG    | Keller I |
| 203 | KG    | Keller III |

---

### 5.3 Sicherheit — main group 4

| GA function | Format  |
|-------------|---------|
| Motion      | `4/0/z` |

| z   | Floor | Room / Device |
|-----|-------|---------------|
| **EG (Ground floor) 0 – 99** |||
| 001 | EG    | Flur (Hall) |

---

### 5.4 Szenen — main group 5

| z   | Floor    | Room / Device |
|-----|----------|---------------|
| 000 | Zentral  | SCN_Away — away mode cmd `5/0/000` · status `5/4/000` |
