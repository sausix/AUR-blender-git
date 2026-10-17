#!/bin/bash
# Maintainer: Fredrick Brennan <copypaste@kittens.ph>
# Submitter: Lukas Jirkovsky <l.jirkovsky@gmail.com>
# Co-maintainer: bartus <arch-user-repoᘓbartus.33mail.com>
# Contributor: Adrian Sausenthaler <aur@sausenthaler.de>

# Optional SDKs and libraries are enabled when installed on the build system.
# Use CUDA_ARCH / HIP_ARCH for a single GPU architecture or a semicolon-separated list.
# SYCL_ROOT_DIR, CUDA_PATH and HIP_ROOT_DIR override the default SDK locations.
# ONEAPI_JOBS / HIPRT_JOBS control GPU compiler parallelism (default: 1).
# Use FRAGMENT='#commit=...' or '#tag=...' to build a particular revision.
_fragment="${FRAGMENT:-#branch=main}"

pkgname=blender-git
pkgver=5.4.r166331.g3fe0f2ab2a29
pkgrel=1
pkgdesc="A fully integrated 3D graphics creation suite (development)"
arch=('x86_64')
url="https://blender.org/"
options=(!lto)
depends+=('alembic' 'ceres-solver' 'eigen' 'embree' 'libgl' 'python' 'python-numpy' 'openjpeg2' 'libharu' 'potrace' 'openxr'
          'ffmpeg' 'fftw' 'openal' 'freetype2' 'libxi' 'manifold' 'openimageio' 'opencolorio'
          'openvdb' 'opensubdiv' 'openshadinglanguage' 'libtiff' 'libpng'
          'python-zstandard' 'google-glog' 'gflags' 'libsndfile' 'pugixml' 'rubberband')
depends+=('libdecor' 'libepoxy' 'shaderc' 'opentimelineio')
optdepends=('cuda: Precompile CUDA kernels for Cycles'
            'optix8: Build OptiX support in Cycles >=8.0.0 <9.0.0'
            'usd: USD export Scene'
            'openpgl: Intel Path Guiding library in Cycles'
            'openimagedenoise: Intel Open Image Denoise support in compositing'
            'materialx: MaterialX materials'
            'level-zero-headers: Build Cycles oneAPI support'
            'intel-compute-runtime: Intel GPU runtime for Cycles oneAPI'
            'intel-oneapi-compiler-shared-runtime: Cycles oneAPI runtime'
            'intel-oneapi-dpcpp-cpp: Build Cycles oneAPI kernels'
            'hip-runtime-amd: Build Cycles HIP kernels'
            'hiprt: Build Cycles HIP-RT kernels'
            'makepkg-cg: Control resources during compilation')

# These libraries are optional at build time, but required by binaries built with them.
for _optional_pkg in usd openpgl openimagedenoise materialx; do
  if pacman -Qq "$_optional_pkg" &>/dev/null; then
    depends+=("$_optional_pkg")
  fi
done
unset _optional_pkg

makedepends+=('git' 'cmake' 'boost' 'mesa' 'llvm' 'clang' 'subversion' 'ninja' 'mold' 'ccache' 'opencl-headers')
makedepends+=('wayland-protocols')
makedepends+=('cython')
makedepends+=('vulkan-headers' 'spirv-headers')
makedepends+=('makepkg-git-lfs-proto') # provides /usr/share/makepkg/source/git-lfs.sh (download_git-lfs, extract_git-lfs)
provides=('blender')
conflicts=('blender')
license=('GPL')
source=("blender::git-lfs+https://projects.blender.org/blender/blender${_fragment}"
        'blender-fix-otio-namespace.patch'  # `extra/opentimelineio` is currently outdated compared to Blender's version . `aur/opentimelineio-git` is orphaned. So a patch is the best choice.
  )
sha256sums=('SKIP'
            '7c8fbc2e429aa8e803d0c290ab8b20679f00288ba8415d5f46645cff85572e29'
  )

pkgver() {
  local blender_version
  blender_version=$(grep -Po "^#define BLENDER_VERSION \K[0-9]+" "$srcdir"/blender/source/blender/blenkernel/BKE_blender_version.h)
  printf "%d.%d.r%s.g%s" \
    $((blender_version/100)) \
    $((blender_version%100)) \
    "$(git -C "$srcdir/blender" rev-list --count HEAD)" \
    "$(git -C "$srcdir/blender" rev-parse --short HEAD)"
}

prepare() {
  local patch
  for patch in "${source[@]}"; do
    patch=${patch%%::*}
    case "$patch" in
      *.patch|*.diff)
        patch="$srcdir/${patch##*/}"
        if git -C "$srcdir/blender" apply --reverse --check "$patch" &>/dev/null; then
          msg2 "${patch##*/} already applied"
        else
          msg2 "apply ${patch##*/}..."
          git -C "$srcdir/blender" apply -v "$patch" || return
        fi
        ;;
    esac
  done
}

build() {
  local _pyver _pkg _option _enabled _sycl_root _candidate
  local _sycl_compiler=''
  local _cuda_root=${CUDA_PATH:-/opt/cuda}
  local _hip_root=${HIP_ROOT_DIR:-/opt/rocm}
  local -a _CMAKE_FLAGS=() _makeflags=()
  _pyver=$(python -c "from sys import version_info; print(\"%d.%d\" % (version_info[0],version_info[1]))")
  msg "python version detected: ${_pyver}"

  # Start with official defaults and override individual options later
  _CMAKE_FLAGS+=( -C "${srcdir}/blender/build_files/cmake/config/blender_release.cmake"
                  -G Ninja
                  -DCMAKE_INSTALL_PREFIX=/usr
                  -DCMAKE_INSTALL_PREFIX_WITH_CONFIG=/usr
                  -DWITH_LINKER_MOLD=ON
                  -DCMAKE_SKIP_INSTALL_RPATH=ON
                  -DCMAKE_SKIP_BUILD_RPATH=ON
                  -DCMAKE_BUILD_TYPE=Release
                  -DWITH_INSTALL_PORTABLE=OFF
                  -DWITH_LIBS_PRECOMPILED=OFF
                  -DWITH_STATIC_LIBS=OFF
                  -DWITH_CYCLES_HYDRA_RENDER_DELEGATE=OFF
                  -DXR_OPENXR_SDK_ROOT_DIR=/usr
                  -DWITH_DOC_MANPAGE=OFF
                  -DWITH_SYSTEM_GLOG=ON
                  -DWITH_SYSTEM_GFLAGS=ON
                  )

  # Python config to build against system Python
  _CMAKE_FLAGS+=( -DPYTHON_VERSION=$_pyver
                  -DPYTHON_LIBRARY="/usr/lib/libpython${_pyver}.so"
                  -DPYTHON_ROOT_DIR=/usr
                  -DPYTHON_EXECUTABLE="/usr/bin/python${_pyver}"
                  -DPYTHON_INCLUDE_DIR="/usr/include/python${_pyver}"
                  -DPYTHON_INCLUDE_CONFIG_DIR="/usr/include/python${_pyver}"
                  -DPYTHON_LIBPATH=/usr/lib
                  -DWITH_PYTHON_INSTALL=OFF
                  -DWITH_PYTHON_SAFETY=OFF )


  # Use the compiler layout of both current and older oneAPI installations.
  _sycl_root=${SYCL_ROOT_DIR:-/opt/intel/oneapi/compiler/latest}
  for _candidate in bin/icpx bin/dpcpp bin/clang++ bin/compiler/clang++ linux/bin/dpcpp; do
    if [[ -x "$_sycl_root/$_candidate" ]]; then
      _sycl_compiler="$_sycl_root/$_candidate"
      [[ $_candidate == linux/* ]] && _sycl_root+=/linux
      break
    fi
  done
  if [[ -n $_sycl_compiler ]]; then
    _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_ONEAPI=ON
                    -DWITH_CYCLES_ONEAPI_BINARIES=ON
                    -DSYCL_COMPILER="$_sycl_compiler"
                    -DSYCL_ROOT_DIR="$_sycl_root"
                    -DOCLOC_INSTALL_DIR=/usr
                    -DIGC_INSTALL_DIR=/usr
                    -DSYCL_OFFLINE_COMPILER_PARALLEL_JOBS="${ONEAPI_JOBS:-1}" )
    # CMake validates SYCL, Level Zero and ocloc; missing ocloc falls back to JIT.
  else
    _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_ONEAPI=OFF
                    -DWITH_CYCLES_ONEAPI_BINARIES=OFF )
  fi

  # The runtime CUDA backend also works without the CUDA build toolkit.
  if [[ -x "$_cuda_root/bin/nvcc" ]]; then
    _CMAKE_FLAGS+=( -DWITH_CYCLES_CUDA_BINARIES=ON
                    -DWITH_COMPILER_ASAN=OFF
                    -DCUDA_TOOLKIT_ROOT_DIR="$_cuda_root"
                    -DCUDA_NVCC_EXECUTABLE="$_cuda_root/bin/nvcc"
                    -DCUDA_HOST_COMPILER="${NVCC_CCBIN:-/usr/bin/g++}" )
  else
    _CMAKE_FLAGS+=( -DWITH_CYCLES_CUDA_BINARIES=OFF )
  fi

  while read -r _pkg _option; do
    _enabled=OFF
    pacman -Qq "$_pkg" &>/dev/null && _enabled=ON
    _CMAKE_FLAGS+=( "-D${_option}=${_enabled}" )
  done <<'OPTIONAL_LIBRARIES'
materialx WITH_MATERIALX
usd WITH_USD
openimagedenoise WITH_OPENIMAGEDENOISE
openpgl WITH_CYCLES_PATH_GUIDING
OPTIONAL_LIBRARIES

  if pacman -Qq optix8 &>/dev/null && [[ -f /opt/optix/include/optix.h ]]; then
    _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_OPTIX=ON -DOPTIX_ROOT_DIR=/opt/optix )
  else
    _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_OPTIX=OFF )
  fi

  # AMD
  if [[ -x "$_hip_root/bin/hipcc" ]]; then
      _CMAKE_FLAGS+=( -DWITH_CYCLES_HIP_BINARIES=ON
                      -DWITH_CYCLES_DEVICE_HIP=ON
                      -DHIP_ROOT_DIR="$_hip_root"
                      -DHIPRT_COMPILER_PARALLEL_JOBS="${HIPRT_JOBS:-1}"
                    )
      if [[ -f "$_hip_root/include/hiprt/hiprt.h" ]]; then
        _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_HIPRT=ON -DHIPRT_ROOT_DIR="$_hip_root" )
      else
        _CMAKE_FLAGS+=( -DWITH_CYCLES_DEVICE_HIPRT=OFF )
      fi
  else
      # Because some defaults are ON.
      _CMAKE_FLAGS+=( -DWITH_CYCLES_HIP_BINARIES=OFF
                      -DWITH_CYCLES_DEVICE_HIP=OFF
                      -DWITH_CYCLES_DEVICE_HIPRT=OFF
                    )
  fi

  # Apply architecture overrides after the release preset, which uses FORCE.
  [[ -v CUDA_ARCH ]] && _CMAKE_FLAGS+=( -DCYCLES_CUDA_BINARIES_ARCH="$CUDA_ARCH" )
  [[ -v HIP_ARCH ]] && _CMAKE_FLAGS+=( -DCYCLES_HIP_BINARIES_ARCH="$HIP_ARCH" )

  cmake -S "$srcdir/blender" -B "$srcdir/build" "${_CMAKE_FLAGS[@]}" || return
  # MAKEFLAGS may contain Ninja options such as -j and -l; pass them to Ninja.
  read -r -a _makeflags <<< "${MAKEFLAGS:--j1}"
  cmake --build "$srcdir/build" -- "${_makeflags[@]}"
}

package() {
  local _suffix=${pkgver%%.r*} _oneapi_lib
  local -a _kernels=()
  BLENDER_SYSTEM_RESOURCES="${pkgdir}/usr/share/blender/${_suffix}" \
    DESTDIR="$pkgdir" cmake --install "$srcdir/build" || return

  cd "$srcdir/blender" || return
  install -Dm755 release/bin/blender-launcher "${pkgdir}/usr/bin/blender-launcher"
  install -Dm644 release/freedesktop/org.blender.Blender.metainfo.xml "${pkgdir}/usr/share/metainfo/org.blender.Blender.metainfo.xml"
  install -vDm 644 doc/license/{BSD-{2,3}-Clause,MIT,Zlib}-license.txt -t "$pkgdir/usr/share/licenses/blender/"

  python -m compileall "${pkgdir}/usr/share/blender"
  python -O -m compileall "${pkgdir}/usr/share/blender"

  # Both AOT and JIT builds link this library into Blender.
  for _oneapi_lib in "$pkgdir"/usr/share/blender/lib/libcycles_kernel_oneapi_{aot,jit}.so; do
    if [[ -f $_oneapi_lib ]]; then
      install -d "$pkgdir/usr/lib"
      mv "$_oneapi_lib" "$pkgdir/usr/lib/" || return
    fi
  done

  # GPU kernel files are data; makepkg must not strip them.
  shopt -s nullglob
  _kernels=("$pkgdir/usr/share/blender/$_suffix/scripts/addons_core/cycles/lib/"*)
  shopt -u nullglob
  if (( ${#_kernels[@]} )); then
    chmod 444 "${_kernels[@]}"
  fi
}

# vim: syntax=bash:et:ts=2:sw=2
