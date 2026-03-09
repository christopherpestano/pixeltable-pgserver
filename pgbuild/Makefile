SHELL := /bin/bash
INSTALL_RELPATH := ../src/pixeltable_pgserver/pginstall
INSTALL_PREFIX := $(shell pwd)/$(INSTALL_RELPATH)
BUILD := $(shell pwd)/pgbuild/

.PHONY: all
all: pgvector postgis postgres

### postgres
POSTGRES_VERSION := 16.10
POSTGRES_URL := https://ftp.postgresql.org/pub/source/v$(POSTGRES_VERSION)/postgresql-$(POSTGRES_VERSION).tar.gz
POSTGRES_SRC := postgresql-$(POSTGRES_VERSION)
POSTGRES_BLD := $(POSTGRES_SRC)

$(POSTGRES_SRC).tar.gz:
	curl -L -O $(POSTGRES_URL)

## extract
$(POSTGRES_SRC)/configure: $(POSTGRES_SRC).tar.gz
	tar xzf $(POSTGRES_SRC).tar.gz
	touch $(POSTGRES_SRC)/configure

## configure
$(POSTGRES_BLD)/config.status: $(POSTGRES_SRC)/configure
	mkdir -p $(POSTGRES_BLD)
	cd $(POSTGRES_BLD) && ../$(POSTGRES_SRC)/configure --prefix=$(INSTALL_PREFIX) --without-icu

## build
# https://stackoverflow.com/questions/68379786/
# for explanation of unsetting make env variables prior to calling postgres' own make
$(POSTGRES_BLD)/src/bin/initdb/initdb: $(POSTGRES_BLD)/config.status
	unset MAKELEVEL && unset MAKEFLAGS && unset MFLAGS && $(MAKE) -C $(POSTGRES_BLD) -j

## install to INSTALL_PREFIX
# On Windows, the Postgres build (in particular, the `zic` tool) fails to compile the timezone database into its proper
# place. I don't know why this is; no error message is generated, though I suspect it's because the pathname is too
# long, or something of that nature. In any case, we add an additional post-build step to manually compile the timezone
# data using a shorter path reference, ensuring that it gets generated properly on Windows.
$(INSTALL_PREFIX)/bin/postgres: $(POSTGRES_BLD)/config.status
	mkdir -p $(INSTALL_PREFIX)
	unset MAKELEVEL && unset MAKEFLAGS && unset MFLAGS && $(MAKE) -C $(POSTGRES_BLD) install
	$(POSTGRES_BLD)/src/timezone/zic -d $(INSTALL_RELPATH)/share/postgresql/timezone $(POSTGRES_BLD)/src/timezone/data/tzdata.zi

.PHONY: postgres
postgres: $(INSTALL_PREFIX)/bin/postgres

### pgvector
PGVECTOR_TAG := v0.8.1
PGVECTOR_URL := https://github.com/pgvector/pgvector/archive/refs/tags/$(PGVECTOR_TAG).tar.gz
PGVECTOR_DIR := pgvector-$(PGVECTOR_TAG)

$(PGVECTOR_DIR).tar.gz:
	curl -L -o $(PGVECTOR_DIR).tar.gz $(PGVECTOR_URL)

$(PGVECTOR_DIR)/Makefile: $(PGVECTOR_DIR).tar.gz
	# tar extract into pgvector-$(PGVECTOR_TAG)
	mkdir -p $(PGVECTOR_DIR)
	tar xzf $(PGVECTOR_DIR).tar.gz -C $(PGVECTOR_DIR) --strip-components=1
	touch $(PGVECTOR_DIR)/Makefile

$(INSTALL_PREFIX)/lib/vector.so: $(PGVECTOR_DIR)/Makefile $(INSTALL_PREFIX)/bin/postgres
	unset MAKELEVEL && unset MAKEFLAGS && unset MFLAGS \
		&& export PG_CONFIG=$(INSTALL_PREFIX)/bin/pg_config \
		&& $(MAKE) -C $(PGVECTOR_DIR) -j \
		&& $(MAKE) -C $(PGVECTOR_DIR) install

.PHONY: pgvector
pgvector: postgres $(INSTALL_PREFIX)/lib/vector.so

### postgis
POSTGIS_VERSION := 3.5.2
POSTGIS_URL := https://download.osgeo.org/postgis/source/postgis-$(POSTGIS_VERSION).tar.gz
POSTGIS_DIR := postgis-$(POSTGIS_VERSION)

$(POSTGIS_DIR).tar.gz:
	curl -L -O $(POSTGIS_URL)

$(POSTGIS_DIR)/configure: $(POSTGIS_DIR).tar.gz
	tar xzf $(POSTGIS_DIR).tar.gz
	touch $(POSTGIS_DIR)/configure

$(POSTGIS_DIR)/config.status: $(POSTGIS_DIR)/configure $(INSTALL_PREFIX)/bin/postgres
	cd $(POSTGIS_DIR) && ./configure \
		--with-pgconfig=$(INSTALL_PREFIX)/bin/pg_config

$(INSTALL_PREFIX)/lib/postgis-3.so: $(POSTGIS_DIR)/config.status
	unset MAKELEVEL && unset MAKEFLAGS && unset MFLAGS \
		&& $(MAKE) -C $(POSTGIS_DIR) -j \
		&& $(MAKE) -C $(POSTGIS_DIR) install

.PHONY: postgis
postgis: postgres $(INSTALL_PREFIX)/lib/postgis-3.so

### other
.PHONY: clean clean-all
clean:
	rm -rf $(INSTALL_PREFIX)
	rm -rf $(POSTGRES_SRC)
	rm -rf $(POSTGRES_BLD)
	rm -rf $(PGVECTOR_DIR)
	rm -rf $(POSTGIS_DIR)

clean-all: clean
	rm -rf *.tar.gz
