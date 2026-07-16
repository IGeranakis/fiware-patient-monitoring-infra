# FIWARE Device Simulator — Remote Patient Monitoring

Virtual medical devices for an IoT infrastructure that remotely monitors
patients' vital signs, built on the [FIWARE](https://www.fiware.org/) platform.

This repository contains the
[FIWARE Device Simulator](https://github.com/telefonicaid/fiware-device-simulator)
by Telefónica I+D (see `LICENSE`), together with the configuration and helper
files developed for this project. All credit for the simulator itself goes to
the original authors; the project-specific additions are listed below.

## What it does

For every patient, three virtual devices are simulated, each sending
measurements every 5 seconds to an Orion Context Broker (NGSI-v2):

| Virtual device        | Entity type            | Measurements                                |
|-----------------------|------------------------|---------------------------------------------|
| Blood pressure monitor| `BloodPressureMonitor` | `systolicPressure`, `diastolicPressure` (mmHg) |
| Pulse oximeter        | `Oximeter`             | `oxygenSaturation` (%), `pulseRate` (bpm)   |
| Holter monitor        | `Holter`               | `heartRate` (bpm), `rrInterval` (s)         |

Entities are named `<PatientId>:<DeviceType>` (e.g. `Patient001:Oximeter`).
Generated values follow a circadian pattern (lower heart rate and blood
pressure at night) using the simulator's `time-random-linear-interpolator`.

Measurements flow through the FIWARE pipeline and end up in MySQL:

```
Simulator → Orion (1026) → Cygnus (5050) → MySQL (hospital.vitals)
                                              ↑
                        Doctors' web application (Node.js / React)
```

## Project files

- `simulation.json` — the simulation configuration (FIWARE service
  `hospital`, service path `/vitals`, devices as described above).
- `docker-compose.yml` — the supporting infrastructure: Orion Context
  Broker, MongoDB, Cygnus, MySQL. MySQL is exposed on host port **3307**.
- `subscription-cygnus.json` — the Orion subscription that forwards every
  entity update to Cygnus for persistence in MySQL.

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Node.js](https://nodejs.org/) (LTS)
- `nodemon` for auto-restarting the simulator: `npm install -g nodemon`

## Setup

1. Install the simulator's dependencies:

   ```bash
   npm install
   ```

2. Start the FIWARE infrastructure:

   ```bash
   docker compose up -d
   docker compose ps        # wait until all 4 containers are running
   ```

3. Create the Orion → Cygnus subscription (one-time step; repeat only after
   the containers are removed with `docker compose down`):

   ```bash
   curl -X POST http://localhost:1026/v2/subscriptions \
     -H "Content-Type: application/json" \
     -H "fiware-service: hospital" \
     -H "fiware-servicepath: /vitals" \
     -d "@subscription-cygnus.json"
   ```

## Running the simulator

One-off run:

```bash
node bin/fiwareDeviceSimulatorCLI -c simulation.json
```

Recommended (auto-restarts whenever the configuration file changes, e.g.
when the web application registers a new patient and regenerates it):

```bash
nodemon --watch simulation.json --exec "node bin/fiwareDeviceSimulatorCLI -c simulation.json"
```

## Verifying the pipeline

Current state in Orion:

```bash
curl http://localhost:1026/v2/entities -H "fiware-service: hospital" -H "fiware-servicepath: /vitals"
```

Persisted history in MySQL:

```bash
docker compose exec mysql mysql -uroot -psecret \
  -e "SELECT recvTime, entityId, attrName, attrValue FROM hospital.vitals ORDER BY recvTime DESC LIMIT 10;"
```

## Troubleshooting

- **`ECONNREFUSED` on port 1026** — Orion is not running. Start Docker
  Desktop and run `docker compose up -d`.
- **Orion logs "A broker seems to be running already"** — stale PID file
  after an unclean shutdown. Fix: `docker compose up -d --force-recreate orion`.
- **Data reaches Orion but not MySQL** — the subscription is missing
  (removed together with the containers by `docker compose down`).
  Re-create it as in Setup step 3.
- **Port 3306 already in use** — a local MySQL installation conflicts with
  the container; this project maps the container to host port 3307 instead.

## Related repository

The doctors' web application (Node.js/Express backend, React frontend,
MySQL) that consumes these measurements is maintained separately.

## License and attribution

The FIWARE Device Simulator is © Telefónica Investigación y Desarrollo and
is distributed under the license in `LICENSE`. This repository redistributes
it unchanged, adding only the configuration and documentation files listed
above.