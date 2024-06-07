# Maintainer: WeirdBeard <obarrtimothy@gmail.com>
# Contributor: Christian Hegerstroem
# Contributor: éclairevoyant
# Contributor: Maxime Gauduin <alucryd@archlinux.org>
# Contributor: Themaister <maister@archlinux.us>

pkgname=pcsx2-git
pkgver=1.7.5867.r0.g5e858fa1bc
pkgrel=1
pkgdesc='A Sony PlayStation 2 emulator'
arch=(x86_64)
url=https://github.com/PCSX2/pcsx2

# https://github.com/PCSX2/pcsx2/blob/master/.github/workflows/scripts/linux/build-dependencies-qt.sh
_shaderc=2024.1
_shaderc_glslang=142052fa30f9eca191aa9dcf65359fcaed09eeec
_shaderc_spirvheaders=5e3ad389ee56fca27c9705d093ae5387ce404df4
_shaderc_spirvtools=dd4b663e13c07fea4fbb3f70c1c91c86731099f7

license=(
    GPL2
    GPL3
    LGPL2.1
    LGPL3
)

depends=(
    libaio
    libpcap
    libglvnd
    libxrandr
    alsa-lib
    ffmpeg
    sdl2
    lld
    qt6-base
    qt6-svg
    soundtouch
    wayland
    libpng
    hicolor-icon-theme
    xcb-util-cursor
    libbacktrace-git
    )
makedepends=(
    cmake
    extra-cmake-modules
    clang
    lld
    llvm
    git
    ninja
    libpulse
    libpipewire
    p7zip
    qt6-wayland
    qt6-tools
)
optdepends=(
    'qt6-wayland: Wayland support'
    'libpulse: Pulseaudio support'
    'libpipewire: Pipewire support'
)
provides=(${pkgname%-git})
conflicts=(${pkgname%-git})
options=(!lto)
source=(
    git+https://github.com/PCSX2/pcsx2.git
    git+https://github.com/PCSX2/pcsx2_patches.git
    git+https://github.com/google/googletest.git
    git+https://github.com/fmtlib/fmt.git
    git+https://github.com/biojppm/rapidyaml.git
    git+https://github.com/biojppm/cmake.git
    git+https://github.com/biojppm/c4core.git
    git+https://github.com/biojppm/debugbreak.git
    git+https://github.com/fastfloat/fast_float.git
    vulkan-headers::git+https://github.com/KhronosGroup/Vulkan-Headers.git
    shaderc::git+https://github.com/google/shaderc.git#tag=v"${_shaderc}"
    glslang::git+https://github.com/KhronosGroup/glslang.git#commit="${_shaderc_glslang}"
    spirv-headers::git+https://github.com/KhronosGroup/SPIRV-Headers.git#commit="${_shaderc_spirvheaders}"
    spirv-tools::git+https://github.com/KhronosGroup/SPIRV-Tools.git#commit="${_shaderc_spirvtools}"
    pcsx2-qt.sh
)
install=pcsx2-git.install

prepare() {
    cd pcsx2
    local submodule
    _pcsx2_submodules=(
        googletest::3rdparty/gtest
        fmt::3rdparty/fmt/fmt
        rapidyaml::3rdparty/rapidyaml/rapidyaml
        vulkan-headers::3rdparty/vulkan-headers
    )
    for submodule in "${_pcsx2_submodules[@]}"; do
        git submodule init "${submodule#*::}"
        git submodule set-url "${submodule#*::}" "${srcdir}"/"${submodule%::*}"
        git -c protocol.file.allow=always submodule update "${submodule#*::}"
    done
    
    cd 3rdparty/rapidyaml/rapidyaml
    for submodule in ext/c4core; do
        git submodule init ${submodule}
        git submodule set-url ${submodule} "${srcdir}/${submodule##*/}"
        git -c protocol.file.allow=always submodule update ${submodule}
    done
    
    cd ext/c4core
    for submodule in cmake src/c4/ext/{debugbreak,fast_float}; do
        git submodule init ${submodule}
        git submodule set-url ${submodule} "${srcdir}/${submodule##*/}"
        git -c protocol.file.allow=always submodule update ${submodule}
    done

    ln -s "${srcdir}"/glslang "${srcdir}"/shaderc/third_party/glslang
    ln -s "${srcdir}"/spirv-headers "${srcdir}"/shaderc/third_party/spirv-headers
    ln -s "${srcdir}"/spirv-tools "${srcdir}"/shaderc/third_party/spirv-tools

    patch -p1 -i "${srcdir}"/pcsx2/.github/workflows/scripts/common/shaderc-changes.patch -d "${srcdir}"/shaderc
}

pkgver() {
    cd pcsx2
    git describe --long --tags | sed 's/\([^-]*-g\)/r\1/;s/-/./g;s/^v//'
}

build() {
    # Build custom shaderc

    cmake -S shaderc -B shaderc_build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_PREFIX_PATH="${srcdir}/shaderc_out" \
    -DCMAKE_INSTALL_PREFIX="${srcdir}/shaderc_out" \
    -DSHADERC_SKIP_TESTS=ON \
    -DSHADERC_SKIP_EXAMPLES=ON \
    -DSHADERC_SKIP_COPYRIGHT_CHECK=ON 

    cmake --build shaderc_build --parallel
    ninja -C shaderc_build install

    # Build PCSX2

    cmake -S pcsx2 -B build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_EXE_LINKER_FLAGS_INIT="-fuse-ld=lld" \
    -DCMAKE_MODULE_LINKER_FLAGS_INIT="-fuse-ld=lld" \
    -DCMAKE_SHARED_LINKER_FLAGS_INIT="-fuse-ld=lld" \
    -DCMAKE_PREFIX_PATH="${srcdir}/shaderc_out" \
    -DCMAKE_BUILD_RPATH='/opt/pcsx2' \
    -DUSE_VULKAN=ON \
    -DENABLE_SETCAP=OFF \
    -DX11_API=ON \
    -DWAYLAND_API=ON \
    -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON \
    -DDISABLE_ADVANCE_SIMD=ON
    ninja -C build

    cd pcsx2_patches
    7z a -r ../patches.zip patches/.
}

package() {
    install -dm755  "${pkgdir}"/opt/
    cp -r build/bin "${pkgdir}"/opt/"${pkgname%-git}"
    install -Dm755 pcsx2-qt.sh "$pkgdir"/usr/bin/pcsx2-qt
    install -Dm644 pcsx2/.github/workflows/scripts/linux/pcsx2-qt.desktop \
    "${pkgdir}"/usr/share/applications/PCSX2.desktop
    install -Dm644 pcsx2/bin/resources/icons/AppIconLarge.png \
    "${pkgdir}"/usr/share/icons/hicolor/512x512/apps/PCSX2.png
    install -Dm644 -t "${pkgdir}"/opt/"${pkgname%-git}"/resources/ patches.zip
    install -Dm644 -t "${pkgdir}"/opt/"${pkgname%-git}"/ "${srcdir}"/shaderc_out/lib/libshaderc_shared.so.1
}

b2sums=('SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        'SKIP'
        '956d3547f316de51de4712e6ad6cf5621efadd222ef6c1aa18508321949e63d6b3dc32cdc7eabbcb8172b4b77593485e3debe0b250ec3d0c6926170d80baf3ef')
