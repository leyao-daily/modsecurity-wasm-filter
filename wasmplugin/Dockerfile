FROM ubuntu:bionic
# basics
SHELL ["/bin/bash", "-c"] 
ARG http_proxy 
ARG https_proxy 
RUN apt-get update && apt-get install -y --no-install-recommends apt-utils ca-certificates \ 
    && apt-get autoremove -y \ 
    && apt-get clean \ 
    && apt-get install -y --no-install-recommends \
    software-properties-common apt-transport-https git wget curl pkg-config \
    autoconf autotools-dev automake libtool cmake python zlib1g-dev libpcre3 \
    libpcre3-dev gcc-7 g++-7 cpp-7 gcc python3 python3-distutils \
    && rm -rf /var/lib/apt/lists/* \
    && cd $HOME

ENV CC=gcc-7 
ENV CXX=g++-7
ENV CPP=cpp-7

WORKDIR /root
RUN git clone -b v3.9.1 https://github.com/protocolbuffers/protobuf \
    && cd protobuf \
    && git submodule update --init --recursive \
    && ./autogen.sh \
    && ./configure   \
    && make \
    && make check \
    && make install \
    && cd .. \
    && rm -rf protobuf

# Commit from SpiderLabs/ModSecurity repository, aiming at v3/dev/wasm-experimental branch
ARG MODSEC_SHA=a3454df9b09a8de16d41be10cdea4fc46ef08e91

WORKDIR /root/
RUN git clone https://github.com/emscripten-core/emsdk.git -b 3.1.15 \ 
    && git clone https://github.com/leyao-daily/pcre2.git \
    && git clone https://github.com/SpiderLabs/ModSecurity.git \
    && git -C ModSecurity checkout $MODSEC_SHA \
    && git clone https://github.com/abseil/abseil-cpp -b 20211102.0 \
    && git clone https://github.com/proxy-wasm/proxy-wasm-cpp-sdk \
    && git clone https://github.com/istio/proxy.git -b 1.13.3

WORKDIR /root/emsdk
RUN ./emsdk install 3.1.7 \ 
    && ./emsdk activate 3.1.7 \
    && echo "source /root/emsdk/emsdk_env.sh" >> ~/.bashrc \  
    && cd ..

RUN wget https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-12/wasi-sdk-12.0-linux.tar.gz \
    && tar xvf wasi-sdk-12.0-linux.tar.gz -C /opt/ \
    && mv /opt/wasi-sdk-12.0/ /opt/wasi-sdk \
    && rm wasi-sdk-12.0-linux.tar.gz 

ENV WASI_SDK_PATH="/opt/wasi-sdk" 

WORKDIR /root/pcre2
RUN autoreconf -ivf \
    && source "/root/emsdk/emsdk_env.sh" \
    && ./build_for_wasm.sh wasm32-emscripten \
    && mkdir -p /usr/local/pcre2 \
    && cp targets/wasm32-emscripten/*.a /usr/local/pcre2 \
    && cd .. 

WORKDIR /root/ModSecurity
RUN ./build.sh \
    && git submodule init && git submodule update \
    && autoreconf -ivf \
    && cd ~/emsdk \
    && source ./emsdk_env.sh \
    && cd ~/ModSecurity \
    && emconfigure ./configure --without-yajl --without-geoip --without-libxml --without-curl \
    --without-lua --disable-shared --disable-examples --disable-libtool-lock \
    --disable-debug-logs  --disable-mutex-on-pm --without-lmdb --without-maxmind \
    --without-ssdeep --with-pcre2 \
    && emmake make -j 4 \
    && emmake make install \
    && cd ..

WORKDIR /root/proxy-wasm-cpp-sdk
RUN mkdir /build /sdk \
    && cp *.cc *.h *.js *.proto Makefile* *.a /sdk/ 

WORKDIR /root 
RUN wget -O /usr/local/bin/bazel https://github.com/bazelbuild/bazelisk/releases/latest/download/bazelisk-linux-amd64 \
    && chmod +x /usr/local/bin/bazel 

RUN mv proxy istio-proxy \
    && cd istio-proxy \ 
    && bazel build //extensions/common/wasm:json_util \ 
    && cp ~/istio-proxy/bazel-bin/extensions/common/wasm/nlohmann_json.hpp ~/istio-proxy/extensions/common/wasm/ \
    && cd ..

WORKDIR /root
ADD Makefile envoy-wasm-modsecurity-dynamic.cc utils.cc utils.h /build/ 

RUN cd ~/emsdk \
    && source ./emsdk_env.sh \
    && cd /build \
    && make 

FROM scratch
COPY --from=0 /build/envoy-wasm-modsecurity-dynamic.wasm /plugin.wasm
