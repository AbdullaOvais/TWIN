# ns-O-RAN Telemetry-Driven Handover Diagnosis

This repository is the local analysis companion for a real **ns-O-RAN handover** study.

The main idea is simple:

1. collect raw telemetry from a real ns-O-RAN / ns-3 experiment,
2. convert those raw traces into a shared KPI schema,
3. compare normal versus anomaly runs around the handover window,
4. train lightweight detectors, and
5. use a **rule-based** model selection for the given intent by operator to explain the fault and suggest handover tuning changes.



## What This Repo Contains

- `scripts/build_real_ns_o_ran_dataset.py`
  - converts raw ns-O-RAN trace bundles into the project CSV layout,
  - builds handover-window summaries,
  - and can optionally run the local diagnosis pipeline.
- `ns_oran_intent_selector/`
  - dataset helpers,
  - model training/evaluation,
  - a small model catalog,
  - and the rule-based selector.
- `README.md`
  - this guide.

The repository does **not** include the full ns-O-RAN simulator stack itself.
You run the simulator and RIC in your experiment environment, then feed the exported trace bundles into this repo.

## Prerequisites

- Python 3.10+ recommended
- `pip install -r requirements.txt`
- A real ns-O-RAN run that produces these raw files in each bundle directory:
  - `CellIdStats.txt`
  - `CellIdStatsHandover.txt`
  - `DlE2PdcpStats.txt`

## Raw Trace Bundle Layout

Each experiment bundle should look like this:

```text
normal/
  CellIdStats.txt
  CellIdStatsHandover.txt
  DlE2PdcpStats.txt

anomaly/
  CellIdStats.txt
  CellIdStatsHandover.txt
  DlE2PdcpStats.txt
```

The `normal` folder is your baseline run.
The `anomaly` folder is the misconfigured run, for example:

- lower hysteresis
- shorter time-to-trigger
- different `hoSinrDifference`

## Data Collection Workflow

The project collects data in two parts:

1. the **RIC side** in Docker, which shows the control and telemetry stack,
2. the **ns-3 side** on the host machine, which generates the raw trace files.

The raw telemetry comes from these files:

- `CellIdStats.txt`
- `CellIdStatsHandover.txt`
- `DlE2PdcpStats.txt`

These files are then copied into experiment folders and converted into CSV and KPI summaries.

### Terminal 1: Start and monitor the RIC stack

This terminal is used to make sure the RIC side is running.

```bash
docker start e2term e2mgr e2rtmansim db sample-xapp-24
docker logs -f e2term
```

Optional xApp logs:

```bash
docker logs -f sample-xapp-24
```

What this does:

- starts the Docker containers if they are stopped,
- shows E2 termination logs,
- proves that the RIC side is live.

### Terminal 2: Open the xApp container

This is optional for raw trace collection, but useful for a live demo.

```bash
docker exec -it sample-xapp-24 bash
cd /home/sample-xapp
./run_xapp.sh 2>&1 | tee /tmp/xapp_live.log
```

What this does:

- opens a shell inside the xApp container,
- lets you inspect the xApp environment,
- is useful if you want to show the RIC side interactively.

### Terminal 3: Run the ns-3 scenario for the normal case

This terminal runs the simulator and generates the raw telemetry files.

```bash
cd ~/Desktop/TWIN/ns-o-ran/ns-3-mmwave-oran
./ns3 run "scratch/scenario-zero.cc --e2TermIp=10.0.2.1 --enableE2FileLogging=1 --simTime=60 --hoSinrDifference=3 --hysteresis=3.0 --timeToTrigger=256"
```

What this does:

- runs the ns-3 / ns-O-RAN simulation,
- connects the simulator to `e2term`,
- generates the raw trace files in the simulator output folder.

For the normal run:

- `hoSinrDifference = 3`
- `Hysteresis = 3.0`
- `TimeToTrigger = 256`

### Terminal 4: Run the anomaly case

This terminal runs the same scenario again, but with one handover parameter changed.

Example 1: `hoSinrDifference` anomaly

```bash
cd ~/Desktop/TWIN/ns-o-ran/ns-3-mmwave-oran
./ns3 run "scratch/scenario-zero.cc --e2TermIp=10.0.2.1 --enableE2FileLogging=1 --simTime=60 --hoSinrDifference=18 --hysteresis=3.0 --timeToTrigger=256"
```

Example 2: hysteresis anomaly

```bash
cd ~/Desktop/TWIN/ns-o-ran/ns-3-mmwave-oran
./ns3 run "scratch/scenario-zero.cc --e2TermIp=10.0.2.1 --enableE2FileLogging=1 --simTime=60 --hoSinrDifference=3 --hysteresis=0.5 --timeToTrigger=256"
```

Example 3: time-to-trigger anomaly

```bash
cd ~/Desktop/TWIN/ns-o-ran/ns-3-mmwave-oran
./ns3 run "scratch/scenario-zero.cc --e2TermIp=10.0.2.1 --enableE2FileLogging=1 --simTime=60 --hoSinrDifference=3 --hysteresis=3.0 --timeToTrigger=40"
```

What this does:

- keeps the setup the same,
- changes only one parameter,
- lets you compare normal vs anomaly cleanly.

### Terminal 5: Save the raw trace files

After each run, copy the raw files immediately, because the next run can overwrite them.

#### Normal run

```bash
mkdir -p ~/Desktop/TWIN/ns-o-ran/results/exp1/normal
cp CellIdStats.txt ~/Desktop/TWIN/ns-o-ran/results/exp1/normal/
cp CellIdStatsHandover.txt ~/Desktop/TWIN/ns-o-ran/results/exp1/normal/
cp DlE2PdcpStats.txt ~/Desktop/TWIN/ns-o-ran/results/exp1/normal/
```

#### Anomaly run

```bash
mkdir -p ~/Desktop/TWIN/ns-o-ran/results/exp1/anomaly
cp CellIdStats.txt ~/Desktop/TWIN/ns-o-ran/results/exp1/anomaly/
cp CellIdStatsHandover.txt ~/Desktop/TWIN/ns-o-ran/results/exp1/anomaly/
cp DlE2PdcpStats.txt ~/Desktop/TWIN/ns-o-ran/results/exp1/anomaly/
```

You can repeat the same folder structure for:

- `exp2` for hysteresis,
- `exp3` for time-to-trigger.



## What The Project Measures

The pipeline focuses on deep telemetry for a **single specific fault class**:

- handover threshold misconfiguration,
- hysteresis misconfiguration,
- time-to-trigger misconfiguration.

The strongest evidence comes from:

- handover event counts,
- cell transition patterns,
- packet loss around the handover window,
- received throughput around the handover window,
- delay around the handover window.

