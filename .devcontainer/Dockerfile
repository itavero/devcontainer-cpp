FROM mcr.microsoft.com/devcontainers/base:jammy as prep
ENV CMAKE_GENERATOR=Ninja
ENV CMAKE_EXPORT_COMPILE_COMMANDS=ON
# Avoid stuck build due to user prompt
ARG DEBIAN_FRONTEND=noninteractive
# Get stuff we need to download/build
RUN apt-get update && \
   apt-get install --no-install-recommends -y git build-essential unzip gcc g++ cmake ninja-build zlib1g-dev ca-certificates curl && \
   apt-get clean && rm -rf /var/lib/apt/lists/*
# Download latest ninja build release and unzip it to /tmp
RUN curl -s https://api.github.com/repos/ninja-build/ninja/releases/latest | \
   grep "browser_download_url.*ninja-linux.zip" | \
   cut -d : -f 2,3 | \
   tr -d \" | \
   xargs curl -sSL -o /tmp/ninja.zip && \
   unzip /tmp/ninja.zip -d /tmp/ninja
# Create Debian package for Google Test
RUN git clone --branch v1.14.0 --depth 1 https://github.com/google/googletest.git /tmp/googletest && \
   echo "include(CPack)" >> /tmp/googletest/CMakeLists.txt && \
   cmake -S /tmp/googletest/ -B /tmp/build/googletest/ -DBUILD_GMOCK=ON -DCMAKE_BUILD_TYPE=MinSizeRel -DCPACK_GENERATOR=DEB -DCPACK_DEBIAN_PACKAGE_MAINTAINER="itavero" \
   -DCPACK_COMPONENTS_GROUPING=ALL_COMPONENTS_IN_ONE -DCPACK_PACKAGE_NAME=googletest && \
   cmake --build /tmp/build/googletest/ --target package

# Actual image
FROM mcr.microsoft.com/devcontainers/base:jammy
ENV CMAKE_GENERATOR=Ninja
ENV CMAKE_EXPORT_COMPILE_COMMANDS=ON
ENV CPM_SOURCE_CACHE=/tmp/cache/cpm
ENV CPM_USE_NAMED_CACHE_DIRECTORIES=ON
# LLVM version
ENV LLVM_VERSION=17

COPY --from=prep /tmp/ninja/ninja /usr/local/bin/
COPY --from=prep /tmp/build/googletest/googletest*.deb /tmp/
COPY assets/requirements.txt /tmp/

RUN apt-get update && \
   apt-get install --no-install-recommends -y \
   git git-lfs gcc g++ unzip ca-certificates lsb-release wget gpg curl gnupg \
   python3 python3-dev python3-venv python3-pip python-is-python3 doxygen graphviz && \
   (wget -qO- https://apt.kitware.com/keys/kitware-archive-latest.asc | tee /etc/apt/trusted.gpg.d/apt.kitware.com.asc) && \
   (wget -qO- https://apt.llvm.org/llvm-snapshot.gpg.key | tee /etc/apt/trusted.gpg.d/apt.llvm.org.asc) && \
   echo "deb https://apt.kitware.com/ubuntu/ $(lsb_release -cs) main" > /etc/apt/sources.list.d/kitware.list && \
   echo "deb https://apt.llvm.org/$(lsb_release -cs)/ llvm-toolchain-$(lsb_release -cs)-${LLVM_VERSION} main" > /etc/apt/sources.list.d/llvm.list && \
   apt-get update && \
   apt-get install --no-install-recommends -y cmake clang-${LLVM_VERSION} clang-tidy-${LLVM_VERSION} clang-format-${LLVM_VERSION} && \
   dpkg -i /tmp/googletest*.deb && \
   apt-get clean autoclean && \
   apt-get autoremove --yes && \
   pip3 install --no-cache-dir -r /tmp/requirements.txt && \
   rm -rf /tmp/* && \
   rm -rf /var/lib/apt/lists/* && \
   rm -rf /var/cache/apt/archives && \
   rm -rf /usr/share/doc/ && \
   rm -rf /usr/share/man/ && \
   rm -rf /usr/share/locale/ \
   rm -rf /usr/share/cmake-*/Help && \
   # Make sure the CPM cache directory is always created
   mkdir -p $CPM_SOURCE_CACHE && \
   # Create symlink for clang-format and clang-tidy that does not include the version
   ln -s /usr/bin/clang-format-${LLVM_VERSION} /usr/bin/clang-format && \
   ln -s /usr/bin/clang-tidy-${LLVM_VERSION} /usr/bin/clang-tidy && \
   # Check if all tools can be executed
   gcc --version && \
   cmake --version && \
   ninja --version && \
   doxygen --version && \
   clang-format --version && \
   clang-tidy --version && \
   cpplint --version && \
   git --version && \
   python --version && \
   gcovr --version && \
   rm -rf /var/log/*
