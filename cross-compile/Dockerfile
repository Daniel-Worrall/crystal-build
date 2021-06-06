FROM crystallang/crystal:1.0.0-alpine-build as builder

RUN git clone --branch 1.0.0 --depth 1 https://github.com/crystal-lang/crystal
RUN cd crystal \
 \
 && make crystal FLAGS="--cross-compile --target aarch64-alpine-linux-musl" \
                 stats=true static=true ${release:+release=true} \
                 CRYSTAL_CONFIG_TARGET=aarch64-alpine-linux-musl CRYSTAL_CONFIG_LIBRARY_PATH=

RUN git clone --branch v0.14.1 --depth 1 https://github.com/crystal-lang/shards \
 && cd shards \
 && make FLAGS="--stats --cross-compile --target aarch64-alpine-linux-musl --static ${release:+--release}"



FROM --platform=linux/arm64/v8 arm64v8/alpine:3.13 as alpine-builder

RUN \
  apk add --update --no-cache --force-overwrite \
    # core dependencies
    gc-dev gcc gmp-dev libatomic_ops libevent-static musl-dev pcre-dev \
    # stdlib dependencies
    libxml2-dev openssl-dev openssl-libs-static tzdata yaml-dev yaml-static zlib-static \
    # dev tools
    make git \
    # compiler
    llvm10-dev llvm10-static g++

WORKDIR /src
COPY --from=builder /crystal /crystal
# Build deps
RUN cp /crystal/.build/crystal.o . \
 && cd /crystal \
 && make clean \
 && make deps

# Link Crystal
RUN cc crystal.o -o crystal -rdynamic -static /crystal/src/llvm/ext/llvm_ext.o  `"/usr/bin/llvm-config" --libs --system-libs --ldflags --link-static 2> /dev/null` -lstdc++ -lpcre -lm -lgc -lpthread /crystal/src/ext/libcrystal.a -levent -lrt

# Link Shards
COPY --from=builder /shards/bin/shards.o .
RUN cc shards.o -o shards -rdynamic -static -lyaml -lpcre -lm -lgc -lpthread /crystal/src/ext/libcrystal.a -levent -lrt



FROM --platform=linux/arm64/v8 arm64v8/alpine:3.13 as runtime
RUN \
  apk add --update --no-cache --force-overwrite \
    # core dependencies
    gc-dev gcc gmp-dev libatomic_ops libevent-static musl-dev pcre-dev \
    # stdlib dependencies
    libxml2-dev openssl-dev openssl-libs-static tzdata yaml-static zlib-static \
    # dev tools
    make git

# Crystal Wrapper
COPY --from=builder /usr/bin/crystal /usr/bin/crystal
# Crystal & Shards binaries
COPY --from=alpine-builder /src/crystal /src/shards /usr/lib/crystal/bin/
# stdlib
COPY --from=alpine-builder /crystal/src /usr/share/crystal/src
# samples
COPY --from=alpine-builder /crystal/samples /usr/share/doc/crystal/examples
# manpages
COPY --from=alpine-builder /crystal/man/crystal.1 /usr/share/man/man1/crystal.1
COPY --from=builder /shards/man/shards.1 /usr/share/man/man1/shards.1
COPY --from=builder /shards/man/shard.yml.5 /usr/share/man/man5/shard.yml.5
# license
COPY --from=alpine-builder /crystal/LICENSE /usr/share/licenses/crystal/LICENSE
RUN \
 # shards link
 ln -s ../lib/crystal/bin/shards /usr/bin/shards

CMD ["/bin/sh"]



FROM runtime as build

RUN \
  apk add --update --no-cache --force-overwrite \
    llvm10-dev llvm10-static g++

CMD ["/bin/sh"]