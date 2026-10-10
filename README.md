# PLC Agent — Two Agents and a Compiler Between Them

## Motivation

Operational Technology (OT) software controls the physical world. Power grids, water treatment plants, oil refineries, pharmaceutical production lines, nuclear reactors — all of them run on programmable logic controllers executing IEC 61131-3 Structured Text. A vulnerability or a misconfiguration in that code is not a data breach. It is a pipeline explosion, a contaminated water supply, a grid outage.

The cybersecurity gap in OT environments is severe and widening. IT security has decades of tooling, threat intelligence, and trained models behind it. OT security has almost none of that. Attackers have noticed: Stuxnet, Triton, Industroyer, and Pipedream are all purpose-built OT weapons, and they work precisely because defenders lack the AI-assisted detection and response capabilities that IT takes for granted.

**The core problem is data.** Every modern security tool — anomaly detection, vulnerability scanning, code auditing, patch recommendation — eventually needs a model, and every model needs training data. For OT, that data does not exist at scale. Structured Text is one of the most consequential programming languages in the world and one of the least represented in any public benchmark. There is no OT equivalent of GitHub Copilot's training corpus, no Stack Overflow for ladder logic, no HuggingFace dataset of verified PLC programs. The language is proprietary, the deployments are air-gapped, and the engineers who write the code do not publish it.

**This project is a direct attack on that data problem.** The approach: use a small language model to generate Structured Text, use a deterministic compiler and a set of structural checks as the only authority on correctness, and let the model learn from verified programs as it goes. No human labels. No proprietary data. No large model required. The loop produces gate-verified, sector-diverse, requirement-tagged Structured Text programs that can seed the training data pipeline for OT security tools.

The broader claim is that this approach generalises. Structured Text is not unique. Any domain with a formal correctness criterion — hardware description languages, medical device firmware, aviation control software, industrial robotics — faces the same data scarcity problem and can use the same bootstrap loop: generate, verify deterministically, learn from what passed. The checker is the hard part and the valuable artifact. Building a good checker for an underrepresented language is the contribution that compounds.

**Hack the Chip — NYU × Annapurna Labs, October 2026**
**Hardware:** AWS Trainium (Qwen3-8B on-chip via vLLM-Neuron)
**Project type:** Propose-your-own (approved before 11:30)

---

A small model writes IEC 61131-3 Structured Text programs. A deterministic gate grades them: IronPLC decides whether the code is valid IEC 61131-3, and 32 structural checks decide whether it contains what was asked for. A second agent turns the gate's verdict into line-level edits with the program's own variable names. The generator tries again. Memory of what worked carries from one spec to the next.

```
spec ──> Agent 1 (generator) ──> GATE: ironplcc check + 32 spec checks ──> PASS ──> dataset
               ^                              │ fail
               └──── Agent 2 (critic): verdict -> numbered edits <──┘
                                  (bootstrap memory feeds round 0 of the next spec)
```

**The rule everything is built on: no LLM ever says PASS.** The critic advises; the gate decides. A model asked "is this correct?" will say yes to broken code, and then the loop optimises for fooling the critic instead of compiling.

| file | what it is |
|---|---|
| `plc_checker.py` | the gate: spec builder, 32 requirement checks, IronPLC wrapper, compiler probe, `--selftest` |
| `agent.py` | the loop: generator, critic, keep-best repair, bootstrap memory, curriculum, attempt log |
| `report.py` | side-by-side comparison of run directories; writes `progress.png` |

---

## What changed between the first run and the final version

| version | round-0 result | reason |
|---|---|---|
| v1 (original GPT script) | 0.30 every round, 0% solve rate | IronPLC's raw token list sent as feedback; loop learned nothing |
| v2 (two agents, raw IronPLC output) | 0.30 every round, 0% solve rate | Same problem: critic saw the same noise as the generator |
| v3 (translated compiler messages) | 0.30 → 1.00 in 2 rounds on first solve | P0002/P4007/P4012/P2008 translated to one-line instructions |
| v4 (specific failure diagnoses) | 0.30 → 1.00 round 1; P2008+specific FB hint | Diagnosers name the program's own variables in every fix |
| v5 (keep-best repair) | No regression from 0.96 back to 0.30 | Always repairs the highest-scoring version seen so far |
| v6 (bootstrap + curriculum) | Round-0 solves as memory fills | Verified exemplars and known fixes carried across specs |

The before/after is visible in `runs/`: the first run directory shows 0.30 for all 4 rounds on every sample. Later runs show round-1 solves.

---

## Setup (inside the pod)

Everything runs in your seat pod. Keep `./serve.sh` running in terminal 1; use terminal 2 for the agent.

### 1. Copy the project in

```bash
# on your laptop
kubectl cp PLC-AGENT seat-39:/workspace/PLC-AGENT
```

```bash
# inside the pod
git config --global --add safe.directory /workspace
cd /workspace/PLC-AGENT
```

### 2. Install IronPLC and the OpenAI client

```bash
curl -fsSL https://www.ironplc.com/install.sh | sh
pip install openai
ironplcc --help
```

If `ironplcc` is not found after install, add its directory to PATH:
`export PATH="$HOME/.ironplc/bin:$PATH"`

### 3. Confirm the model server is up

```bash
curl -s http://localhost:8000/v1/models
```

Must return JSON listing `Qwen/Qwen3-8B`. If it returns nothing, `serve.sh` is not running or still loading (wait for `READY`).

### 4. Run the selftest

```bash
python plc_checker.py --selftest
```

Must print `SELFTEST PASSED`. Part A checks 32 requirement checks plus compiler-message translation. Part B probes every IronPLC language capability (CASE, FOR, WHILE, FUNCTION_BLOCK, ARRAY, STRUCT, TON) and verifies that three invalid programs are rejected. If a capability probe fails, the agent automatically stops sampling specs that need it.

---

## Running

### Smoke test

```bash
python agent.py --n 2 --model Qwen/Qwen3-8B
```

Round 0 may still score 0.30 (syntax error). Round 1 should reach ≥ 0.60. A 1.00 on round 0 or 1 means the loop is working.

If you see `best 0.00 of 4 [no program in output]`: the server crashed (`pgrep -af vllm` shows `<defunct>`). Restart `serve.sh` and wait for `READY`.

### Full experiment

```bash
cd /workspace/PLC-AGENT
nohup sh -c '
python agent.py --n 10 --repeat 3 --seed 99 --out-dir runs/baseline &&
python agent.py --n 30 --curriculum --out-dir runs/curriculum_train &&
python agent.py --n 10 --repeat 3 --seed 99 --memory-in runs/curriculum_train/memory_run0.json --freeze-memory --out-dir runs/curriculum_eval
' > main.log 2>&1 < /dev/null &
echo "PID $!"
tail -f main.log
```

Disconnected? `tail -f /workspace/PLC-AGENT/main.log`. Still running? `pgrep -af agent.py`.

### Compare results

```bash
python report.py runs/baseline runs/curriculum_eval
```

### Copy results to laptop when done

```bash
# on your laptop
kubectl cp seat-39:/workspace/PLC-AGENT/runs ./Hack-Chip-26/runs
kubectl cp seat-39:/workspace/PLC-AGENT/main.log ./Hack-Chip-26/main.log
```

---

## The gate: 32 checks across 6 dimensions

### Spec dimensions sampled per run

| dimension | examples |
|---|---|
| sector | Water Treatment, Oil Refinery, Pharmaceutical, Nuclear, Agriculture (21 total) |
| control strategy | State Machine, PID Loop, Cascade Control, Pump Alternation (8 total) |
| IEC language feature | CASE Statement, FOR Loop, WHILE Loop, Function Block, Array, Structures |
| safety feature | Emergency Stop, SIL Logic, Watchdog Monitoring, Redundant Sensors |
| failure scenario | Sensor Failure, Motor Overload, High Pressure, Power Loss |
| security profile | Secure (clamp all setpoints) or Insecure (one marked vulnerability) |

### Why the spec dictates identifiers

The generator is told *"Declare EStop : BOOL. Inside IF EStop THEN set every actuator output to FALSE."* The check then looks for exactly that variable name. This turns a fuzzy question ("does this have an E-stop?") into a structural one, and lets every failure map to a precise instruction: *"Declare EStop : BOOL and add IF EStop THEN <output> := FALSE; END_IF; at the end of the program."*

### What each safety check actually requires

- **Emergency stop / tank overflow / motor overload:** variable declared with the right type AND some IF branch testing it assigns FALSE to an actuator. An inverted test (`IF NOT EStop`) is rejected. A comment containing the logic is rejected.
- **SIL voting (2-of-3):** `VoteTrip := (TripA AND TripB) OR (TripA AND TripC) OR (TripB AND TripC);` exactly, and something forces outputs off when VoteTrip is TRUE.
- **Safety shutdown:** latches on a trip, may only clear inside a branch testing `ResetCmd`, and forces outputs off while active.
- **Watchdog:** `TON` timer declared and called every scan; `WatchdogTimer.Q` used in a condition or assignment.
- **Power loss:** `NOT PowerOK` tested (not `PowerOK`).
- **Redundant sensors:** `ABS(SensorA - SensorB) > MaxDeviation` exactly.

### Loops must terminate

A WHILE loop passes only if its condition compares a counter with `<` or `<=` and the body increments it. An infinite loop is a watchdog trip on real hardware.

### Scoring

| score | meaning |
|---|---|
| 0.00 | no program in output, or truncated at token limit |
| 0.30 | IronPLC rejects it (syntax error) |
| 0.60–0.99 | compiles; 0.60 + 0.40 × share of checks passed |
| 1.00 | compiles and every check passes — **PASS** |

### `best X of 4` explained

`best 0.96 of 4` means 4 candidates were generated in parallel that round; the best scored 0.96. Only 1 needs to score 1.00 for the spec to be solved. `--samples 4` is the parallel count.

---

## Compiler-message translation

Raw IronPLC output is a list of expected tokens, useless as repair instructions. The gate parses it into one-line instructions before the critic or the raw-feedback path sees it:

| error code | raw message (truncated) | translated instruction |
|---|---|---|
| P0002 found 'VAR' | `Expected 'PROGRAM' \| 'FUNCTION_BLOCK' \| ...` | `This VAR block is outside any PROGRAM. Put PROGRAM Main directly above it.` |
| P2008 | `Cannot determine kind of type identifier (identifier=MyBlock)` | `'MyBlock' is used as a type but not defined. FUNCTION_BLOCK MyBlock must appear BEFORE PROGRAM Main.` |
| P4007 | `Variable not defined before used (variable=n)` | `'n' is used but never declared. Add 'n : INT;' to the VAR block.` |
| P4012 | `Function block invocation is not a variable in scope` | `'Ctl1(...)' calls an instance not declared. Add 'Ctl1 : MyBlock;' to the VAR block.` |

### Failure diagnosers

When a check fails, the gate inspects the program and returns a specific instruction using the program's own variable names:

- **Function block never called:** *"'Ctl1' is declared as MyBlock but never called. Assigning Ctl1.In1 is not a call. Add Ctl1(In1 := \<value\>); in the program body."*
- **Function block body inside PROGRAM:** *"The FUNCTION_BLOCK definition must come BEFORE PROGRAM Main. The file must look like: FUNCTION_BLOCK \<Name\> ... END_FUNCTION_BLOCK followed by PROGRAM Main."*
- **Struct type declared but fields not used:** *"'\<var\>' is declared as \<Type\> but its fields are never used. Read or write them as \<var\>.\<field\>."*

---

## Bootstrap memory

Without `--bootstrap`, every spec starts from zero. With it, Agent 1 carries three things from each solved spec to the next:

| what | source | use |
|---|---|---|
| verified exemplars | programs the gate accepted | the closest one by shared requirements goes into the round-0 prompt |
| known fixes | a compiler error that disappeared after feedback | *"IronPLC error X was fixed by: ..."* prepended to the prompt |
| often-missed requirements | round-0 failure rate tracked per check | a warning added when that check appears in the new spec |

**Nothing enters memory unless the gate accepted it.** An exemplar only exists if IronPLC compiled it and every check passed. The memory cannot teach the model a wrong pattern.

Each `--repeat` run starts with an empty memory, so runs stay independent and the spread is honest. Memory is saved as `memory_run<k>.json` after each run.

---

## Curriculum mode

`--curriculum` starts with small specs and levels up as the agent succeeds. Bootstrap memory is on automatically.

| level | spec contents | requirements |
|---|---|---|
| 1 | operating mode + one simple safety interlock | ~3 checks |
| 2 | + control strategy + second safety feature | ~5 checks |
| 3 | + one IEC feature + failure scenario + security profile | ~7 checks |
| 4 | full random sampling (default without `--curriculum`) | ~9 checks |

Level-up rule: 2 of the last 3 specs at a level solved within 2 rounds. Never levels down.

### Fair evaluation

Curriculum numbers alone are inflated because level-1 specs are easy. The valid comparison is: train with the curriculum, then freeze the memory and test on full-difficulty specs that were never seen:

- `runs/baseline` — full-difficulty specs, no memory
- `runs/curriculum_train` — curriculum run that builds the memory
- `runs/curriculum_eval` — same full-difficulty specs as baseline, memory frozen

Any gap between baseline and curriculum_eval is what the curriculum taught, not the easy level-1 specs inflating the number.

---

## Keep-best repair

If a repair round produces a worse program (0.96 → 0.30 happens when the model rewrites a mostly-correct program and introduces a syntax error), the regression is discarded. The agent re-sends the best program seen so far with a note: *"Your latest version scored 0.30, worse than this one (0.96), and was discarded. Apply the fixes to the program shown above."*

---

## Known limits

The checks are structural regexes, not a full parser. They prove required constructs are present and wired to the right variables; they do not prove the program behaves correctly over time. IronPLC also ships a runtime (`ironplcvm`); running a passing program through scripted input scans ("set EStop TRUE, assert MotorRun is FALSE") would upgrade the gate from structure to behaviour and is the natural next step.

The memory stores up to `--max-exemplar-chars` characters per exemplar (default 4000). If prompts grow long and round-0 results drop, lower it.

---

## Deliverables

1. **The checker** — `plc_checker.py`: 32 requirement checks, compiler-message translator, failure diagnosers, compiler capability probe, selftest. Run `python plc_checker.py --selftest` to verify.
2. **The attempt log** — every run directory contains `attempts.jsonl` with every candidate, every score, every compiler message, every feedback message sent.
3. **Results** — `runs/baseline/summary.json` and `runs/curriculum_eval/summary.json`, compared with `python report.py runs/baseline runs/curriculum_eval`.
