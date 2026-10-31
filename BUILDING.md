# Building PennyLane-Lightning AMDGPU for `gfx1201`

This document describes how to reproduce the community build and package a relocatable Linux wheel for AMD GPUs using the `gfx1201` architecture.

The commands below reflect the build procedure validated for the initial configuration. When updating any upstream component, review its build instructions and compatibility notes first. A new version number does not guarantee that the same CMake options or binary layout remain valid.

> **Important:** Run the commands in one Bash terminal so that the exported variables remain available. This guide deliberately does **not** use `set -euo pipefail`; do not add it if your terminal configuration closes the session when that option is enabled. Check the output of each step before proceeding.

## 1. Initial build configuration

The initial validated combination was:

| Component | Version / configuration |
|---|---|
| PennyLane | 0.45.1 |
| PennyLane-Lightning source tag | `v0.45.0` |
| Lightning-AMDGPU | 0.45.0 |
| Kokkos source tag | `5.1.0` |
| Kokkos source commit | `3ec81abe1816109f6f62ac48cef41921f91a4d00` |
| ROCm | 7.1.1 |
| HIP | 7.1.52802 |
| HIP compiler | Clang 20.0.0 |
| Python | CPython 3.14 |
| Build OS | Fedora Linux 44 |
| GPU target | `AMD_GFX1201` |
| Platform | Linux x86_64 |

For a future release, update the version variables in section 2 and verify that the selected PennyLane-Lightning version supports the selected Kokkos and ROCm/HIP combination.

## 2. Version variables and build directories

Choose a new, unique build directory for every release. The example below adds the current date and a revision suffix. If you repeat a build on the same day, change `r1` to `r2`, etc. Do not reuse an old CMake build directory when changing compiler, Kokkos, ROCm, or architecture settings.

```bash
export KOKKOS_TAG="5.1.0"
export LIGHTNING_TAG="v0.45.0"
export PENNYLANE_VERSION="0.45.1"
export LIGHTNING_PYTHON_VERSION="0.45.0"
export ROCM_VERSION="7.1.1"
export HIP_VERSION="7.1.52802"
export PYTHON_BIN="python3.14"
export GPU_ARCH="gfx1201"

export BUILD_ROOT="/tmp/pl-gfx1201-build-$(date +%Y%m%d)-r1"

export KOKKOS_SRC="$BUILD_ROOT/kokkos"
export KOKKOS_BUILD="$BUILD_ROOT/kokkos-build"
export KOKKOS_ROOT="$BUILD_ROOT/kokkos-install"
export LIGHTNING_SRC="$BUILD_ROOT/pennylane-lightning"
export RAW_WHEEL_DIR="$BUILD_ROOT/raw-wheel"
export UNPACK_DIR="$BUILD_ROOT/unpacked"
export FINAL_WHEEL_DIR="$BUILD_ROOT/final-wheel"
export TMPDIR="$BUILD_ROOT/tmp"
export PIP_CACHE_DIR="$BUILD_ROOT/pip-cache"

mkdir -p "$RAW_WHEEL_DIR" "$UNPACK_DIR" "$FINAL_WHEEL_DIR" \
         "$TMPDIR" "$PIP_CACHE_DIR"
```

The path `/tmp/pl-gfx1201-build-...` is intentionally generic. Do not move the source or build directories under a personal home directory during compilation. The final wheel must not contain the local build path or a user's home-directory path.

## 3. Install system build requirements

On Fedora, install the required build utilities and development packages:

```bash
sudo dnf install \
    git \
    curl \
    cmake \
    ninja-build \
    gcc-c++ \
    python3.14 \
    python3.14-devel \
    libomp \
    libomp-devel \
    rocthrust-devel \
    rocprim-devel \
    patchelf
```

ROCm/HIP must already be installed. The initial environment used ROCm 7.1.1. Check the compiler and GPU visibility:

```bash
/usr/bin/hipcc --version
rocminfo
```

The target GPU agent must report `gfx1201`. Check that the HIP runtime is registered with the system dynamic linker:

```bash
ldconfig -p | grep libamdhip64.so.7
```

If a command is not found or the target architecture is absent, stop and correct the system setup before building.

## 4. Download pinned source versions

For a new, empty build root, clone the pinned Kokkos and PennyLane-Lightning source tags:

```bash
git clone \
    --depth 1 \
    --branch "$KOKKOS_TAG" \
    https://github.com/kokkos/kokkos.git \
    "$KOKKOS_SRC"
```

```bash
git clone \
    --depth 1 \
    --branch "$LIGHTNING_TAG" \
    https://github.com/PennyLaneAI/pennylane-lightning.git \
    "$LIGHTNING_SRC"
```

Verify both tags and record their commit IDs:

```bash
git -C "$KOKKOS_SRC" describe --tags --exact-match
git -C "$KOKKOS_SRC" rev-parse HEAD

git -C "$LIGHTNING_SRC" describe --tags --exact-match
git -C "$LIGHTNING_SRC" rev-parse HEAD
```

For the initial Kokkos 5.1.0 source, the tested commit was:

```text
3ec81abe1816109f6f62ac48cef41921f91a4d00
```

A `detached HEAD` notice is normal when cloning a tag. Git may also print a warning that an annotated tag is not itself a commit and then check out the commit to which it points. Verify the tag and commit rather than treating that notice alone as a build failure.

If GitHub cannot be resolved (`Could not resolve host: github.com`), this is a network/DNS issue. Do not delete a source directory that was already cloned successfully. Restore connectivity and retry only the failed clone.

## 5. Create an isolated Python build environment

```bash
"$PYTHON_BIN" -m venv "$BUILD_ROOT/venv-build"
source "$BUILD_ROOT/venv-build/bin/activate"

python -m pip install --upgrade pip setuptools wheel cmake ninja tomli tomlkit
```

Install the Python dependencies for the selected release. The following pins reproduce the initial tested environment:

```bash
python -m pip install \
    "pennylane==$PENNYLANE_VERSION" \
    "pennylane-lightning==$LIGHTNING_PYTHON_VERSION" \
    "scipy-openblas32>=0.3.26"
```

For later upstream versions, review the Lightning project's `pyproject.toml` and release notes. Update these pins together; do not assume a new Lightning source tag can be built against the old Python package versions.

Check the environment:

```bash
python -m pip check
```

## 6. Configure compiler path remapping

The native binaries must not retain personal home-directory paths or the original build directory. The source trees, build trees, temporary files, and pip cache are all placed under `BUILD_ROOT` so the compiler can map that common prefix to a neutral path.

```bash
export PRIVACY_FLAGS="-ffile-prefix-map=$BUILD_ROOT=/usr/src/pl-gfx1201 -fdebug-prefix-map=$BUILD_ROOT=/usr/src/pl-gfx1201 -fmacro-prefix-map=$BUILD_ROOT=/usr/src/pl-gfx1201"
```

Create a CMake toolchain file that applies the flags to both Kokkos and PennyLane-Lightning:

```bash
cat > "$BUILD_ROOT/privacy-toolchain.cmake" <<EOF
set(CMAKE_CXX_FLAGS_INIT "$PRIVACY_FLAGS")
set(CMAKE_BUILD_RPATH_USE_ORIGIN ON)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH OFF)
set(CMAKE_INSTALL_RPATH "\$ORIGIN")
EOF
```

Inspect the file:

```bash
cat "$BUILD_ROOT/privacy-toolchain.cmake"
```

The three prefix-map options are complementary. They reduce the chance of source, debug, or macro-expanded paths being embedded in native binaries. They do not replace the final binary inspection later in this guide.

## 7. Build and install Kokkos for `gfx1201`

Configure Kokkos in a fresh build directory:

```bash
cmake \
    -S "$KOKKOS_SRC" \
    -B "$KOKKOS_BUILD" \
    -G Ninja \
    -DCMAKE_TOOLCHAIN_FILE="$BUILD_ROOT/privacy-toolchain.cmake" \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX="$KOKKOS_ROOT" \
    -DCMAKE_CXX_COMPILER=/usr/bin/hipcc \
    -DCMAKE_CXX_STANDARD=20 \
    -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
    -DBUILD_SHARED_LIBS=ON \
    -DBUILD_TESTING=OFF \
    -DKokkos_ENABLE_SERIAL=ON \
    -DKokkos_ENABLE_HIP=ON \
    -DKokkos_ARCH_AMD_GFX1201=ON \
    -DKokkos_ENABLE_EXAMPLES=OFF \
    -DKokkos_ENABLE_TESTS=OFF \
    -DKokkos_ENABLE_COMPLEX_ALIGN=OFF \
    -DKokkos_ENABLE_LIBDL=OFF \
    -DCMAKE_INSTALL_RPATH='$ORIGIN' \
    -DCMAKE_BUILD_RPATH_USE_ORIGIN=ON \
    -DCMAKE_INSTALL_RPATH_USE_LINK_PATH=OFF
```

Check the target configuration:

```bash
grep -E 'Kokkos_ARCH_AMD_GFX1201:BOOL|Kokkos_ENABLE_HIP:BOOL' \
    "$KOKKOS_BUILD/CMakeCache.txt"
```

Expected values:

```text
Kokkos_ARCH_AMD_GFX1201:BOOL=ON
Kokkos_ENABLE_HIP:BOOL=ON
```

Check that the compiler flags made it into the generated Ninja build rules:

```bash
grep -F -- '-ffile-prefix-map=' "$KOKKOS_BUILD/build.ninja"
```

If the expected flags are absent, stop and correct the toolchain configuration before compiling.

Build and install:

```bash
cmake --build "$KOKKOS_BUILD" --parallel 6
cmake --install "$KOKKOS_BUILD"
```

Check that Kokkos Core and Kokkos Containers were installed:

```bash
ls -l "$KOKKOS_ROOT/lib64/"libkokkos*
```

The installed directory should contain both libraries and their versioned shared-library files/symbolic links.

## 8. Configure PennyLane-Lightning AMDGPU

Set the paths to the newly built Kokkos installation:

```bash
export KOKKOS_DIR="$KOKKOS_ROOT/lib64/cmake/Kokkos"
export CMAKE_PREFIX_PATH="$KOKKOS_ROOT:/usr"
export LD_LIBRARY_PATH="$KOKKOS_ROOT/lib64:/usr/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

Configure the AMDGPU backend:

```bash
cd "$LIGHTNING_SRC"

PL_BACKEND=lightning_amdgpu \
    python scripts/configure_pyproject_toml.py
```

Pass the HIP, Kokkos, and privacy configuration to the Lightning build:

```bash
export CMAKE_ARGS="-DCMAKE_TOOLCHAIN_FILE=$BUILD_ROOT/privacy-toolchain.cmake -DCMAKE_CXX_COMPILER=/usr/bin/hipcc -DKokkos_ENABLE_HIP=ON -DKokkos_ARCH_AMD_GFX1201=ON -DKokkos_DIR=$KOKKOS_DIR -DCMAKE_PREFIX_PATH=$KOKKOS_ROOT:/usr -DCMAKE_BUILD_RPATH_USE_ORIGIN=ON -DCMAKE_INSTALL_RPATH_USE_LINK_PATH=OFF"
```

The toolchain file is important: it carries the compiler prefix-map flags into the Lightning CMake configuration. The build is expected to use the matching pinned `pennylane-lightning` Python dependency installed in the virtual environment.

## 9. Build the intermediate wheel

Build from the Lightning source tree:

```bash
python -m pip wheel . \
    --no-deps \
    --no-build-isolation \
    --wheel-dir "$RAW_WHEEL_DIR" \
    -v
```

Locate the generated wheel:

```bash
RAW_WHEEL="$(find "$RAW_WHEEL_DIR" -maxdepth 1 -type f \
    -name 'pennylane_lightning_amdgpu-*.whl' -print -quit)"

echo "$RAW_WHEEL"
```

This is an **intermediate wheel**. Do not publish it: the Kokkos libraries must first be bundled and its ELF runtime paths corrected.

## 10. Unpack the wheel and locate its package directories

```bash
python -m wheel unpack "$RAW_WHEEL" -d "$UNPACK_DIR"
```

Locate the distribution metadata and package directories without hard-coding the wheel's internal directory name:

```bash
DIST_INFO_DIR="$(find "$UNPACK_DIR" -mindepth 2 -maxdepth 2 \
    -type d -name '*.dist-info' -print -quit)"

WHEEL_ROOT="${DIST_INFO_DIR%/*}"
PACKAGE_DIR="$WHEEL_ROOT/pennylane_lightning"

printf 'Wheel root: %s\n' "$WHEEL_ROOT"
printf 'Package: %s\n' "$PACKAGE_DIR"
printf 'Metadata: %s\n' "$DIST_INFO_DIR"
```

Confirm that `PACKAGE_DIR` and `DIST_INFO_DIR` exist before continuing:

```bash
ls -ld "$PACKAGE_DIR" "$DIST_INFO_DIR"
```

## 11. Bundle Kokkos shared libraries

Create the private library directory inside the Python package:

```bash
mkdir -p "$PACKAGE_DIR/.libs"
```

Copy Kokkos Core and Kokkos Containers, preserving their symbolic links:

```bash
cp -a "$KOKKOS_ROOT"/lib64/libkokkoscore.so* \
    "$PACKAGE_DIR/.libs/"

cp -a "$KOKKOS_ROOT"/lib64/libkokkoscontainers.so* \
    "$PACKAGE_DIR/.libs/"
```

Inspect the result:

```bash
ls -la "$PACKAGE_DIR/.libs/"
```

The `.libs` directory should contain both libraries, including the real versioned shared objects and their SONAME symlinks.

## 12. Set relocatable ELF runtime paths

The Lightning native libraries must find Kokkos in the bundled `.libs` directory. They must not refer to the original Kokkos installation path.

For the initial wheel layout, patch the Lightning Catalyst library and AMDGPU extension:

```bash
patchelf --set-rpath \
    '$ORIGIN/.libs:$ORIGIN/../scipy_openblas32/lib' \
    "$PACKAGE_DIR/liblightning_kokkos_catalyst.so"
```

```bash
patchelf --set-rpath \
    '$ORIGIN/.libs:$ORIGIN/../scipy_openblas32/lib' \
    "$PACKAGE_DIR"/lightning_amdgpu_ops*.so
```

For Kokkos, each real shared library must be able to locate its sibling library:

```bash
find "$PACKAGE_DIR/.libs" -maxdepth 1 -type f \
    -name 'libkokkos*.so.*' -print0 |
while IFS= read -r -d '' LIB; do
    patchelf --set-rpath '$ORIGIN' "$LIB"
done
```

If a future upstream version changes the native library names or package layout, inspect the unpacked wheel and adjust these targets. Do not silently skip a missing Lightning binary: verify the actual `.so` filenames before proceeding.

## 13. Include Kokkos license information

Copy the license file from the exact Kokkos source tree used for this build:

```bash
mkdir -p "$DIST_INFO_DIR/licenses"
cp "$KOKKOS_SRC/LICENSE" \
    "$DIST_INFO_DIR/licenses/Kokkos-LICENSE.txt"
```

Register the license file in the wheel metadata:

```bash
python - "$DIST_INFO_DIR/METADATA" <<'PY'
from pathlib import Path
import sys

metadata = Path(sys.argv[1])
content = metadata.read_text()
license_entry = "License-File: licenses/Kokkos-LICENSE.txt"

if license_entry not in content.splitlines():
    header, separator, body = content.partition("\n\n")
    if not separator:
        raise SystemExit("Could not locate the METADATA header/body separator")
    header += "\n" + license_entry
    content = header + separator + body
    metadata.write_text(content)

print("Kokkos license registered in wheel metadata.")
PY
```

The repository should also contain the Kokkos license and third-party notices. See the README's licensing section and the `THIRD_PARTY_NOTICES.md` file.

## 14. Remove native debug information

Use the LLVM strip utility installed with ROCm. For the initial Fedora/ROCm layout:

```bash
LLVM_STRIP=/usr/lib64/rocm/llvm/bin/llvm-strip
```

Check the path:

```bash
test -x "$LLVM_STRIP" && echo "llvm-strip found"
```

Strip debug sections from the Lightning native libraries:

```bash
"$LLVM_STRIP" --strip-debug \
    "$PACKAGE_DIR/liblightning_kokkos_catalyst.so"

"$LLVM_STRIP" --strip-debug \
    "$PACKAGE_DIR"/lightning_amdgpu_ops*.so
```

Strip the real Kokkos shared libraries:

```bash
find "$PACKAGE_DIR/.libs" -maxdepth 1 -type f \
    -name 'libkokkos*.so.*' -print0 |
while IFS= read -r -d '' LIB; do
    "$LLVM_STRIP" --strip-debug "$LIB"
done
```

Use `--strip-debug`, not `--strip-all`. Stripping debug sections is not a substitute for path remapping; both are used, and the next section verifies the result.

## 15. Audit native binaries for personal and build-machine paths

Before packaging, scan all native shared objects under the package directory. This check looks for Linux and macOS home-directory paths, the current build directory, and the current account name.

```bash
python - "$PACKAGE_DIR" "$BUILD_ROOT" <<'PY'
from pathlib import Path
import getpass
import re
import sys

package = Path(sys.argv[1])
build_root = sys.argv[2].encode()
username = getpass.getuser().encode()

patterns = [
    (re.compile(rb"/home/[^/\x00\s]+"), "Linux home-directory path"),
    (re.compile(rb"/Users/[^/\x00\s]+"), "macOS home-directory path"),
    (re.compile(re.escape(build_root)), "local build directory"),
    (re.compile(re.escape(username)), "current account name"),
]

native_files = []
for path in package.rglob("*"):
    if path.is_file() and (".so" in path.name):
        native_files.append(path)

if not native_files:
    raise SystemExit("FAIL: no native shared libraries found to audit")

matches = []
for path in native_files:
    data = path.read_bytes()
    for pattern, description in patterns:
        if pattern.search(data):
            matches.append((path.name, description))

if matches:
    print("FAIL: possible personal/build-machine paths detected:")
    for filename, description in matches:
        print(f"  {filename}: {description}")
    raise SystemExit(1)

print("PASS: no matching personal or original build paths found in native shared libraries.")
PY
```

If this check fails, **do not publish the wheel**. Identify which binary contains the path and determine why the prefix-map flags were not applied. Do not try to hide the path by merely running `strip` again.

## 16. Audit ELF runtime paths

Inspect the two Lightning native libraries:

```bash
readelf -d "$PACKAGE_DIR/liblightning_kokkos_catalyst.so" \
    | grep -E 'RPATH|RUNPATH'

readelf -d "$PACKAGE_DIR"/lightning_amdgpu_ops*.so \
    | grep -E 'RPATH|RUNPATH'
```

Inspect Kokkos:

```bash
for LIB in "$PACKAGE_DIR"/.libs/libkokkos*.so.*; do
    [ -f "$LIB" ] || continue
    echo "=== $(basename "$LIB") ==="
    readelf -d "$LIB" | grep -E 'RPATH|RUNPATH'
done
```

Expected paths:

- Lightning: `$ORIGIN/.libs` and `$ORIGIN/../scipy_openblas32/lib`.
- Kokkos: `$ORIGIN`.

No runtime path should point to `/home/...`, `/tmp/pl-gfx1201-build...`, or the original Kokkos installation directory.

## 17. Repack the final wheel

Only proceed after the privacy, license, and ELF path checks succeed:

```bash
python -m wheel pack \
    "$WHEEL_ROOT" \
    --dest-dir "$FINAL_WHEEL_DIR"
```

Locate the final artifact:

```bash
FINAL_WHEEL="$(find "$FINAL_WHEEL_DIR" -maxdepth 1 -type f \
    -name 'pennylane_lightning_amdgpu-*.whl' -print -quit)"

printf 'Final wheel: %s\n' "$FINAL_WHEEL"
```

Do not rename the wheel manually. Its filename contains Python, ABI, and platform tags generated by the packaging tool.

## 18. Inspect the final wheel contents and dynamic dependencies

List the archive contents:

```bash
python -m zipfile -l "$FINAL_WHEEL"
```

Confirm that it contains:

- `pennylane_lightning/liblightning_kokkos_catalyst.so`.
- `pennylane_lightning/lightning_amdgpu_ops...so`.
- `pennylane_lightning/.libs/` with Kokkos Core and Kokkos Containers.
- `*.dist-info/licenses/Kokkos-LICENSE.txt`.

Unpack the final wheel to a separate inspection directory:

```bash
FINAL_CHECK_DIR="$BUILD_ROOT/final-wheel-inspection"
mkdir -p "$FINAL_CHECK_DIR"
python -m wheel unpack "$FINAL_WHEEL" -d "$FINAL_CHECK_DIR"

CHECK_DIST_INFO="$(find "$FINAL_CHECK_DIR" -mindepth 2 -maxdepth 2 \
    -type d -name '*.dist-info' -print -quit)"
CHECK_ROOT="${CHECK_DIST_INFO%/*}"
CHECK_PACKAGE="$CHECK_ROOT/pennylane_lightning"
```

Check dynamic dependencies without relying on the temporary Kokkos build path:

```bash
env -u LD_LIBRARY_PATH ldd \
    "$CHECK_PACKAGE/liblightning_kokkos_catalyst.so"

env -u LD_LIBRARY_PATH ldd \
    "$CHECK_PACKAGE"/lightning_amdgpu_ops*.so
```

Expected:

- Kokkos libraries resolve from the wheel's `.libs` directory.
- `libamdhip64.so.7` resolves from the ROCm installation on the system.
- No library is reported as `not found`.

If HIP is reported as missing, inspect the system ROCm installation and dynamic-linker configuration. Do not solve this by reintroducing the build machine's Kokkos path.

### Record glibc and libstdc++ symbol requirements

The wheel's `linux_x86_64` tag does not encode a minimum glibc or libstdc++ version. Inspect the native binaries and record the highest required symbol versions for every release:

```bash
for SO in \
    "$CHECK_PACKAGE/liblightning_kokkos_catalyst.so" \
    "$CHECK_PACKAGE"/lightning_amdgpu_ops*.so \
    "$CHECK_PACKAGE"/.libs/libkokkos*.so.*
do
    [ -f "$SO" ] || continue
    echo "=== $(basename "$SO") ==="
    readelf --version-info "$SO" | grep -oE 'GLIBC_[0-9.]+|GLIBCXX_[0-9.]+' | sort -Vu | tail -n 12
done
```

Compare the requirements with the target systems and report the actual requirements in `BUILD-INFO.txt` and the release notes. The initial wheel was inspected and required `GLIBC_2.43` and `GLIBCXX_3.4.32`; do not copy those values into a future release without checking its binaries.

## 19. Install and test in a clean virtual environment

Create a separate test environment. It must not inherit the build environment's Kokkos variables or `LD_LIBRARY_PATH`:

```bash
TEST_VENV="$BUILD_ROOT/venv-smoke-test"

"$PYTHON_BIN" -m venv "$TEST_VENV"
"$TEST_VENV/bin/python" -m pip install --upgrade pip

"$TEST_VENV/bin/python" -m pip install \
    "pennylane==$PENNYLANE_VERSION" \
    "pennylane-lightning==$LIGHTNING_PYTHON_VERSION" \
    "scipy-openblas32>=0.3.26"

"$TEST_VENV/bin/python" -m pip install --no-deps "$FINAL_WHEEL"
"$TEST_VENV/bin/python" -m pip check
```

Run a smoke test with the temporary Kokkos library path removed:

```bash
env -u LD_LIBRARY_PATH \
    -u KOKKOS_ROOT \
    -u KOKKOS_DIR \
    -u CMAKE_PREFIX_PATH \
    "$TEST_VENV/bin/python" - <<'PY'
import numpy as np
import pennylane as qml


dev = qml.device("lightning.amdgpu", wires=2)


@qml.qnode(dev)
def circuit(theta):
    qml.RY(theta, wires=0)
    qml.CNOT(wires=[0, 1])
    return qml.state()


result = circuit(0.5)
expected = np.array(
    [np.cos(0.25), 0.0, 0.0, np.sin(0.25)],
    dtype=complex,
)

print("Device:", dev)
print("State:", result)
np.testing.assert_allclose(result, expected, atol=1e-8)
print("SUCCESS: clean virtual environment smoke test passed.")
PY
```

The expected final line is:

```text
SUCCESS: clean virtual environment smoke test passed.
```

This verifies the basic installation path in a fresh Python environment on the build machine. Testing on a second physical computer is still desirable before describing the wheel as independently verified on other systems.

## 20. Create permanent release assets

Do not leave the only copy of a release under `/tmp`, because temporary directories can be cleaned by the operating system. Copy the final wheel and generated metadata into a permanent local directory associated with the repository.

Set the local repository path (adjust it if the repository is stored elsewhere):

```bash
export REPO_DIR="$HOME/src/pennylane-lightning-amdgpu-gfx1201"
export RELEASE_DIR="$REPO_DIR/release-assets"
mkdir -p "$RELEASE_DIR"

cp "$FINAL_WHEEL" "$RELEASE_DIR/"
export RELEASE_WHEEL="$RELEASE_DIR/$(basename "$FINAL_WHEEL")"
```

Generate the SHA-256 checksum from inside the release directory so that the checksum file contains only the wheel filename, not a local absolute path:

```bash
cd "$RELEASE_DIR"
sha256sum "$(basename "$RELEASE_WHEEL")" > SHA256SUMS.txt
sha256sum -c SHA256SUMS.txt
```

Expected result:

```text
<wheel-filename>: OK
```

### Generate `BUILD-INFO.txt`

The following command creates a metadata file without including the local username, hostname, home directory, or build directory. Update the version variables in section 2 for each new release before generating it.

```bash
OS_NAME="$(. /etc/os-release; printf '%s' "$PRETTY_NAME")"
PYTHON_INFO="$("$PYTHON_BIN" --version 2>&1)"
GLIBC_INFO="$(getconf GNU_LIBC_VERSION)"
KOKKOS_COMMIT="$(git -C "$KOKKOS_SRC" rev-parse HEAD)"
LIGHTNING_COMMIT="$(git -C "$LIGHTNING_SRC" rev-parse HEAD)"
HIPCC_INFO="$(/usr/bin/hipcc --version 2>&1 | tail -n 2 | tr '\n' ' ')"
BUILD_DATE="$(date -u '+%Y-%m-%dT%H:%M:%SZ')"
WHEEL_NAME="$(basename "$RELEASE_WHEEL")"

cat > "$RELEASE_DIR/BUILD-INFO.txt" <<EOF
PROJECT: PennyLane-Lightning AMDGPU gfx1201 community build
KOKKOS SOURCE TAG: $KOKKOS_TAG
KOKKOS SOURCE COMMIT: $KOKKOS_COMMIT
PENNYLANE-LIGHTNING SOURCE TAG: $LIGHTNING_TAG
PENNYLANE-LIGHTNING SOURCE COMMIT: $LIGHTNING_COMMIT

PYTHON DEPENDENCIES
PennyLane: $PENNYLANE_VERSION
PennyLane-Lightning: $LIGHTNING_PYTHON_VERSION

BUILD CONFIGURATION
Operating system: $OS_NAME
Python: $PYTHON_INFO
ROCm: $ROCM_VERSION
HIP: $HIP_VERSION
HIP compiler: $HIPCC_INFO
GPU architecture target: $GPU_ARCH
Platform: Linux x86_64
Kokkos HIP backend: enabled
Kokkos Serial backend: enabled
C++ standard: C++20

SYSTEM ABI
Build-system glibc: $GLIBC_INFO
Initial wheel ABI audit: GLIBC_2.43 and GLIBCXX_3.4.32
Re-check ABI requirements for every new build.

WHEEL
Filename: $WHEEL_NAME
Kokkos Core and Kokkos Containers: bundled
AMD HIP runtime: external system dependency

VALIDATION
Build completed: yes
Native path audit: record the result of section 15
ELF RUNPATH audit: record the result of section 16
Clean virtual environment smoke test: record the result of section 19
Second physical computer tested: record yes/no and system details

BUILD DATE (UTC): $BUILD_DATE
EOF
```

Review `BUILD-INFO.txt` and replace the validation placeholders with the actual results. Do not leave a test marked as passed unless it was run against this exact final wheel.

The release-assets directory should contain:

```text
release-assets/
├── pennylane_lightning_amdgpu-...whl
├── SHA256SUMS.txt
└── BUILD-INFO.txt
```

Add `/release-assets/` to the repository's `.gitignore` if these local release files should not be committed to the normal Git history. Upload the three files as assets attached to the corresponding GitHub Release instead.

## 21. Final release checklist

Before publishing, verify all of the following:

- [ ] Source tags and commit IDs are recorded.
- [ ] Kokkos is compiled with HIP enabled and `AMD_GFX1201` selected.
- [ ] Compiler prefix-map flags appear in the build rules for both projects.
- [ ] Kokkos Core and Kokkos Containers are bundled in the wheel.
- [ ] Lightning and Kokkos ELF runtime paths are relative and contain no build-machine path.
- [ ] The Kokkos license is included in the wheel metadata directory.
- [ ] Native binaries pass the personal-path audit.
- [ ] `ldd` reports no missing libraries; HIP resolves from the target system.
- [ ] The final wheel installs in a clean virtual environment.
- [ ] The basic `lightning.amdgpu` circuit passes in that clean environment.
- [ ] `SHA256SUMS.txt` validates the exact wheel being uploaded.
- [ ] `BUILD-INFO.txt` accurately describes the build and tests.

## 22. Updating the project approximately every six months

For each maintenance cycle:

1. Review the upstream PennyLane-Lightning release notes and source build configuration.
2. Review Kokkos and ROCm/HIP compatibility. Do not upgrade these components independently without checking their compatibility with the selected Lightning release.
3. Update the version variables in section 2 and the Python dependency pins in section 5.
4. Use a new `BUILD_ROOT`; do not reuse old CMake build directories.
5. Repeat the full build, packaging, privacy audit, ABI inspection, and clean-environment smoke test.
6. Generate a new `BUILD-INFO.txt` and checksum from the exact final wheel.
7. Publish a new GitHub Release with a tag that distinguishes the upstream Lightning version, GPU target, and ROCm version, for example `v0.45.0-gfx1201-rocm7.1.1`.

A six-month review is a maintenance target, not a promise that a newer upstream release will always be compatible. Retain the last known-good release until the next build has passed its validation checklist.
