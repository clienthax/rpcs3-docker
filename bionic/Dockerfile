FROM ubuntu:18.04
LABEL maintainer="hcorion"


ENV QTVER=5.15.2
ENV QTVERMIN=515
ENV LLVMVER=14
ENV GCCVER=11

ENV CLANG_BINARY=clang-${LLVMVER}
ENV CLANGXX_BINARY=clang++-${LLVMVER}
ENV LLD_BINARY=lld-${LLVMVER}
ENV GCC_BINARY=gcc-${GCCVER}
ENV GXX_BINARY=g++-${GCCVER}

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update -y

RUN apt-get install -y curl libcurl4-openssl-dev software-properties-common apt-transport-https
RUN curl -Lvs https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add -
RUN add-apt-repository -y ppa:beineri/opt-qt-${QTVER}-bionic
RUN add-apt-repository -y ppa:ubuntu-toolchain-r/test
RUN add-apt-repository -y "deb http://apt.llvm.org/bionic/ llvm-toolchain-bionic-${LLVMVER} main"
RUN apt-get update -y --fix-missing
	
RUN apt-get install -y \
	build-essential binutils $GCC_BINARY $GXX_BINARY git ccache bc $CLANG_BINARY $CLANGXX_BINARY $LLD_BINARY \
	libasound2-dev libopenal-dev freeglut3-dev libevdev-dev libpulse-dev libxrandr-dev libglvnd-dev \
	libwayland-dev libwayland-cursor0 libwayland-client0 libwayland-server0 \
	desktop-file-utils dput gnupg pbuilder ubuntu-dev-tools apt-file \
	qt${QTVERMIN}base qt${QTVERMIN}declarative qt${QTVERMIN}xmlpatterns qt${QTVERMIN}svg qt${QTVERMIN}multimedia

# libudev-dev depenency is broken
# libudev-dev : Depends: libudev1 (= 237-3ubuntu10.53) but 237-3ubuntu10.54 is to be installed


RUN apt-get install --only-upgrade dpkg
RUN apt-get clean autoclean
RUN apt-get autoremove --yes
RUN rm -rf /var/lib/apt
	
RUN curl -sSfLO http://http.us.debian.org/debian/pool/main/g/glew/libglew2.2_2.2.0-4_amd64.deb
RUN curl -sSfLO http://http.us.debian.org/debian/pool/main/g/glew/libglew-dev_2.2.0-4_amd64.deb
RUN dpkg -i ./*.deb
RUN rm ./*.deb

# Setup newer CMake
ENV CMAKEVER=3.16.9
RUN curl -sSfLO https://cmake.org/files/v${CMAKEVER%.*}/cmake-${CMAKEVER}-Linux-x86_64.sh
RUN sh cmake-${CMAKEVER}-Linux-x86_64.sh --skip-license
RUN rm ./cmake*.sh
RUN cmake --version

# We gotta build our own version of Ninja as well
ENV NINJAVER=1.11.0
RUN curl -sSfL -o ninja.tar.gz https://github.com/ninja-build/ninja/archive/v${NINJAVER}.tar.gz
RUN tar -xf ninja.tar.gz
RUN cd ninja-*
RUN cmake -Bbuild-cmake -DCMAKE_C_COMPILER=$CLANG_BINARY -DCMAKE_CXX_COMPILER=$CLANGXX_BINARY -H.
RUN cmake --build build-cmake
RUN cp ./build-cmake/ninja /usr/bin/ninja
RUN cd ../
RUN rm -r ./ninja-*
RUN rm ./*.tar.gz

# Need to manually build vulkan headers and libvulkan
ENV VULKAN_HEADER_VER=1.2.198.0
ENV VULKAN_LOADER_VER=1.2.198.1
RUN curl -sSfL -o Vulkan-Headers.tar.gz https://github.com/KhronosGroup/Vulkan-Headers/archive/sdk-${VULKAN_HEADER_VER}.tar.gz
RUN tar -xf Vulkan-Headers*.tar.gz
RUN cd Vulkan-Headers*
RUN mkdir build && cd build
RUN cmake ../ -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=$CLANG_BINARY -DCMAKE_CXX_COMPILER=$CLANGXX_BINARY \
	-DCMAKE_INSTALL_PREFIX=/usr
RUN ninja
RUN ninja install
RUN cd ../..
RUN rm ./*.tar.gz
RUN rm -r ./Vulkan-Headers*

RUN curl -sSfL -o Vulkan-Loader.tar.gz https://github.com/KhronosGroup/Vulkan-Loader/archive/sdk-${VULKAN_LOADER_VER}.tar.gz
RUN tar -xf Vulkan-Loader.tar.gz
RUN cd Vulkan-Loader*
RUN mkdir build && cd build
RUN cmake ../ -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=$CLANG_BINARY -DCMAKE_CXX_COMPILER=$CLANGXX_BINARY \
	-DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=/usr/lib -DCMAKE_INSTALL_SYSCONFDIR=/etc -DCMAKE_INSTALL_DATADIR=/share
RUN ninja
RUN ninja install
RUN cd ../..
RUN rm ./*.tar.gz
RUN rm -r ./Vulkan-Loader*

# We need to build a custom version of SDL2 as well
ENV SDL2VER=2.0.22
RUN curl -sSfL -o SDL2.tar.gz https://www.libsdl.org/release/SDL2-${SDL2VER}.tar.gz
RUN tar -xf SDL2.tar.gz
RUN cd SDL2*
RUN mkdir build && cd build
RUN cmake ../ -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=$CLANG_BINARY -DCMAKE_CXX_COMPILER=$CLANGXX_BINARY \
    -DCMAKE_INSTALL_PREFIX=/usr -DSDL_DLOPEN=ON -DARTS=OFF -DESD=OFF -DNAS=OFF -DALSA=ON -DPULSEAUDIO_SHARED=ON -DRPATH=OFF -DCLOCK_GETTIME=ON
RUN ninja
RUN ninja install
RUN cd ../..
RUN rm ./*.tar.gz
RUN rm -r ./SDL2*

ENV SHELLCHECKVER=0.8.0
RUN curl -sSfL -o shellcheck.tar.gz https://github.com/koalaman/shellcheck/releases/download/v${SHELLCHECKVER}/shellcheck-v${SHELLCHECKVER}.linux.x86_64.tar.xz
RUN tar -xf shellcheck.tar.gz
RUN cd shellcheck*
RUN cp ./shellcheck /usr/bin/
RUN cd ..
RUN rm -r ./shellcheck*
RUN shellcheck --version

# Need to patch Qt for compatibility with GCC 11
# See https://codereview.qt-project.org/c/qt/qtbase/+/339417 and https://github.com/RPCS3/rpcs3/issues/10291
RUN \
	sed -i 's/ThreadEngineStarter<void>(ThreadEngine<void> \*_threadEngine)/ThreadEngineStarter(ThreadEngine<void> \*_threadEngine)/' \
	/opt/qt515/include/QtConcurrent/qtconcurrentthreadengine.h
