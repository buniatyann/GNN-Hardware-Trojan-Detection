# Hardware Trojan Detector

Automated detection of hardware trojans in gate-level Verilog and SystemVerilog netlists using a hybrid Graph Neural Network + algorithmic analysis approach.

The tool processes HDL source files through a 6-stage pipeline — from file ingestion to synthesis, graph construction, GNN-based classification augmented by SCOAP + Cone of Influence analysis, and report generation — to identify malicious logic inserted into integrated circuits. It provides per-gate suspicion scores and exact source locations of detected trojans.

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Pipeline Architecture](#pipeline-architecture)
- [Detection Approach](#detection-approach)
- [GNN Models](#gnn-models)
- [Training](#training)
  - [Training Data Sources](#training-data-sources)
- [GUI](#gui)
- [Project Structure](#project-structure)
- [Development](#development)
- [License](#license)

## Features

- **Multi-format parsing** — supports Verilog (`.v`, `.vh`) and SystemVerilog (`.sv`) via pyslang
- **Yosys-based synthesis** — elaborates netlists into a normalized JSON representation
- **GNN ensemble classification** — three interchangeable architectures (GCN, GAT, GIN) with pretrained weights; cascade with early-exit *or* full-ensemble weighted-average fusion (user-selectable)
- **Algorithmic analysis** — SCOAP controllability/observability metrics and Cone of Influence reachability scoring, integrated with GNN for combined verdict
- **Trojan localization** — reports suspicious gates with exact `file:line` source locations, SCOAP profile, and output cone
- **Multiple export formats** — JSON, PDF, and plain text reports
- **CLI and GUI interfaces** — command-line pipeline and PySide6 desktop application
- **Batch processing** — analyze entire directories of HDL files independently
- **CUDA support** — automatic GPU detection for accelerated inference

## Requirements

- Python >= 3.10
- [Yosys](https://github.com/YosysHQ/yosys) — must be installed and available in `PATH`
- CUDA-capable GPU (optional, for faster inference)

## Installation

### Linux / macOS

```bash
git clone https://github.com/buniatyann/GNN-Hardware-Trojan-Detection#features
cd trojan-detector

# Automated setup (creates venv, installs dependencies, downloads datasets)
./setup.sh            # Full install (core + GUI + training)
./setup.sh --core     # Core pipeline only
./setup.sh --dev      # Everything + development tools
```

### Manual installation

```bash
python -m venv .venv
source .venv/bin/activate

pip install -e "."            # Core dependencies
pip install -e ".[gui]"       # Add GUI support
pip install -e ".[training]"  # Add training dependencies
pip install -e ".[dev]"       # Add development tools
pip install -e ".[all]"       # Everything
```

### Windows

See [windows/README.md](windows/README.md) for Windows-specific setup instructions and PowerShell scripts.

## Usage

### CLI

```bash
# Analyze a single file
python -m main circuit.v -o reports/ -f json text pdf

# Analyze a directory
python -m main designs/ -o reports/

# Batch mode (process each file independently)
python -m main designs/ --batch -o reports/

# Choose GNN architecture
python -m main circuit.v -a gat

# Adjust confidence threshold
python -m main circuit.v -t 0.85

# Force CPU execution
python -m main circuit.v --device cpu

# Verbose output
python -m main circuit.v -v    # INFO level
python -m main circuit.v -vv   # DEBUG level
```

### Full CLI reference

```
trojan-detector [input] [-o OUTPUT_DIR] [-f FORMAT...] [-a ARCH] [-t THRESHOLD]
                        [--device DEVICE] [--batch] [-v|-vv]

positional arguments:
  input                   Path to a Verilog file or directory. Omit to launch the GUI.

options:
  -o, --output-dir DIR    Directory for report output (default: current directory)
  -f, --format FMT        Export format(s): json, pdf, text (default: json)
  -a, --architecture ARCH GNN architecture: gcn, gat, gin (default: gcn)
  -t, --confidence-threshold N
                          Confidence threshold for classification (default: 0.7)
  --device DEVICE         Computation device: cpu, cuda, cuda:N (default: auto-detect)
  --batch                 Process all files in directory independently
  -v, --verbose           Increase verbosity (-v INFO, -vv DEBUG)
```

### GUI

```bash
# Launch the graphical interface
python -m main
```

The GUI provides file and directory selection with recursive tree display, real-time pipeline progress, log viewing, and report preview. Folders can be dragged and dropped directly into the file list.

## Pipeline Architecture

The detection system is a sequential 6-stage pipeline. Each stage receives a shared `History` object for inter-stage communication and returns a `StageOutcome[T]` indicating success or failure. The pipeline stops at the first failure but always runs the final stage to produce at least a partial report.

```
Input (.v / .sv / .vh)
        |
        v
+-------------------+
| 1. File Ingestion | --> DirectoryManifest (discovered files, checksums)
+-------------------+
        |
        v
+-------------------+
| 2. Syntax Parser  | --> list[ParsedModule] (gates, wires, ports)
|    (pyslang)      |
+-------------------+
        |
        v
+-------------------+
| 3. Netlist         | --> SynthesisResult (Yosys JSON netlist)
|    Synthesizer     |
+-------------------+
        |
        v
+-------------------+
| 4. Graph Builder  | --> CircuitGraph (PyTorch Geometric Data object)
+-------------------+
        |
        v
+-------------------+
| 5. Trojan          | --> ClassificationResult (verdict, confidence,
|    Classifier      |     per-node scores, trojan locations,
|    (GNN + SCOAP)   |     algorithmic result)
+-------------------+
        |
        v
+-------------------+
| 6. Analysis        | --> AnalysisReport (JSON / PDF / text)
|    Summarizer      |
+-------------------+
```

## Detection Approach

Classification uses a two-component hybrid:

### GNN Ensemble

Three GNN architectures (GCN, GAT, GIN) are available, with two fusion strategies:

- **Cascade (all)** — models run in cascade order; if any model produces a confidence ≥ 0.92, the remaining models are skipped. Otherwise the partial results are combined via weighted average. Faster on confident inputs.
- **Ensemble (all)** — every model always runs and the per-model graph- and node-level probabilities are combined via weighted average (GCN: 0.30, GAT: 0.35, GIN: 0.35). Slower but uses all three signals on every input.

Either mode produces a graph-level verdict (CLEAN / INFECTED / UNCERTAIN) and per-node suspicion scores. Single architectures (GCN-only, GAT-only, GIN-only) and pairs (GCN+GAT, GCN+GIN, GAT+GIN) are also selectable from the GUI's Model dropdown.

### Algorithmic Analysis (SCOAP + CoI)

Runs in parallel with the GNN on every circuit, computing circuit-theoretic metrics:

- **SCOAP CC1** (controllability): how hard it is to set a gate to logic 1. High CC1 = rare activation = trojan trigger pattern.
- **SCOAP CO** (observability): how hard it is to observe a gate at primary outputs. High CO = masked effect = trojan payload pattern.
- **Cone of Influence**: bitmask propagation to determine which primary inputs activate and which primary outputs a gate drives. Sparse output cone = suspicious.
- **Subgraph isolation**: weakly-connected component analysis. Gates in small disconnected fragments score high.

All metrics are computed in O(N + E) time using Kahn's topological sort and integer bitmask operations.

### Combined Decision

Node scores are fused: `merged = 0.6 × GNN_score + 0.4 × algo_score`. The combined verdict follows a decision matrix:

| GNN verdict | Algorithmic signal | Final verdict |
|---|---|---|
| INFECTED | algo confirms (graph_algo_score ≥ threshold) | INFECTED |
| INFECTED | algo disagrees strongly (small circuits only) | UNCERTAIN |
| CLEAN | algo confirms strongly (small circuits only) | UNCERTAIN |
| UNCERTAIN | algo confirms + trojan probability > 0.4 | INFECTED |
| UNCERTAIN | algo disagrees | CLEAN |

For large circuits (≥ 200 nodes), the algorithmic confirmation threshold is raised (0.60 vs 0.15) because SCOAP-only scoring is less discriminative in flat synthesized graphs.

### Known Limitations

- **Stealthy data-dependent trojans** (e.g., LFSR-based trigger logic) are structurally indistinguishable from normal sequential logic. Both GNN and SCOAP will assign low suspicion scores. Golden reference comparison is required for reliable detection of this class.
- **Synthesis failures**: some RTL-style Verilog files cannot be elaborated by Yosys (behavioral constructs, unsupported primitives). These files are excluded with an error verdict.

## GNN Models

Three graph neural network architectures are available, all sharing the same interface:

| Architecture | Description | Strengths |
|---|---|---|
| **GCN** (default) | Graph Convolutional Network | Fast inference, good general performance |
| **GAT** | Graph Attention Network | Attention-based; better at capturing local structure |
| **GIN** | Graph Isomorphism Network | Most expressive; best at distinguishing graph structures |

All models use:
- 3 convolutional layers with layer normalization, dropout, and residual connections
- 128-dimensional hidden representations
- 31-dimensional node feature vectors (gate type one-hot, fan-in/fan-out, depth proxy, structural features)
- Dual-pooling readout: global mean + global max concatenated before the classification head
- Graph-level output head: 2-class (clean / infected) softmax

Pretrained weights are stored in `backend/trojan_classifier/weights/`.

## Training

Train GNN models on the combined hardware trojan benchmark suite:

```bash
# Train individual architectures
python -m backend.training.train --data-dir ./data/trusthub --architecture gcn
python -m backend.training.train --data-dir ./data/trusthub --architecture gat --epochs 100
python -m backend.training.train --data-dir ./data/trusthub --architecture gin --batch-size 16

# Or use the training scripts
./training_scripts/train_all.sh    # Train all three models
./training_scripts/train_gcn.sh    # Train GCN only
```

### Training data sources

Training uses publicly available hardware trojan benchmarks and clean circuit collections:

| Dataset | Type | Description |
|---|---|---|
| **TrustHub** | Trojan + golden pairs | Chip-level benchmarks (AES, RS232, PIC16F84, wb_conmax, BasicRSA, ISCAS'89 trojans) |
| **TRIT (tc/ts)** | Synthetic trojans | Trojan-inserted ASIC benchmarks at LEDA 250nm and Skywater 130nm |
| **ISCAS'85** | Clean | 10 combinational benchmark circuits |
| **ISCAS'89** | Clean + trojan | 31 sequential benchmark circuits |
| **EPFL arithmetic** | Clean | adder, multiplier, divider, and 7 other arithmetic circuits |
| **EPFL random_control** | Clean | arbiter, i2c, mem_ctrl, and 7 other control circuits |
| **HDL benchmarks** | Clean | Open-source IP cores for verification |

**TrustHub** provides the primary trojan training data, each circuit paired with a golden (trojan-free) reference:
- **AES** — AES-T100 through AES-T2000 (cryptographic trojans)
- **RS232** — RS232-T100 through RS232-T500 (UART trojans)
- **PIC16F84** — Microcontroller trojans
- **wb_conmax** — Wishbone interconnect trojans
- **BasicRSA** — RSA cryptographic core trojans
- **ISCAS'89** — s38417, s35932, s15850 benchmark trojans

TrustHub and TRIT require manual download after registration. ISCAS, EPFL, and HDL benchmarks are downloaded automatically. See [backend/training/data/README.md](backend/training/data/README.md) for the expected directory layout.

```bash
./training_scripts/download_datasets.sh
```

### Training hyperparameters

| Parameter | Default |
|---|---|
| Epochs | 200 |
| Hidden dimension | 128 |
| Layers | 3 |
| Learning rate | 1e-3 |
| Weight decay | 1e-4 |
| Dropout | 0.3 |
| Batch size | 32 |
| Early stopping patience | 30 |

## GUI

The PySide6-based graphical interface provides:

- File and directory selection with recursive tree display (folders shown with full subfolder/file hierarchy)
- Drag-and-drop support for files and directories
- Real-time pipeline progress and status reporting
- Interactive log viewer with per-file report tabs (double-click a file to open its report)
- Report preview in all export formats
- Toolbar **Model** dropdown to pick GCN / GAT / GIN, any pair, **Cascade (all)** (early-exit), or **Ensemble (all)** (always run all three)
- "Run as Design" mode: synthesize multiple checked files together as a single design (cross-module connections visible to the GNN); the resulting "Design Report" tab is also reachable by double-clicking any contributing file
- Asynchronous processing with cancellation support

Install GUI dependencies:

```bash
pip install -e ".[gui]"
```

## Project Structure

```
.
├── backend/
│   ├── core/                       # Pipeline orchestration, History, StageOutcome
│   ├── file_ingestion/             # Stage 1: file discovery and validation
│   ├── syntax_parser/              # Stage 2: Verilog/SystemVerilog parsing (pyslang)
│   ├── netlist_synthesizer/        # Stage 3: Yosys synthesis
│   ├── netlist_graph_builder/      # Stage 4: graph construction
│   ├── trojan_classifier/          # Stage 5: GNN + algorithmic classification
│   │   ├── architectures/          # GCN, GAT, GIN implementations
│   │   ├── algorithmic_analyzer.py # SCOAP + Cone of Influence analyzer
│   │   ├── structural_verifier.py  # Graph invariant baseline comparison
│   │   └── weights/                # Pretrained model weights + structural baseline
│   ├── analysis_summarizer/        # Stage 6: report generation
│   │   └── exporters/              # JSON, PDF, text exporters
│   ├── api/                        # DetectorAPI facade for external consumers
│   └── training/                   # Dataset loading, labeling, training loop
├── gui/                            # PySide6 desktop application
├── tests/                          # pytest test suite
├── training_scripts/               # Shell scripts for model training
├── windows/                        # Windows setup and batch scripts
├── main.py                         # CLI / GUI entry point
├── config.py                       # Global configuration
├── pyproject.toml                  # Project metadata and dependencies
└── setup.sh                        # Automated setup script
```

## Development

```bash
# Install with dev dependencies
pip install -e ".[dev]"

# Run tests
pytest tests/
pytest tests/test_file_ingestion/                                          # single module
pytest tests/test_file_ingestion/test_collector.py                         # single file
pytest tests/test_file_ingestion/test_collector.py::TestCollector::test_single_file  # single test

# Lint
ruff check .

# Type check
mypy backend/
```

## License

MIT License. See [LICENSE](LICENSE) for details.
