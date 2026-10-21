# EnergyPlus simulation

The simulation stack is defined in `docker-compose-sim.yml`. It starts PostgreSQL and a minimal VOLTTRON platform with the SQL historian and the EnergyPlus agent. The EnergyPlus input model and generated output files live under `simulation-data/building1` on the host. The inputs are copied from the agent's example model before each start so EnergyPlus can modify its working copy without changing the source model.

## Start

From this directory, start the simulation stack:

```sh
mkdir -p simulation-data/building1
cp agents/EnergyplusAgent/eplus_config/building1/BUILDING1.idf simulation-data/building1/
cp agents/EnergyplusAgent/eplus_config/building1/USA_WA_Pasco-Tri.Cities.AP.727845_TMY3.epw simulation-data/building1/
LOCAL_USER_ID="$(id -u)" docker compose -f docker-compose-sim.yml up --build -d postgres volttron1
```

Follow VOLTTRON startup and agent logs:

```sh
docker compose -f docker-compose-sim.yml logs -f volttron1
```

Wait for platform setup and the EnergyPlus agent to finish installing and start before running `vctl status`; checking status during the first-time installer can interfere with setup. The expected startup includes the historian and EnergyPlus agents, then a message that the EnergyPlus process is running.

When updating the EnergyPlus agent source after it has already been installed, reinstall the package so VOLTTRON uses the updated code:

```sh
docker exec -itu volttron volttron1 vctl remove --tag energyplus
docker exec -itu volttron volttron1 vctl install /home/volttron/energyplus-agent \
  --agent-config /home/volttron/configs/energyplus.config \
  --tag energyplus --vip-identity energyplus --start
```

## Verify simulation and historian data

The configured run covers August 1–31 at a 90× time scale, so the full simulation is expected to take about eight hours. Successful completion should leave `simulation-data/building1/eplusout.csv`.

Check that the CSV exists and has rows:

```sh
ls -lh simulation-data/building1/eplusout.csv
head -n 3 simulation-data/building1/eplusout.csv
```

Check that PostgreSQL is receiving EnergyPlus topics:

```sh
docker compose -f docker-compose-sim.yml exec postgres \
  psql -U volttron -d volttron -c 'SELECT COUNT(*) FROM data;'
```

Inspect recent values and confirm the topic names come from EnergyPlus:

```sh
docker compose -f docker-compose-sim.yml exec postgres \
  psql -U volttron -d volttron -c \
  'SELECT t.topic_name, d.ts, to_jsonb(d) AS stored_row FROM data d JOIN topics t USING (topic_id) ORDER BY d.ts DESC LIMIT 20;'
```

The total row count should grow while the simulation runs. To count just the EnergyPlus points:

```sh
docker compose -f docker-compose-sim.yml exec postgres \
  psql -U volttron -d volttron -c \
  "SELECT COUNT(*) FROM data d JOIN topics t USING (topic_id) WHERE t.topic_name LIKE 'devices/PNNL/BUILDING1/%';"
```

That count should grow, and the recent topic query above should show `devices/PNNL/BUILDING1/...` values.
The PostgreSQL volume may contain older `fake-campus/...` rows from a previous deployment; those do not count as EnergyPlus data.

## Stop

Stop the simulation without deleting its database volume:

```sh
docker compose -f docker-compose-sim.yml stop volttron1
```

Use `docker compose ... down -v` only if you intentionally want to delete persisted database data.
