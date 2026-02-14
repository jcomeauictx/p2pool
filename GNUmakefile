# GNUmakefile for p2pool
# Builds p2pool without CMake, using system libraries where available
# and building from submodules as fallback.
#
# Prerequisites (Debian/Ubuntu):
#   apt install build-essential libuv1-dev libcurl4-openssl-dev \
#               libzmq3-dev librandomx-dev
#
# Submodules are still needed for header-only libs (rapidjson,
# robin-hood-hashing, cppzmq) and RandomX internal headers (cpu.hpp).
#
# Usage:
#   make              - build p2pool (release)
#   make debug        - build p2pool (debug)
#   make clean        - remove build artifacts
#   make distclean    - remove build artifacts and submodule builds
#   make submodules   - initialize git submodules

SHELL := /bin/bash

# Project
TARGET := p2pool
BUILDDIR := build

# Compilers
CXX ?= g++
CC ?= gcc

# Architecture detection
ARCH := $(shell uname -m)
AMD64 := $(filter x86_64 x86-64 amd64,$(ARCH))

# Git commit for version embedding
GIT_COMMIT := $(shell git rev-parse --short=7 HEAD 2>/dev/null || echo "unknown")
GIT_DIRTY := $(shell git status --porcelain 2>/dev/null)
ifneq ($(GIT_DIRTY),)
  GIT_COMMIT := $(GIT_COMMIT) (dirty)
endif

# Feature flags (set to 0 to disable)
WITH_RANDOMX ?= 1
WITH_UPNP ?= 0
WITH_GRPC ?= 0
WITH_TLS ?= 0
WITH_INDEXED_HASHES ?= 0
WITH_MERGE_MINING_DONATION ?= 1
WITH_LTO ?= 1

# Standards
CXXSTD := -std=c++17
CSTD := -std=c11

# Base flags
COMMON_FLAGS := -pthread
WARNFLAGS := -Wall -Wextra -Wcast-qual -Wlogical-op -Wundef -Wformat=2 \
             -Wpointer-arith -Wstrict-overflow=2

# Release vs debug
ifdef DEBUG
  OPTFLAGS := -Og -g3 -ftrapv
  DEFINES += -DDEBUG_BUILD
else
  OPTFLAGS := -O3 -ffast-math
  ifeq ($(WITH_LTO),1)
    OPTFLAGS += -flto=auto -fuse-linker-plugin
  endif
endif

CXXFLAGS := $(CXXSTD) $(COMMON_FLAGS) $(WARNFLAGS) $(OPTFLAGS) \
            -Wno-error=inline -Wno-error=unused-function -Wno-error=strict-overflow
CFLAGS := $(CSTD) $(COMMON_FLAGS) $(WARNFLAGS) $(OPTFLAGS)
LDFLAGS := -static-libgcc -static-libstdc++ -pthread

# Include paths
INCLUDES := -Isrc \
            -Iexternal/src \
            -Iexternal/src/crypto \
            -Iexternal/src/cryptonote

# Preprocessor definitions
DEFINES += -DZMQ_STATIC \
           '-DRAPIDJSON_PARSE_DEFAULT_FLAGS=kParseTrailingCommasFlag' \
           '-DGIT_COMMIT="$(GIT_COMMIT)"'

# Feature-detection defines (Linux/glibc assumed)
DEFINES += -DHAVE_BUILTIN_CLZLL -DHAVE_SCHED -DHAVE_PTHREAD_SETNAME_NP -DHAVE_GLIBC

# Check for asm/hwcap.h
ifneq ($(wildcard /usr/include/asm/hwcap.h),)
  DEFINES += -DHAVE_HWCAP
endif

# Check for resolv library (for DNS TXT lookups)
HAVE_RESOLV := $(shell echo 'int main(){return 0;}' | $(CC) -x c - -lresolv -o /dev/null 2>/dev/null && echo 1)
ifeq ($(HAVE_RESOLV),1)
  # Full res_query check is complex; assume available if resolv links
  DEFINES += -DHAVE_RES_QUERY
  LIBS += -lresolv
endif

# ---------- Source files ----------

C_SOURCES := \
	external/src/crypto/sha256.c \
	external/src/cryptonote/crypto-ops-data.c \
	external/src/cryptonote/crypto-ops.c

CXX_SOURCES := \
	external/src/cryptonote/fcmp_pp_crypto.cpp \
	external/src/hardforks/hardforks.cpp \
	src/block_cache.cpp \
	src/block_template.cpp \
	src/console_commands.cpp \
	src/crypto.cpp \
	src/json_rpc_request.cpp \
	src/keccak.cpp \
	src/log.cpp \
	src/main.cpp \
	src/memory_leak_debug.cpp \
	src/mempool.cpp \
	src/merge_mining_client.cpp \
	src/merge_mining_client_json_rpc.cpp \
	src/merkle.cpp \
	src/p2p_server.cpp \
	src/p2pool.cpp \
	src/p2pool_api.cpp \
	src/params.cpp \
	src/pool_block.cpp \
	src/pow_hash.cpp \
	src/side_chain.cpp \
	src/stratum_server.cpp \
	src/tcp_server.cpp \
	src/util.cpp \
	src/wallet.cpp \
	src/zmq_reader.cpp

# AMD64-only: keccak with BMI instructions
ifneq ($(AMD64),)
  CXX_SOURCES += src/keccak_bmi.cpp
endif

# ---------- Submodule header-only libraries ----------

# rapidjson (header-only)
RAPIDJSON_INCLUDE := external/src/rapidjson/include
INCLUDES += -I$(RAPIDJSON_INCLUDE)

# robin-hood-hashing (header-only)
ROBIN_HOOD_INCLUDE := external/src/robin-hood-hashing/src/include
INCLUDES += -I$(ROBIN_HOOD_INCLUDE)

# cppzmq (header-only)
CPPZMQ_INCLUDE := external/src/cppzmq
INCLUDES += -I$(CPPZMQ_INCLUDE)

# ---------- System vs submodule libraries ----------

# libuv - prefer system
UV_CFLAGS := $(shell pkg-config --cflags libuv 2>/dev/null)
UV_LIBS := $(shell pkg-config --libs libuv 2>/dev/null)
ifeq ($(UV_LIBS),)
  # Try default paths
  UV_LIBS := -luv
endif
INCLUDES += $(UV_CFLAGS)
LIBS += $(UV_LIBS)

# libcurl - prefer system
CURL_CFLAGS := $(shell pkg-config --cflags libcurl 2>/dev/null)
CURL_LIBS := $(shell pkg-config --libs libcurl 2>/dev/null)
ifeq ($(CURL_LIBS),)
  CURL_LIBS := -lcurl
endif
INCLUDES += $(CURL_CFLAGS)
LIBS += $(CURL_LIBS)

# libzmq - prefer system, fall back to submodule
ZMQ_SYSTEM_CFLAGS := $(shell pkg-config --cflags libzmq 2>/dev/null)
ZMQ_SYSTEM_LIBS := $(shell pkg-config --libs libzmq 2>/dev/null)
ifneq ($(ZMQ_SYSTEM_LIBS),)
  # System libzmq found
  INCLUDES += $(ZMQ_SYSTEM_CFLAGS)
  LIBS += $(ZMQ_SYSTEM_LIBS)
  ZMQ_DEP :=
else
  # Build from submodule
  ZMQ_BUILDDIR := external/src/libzmq/build
  ZMQ_LIB := $(ZMQ_BUILDDIR)/lib/libzmq.a
  INCLUDES += -Iexternal/src/libzmq/include
  LIBS += $(ZMQ_LIB) -lsodium 2>/dev/null || true
  ZMQ_DEP := $(ZMQ_LIB)
endif

# ---------- Optional features ----------

# RandomX
ifeq ($(WITH_RANDOMX),1)
  DEFINES += -DWITH_RANDOMX
  INCLUDES += -Iexternal/src/RandomX/src
  CXX_SOURCES += src/miner.cpp
  RANDOMX_BUILDDIR := external/src/RandomX/build
  RANDOMX_LIB := $(RANDOMX_BUILDDIR)/librandomx.a
  LIBS += $(RANDOMX_LIB)
  RANDOMX_DEP := $(RANDOMX_LIB)
else
  CXX_SOURCES += external/src/RandomX/src/cpu.cpp
  RANDOMX_DEP :=
endif

# UPnP
ifeq ($(WITH_UPNP),1)
  DEFINES += -DWITH_UPNP
  INCLUDES += -Iexternal/src/miniupnp/miniupnpc/include
  UPNP_BUILDDIR := external/src/miniupnp/miniupnpc/build
  UPNP_LIB := $(UPNP_BUILDDIR)/libminiupnpc.a
  LIBS += $(UPNP_LIB)
  UPNP_DEP := $(UPNP_LIB)
else
  UPNP_DEP :=
endif

# gRPC (Tari merge mining) - complex, disabled by default
ifeq ($(WITH_GRPC),1)
  DEFINES += -DWITH_GRPC -DWITH_TLS
  CXX_SOURCES += src/merge_mining_client_tari.cpp src/tls.cpp
  # gRPC requires its own elaborate build; use cmake for that
  $(error WITH_GRPC=1 requires cmake. Use: cmake -DWITH_GRPC=ON . && make)
endif

# TLS (without gRPC, needs BoringSSL)
ifeq ($(WITH_TLS),1)
  ifeq ($(WITH_GRPC),0)
    DEFINES += -DWITH_TLS
    CXX_SOURCES += src/tls.cpp
    LIBS += -lssl -lcrypto
  endif
endif

# Indexed hashes
ifeq ($(WITH_INDEXED_HASHES),1)
  DEFINES += -DWITH_INDEXED_HASHES
  CXX_SOURCES += src/indexed_hash.cpp
endif

# Merge mining donation
ifeq ($(WITH_MERGE_MINING_DONATION),1)
  DEFINES += -DWITH_MERGE_MINING_DONATION
endif

# Platform libs
LIBS += -lpthread

# ---------- Object files ----------

C_OBJS := $(patsubst %.c,$(BUILDDIR)/%.o,$(C_SOURCES))
CXX_OBJS := $(patsubst %.cpp,$(BUILDDIR)/%.o,$(CXX_SOURCES))
ALL_OBJS := $(C_OBJS) $(CXX_OBJS)
DEPS := $(ALL_OBJS:.o=.d)

# Submodule sentinel
SUBMODULE_STAMP := .submodules_initialized

# ---------- Required submodule directories ----------
# These are header-only or source-only deps that must exist
REQUIRED_SUBMOD_DIRS := $(RAPIDJSON_INCLUDE) $(ROBIN_HOOD_INCLUDE) $(CPPZMQ_INCLUDE)
ifeq ($(WITH_RANDOMX),1)
  REQUIRED_SUBMOD_DIRS += external/src/RandomX/CMakeLists.txt
endif

# ---------- Targets ----------

.PHONY: all debug clean distclean submodules

all: $(TARGET)

debug:
	$(MAKE) DEBUG=1

$(TARGET): $(SUBMODULE_STAMP) $(ZMQ_DEP) $(RANDOMX_DEP) $(UPNP_DEP) $(ALL_OBJS)
	$(CXX) $(LDFLAGS) $(OPTFLAGS) -o $@ $(ALL_OBJS) $(LIBS)
ifndef DEBUG
	-strip $@
endif
	@echo "Build complete: $(TARGET)"

# Submodule initialization
$(SUBMODULE_STAMP):
	git submodule update --init --recursive
	touch $@

submodules: $(SUBMODULE_STAMP)

# ---------- Compile rules ----------

$(BUILDDIR)/%.o: %.cpp | $(SUBMODULE_STAMP)
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(DEFINES) $(INCLUDES) -MMD -MP -c -o $@ $<

$(BUILDDIR)/%.o: %.c | $(SUBMODULE_STAMP)
	@mkdir -p $(dir $@)
	$(CC) $(CFLAGS) $(DEFINES) $(INCLUDES) -MMD -MP -c -o $@ $<

# Special rule for keccak_bmi.cpp (needs -mbmi flag)
ifneq ($(AMD64),)
$(BUILDDIR)/src/keccak_bmi.o: src/keccak_bmi.cpp | $(SUBMODULE_STAMP)
	@mkdir -p $(dir $@)
	$(CXX) $(CXXFLAGS) $(DEFINES) $(INCLUDES) -mbmi -MMD -MP -c -o $@ $<
endif

# ---------- Library builds ----------

# RandomX (built with cmake)
ifeq ($(WITH_RANDOMX),1)
$(RANDOMX_LIB): $(SUBMODULE_STAMP)
	@mkdir -p $(RANDOMX_BUILDDIR)
	cd $(RANDOMX_BUILDDIR) && cmake .. -DCMAKE_BUILD_TYPE=Release
	$(MAKE) -C $(RANDOMX_BUILDDIR)
endif

# libzmq from submodule (built with cmake, only if system lib not found)
ifneq ($(ZMQ_DEP),)
$(ZMQ_LIB): $(SUBMODULE_STAMP)
	@mkdir -p $(ZMQ_BUILDDIR)
	cd $(ZMQ_BUILDDIR) && cmake .. \
		-DCMAKE_BUILD_TYPE=Release \
		-DBUILD_TESTS=OFF \
		-DBUILD_SHARED=OFF \
		-DBUILD_STATIC=ON \
		-DWITH_DOCS=OFF
	$(MAKE) -C $(ZMQ_BUILDDIR)
endif

# miniupnpc from submodule
ifeq ($(WITH_UPNP),1)
$(UPNP_LIB): $(SUBMODULE_STAMP)
	@mkdir -p $(UPNP_BUILDDIR)
	cd $(UPNP_BUILDDIR) && cmake .. \
		-DCMAKE_BUILD_TYPE=Release \
		-DUPNPC_BUILD_SHARED=OFF \
		-DUPNPC_BUILD_TESTS=OFF \
		-DUPNPC_BUILD_SAMPLE=OFF
	$(MAKE) -C $(UPNP_BUILDDIR)
endif

# ---------- Dependency tracking ----------

-include $(DEPS)

# ---------- Clean ----------

clean:
	rm -rf $(BUILDDIR) $(TARGET)

distclean: clean
	rm -rf external/src/RandomX/build
	rm -rf external/src/libzmq/build
	rm -rf external/src/miniupnp/miniupnpc/build
	rm -f $(SUBMODULE_STAMP)
