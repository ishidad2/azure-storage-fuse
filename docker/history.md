```
docker build -f jenkins/catapult/compilers/ubuntu-gcc/Dockerfile -t ishidad2/symbol-server-compiler:ubuntu-gcc-12 .
```

# 1つ目のイメージ
```
python3 ./jenkins/catapult/baseImageDockerfileGenerator.py \
    --compiler-configuration jenkins/catapult/configurations/gcc-latest.yaml \
    --operating-system ubuntu \
    --versions ./jenkins/catapult/versions.properties \
    --layer os
```

Dockerfile-preimage1
```
FROM ishidad2/symbol-server-compiler:ubuntu-gcc-12
ARG DEBIAN_FRONTEND=noninteractive
LABEL maintainer="Catapult Development Team"
RUN apt-get -y update \
    && apt-get install -y autoconf ca-certificates ccache curl gdb git libatomic-ops-dev libgflags-dev libsnappy-dev libtool make ninja-build pkg-config python3 python3-ply xz-utils \
    && rm -rf /var/lib/apt/lists/* \
    && curl -o cmake-3.23.2-Linux-x86_64.sh -SL "https://github.com/Kitware/CMake/releases/download/v3.23.2/cmake-3.23.2-Linux-x86_64.sh" \
    && chmod +x cmake-3.23.2-Linux-x86_64.sh \
    && ./cmake-3.23.2-Linux-x86_64.sh --skip-license --prefix=/usr \
    && rm -rf cmake-3.23.2-Linux-x86_64.sh
```
ビルド
```
docker build -f Dockerfile-preimage1 -t ishidad2/symbol-server-build-base:ubuntu-gcc-12-preimage1 .
```

# 2つ目のイメージ
```
python3 ./jenkins/catapult/baseImageDockerfileGenerator.py \
    --compiler-configuration jenkins/catapult/configurations/gcc-latest.yaml \
    --operating-system ubuntu \
    --versions ./jenkins/catapult/versions.properties \
    --layer boost
```

Dockerfile-preimage2
```
FROM ishidad2/symbol-server-build-base:ubuntu-gcc-12-preimage1
RUN curl -o /usr/local/bin/gosu -SL "https://github.com/tianon/gosu/releases/download/1.14/gosu-$(dpkg --print-architecture)" \
    && chmod +x /usr/local/bin/gosu
RUN curl -o boost_1_80_0.tar.gz -SL https://boostorg.jfrog.io/artifactory/main/release/1.80.0/source/boost_1_80_0.tar.gz \
    && tar -xzf boost_1_80_0.tar.gz \
    && mkdir /mybuild \
    && cd boost_1_80_0 \
    && ./bootstrap.sh  --prefix=/mybuild \
    && ./b2 address-model=64 runtime-link=shared threading=multi link=shared --layout=system variant=release cxxflags='-march=skylake' --prefix=/mybuild --without-context --without-contract --without-coroutine --without-fiber --without-graph --without-graph_parallel --without-headers --without-iostreams --without-json --without-mpi --without-nowide --without-python --without-serialization --without-stacktrace --without-test --without-timer --without-type_erasure --without-wave -j 8 stage release \
    && ./b2 address-model=64 runtime-link=shared threading=multi link=shared --layout=system variant=release cxxflags='-march=skylake' --without-context --without-contract --without-coroutine --without-fiber --without-graph --without-graph_parallel --without-headers --without-iostreams --without-json --without-mpi --without-nowide --without-python --without-serialization --without-stacktrace --without-test --without-timer --without-type_erasure --without-wave install
```

ビルド
```
docker build -f Dockerfile-preimage2 -t ishidad2/symbol-server-build-base:ubuntu-gcc-12-preimage2 .
```

# 3つ目のイメージ
```
python3 ./jenkins/catapult/baseImageDockerfileGenerator.py \
    --compiler-configuration jenkins/catapult/configurations/gcc-latest.yaml \
    --operating-system ubuntu \
    --versions ./jenkins/catapult/versions.properties \
    --layer deps
```

Dockerfile-preimage3
``
FROM ishidad2/symbol-server-build-base:ubuntu-gcc-12-preimage2
RUN git clone https://github.com/openssl/openssl.git -b openssl-3.0.7 \
    && cd openssl \
    && CFLAGS='-march=skylake' perl ./Configure linux-x86_64  --prefix=/usr/catapult/deps --openssldir=/usr/catapult/deps --libdir=/usr/catapult/deps \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf openssl
RUN git clone https://github.com/mongodb/mongo-c-driver.git -b 1.22.0 \
    && cd mongo-c-driver \
    && mkdir _build \
    && cd _build \
    && cmake -DOPENSSL_ROOT_DIR=/usr/catapult/deps -DENABLE_AUTOMATIC_INIT_AND_CLEANUP=OFF -DENABLE_MONGODB_AWS_AUTH=OFF -DENABLE_TESTS=OFF -DENABLE_EXAMPLES=OFF -DENABLE_SASL=OFF -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_CXX_FLAGS='-march=skylake' .. \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf mongo-c-driver \
    && echo "force rebuild revision 1"
RUN git clone https://github.com/mongodb/mongo-cxx-driver.git -b r3.6.7 \
    && cd mongo-cxx-driver \
    && mkdir _build \
    && cd _build \
    && cmake -DOPENSSL_ROOT_DIR=/usr/catapult/deps -DCMAKE_CXX_STANDARD=17 -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_CXX_FLAGS='-march=skylake' .. \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf mongo-cxx-driver \
    && echo "force rebuild revision 1"
RUN git clone https://github.com/zeromq/libzmq.git -b v4.3.4 \
    && cd libzmq \
    && mkdir _build \
    && cd _build \
    && cmake -DWITH_TLS=OFF -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_CXX_FLAGS='-march=skylake' .. \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf libzmq \
    && echo "force rebuild revision 1"
RUN git clone https://github.com/zeromq/cppzmq.git -b v4.8.1 \
    && cd cppzmq \
    && mkdir _build \
    && cd _build \
    && cmake -DCPPZMQ_BUILD_TESTS=OFF -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_CXX_FLAGS='-march=skylake' .. \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf cppzmq \
    && echo "force rebuild revision 1"
RUN git clone https://github.com/facebook/rocksdb.git -b v7.4.3 \
    && cd rocksdb \
    && mkdir _build \
    && cd _build \
    && cmake -DPORTABLE=1 -DWITH_TESTS=OFF -DWITH_TOOLS=OFF -DWITH_BENCHMARK_TOOLS=OFF -DWITH_CORE_TOOLS=OFF -DWITH_GFLAGS=OFF -DUSE_RTTI=1 -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_CXX_FLAGS='-Wno-error=maybe-uninitialized -march=skylake' .. \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf rocksdb \
    && echo "force rebuild revision 1"
``

ビルド
```
docker build -f Dockerfile-preimage3 -t ishidad2/symbol-server-build-base:ubuntu-gcc-12-preimage3 . --memory-swap -1

```

#最終レイヤー

```
python3 ./jenkins/catapult/baseImageDockerfileGenerator.py \
    --compiler-configuration jenkins/catapult/configurations/gcc-latest.yaml \
    --operating-system ubuntu \
    --versions ./jenkins/catapult/versions.properties \
    --layer test
```

```
FROM ishidad2/symbol-server-build-base:ubuntu-gcc-12-preimage3
RUN git clone https://github.com/google/googletest.git -b release-1.12.1 \
    && cd googletest \
    && mkdir _build \
    && cd _build \
    && cmake -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DBUILD_GMOCK=OFF -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_CXX_FLAGS='-march=skylake' .. \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf googletest \
    && echo "force rebuild revision 1"
RUN git clone https://github.com/google/benchmark.git -b v1.7.0 \
    && cd benchmark \
    && mkdir _build \
    && cd _build \
    && cmake -DBENCHMARK_ENABLE_GTEST_TESTS=OFF -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_CXX_FLAGS='-march=skylake' .. \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf benchmark \
    && echo "force rebuild revision 1"
RUN apt-get -y update \
    && apt-get remove -y --purge pylint \
    && apt-get install -y python3-pip lcov libssl-dev \
    && python3 -m pip install -U pycodestyle pylint pyyaml
RUN git clone https://github.com/openssl/openssl.git -b openssl-3.0.7 \
    && cd openssl \
    && CFLAGS='-march=skylake' perl ./Configure linux-x86_64  --prefix=/usr/catapult/deps --openssldir=/usr/catapult/deps --libdir=/usr/catapult/deps \
    && make -j 8 \
    && make install \
    && cd .. \
    && rm -rf openssl
RUN echo "docker image build $BUILD_NUMBER"
CMD ["/bin/bash"]
```

# 最終イメージ
```
docker build -f Dockerfile-preimage4 -t ishidad2/symbol-server-build-base:ubuntu-gcc-12 .
```

# TODO この後にblobfuse2の処理レイヤーを追加する
```
# Create container based on Ubuntu-22.04 Jammy Jellyfish image
FROM ishidad2/symbol-server-build-base:ubuntu-gcc-12

#Define arg to determine whether to build for fuse2 or fuse3
ARG FUSE2

# Create directory to hold samples
RUN mkdir -p /usr/share/blobfuse2

# Copy blobfuse2 binary to executable path
COPY ./blobfuse2 /usr/local/bin/
COPY ./config.yaml /usr/share/blobfuse2/

# Install fuse library
RUN \
	apt update && \
	apt-get install -y ca-certificates vim rsyslog

RUN if [ "$FUSE2" = "TRUE" ] ; then apt-get install -y fuse ; else apt-get install -y fuse3 ; fi 

RUN echo "user_allow_other" >> /etc/fuse.conf

# Create syslog filter files
COPY ./11-blobfuse2.conf /etc/rsyslog.d
COPY ./blobfuse2-logrotate /etc/logrotate.d/blobfuse2


# Create mount directory structure
RUN \
	mkdir -p /mnt/blobfuse_mnt && \
	mkdir -p /tmp/blobfuse_temp && \
	chmod 777 /mnt/blobfuse_mnt && \
	chmod 777 /tmp/blobfuse_temp


# Create the mount script and set it to entry point once container start
RUN \
	echo "/sbin/rsyslogd" > /usr/share/blobfuse2/blobfuse2-mount.sh && \
	echo "blobfuse2 mount /symbol-workdir --config-file=/usr/share/blobfuse2/config.yaml  --ignore-open-flags --foreground=true" >> /usr/share/blobfuse2/blobfuse2-mount.sh && \
	echo "blobfuse2 unmount all" > /usr/share/blobfuse2/blobfuse2-umount.sh && \
	chmod 777 /usr/share/blobfuse2/blobfuse2-mount.sh && \
	chmod 777 /usr/share/blobfuse2/blobfuse2-umount.sh && \
	ln -s  /usr/share/blobfuse2/blobfuse2-mount.sh /usr/local/bin/fuse && \
	ln -s  /usr/share/blobfuse2/blobfuse2-umount.sh /usr/local/bin/unfuse 


ENTRYPOINT ["bash", "fuse"]
```

# blobfuse2を追加したイメージ作成
blobfuse2のdockerフォルダ内で実行する
```
./buildcontainer-for-symbol.sh
```