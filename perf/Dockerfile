# Multi-stage build: First the full builder image:

# Default location where all binaries wind up:
ARG INSTALLDIR=/opt/oqssa

# liboqs build defines (https://github.com/open-quantum-safe/liboqs/wiki/Customizing-liboqs)
# for minimal build:
#ARG LIBOQS_BUILD_DEFINES="-DOQS_MINIMAL_BUILD=ON"
ARG LIBOQS_BUILD_DEFINES=""

# openssl build defines (https://github.com/open-quantum-safe/openssl#build-options)
ARG OPENSSL_BUILD_DEFINES="-DOQS_DEFAULT_GROUPS=p384_kyber768:X25519:kyber768:newhope1024cca"

# Default root CA signature algorithm; can be set to any listed at https://github.com/open-quantum-safe/openssl#authentication
ARG SIG_ALG="dilithium2"

# Define the degree of parallelism when building the image; leave the number away only if you know what you are doing
ARG MAKE_DEFINES="-j 2"

# Define main performance architecture(s)
ARG PERF_ARCH_X64="skylake-avx512"

# default architecture
ARG ARCH="amd64"

FROM multiarch/debian-debootstrap:${ARCH}-buster as intermediate
# Take in all global args
ARG INSTALLDIR
ARG LIBOQS_BUILD_DEFINES
ARG OPENSSL_BUILD_DEFINES
ARG SIG_ALG
ARG PERF_ARCH_X64
ARG MAKE_DEFINES

LABEL version="2"

ENV DEBIAN_FRONTEND noninteractive

# Get all software packages required for builing all components:
RUN dpkg --add-architecture arm64 && \
    apt-get update -qq && \
    apt-get upgrade -y && \
    apt-get dist-upgrade -y && \
    apt-get install -y gcc \
       cmake ninja-build \
       autoconf automake git libssl-dev libtool make unzip wget zlib1g-dev \
       doxygen  \
       python3 python3-nose python3-rednose python3-pytest python3-pytest-xdist docker.io \
       python3-psutil \
       maven openjdk-11-jdk \
       gcc-arm-linux-gnueabi gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu qemu-user-static \
       docker \
       libssl-dev:arm64 s3fs


# get all sources
WORKDIR /opt
RUN git clone --depth 1 --branch main https://github.com/open-quantum-safe/liboqs && \
    git clone --depth 1 --branch OQS-OpenSSL_1_1_1-stable https://github.com/open-quantum-safe/openssl ossl-src

# build liboqs shared and static, distributable (x86_64)
WORKDIR /opt/liboqs
# base platform build (x86_64)
RUN mkdir build && cd build && cmake .. ${LIBOQS_BUILD_DEFINES} -DOQS_DIST_BUILD=ON -DBUILD_SHARED_LIBS=ON -DCMAKE_INSTALL_PREFIX=/opt/ossl-src/oqs && make ${MAKE_DEFINES} && make install
RUN mkdir build-static && cd build-static && cmake .. ${LIBOQS_BUILD_DEFINES} -DOQS_DIST_BUILD=ON -DBUILD_SHARED_LIBS=OFF -DCMAKE_INSTALL_PREFIX=/opt/ossl-src/oqs && make ${MAKE_DEFINES} && make install

# Build ref-only variants and store in INSTALLDIR-ref (x86_64)
RUN mkdir build-shared-ref && cd build-shared-ref && cmake .. ${LIBOQS_BUILD_DEFINES} -DBUILD_SHARED_LIBS=ON -DOQS_OPT_TARGET="generic" -DCMAKE_INSTALL_PREFIX=${INSTALLDIR}/oqs-ref && make ${MAKE_DEFINES} && make install
RUN mkdir build-static-ref && cd build-static-ref && cmake .. ${LIBOQS_BUILD_DEFINES} -DBUILD_SHARED_LIBS=OFF -DOQS_OPT_TARGET="generic" -DCMAKE_INSTALL_PREFIX=${INSTALLDIR}/oqs-ref && make ${MAKE_DEFINES} && make install

# Build noportable fast variants and store in INSTALLDIR-noport (liboqs build default) (x86_64)
RUN mkdir build-shared-noport && cd build-shared-noport && cmake .. ${LIBOQS_BUILD_DEFINES} -DBUILD_SHARED_LIBS=ON -DOQS_OPT_TARGET=${PERF_ARCH_X64} -DCMAKE_INSTALL_PREFIX=${INSTALLDIR}/oqs-noport && make ${MAKE_DEFINES} && make install
RUN mkdir build-static-noport && cd build-static-noport && cmake .. ${LIBOQS_BUILD_DEFINES} -DBUILD_SHARED_LIBS=OFF -DOQS_OPT_TARGET=${PERF_ARCH_X64} -DCMAKE_INSTALL_PREFIX=${INSTALLDIR}/oqs-noport && make ${MAKE_DEFINES} && make install

# build OQS-OpenSSL (x86_64)
WORKDIR /opt/ossl-src
RUN LDFLAGS="-Wl,-rpath -Wl,${INSTALLDIR}/lib" ./config shared ${OPENSSL_BUILD_DEFINES} --prefix=${INSTALLDIR} && \
    make ${MAKE_DEFINES} && make install_sw && make install_ssldirs;

# set path to use 'new' openssl. Dyn libs have been properly linked in to match
ENV PATH="${INSTALLDIR}/bin:${PATH}"

# generate certificates for openssl s_server, which is what we will test curl against
ENV OPENSSL=${INSTALLDIR}/bin/openssl
ENV OPENSSL_CNF=${INSTALLDIR}/ssl/openssl.cnf

WORKDIR ${INSTALLDIR}/bin
# generate CA key and cert
RUN set -x && \
    ${OPENSSL} req -x509 -new -newkey ${SIG_ALG} -keyout CA.key -out CA.crt -nodes -subj "/CN=oqstest CA" -days 365 -config ${OPENSSL_CNF} 

WORKDIR /

## second stage: Only create minimal image without build tooling and intermediate build results generated above:
ARG ARCH
FROM debian:buster-slim as dev
# Take in all global args
ARG INSTALLDIR
ARG SIG_ALG

# Dependencies for logfile analysis and S3FS:
# Get all software packages required for builing all components:
RUN apt update && apt upgrade -y && \
    apt dist-upgrade -y && \
    apt install -y python3 fuse valgrind libssl-dev s3fs git && \
    apt autoremove && rm -rf /var/cache/apt/*

# Retain the ${INSTALLDIR} contents in the final image
COPY --from=intermediate ${INSTALLDIR} ${INSTALLDIR}
# Also retain liboqs speed-executables
COPY --from=intermediate /opt/liboqs/build-static/tests/speed_kem ${INSTALLDIR}/bin/speed_kem
COPY --from=intermediate /opt/liboqs/build-static/tests/speed_sig ${INSTALLDIR}/bin/speed_sig
COPY --from=intermediate /opt/liboqs/build-static-ref/tests/speed_kem ${INSTALLDIR}/bin/speed_kem-ref
COPY --from=intermediate /opt/liboqs/build-static-ref/tests/speed_sig ${INSTALLDIR}/bin/speed_sig-ref
COPY --from=intermediate /opt/liboqs/build-static-noport/tests/speed_kem ${INSTALLDIR}/bin/speed_kem-noport
COPY --from=intermediate /opt/liboqs/build-static-noport/tests/speed_sig ${INSTALLDIR}/bin/speed_sig-noport
# Also retain liboqs test_mem-executables
COPY --from=intermediate /opt/liboqs/build-static/tests/test_kem_mem ${INSTALLDIR}/bin/test_kem_mem
COPY --from=intermediate /opt/liboqs/build-static/tests/test_sig_mem ${INSTALLDIR}/bin/test_sig_mem
COPY --from=intermediate /opt/liboqs/build-static-ref/tests/test_kem_mem ${INSTALLDIR}/bin/test_kem_mem-ref
COPY --from=intermediate /opt/liboqs/build-static-ref/tests/test_sig_mem ${INSTALLDIR}/bin/test_sig_mem-ref
COPY --from=intermediate /opt/liboqs/build-static-noport/tests/test_kem_mem ${INSTALLDIR}/bin/test_kem_mem-noport
COPY --from=intermediate /opt/liboqs/build-static-noport/tests/test_sig_mem ${INSTALLDIR}/bin/test_sig_mem-noport

COPY scripts/* /opt/test/

# set path to use 'new' openssl. Dyn libs have been properly linked in to match
ENV PATH="${INSTALLDIR}/bin:${PATH}"

# config locations
ENV OPENSSL=${INSTALLDIR}/bin/openssl
ENV OPENSSL_CNF=${INSTALLDIR}/ssl/openssl.cnf

WORKDIR ${INSTALLDIR}

FROM dev
ARG INSTALLDIR

# Enable a normal user to create new server keys off set CA
RUN addgroup --gid 1000 oqs && adduser --system --disabled-password --gid 1000 --uid 1000 oqs && chown -R oqs.oqs /opt/test

# permit changes to liboqs lib by normal oqs user:
RUN chmod gou+rwx /opt/oqssa/lib && chmod gou+rwx /opt/oqssa/lib/*

USER oqs
WORKDIR /opt/test
CMD ["/opt/test/run-tests.sh"]
STOPSIGNAL SIGTERM
