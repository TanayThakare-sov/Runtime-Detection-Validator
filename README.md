# Runtime Detection Validator

A small framework that **attacks a Docker container on purpose and proves that an eBPF sensor detected the attack.**
It is a miniature version of what a cloud-security validation engineer does: build a sensor, simulate threats, and automatically verify detection.

## What it does

| Attack simulated in the container | Syscall watched by eBPF | Test |
|---|---|---|
| Shell spawned (`sh -c ...`) | `execve` | `test_shell_spawn_detected` |
| Credential file read (`cat /etc/shadow`) | `openat` | `test_shadow_read_detected` |
| Unexpected outbound connection | `connect` | `test_outbound_connection_detected` |

Every test follows the same pattern: **mark the log, run the attack, wait for a matching event, assert.**

## Architecture

```
pytest ──► attacks/ (docker exec) ──► target container
   │                                        │ syscalls
   │                                        ▼
   │                          sensor/tracer.bt (bpftrace, eBPF)
   │                                        │ event lines
   │                                        ▼
   └──────► validator/ ◄──────────── logs/events.log
        (parse, match rules, assert)
```

## Project structure

```
runtime-detection-validator/
├── sensor/
│   ├── tracer.bt            # eBPF program (bpftrace): execve, openat, connect
│   └── sensor.py            # runs the tracer, writes logs/events.log
├── attacks/
│   └── docker_attacks.py    # attack simulations (docker exec)
├── validator/
│   ├── events.py            # parse sensor lines into Event objects
│   ├── detector.py          # EventLog: mark, read, wait_for
│   └── rules.py             # what counts as "detected"
├── tests/
│   ├── conftest.py          # fixtures: target container, event log
│   ├── test_event_parsing.py   # unit tests (no eBPF needed)
│   ├── test_detector.py        # unit tests (no eBPF needed)
│   └── test_attack_detection.py # integration tests (eBPF + Docker)
├── docs/
│   ├── CONCEPTS.md          # concepts to know for interviews
│   └── LEARNING_LOG.md      # your own notes
├── .github/workflows/ci.yml # lint + unit tests + integration
├── Dockerfile.target        # victim container
├── Makefile
├── pytest.ini
├── ruff.toml
├── requirements.txt
└── LICENSE
```

## Prerequisites

- Linux (native, VM, or WSL2 with a recent kernel), kernel 5.x or newer
- Docker
- `bpftrace` (`sudo apt install bpftrace`)
- Python 3.10+

## Quick start

```bash
git clone <your-repo-url> && cd runtime-detection-validator
make setup && source .venv/bin/activate

make unit            # works anywhere, no eBPF needed

# Terminal 1: start the sensor (root needed for eBPF)
make sensor

# Terminal 2: run the attacks and validation
make integration
```

Peek at what the sensor sees: `tail -f logs/events.log`

## CI

GitHub Actions runs lint and unit tests on every push. The integration job runs the eBPF tests on the hosted runner; it is marked `continue-on-error` because eBPF permissions on hosted runners can vary.

## Challenges I faced
<!-- Write this in your OWN words as you build. Examples: bpftrace install problems,
     kernel/header errors, noisy openat output, sudo + venv issues, timing/flaky tests. -->
- 

## What I learned
- 

## Roadmap
- [ ] More detections (`chmod`, writes to `/etc/cron.d`, `ptrace`)
- [ ] Filter events by container (cgroup id) instead of by token/time window
- [ ] Run the sensor as a Kubernetes DaemonSet (kind)
- [ ] Provision a cloud host with Terraform (AWS EC2)
- [ ] Export results to Prometheus and a Grafana dashboard
