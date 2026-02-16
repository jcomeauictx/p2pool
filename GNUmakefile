# GNUmakefile for p2pool - thin wrapper around CMake
#
# Usage:
#   make              - build p2pool (release)
#   make debug        - build p2pool (debug)
#   make clean        - remove build artifacts
#   make distclean    - clean + remove submodule stamp
#   make submodules   - initialize git submodules

SHELL := /bin/bash
BUILDDIR := build
SUBMODULE_STAMP := .submodules_initialized

# CMake options to pass through
CMAKE_OPTS ?=

.PHONY: all debug clean distclean submodules

all: $(BUILDDIR)/Makefile
	$(MAKE) -C $(BUILDDIR)
	cp $(BUILDDIR)/p2pool .

debug: CMAKE_OPTS += -DDEV_DEBUG=ON
debug: clean-build $(BUILDDIR)/Makefile
	$(MAKE) -C $(BUILDDIR)
	cp $(BUILDDIR)/p2pool .

$(BUILDDIR)/Makefile: $(SUBMODULE_STAMP) CMakeLists.txt
	mkdir -p $(BUILDDIR)
	cd $(BUILDDIR) && cmake .. $(CMAKE_OPTS)

# Submodule initialization
$(SUBMODULE_STAMP):
	git submodule update --init --recursive
	touch $@

submodules: $(SUBMODULE_STAMP)

clean-build:
	rm -rf $(BUILDDIR)

clean:
	rm -rf $(BUILDDIR) p2pool

distclean: clean
	rm -f $(SUBMODULE_STAMP)
