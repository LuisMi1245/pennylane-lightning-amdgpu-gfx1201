# PennyLane-Lightning AMDGPU — gfx1201

Unofficial community build of [PennyLane-Lightning AMDGPU](https://github.com/PennyLaneAI/pennylane-lightning) for AMD GPUs based on the `gfx1201` architecture.

This repository provides reproducible build instructions and release wheels for using the `lightning.amdgpu` device with supported AMD `gfx1201` GPUs on Linux. The primary target tested by this project is the AMD Radeon RX 9070 XT.

> **Disclaimer:** This is an independent community project. It is not affiliated with, endorsed by, or officially maintained by PennyLaneAI, AMD, or the Kokkos project.

## Project goals

- Provide a binary CPython wheel targeting AMD `gfx1201` GPUs.
- Bundle the Kokkos shared libraries required by the backend.
- Avoid embedding build-machine-specific absolute paths in native binaries.
- Document a repeatable build and packaging procedure.
- Publish the exact software and hardware configuration used for each release.
- Review dependencies and publish updates approximately every six months, subject to validation.

## Current validated configuration

The initial build was produced and tested using the following configuration:

| Component | Version / configuration |
|---|---|
| PennyLane | 0.45.1 (tested Python environment) |
| PennyLane-Lightning | 0.45.0 |
| Lightning-AMDGPU | 0.45.0 |
| Kokkos | 5.1.0 |
| ROCm | 7.1.1 |
| HIP | 7.1.52802 |
| HIP compiler | Clang 20.0.0 |
| Python | CPython 3.14 |
| Operating system | Fedora Linux 44 |
| Platform | Linux x86_64 |
| GPU architecture | AMD `gfx1201` |
| Tested GPU | AMD Radeon RX 9070 XT |

This table records the configuration actually used for the initial build and validation. It does not imply that every Linux distribution, ROCm version, or AMD GPU is compatible. Each later release must include its own `BUILD-INFO.txt` describing the configuration used for that release.

## Releases and wheel compatibility

Download binary wheels from the repository's [GitHub Releases](../../releases) page. Each release should include:

- The final `.whl` file.
- `SHA256SUMS.txt`, containing its SHA-256 checksum.
- `BUILD-INFO.txt`, describing the build and validation environment.

The initial wheel is named:

```text
pennylane_lightning_amdgpu-0.45.0-cp314-cp314-linux_x86_64.whl
```

The wheel tag indicates CPython 3.14 and Linux x86_64. It does **not** encode the GPU architecture or ROCm version. The initial build targets `gfx1201` and requires a compatible AMD ROCm/HIP installation.

### Initial binary requirements

- Linux x86_64.
- CPython 3.14.
- AMD GPU with architecture `gfx1201`.
- A compatible ROCm installation providing `libamdhip64.so.7`.
- glibc 2.43 or newer.
- libstdc++ providing `GLIBCXX_3.4.32` or newer.

Kokkos Core and Kokkos Containers are bundled in the wheel. The AMD HIP runtime, GPU driver, and other system libraries are not bundled and must be supplied by the host system.

The ABI requirements above apply to the initial wheel. They must be checked again for every rebuilt release; do not assume a future build has identical requirements.

## Installation

The following commands describe the initial release. Replace the wheel filename with the exact asset attached to the release you are installing.

### 1. Check the GPU and ROCm installation

Ensure ROCm is installed and the GPU is visible:

```bash
rocminfo
```

The GPU agent should identify the architecture as `gfx1201`. Also check that the HIP runtime is visible to the dynamic linker:

```bash
ldconfig -p | grep libamdhip64.so.7
```

### 2. Create a virtual environment

```bash
python3.14 -m venv ~/venvs/pennylane-gfx1201
source ~/venvs/pennylane-gfx1201/bin/activate
python -m pip install --upgrade pip
```

### 3. Install the tested Python dependencies

```bash
python -m pip install \
    "pennylane==0.45.1" \
    "pennylane-lightning==0.45.0" \
    "scipy-openblas32>=0.3.26"
```

These are the versions used for the initial release. For a later release, follow its release notes and `BUILD-INFO.txt`.

### 4. Install the wheel

Download the wheel asset from GitHub Releases, change to the directory containing the downloaded file, and run:

```bash
python -m pip install --no-deps \
    ./pennylane_lightning_amdgpu-0.45.0-cp314-cp314-linux_x86_64.whl
```

`--no-deps` is used because the dependencies are installed explicitly in the previous step.

### 5. Check Python dependencies

```bash
python -m pip check
```

Expected output:

```text
No broken requirements found.
```

### 6. Run a basic circuit

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

Expected state, up to normal floating-point formatting:

```text
[0.96891242+0.j  0.+0.j  0.+0.j  0.24740396+0.j]
```

This is a basic functional smoke test. It is not a comprehensive test of every operation, circuit configuration, gradient method, or machine-learning workload.

## Build from source

See [`BUILDING.md`](BUILDING.md) for the full procedure, including:

- Pinning source versions.
- Building Kokkos for `AMD_GFX1201`.
- Configuring and compiling PennyLane-Lightning AMDGPU.
- Bundling Kokkos shared libraries in the wheel.
- Applying relocatable ELF runtime paths.
- Removing build-machine paths and debug information.
- Including third-party license notices.
- Creating release assets and validating the final wheel in a clean virtual environment.

## Validation history

The initial backend build was exercised on the target `gfx1201` GPU with:

- Basic state-vector circuit execution.
- Multi-qubit state-vector simulation, including a 24-qubit test.
- Adjoint differentiation.
- GPU activity observed during simulation.

These checks establish that the backend executed on the target hardware in the original build environment. They do not establish compatibility with every PennyLane operation or all AMD GPUs. Each release should distinguish tests run on the original build environment from tests run after installing the final wheel into a clean virtual environment.

## Maintenance policy

The project aims to review the build approximately every six months. An update is not automatic: every new combination of PennyLane-Lightning, Kokkos, ROCm/HIP, Python, and compiler versions must be built and validated before publication.

For every release, record:

1. PennyLane and PennyLane-Lightning versions.
2. Kokkos version and source commit.
3. ROCm, HIP, and HIP compiler versions.
4. Python version, Linux platform, glibc and libstdc++ ABI requirements.
5. GPU architecture and physical GPU models actually tested.
6. Clean-environment installation and smoke-test results.
7. SHA-256 checksum of the published wheel.

Do not claim support for a GPU architecture or software configuration that has not been tested.

## Known limitations

- The initial wheel is compiled specifically for AMD `gfx1201`.
- The initial wheel targets CPython 3.14 and Linux x86_64.
- A compatible system ROCm/HIP runtime is required.
- The generic `linux_x86_64` wheel tag does not make the binary portable to every Linux distribution.
- Other AMD GPU architectures require separate compilation and validation.
- A successful import or installation does not establish that every PennyLane operation is supported by this backend.

## Upstream projects

- [PennyLane](https://github.com/PennyLaneAI/pennylane)
- [PennyLane-Lightning](https://github.com/PennyLaneAI/pennylane-lightning)
- [Kokkos](https://github.com/kokkos/kokkos)
- [AMD ROCm documentation](https://rocm.docs.amd.com/)
- [PennyLane-Lightning documentation](https://docs.pennylane.ai/projects/lightning/)

## Licensing and third-party notices

This repository must include its own `LICENSE` for original repository materials and `THIRD_PARTY_NOTICES.md` for redistributed components. Include the license text shipped with the exact Kokkos source version in `licenses/`, and include the corresponding Kokkos license file in the wheel's `.dist-info/licenses/` directory when bundling its shared libraries.

See [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md) for the third-party notices. Do not imply that this independent build is an official PennyLaneAI or AMD binary distribution.
