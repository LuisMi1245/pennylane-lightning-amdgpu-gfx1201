# PennyLane-Lightning AMDGPU — gfx1201

Unofficial community build of [PennyLane-Lightning AMDGPU](https://github.com/PennyLaneAI/pennylane-lightning) for AMD GPUs based on the `gfx1201` architecture.

This repository provides a reproducible build configuration and relocatable binary wheel for using the `lightning.amdgpu` device with AMD `gfx1201` GPUs on Linux.

The primary target is the AMD Radeon RX 9070 XT.

> **Disclaimer:** This is an independent community project. It is not affiliated with, endorsed by, or officially maintained by PennyLaneAI, AMD, or the Kokkos project.

## Project goals

- Provide a precompiled AMDGPU wheel for `gfx1201`.
- Bundle the Kokkos shared libraries required by the backend.
- Avoid hard-coded paths to the original build machine.
- Document the complete build and packaging procedure.
- Maintain a reproducible and documented compatibility matrix.
- Review dependencies and publish updated builds approximately every six months.

## Current build

| Component | Version / Configuration |
|---|---|
| PennyLane | 0.45.1 |
| PennyLane-Lightning | 0.45.0 |
| Lightning-AMDGPU | 0.45.0 |
| Kokkos | 5.1.0 |
| ROCm / HIP | 7.1.1 |
| Python | CPython 3.14 |
| Operating system | Fedora Linux 44 |
| Platform | Linux x86_64 |
| GPU architecture | AMD `gfx1201` |
| Tested GPU | Radeon RX 9070 XT |

This compatibility matrix describes the environment used to build and validate the current release. Other Linux distributions, ROCm versions, Python versions, and AMD GPU architectures are not considered verified unless explicitly listed in a subsequent release.

## Current release

**Wheel:**

`pennylane_lightning_amdgpu-0.45.0-cp314-cp314-linux_x86_64.whl`

The wheel includes the Lightning AMDGPU native extensions and the Kokkos shared libraries required by this build.

The AMD HIP runtime remains a system dependency and must be provided by a compatible ROCm installation.

### Important compatibility requirements

- Linux x86_64.
- CPython 3.14.
- AMD ROCm/HIP runtime compatible with `libamdhip64.so.7`.
- AMD GPU with architecture `gfx1201`.
- glibc 2.43 or newer.
- libstdc++ providing `GLIBCXX_3.4.32` or newer.

The wheel uses the generic `linux_x86_64` platform tag. This tag does not guarantee compatibility with every Linux distribution or AMD GPU.

## Installation

### 1. Check the AMD GPU

Ensure ROCm is installed and that the GPU is recognized:

```bash
rocminfo
```

The GPU agent should report:

```text
Name: gfx1201
Marketing Name: AMD Radeon RX 9070 XT
```

### 2. Create a virtual environment

```bash
python3.14 -m venv ~/venvs/pennylane-gfx1201

source ~/venvs/pennylane-gfx1201/bin/activate

python -m pip install --upgrade pip
```

### 3. Install Python dependencies

```bash
python -m pip install \
    "pennylane==0.45.1" \
    "pennylane-lightning==0.45.0" \
    "scipy-openblas32>=0.3.26"
```

### 4. Download and install the wheel

Download the wheel from the Assets section of the corresponding GitHub Release.

Then install it:

```bash
python -m pip install --no-deps \
    ./pennylane_lightning_amdgpu-0.45.0-cp314-cp314-linux_x86_64.whl
```

### 5. Verify the installation

```bash
python -m pip check
```

A successful installation should report:

```text
No broken requirements found.
```

### 6. Run a basic quantum circuit

```python
import pennylane as qml

dev = qml.device("lightning.amdgpu", wires=2)

@qml.qnode(dev)
def circuit(theta):
    qml.RY(theta, wires=0)
    qml.CNOT(wires=[0, 1])
    return qml.state()

print(circuit(0.5))
```

Expected state:

```text
[0.96891242+0.j  0.+0.j  0.+0.j  0.24740396+0.j]
```

This is a basic functional smoke test, not a comprehensive performance or correctness benchmark.

## Build from source

The complete reproducible build procedure, including Kokkos compilation, HIP configuration, wheel packaging, runtime library bundling, and binary sanitization, is documented in [`BUILDING.md`](BUILDING.md).

The current build uses:

- PennyLane-Lightning `v0.45.0`
- Kokkos `5.1.0`
- ROCm/HIP `7.1.1`
- Kokkos architecture flag `AMD_GFX1201`

## Validation

The backend has been validated on the target GPU with:

- Basic state-vector circuit execution.
- Multi-qubit state-vector simulation.
- Adjoint differentiation.
- GPU activity observed during simulation.

These checks establish that the custom backend can execute on the target hardware. They do not imply that every PennyLane operation, circuit configuration, or quantum machine-learning workload has been exhaustively tested.

## Maintenance policy

This project aims to review and update its build approximately every six months.

Each new release should document:

1. PennyLane and PennyLane-Lightning versions.
2. Kokkos version.
3. ROCm/HIP version.
4. Python version and Linux ABI requirements.
5. GPU architectures actually tested.
6. Installation and smoke-test results.
7. Wheel SHA-256 checksum.

Dependency upgrades must be validated before publishing a new wheel. A newer upstream release does not automatically imply compatibility with this custom backend.

## Known limitations

- The current binary is specifically compiled for AMD `gfx1201`.
- The wheel targets CPython 3.14 and Linux x86_64.
- A compatible AMD ROCm/HIP installation is required.
- Other AMD GPU architectures require a separate build.
- Older Linux systems may lack the required glibc or libstdc++ symbols.
- The current release is validated on one hardware and software configuration.

## Upstream projects and references

- [PennyLane](https://github.com/PennyLaneAI/pennylane)
- [PennyLane-Lightning](https://github.com/PennyLaneAI/pennylane-lightning)
- [Kokkos](https://github.com/kokkos/kokkos)
- [ROCm documentation](https://rocm.docs.amd.com/)
- [Lightning documentation](https://docs.pennylane.ai/projects/lightning/)

## License and acknowledgements

PennyLane-Lightning is distributed under the Apache License 2.0.

This project bundles Kokkos Core shared libraries, which have their own license terms. See `LICENSE` and `THIRD_PARTY_NOTICES.md` for the applicable notices.

All trademarks and product names belong to their respective owners.
