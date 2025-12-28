# EMLOX
EMHASS Battery Control for Loxone &amp; Sungrow

Dynamic battery optimization with EMHASS, Tibber dynamic pricing, and Solcast PV forecast.

🇩🇪 [Deutsche Version unten](#deutsche-version)

---

## What does this system do?

EMHASS (Energy Management for Home Assistant) optimizes battery usage based on:
- **Dynamic electricity prices** (Tibber)
- **PV forecast** (Solcast)
- **Load forecast** (historical consumption data)
- **Battery costs** (degradation + efficiency losses)

### Automatic decisions

| Situation | Action |
|-----------|--------|
| Very cheap electricity price | Charge battery from grid |
| PV surplus available | Charge battery with PV (Sungrow handles this automatically) |
| High price, battery full | Use battery to power house |
| Expensive prices expected | Hold battery for later |

### Important: Sungrow stays in Self-Consumption Mode

The Sungrow inverter remains in **Self-Consumption Mode** by default:
- PV surplus is automatically stored in battery
- House load is automatically supplied from battery
- **EMHASS only intervenes when grid charging is profitable!**

Forced Charge (grid charging) is only activated when EMHASS calculates that the price advantage exceeds battery costs (degradation + losses).

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES                                │
├─────────────────────────────────────────────────────────────────────┤
│  Tibber API          Solcast (Loxberry)      Loxone Miniserver      │
│  (electricity prices) (PV forecast)          (real-time data)       │
└────────┬─────────────────────┬───────────────────────┬──────────────┘
         │                     │                       │
         │                     │ MQTT                  │ MQTT
         │                     ▼                       ▼
         │              ┌─────────────────────────────────────┐
         │              │         MQTT Broker                 │
         │              │        (Loxberry)                   │
         │              └─────────────────┬───────────────────┘
         │                                │
         ▼                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          NODE-RED                                   │
│                      (Orchestration)                                │
├─────────────────────────────────────────────────────────────────────┤
│  • Collects Solcast PV forecast                                     │
│  • Collects battery SOC from Loxone                                 │
│  • Fetches Tibber prices via API                                    │
│  • Triggers EMHASS (Day-Ahead + MPC)                                │
│  • Translates results into battery commands                         │
│  • Publishes control commands via MQTT → Loxone                     │
└─────────────────────────────────────────────────────────────────────┘
         │                                │
         │ HTTP API                       │ MQTT
         ▼                                ▼
┌─────────────────────┐          ┌─────────────────────┐
│       EMHASS        │          │   Loxone Miniserver │
├─────────────────────┤          ├─────────────────────┤
│ • Optimization      │          │ • Receives commands │
│ • Day-Ahead (24h)   │          │ • Status blocks     │
│ • MPC (5 min)       │          │ • Modbus → Sungrow  │
└─────────────────────┘          └──────────┬──────────┘
         │                                  │
         │ Sensor data                      │ Modbus TCP
         ▼                                  ▼
┌─────────────────────┐          ┌─────────────────────┐
│   Home Assistant    │          │   Sungrow SHxxRT    │
├─────────────────────┤          ├─────────────────────┤
│ • MQTT → HA sensors │          │ • Hybrid inverter   │
│ • HA → InfluxDB     │          │ • Battery storage   │
└─────────────────────┘          └─────────────────────┘
         │
         ▼
┌─────────────────────┐
│     InfluxDB 2      │
├─────────────────────┤
│ • Historical data   │
│ • For ML training   │
│   (future)          │
└─────────────────────┘
```

---

## Day-Ahead vs. MPC - Two Optimization Stages

This confused me at first: EMHASS works with **two optimization stages**.

### Day-Ahead Optimization (daily at 05:30)

- **Planning horizon:** 24 hours
- **Purpose:** Rough schedule for the day
- **Data:** Tibber prices (today + tomorrow), Solcast PV forecast, historical load
- **Result:** Optimal battery schedule

**Day-Ahead alone does NOT control the battery!** It only creates the plan.

### MPC Optimization (every 5 minutes)

- **Planning horizon:** 12 hours
- **Purpose:** Fine-tuning based on actual state
- **Data:** Current SOC, current load, current PV
- **Result:** Actual control commands

**MPC is the operational control!** It corrects the Day-Ahead plan based on reality.

### Why both?

| Aspect | Day-Ahead | MPC |
|--------|-----------|-----|
| When | Once daily | Every 5 min |
| Horizon | 24h | 12h |
| Accuracy | Rough (forecast) | Fine (actual values) |
| Purpose | Strategy | Tactics |

**Example:** Day-Ahead plans at 05:30: "Start charging at 14:00". At 13:55, MPC detects: "PV delivering less than expected, SOC only 20% instead of 30%". MPC adjusts: "Start charging at 13:30".

---

## Requirements

### Hardware
- Loxone Miniserver (Gen1 or Gen2)
- Hybrid inverter with Modbus TCP (tested: Sungrow SH5RT/SH6RT/SH8RT/SH10RT)
- Battery storage (tested: BYD HVS/HVM)
- PV system
- Docker-capable server (Unraid, Synology, Raspberry Pi 4+, etc.)

### Software & Accounts
- Docker host
- Tibber account with API token
- Solcast account (free for hobbyists)
- LoxBerry with MQTT plugin and Solcast plugin

### Tested Versions
- LoxBerry 3.0.1.3
- PV Solcast Plugin 0.6.1 (by Dieter Schmidberger)
- Unraid 7.2.2
- EMHASS 0.15.1
- Node-RED 4.1.2
- Home Assistant 2025.12.1
- InfluxDB 2.8

---

## Installation

### 1. InfluxDB 2

```bash
docker run -d \
  --name influxdb \
  --restart unless-stopped \
  -p 8086:8086 \
  -v /path/to/influxdb/data:/var/lib/influxdb2 \
  -v /path/to/influxdb/config:/etc/influxdb2 \
  -e DOCKER_INFLUXDB_INIT_MODE=setup \
  -e DOCKER_INFLUXDB_INIT_USERNAME=admin \
  -e DOCKER_INFLUXDB_INIT_PASSWORD=<YOUR_PASSWORD> \
  -e DOCKER_INFLUXDB_INIT_ORG=home \
  -e DOCKER_INFLUXDB_INIT_BUCKET=smarthome \
  -e DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=<YOUR_TOKEN> \
  influxdb:latest
```

**Enable InfluxDB v1 compatibility for EMHASS:**

```bash
docker exec -it influxdb bash
influx bucket list  # Note the bucket ID
influx v1 auth create \
  --username emhass \
  --password <YOUR_PASSWORD> \
  --org home \
  --read-bucket <BUCKET_ID> \
  --write-bucket <BUCKET_ID>
```

### 2. Home Assistant

```bash
docker run -d \
  --name homeassistant \
  --restart unless-stopped \
  --privileged \
  -v /path/to/homeassistant/config:/config \
  -v /etc/localtime:/etc/localtime:ro \
  --network host \
  ghcr.io/home-assistant/home-assistant:stable
```

### 3. EMHASS

```bash
docker run -d \
  --name emhass \
  --restart unless-stopped \
  -p 5000:5000 \
  -v /path/to/emhass/data:/data \
  -v /path/to/emhass/share:/share \
  -e TZ=Europe/Berlin \
  -e LOCAL_COSTFUN=profit \
  ghcr.io/davidusb-geek/emhass:latest
```

### 4. Node-RED

```bash
docker run -d \
  --name nodered \
  --restart unless-stopped \
  -p 1880:1880 \
  -v /path/to/nodered:/data \
  -v /path/to/emhass/data:/emhass/data:rw \
  -e TZ=Europe/Berlin \
  -u root \
  nodered/node-red
```

**Important:** The EMHASS data directory is mounted into Node-RED (`/emhass/data`) so Node-RED can read the CSV results!

---

## Configuration

### Configuration Files

| File | Description |
|------|-------------|
| `config/emhass.json` | EMHASS main configuration |
| `config/homeassistant.yaml` | Home Assistant configuration.yaml |
| `config/influxdb.yaml` | Home Assistant InfluxDB connection |
| `nodered/flow.json` | Node-RED flow |
| `docs/loxone_mqtt_topics.md` | MQTT topics for Loxone |
| `docs/sungrow_modbus.md` | Sungrow Modbus register reference |

### EMHASS config.json - What needs to be changed?

The `config.json` contains many placeholders since most data is passed at **runtime** (PV forecast, prices, SOC). However, some values must be adapted to your system.

#### Must be adapted:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `battery_nominal_energy_capacity` | Usable battery capacity in Wh | 8000 |
| `battery_discharge_power_max` | Max discharge power in W | 5000 |
| `battery_charge_power_max` | Max charge power in W | 5000 |
| `photovoltaic_production_sell_price` | Feed-in tariff €/kWh | 0.0711 |
| `influxdb_*` | InfluxDB credentials | - |

#### Important for Sungrow: Relative SOC!

Sungrow reports SOC **relative** between Min-SOC and Max-SOC:
- Register 13059: Min SOC (e.g., 20%)
- Register 13058: Max SOC (e.g., 90%)
- Register 13023: Current SOC (always 0-100% between Min and Max!)

**Example:** With Min=20%, Max=90%, SOC=50% means an actual charge level of ~55%.

#### Battery costs - Why important?

```json
"weight_battery_discharge": 0.01,
"weight_battery_charge": 0.10
```

These values represent **costs per kWh** for battery usage:
- Degradation: ~8-10 cents/kWh (based on cycle costs)
- Efficiency losses: ~5% per direction

With `weight_battery_charge: 0.10` (10 cents), EMHASS only charges from grid when the price advantage exceeds 10 cents. This prevents unprofitable arbitrage trading!

### Node-RED Configuration

In the flow, adapt the following variables in the **Configuration Node** (at the top):

```javascript
const CONFIG = {
    // IP addresses
    EMHASS_IP: "emhass-ip",
    EMHASS_PORT: 5000,
    
    // Tibber API
    TIBBER_TOKEN: "<YOUR_TIBBER_TOKEN>",
    
    // Feed-in tariff
    FEED_IN_TARIFF: 0.0711,
    
    // Timezone
    TIMEZONE: "+01:00"
};
```

---

## Roadmap & Future Plans

### Current Status

✅ Day-Ahead + MPC optimization running  
✅ Tibber prices fetched  
✅ Solcast PV forecast used  
✅ Battery control via Loxone → Modbus → Sungrow  
✅ Historical data collected in InfluxDB  

### Planned

#### 1. Telegraf Integration
Direct writing from Loxone → MQTT → Telegraf → InfluxDB without Home Assistant as intermediate layer.

#### 2. Remove Home Assistant
EMHASS can read directly from InfluxDB. Home Assistant will no longer be needed.

#### 3. Machine Learning Load Forecast
EMHASS supports ML-based load forecasts. Historical data is already being collected! The ML model can be trained later for better predictions.

---

## License

MIT License - see [LICENSE](LICENSE)

## Credits

- [EMHASS](https://github.com/davidusb-geek/emhass) by davidusb-geek
- Loxone Community 
- LoxBerry Team
- ChatGPT & ClaudeAI

---

---

# Deutsche Version

## Was macht dieses System?

EMHASS (Energy Management for Home Assistant) optimiert die Batterienutzung basierend auf:
- **Dynamischen Strompreisen** (Tibber)
- **PV-Prognose** (Solcast)
- **Lastprognose** (historische Verbrauchsdaten)
- **Batteriekosten** (Degradation + Wirkungsgradverluste)

### Automatische Entscheidungen

| Situation | Aktion |
|-----------|--------|
| Strompreis sehr günstig | Batterie aus Netz laden |
| PV-Überschuss vorhanden | Batterie mit PV laden (Sungrow regelt automatisch) |
| Hoher Preis, Batterie voll | Batterie für Hausversorgung nutzen |
| Teurer Strom erwartet | Batterie für später halten |

### Wichtig: Sungrow bleibt im Self-Consumption Modus

Der Sungrow Wechselrichter bleibt standardmäßig im **Self-Consumption Modus**:
- PV-Überschuss wird automatisch in die Batterie geladen
- Hauslast wird automatisch aus der Batterie versorgt
- **EMHASS greift nur ein, wenn Netzladung profitabel ist!**

Forced Charge (Netzladung) wird nur aktiviert, wenn EMHASS berechnet hat, dass der Preisvorteil die Batteriekosten (Degradation + Verluste) übersteigt.

---

## Day-Ahead vs. MPC - Zwei Optimierungsstufen

Das hat mich anfangs verwirrt: EMHASS arbeitet mit **zwei Optimierungsstufen**.

### Day-Ahead Optimierung (täglich 05:30)

- **Planungshorizont:** 24 Stunden
- **Zweck:** Grober Fahrplan für den Tag
- **Daten:** Tibber-Preise, Solcast PV-Prognose, historische Last
- **Ergebnis:** Optimaler Batteriefahrplan

**Day-Ahead alleine steuert NICHT die Batterie!** Es erstellt nur den Plan.

### MPC Optimierung (alle 5 Minuten)

- **Planungshorizont:** 12 Stunden
- **Zweck:** Feinjustierung basierend auf Ist-Zustand
- **Daten:** Aktueller SOC, aktuelle Last, aktuelle PV
- **Ergebnis:** Konkrete Steuerbefehle

**MPC ist die operative Steuerung!** Es korrigiert den Day-Ahead Plan anhand der Realität.

---

## Voraussetzungen

### Hardware
- Loxone Miniserver (Gen1 oder Gen2)
- Hybrid-Wechselrichter mit Modbus TCP (getestet: Sungrow SHxxRT)
- Batteriespeicher (getestet: BYD HVS/HVM)
- PV-Anlage
- Docker-fähiger Server (Unraid, Synology, Raspberry Pi 4+)

### Software & Accounts
- Docker Host
- Tibber-Account mit API-Token
- Solcast-Account (kostenlos für Hobbyisten)
- LoxBerry mit MQTT Plugin und Solcast Plugin

### Getestete Versionen
- LoxBerry 3.0.1.3
- PV Solcast Plugin 0.6.1 (by Dieter Schmidberger)
- Unraid 7.2.2
- EMHASS 0.15.1
- Node-RED 4.1.2
- Home Assistant 2025.12.1
- InfluxDB 2.8

---

## Konfiguration

### EMHASS config.json - Was muss angepasst werden?

Die `config.json` enthält viele Platzhalter, da die meisten Daten zur **Laufzeit** übergeben werden (PV-Prognose, Preise, SOC). Trotzdem müssen einige Werte angepasst werden.

#### Unbedingt anpassen:

| Parameter | Beschreibung | Beispiel |
|-----------|--------------|----------|
| `battery_nominal_energy_capacity` | Nutzbare Batteriekapazität in Wh | 8000 |
| `battery_discharge_power_max` | Max. Entladeleistung in W | 5000 |
| `battery_charge_power_max` | Max. Ladeleistung in W | 5000 |
| `photovoltaic_production_sell_price` | Einspeisevergütung €/kWh | 0.0711 |
| `influxdb_*` | InfluxDB Zugangsdaten | - |

#### Wichtig bei Sungrow: Relativer SOC!

Sungrow gibt den SOC **relativ** zwischen Min-SOC und Max-SOC an:
- Register 13059: Min SOC (z.B. 20%)
- Register 13058: Max SOC (z.B. 90%)
- Register 13023: Aktueller SOC (immer 0-100% zwischen Min und Max!)

---

## Roadmap

### Aktueller Status

✅ Day-Ahead + MPC Optimierung läuft  
✅ Tibber Preise werden abgerufen  
✅ Solcast PV-Prognose wird verwendet  
✅ Batteriesteuerung via Loxone → Modbus → Sungrow  
✅ Historische Daten werden in InfluxDB gesammelt  

### Geplant

1. **Telegraf Integration** - Direktes Schreiben ohne Home Assistant
2. **Home Assistant entfernen** - EMHASS liest direkt aus InfluxDB
3. **Machine Learning Lastprognose** - Daten werden bereits gesammelt!

---

## Lizenz

MIT License - siehe [LICENSE](LICENSE)
