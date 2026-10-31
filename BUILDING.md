# Building PennyLane-Lightning AMDGPU for gfx1201

This document describes how to reproduce the custom AMDGPU build and package a relocatable wheel for Linux x86_64 and AMD `gfx1201` GPUs.

## 1. Build configuration

The validated configuration for this release is:

| Component | Version |
|---|---|
| Python | 3.14 |
| PennyLane | 0.45.1 |
| PennyLane-Lightning | 0.45.0 |
| Kokkos | 5.1.0 |
| ROCm / HIP | 7.1.1 |
| GPU architecture | AMD GFX1201 |
| Build system | CMake + Ninja |
| Python packaging | pip + wheel |

A compatible ROCm installation, `hipcc`, CMake, Ninja, a C++ compiler, and the ROCm development libraries are required.

The build was originally validated on Fedora Linux 44.

## 2. Install build tools

On Fedora, install the required system tools and libraries:

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

Ensure that ROCm 7.1.1 is installed and that:

```bash
hipcc --version
rocminfo
```

work correctly.

The GPU agent should report `gfx1201`.

## 3. Create a clean build workspace

Use a neutral temporary directory rather than a personal home directory:

```bash
export BUILD_ROOT=/tmp/pl-gfx1201-build

mkdir -p "$BUILD_ROOT"

export KOKKOS_SRC="$BUILD_ROOT/kokkos"
export KOKKOS_BUILD="$BUILD_ROOT/kokkos-build"
export KOKKOS_ROOT="$BUILD_ROOT/kokkos-install"

export LIGHTNING_SRC="$BUILD_ROOT/pennylane-lightning"
export RAW_WHEEL_DIR="$BUILD_ROOT/raw-wheel"
export UNPACKED_DIR="$BUILD_ROOT/unpacked"
export FINAL_WHEEL_DIR="$BUILD_ROOT/final-wheel"

mkdir -p "$RAW_WHEEL_DIR" \
         "$UNPACKED_DIR" \
         "$FINAL_WHEEL_DIR"
```

## 4. Configure privacy-preserving compiler flags

Use compiler path-remapping flags so that source and build directories are not embedded in debug information or compiler-generated file paths.

```bash
export PRIVACY_FLAGS="\
-ffile-prefix-map=${BUILD_ROOT}=/usr/src/pl-gfx1201 \
-fdebug-prefix-map=${BUILD_ROOT}=/usr/src/pl-gfx1201 \
-fmacro-prefix-map=${BUILD_ROOT}=/usr/src/pl-gfx1201"
```

These flags must be applied to both Kokkos and PennyLane-Lightning.

The build workspace is deliberately located outside the user's home directory.

## 5. Create the Python build environment

```bash
python3.14 -m venv "$BUILD_ROOT/venv"

source "$BUILD_ROOT/venv/bin/activate"

python -m pip install --upgrade pip

python -m pip install \
    "setuptools>=75.8.1" \
    wheel \
    cmake \
    ninja \
    tomli \
    tomlkit \
    "pennylane==0.45.1" \
    "pennylane-lightning==0.45.0" \
    "scipy-openblas32>=0.3.26"
```

## 6. Download the pinned source versions

Download Kokkos 5.1.0:

```bash
git clone \
    --depth 1 \
    --branch 5.1.0 \
    https://github.com/kokkos/kokkos.git \
    "$KOKKOS_SRC"
```

Download PennyLane-Lightning 0.45.0:

```bash
git clone \
    --depth 1 \
    --branch v0.45.0 \
    https://github.com/PennyLaneAI/pennylane-lightning.git \
    "$LIGHTNING_SRC"
```

Do not use an unpinned development branch for a reproducible release.

## 7. Compile Kokkos for AMD GFX1201

Configure the Kokkos build:

```bash
cmake \
    -S "$KOKKOS_SRC" \
    -B "$KOKKOS_BUILD" \
    -G Ninja \
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
    -DCMAKE_CXX_FLAGS="$PRIVACY_FLAGS" \
    -DCMAKE_INSTALL_RPATH='$ORIGIN'
```

Compile and install:

```bash
cmake --build "$KOKKOS_BUILD" --parallel 6

cmake --install "$KOKKOS_BUILD"
```

Confirm the installation:

```bash
ls -l "$KOKKOS_ROOT/lib64/"
```

The installation should include:

```text
libkokkoscore.so.5.1
libkokkoscontainers.so.5.1
```

## 8. Configure PennyLane-Lightning

Set the Kokkos paths:

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

Provide CMake with the HIP and Kokkos configuration:

```bash
export CMAKE_ARGS="\
-DCMAKE_CXX_COMPILER=/usr/bin/hipcc \
-DKokkos_ENABLE_HIP=ON \
-DKokkos_ARCH_AMD_GFX1201=ON \
-DKokkos_DIR=${KOKKOS_DIR} \
-DCMAKE_PREFIX_PATH=${KOKKOS_ROOT}:/usr \
-DCMAKE_CXX_FLAGS='${PRIVACY_FLAGS}'"
```

## 9. Build the raw wheel

```bash
python -m pip wheel . \
    --no-deps \
    --no-build-isolation \
    --wheel-dir "$RAW_WHEEL_DIR"
```

The resulting wheel is the intermediate build artifact. It must not be published yet.

## 10. Unpack and bundle Kokkos

Unpack the generated wheel:

```bash
RAW_WHEEL="$(find "$RAW_WHEEL_DIR" \
    -maxdepth 1 \
    -type f \
    -name 'pennylane_lightning_amdgpu-*.whl' \
    -print -quit)"

python -m wheel unpack "$RAW_WHEEL" -d "$UNPACKED_DIR"
```

Define the unpacked package directories:

```bash
DIST_DIR="$UNPACKED_DIR/pennylane_lightning_amdgpu-0.45.0"

PACKAGE_DIR="$DIST_DIR/pennylane_lightning"

DIST_INFO_DIR="$DIST_DIR/pennylane_lightning_amdgpu-0.45.0.dist-info"
```

Create the private library directory:

```bash
mkdir -p "$PACKAGE_DIR/.libs"
```

Copy both Kokkos shared libraries, preserving their symbolic links:

```bash
cp -a "$KOKKOS_ROOT"/lib64/libkokkoscore.so.5.1* \
    "$PACKAGE_DIR/.libs/"

cp -a "$KOKKOS_ROOT"/lib64/libkokkoscontainers.so.5.1* \
    "$PACKAGE_DIR/.libs/"
```

## 11. Correct the ELF runtime paths

The Lightning binaries must find Kokkos inside the installed Python package, not in the original build directory.

Patch both Lightning binaries:

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

The Kokkos libraries must be able to locate their sibling libraries:

```bash
patchelf --set-rpath '$ORIGIN' \
    "$PACKAGE_DIR/.libs/libkokkoscore.so.5.1.0"

patchelf --set-rpath '$ORIGIN' \
    "$PACKAGE_DIR/.libs/libkokkoscontainers.so.5.1.0"
```

## 12. Preserve third-party license information

Copy the Kokkos license into the wheel metadata:

```bash
mkdir -p "$DIST_INFO_DIR/licenses"

cp "$KOKKOS_SRC/LICENSE" \
    "$DIST_INFO_DIR/licenses/Kokkos-LICENSE.txt"
```

The repository should also retain the corresponding Kokkos license and third-party notices.

## 13. Remove debug information

Use the LLVM strip utility distributed with ROCm:

```bash
LLVM_STRIP=/usr/lib64/rocm/llvm/bin/llvm-strip
```

Remove debug sections from the Lightning and Kokkos binaries:

```bash
"$LLVM_STRIP" --strip-debug \
    "$PACKAGE_DIR/liblightning_kokkos_catalyst.so"

"$LLVM_STRIP" --strip-debug \
    "$PACKAGE_DIR"/lightning_amdgpu_ops*.so

"$LLVM_STRIP" --strip-debug \
    "$PACKAGE_DIR/.libs/libkokkoscore.so.5.1.0"

"$LLVM_STRIP" --strip-debug \
    "$PACKAGE_DIR/.libs/libkokkoscontainers.so.5.1.0"
```

`--strip-debug` removes debug information without intentionally removing executable code.

## 14. Repack the wheel

```bash
python -m wheel pack \
    "$DIST_DIR" \
    --dest-dir "$FINAL_WHEEL_DIR"
```

The final output should be a CPython 3.14 Linux x86_64 wheel.

## 15. Check for personal paths and absolute runtime paths

Inspect the Lightning runtime path:

```bash
readelf -d \
    "$PACKAGE_DIR/liblightning_kokkos_catalyst.so" \
    | grep -E 'RPATH|RUNPATH'
```

Inspect the AMDGPU extension:

```bash
readelf -d \
    "$PACKAGE_DIR"/lightning_amdgpu_ops*.so \
    | grep -E 'RPATH|RUNPATH'
```

The paths should be relative to `$ORIGIN`. They must not reference the original build machine.

Search the compiled binaries for home-directory paths without printing the matching content:

```bash
for file in \
    "$PACKAGE_DIR/liblightning_kokkos_catalyst.so" \
    "$PACKAGE_DIR"/lightning_amdgpu_ops*.so \
    "$PACKAGE_DIR"/.libs/libkokkos*.so.5.1.0
do
    if strings -a "$file" | grep -Eq '/home/[^/]+|/Users/[^/]+|[A-Z]:\\Users\\'
    then
        echo "WARNING: path-like personal information found in $(basename "$file")"
    fi
done
```

A warning requires further inspection before publication.

## 16. Generate the checksum

```bash
cd "$FINAL_WHEEL_DIR"

sha256sum ./*.whl > SHA256SUMS.txt

cat SHA256SUMS.txt
```

Keep this checksum with the corresponding GitHub Release.

## 17. Final validation

Install the final wheel into a clean virtual environment with the pinned Python dependencies.

Verify:

```bash
python -m pip check
```

Then execute a basic `lightning.amdgpu` quantum circuit.

Do not publish a release until the installation and functional smoke test have succeeded.

## 18. Release artifacts

Each release should include:

- The final binary `.whl`.
- `SHA256SUMS.txt`.
- A build manifest recording compiler, ROCm, Kokkos, Lightning, Python, OS, and GPU architecture versions.

Do not upload intermediate build directories, personal environment paths, CMake caches, or raw unpatched wheels.
