# GNUmakefile for p2pool - thin wrapper around CMake
#
# Usage:
#   make              - build p2pool (release)
#   make debug        - build p2pool (debug)
#   make clean        - remove build artifacts
#   make distclean    - clean + remove submodule stamp
#   make submodules   - initialize git submodules
#   make install-services - install systemd user service files

SHELL := /bin/bash
BUILDDIR := build
SUBMODULE_STAMP := .submodules_initialized

# systemd user service files
SERVICE_FILES := monerod.service p2pool.service xmrig.service
SYSTEMD_USER_DIR := $(HOME)/.config/systemd/user

# CMake options to pass through
CMAKE_OPTS ?=

.PHONY: all debug clean distclean submodules install-services

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

install-services: $(SERVICE_FILES)
	@mkdir -p $(SYSTEMD_USER_DIR)
	cp $(SERVICE_FILES) $(SYSTEMD_USER_DIR)/
	systemctl --user daemon-reload
	@echo "Installed service files to $(SYSTEMD_USER_DIR)"
	@echo "Configure wallet in ~/.config/p2pool/env"
	@echo "Then: systemctl --user enable --now monerod p2pool xmrig"
	@if loginctl show-user $(USER) --property=Linger 2>/dev/null \
	    | grep -q 'Linger=no'; then \
	  echo "WARNING: Linger is not enabled for user $(USER)."; \
	  echo "  Services will NOT start at boot without it."; \
	  echo "  Enable with: sudo loginctl enable-linger $(USER)"; \
	elif ! loginctl show-user $(USER) --property=Linger 2>/dev/null \
	    | grep -q 'Linger=yes'; then \
	  echo "WARNING: Could not determine Linger status for user $(USER)."; \
	  echo "  Ensure Linger is enabled: sudo loginctl enable-linger $(USER)"; \
	fi
