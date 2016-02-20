# Do not use make's built-in rules
# (this improves performance and avoids hard-to-debug behaviour);
# also do not print "Entering directory..." messages from make
.SUFFIXES:
MAKEFLAGS += -r --no-print-directory 

ifeq ($(V),1)
  Q =
else
  Q = @
endif

NON_CONFIG_TARGETS := clean config %config

# By default we want to use gcc (as does Linux), especially as the clang
# cross toolchain does not use prefixed names
CC    := $(CROSS_COMPILE)gcc
LD    := $(CROSS_COMPILE)$(LD)
AR    := $(CROSS_COMPILE)$(AR)

export CC LD AR

EXESUF   := 
SOSUF    := .so

all:

-include ../scripts/Makefile.include

ifeq ($(srctree),)
srctree := $(patsubst %/,%,$(dir $(shell pwd)))
srctree := $(patsubst %/,%,$(dir $(srctree)))
endif
export srctree

build := -f $(srctree)/tools/build/Makefile.build dir=. obj

# we rely on the global Kconfig for the lkl host library
KCONFIG_CONFIG        ?= $(srctree)/.config
KCONFIG_AUTOHEADER    := $(srctree)/include/generated/autoconf.h
KCONFIG_AUTOCONFIG    := $(srctree)/include/config/auto.conf

export KCONFIG_AUTOCONFIG
export KCONFIG_AUTOHEADER

# forward all config-related targets to the main Kernel makefile
$(KCONFIG_AUTOCONFIG) $(KCONFIG_AUTOHEADER): $(KCONFIG_CONFIG)
	$(Q)$(MAKE) -C $(srctree) ARCH=lkl include/config/auto.conf

$(KCONFIG_CONFIG): ;

config: FORCE
	$(Q)$(MAKE) -C $(srctree) ARCH=lkl $@

%config: FORCE
	$(Q)$(MAKE) -C $(srctree) ARCH=lkl $@

ifneq ($(filter-out $(NON_CONFIG_TARGETS),$(MAKECMDGOALS)),)
  # include kernel config
  include $(KCONFIG_AUTOCONFIG)
else ifeq ($(MAKECMDGOALS),)
  # include kernel config
  include $(KCONFIG_AUTOCONFIG)
endif

export CFLAGS += -Iinclude -Wall -g -O2 -Wextra -Wno-unused-parameter \
	  -Wno-missing-field-initializers -fno-strict-aliasing

export OUTPUT_FORMAT=$(shell $(LD) -r -print-output-format)

TEST_TARGETS=test valgrind gdb

ifneq (,$(filter $(OUTPUT_FORMAT),elf64-x86-64 elf32-i386 elf64-x86-64-freebsd))
CFLAGS += -fPIC
LDLIBS += -lpthread -lrt
export CONFIG_LKL_AUTO_POSIX_HOST=y
else ifeq ($(OUTPUT_FORMAT),elf32-littlearm)
CFLAGS += -fPIC
export CONFIG_LKL_AUTO_POSIX_HOST=y
else ifeq ($(OUTPUT_FORMAT),pe-i386)
KOPT="KALLSYMS_EXTRA_PASS=1"
LDLIBS += -lws2_32
EXESUF   := .exe
SOSUF    := .dll
export CONFIG_LKL_AUTO_NT_HOST=y
else
$(error Unrecognized platform: $(OUTPUT_FORMAT))
endif


ALL_PROGRAMS := 
ifeq (y,$(CONFIG_LKL_FUSE))
ALL_PROGRAMS += lklfuse$(EXESUF)
endif
ifeq (y,$(CONFIG_LKL_CPTOFS))
ALL_PROGRAMS += cptofs$(EXESUF) cpfromfs$(EXESUF)
endif
ifeq (y,$(CONFIG_LKL_FS2TAR))
ALL_PROGRAMS += fs2tar$(EXESUF)
endif

ALL_LIBRARIES := 
ifeq (y,$(CONFIG_LKL_STATIC))
ALL_LIBRARIES += liblkl.a
endif
ifeq (y,$(CONFIG_LKL_SHARED))
ALL_LIBRARIES += liblkl$(SOSUF)
endif
ifeq (y,$(CONFIG_LKL_HIJACK))
ALL_LIBRARIES += liblkl-hijack$(SOSUF)
endif

all: $(ALL_PROGRAMS) $(ALL_LIBRARIES)

%-in.o: lib/lkl.o FORCE
	$(Q)$(MAKE) $(build)=$*

lib/lkl.o: $(KCONFIG_AUTOCONFIG)
	$(Q)$(MAKE) -C ../.. ARCH=lkl $(KOPT) install INSTALL_PATH=$(PWD)

static: liblkl.a
shared: liblkl$(SOSUF)
hijack: liblkl-hijack$(SOSUF)

liblkl.a: lkl-in.o lib/lkl.o
	$(QUIET_AR)$(AR) -rc $@ $^

liblkl$(SOSUF): lkl-in.o lib/lkl.o
	$(QUIET_LINK)$(CC) -shared $^ $(LDLIBS) -o $@

liblkl-hijack$(SOSUF): hijack-in.o liblkl.a
	$(QUIET_LINK)$(CC) -shared -nodefaultlibs $^ -ldl $(LDLIBS) -o $@

lklfuse$(EXESUF): lklfuse-in.o liblkl.a
	$(QUIET_LINK)$(CC) -o $@ $(LDLIBS) -lfuse $^

ifneq (,$(filter $(OUTPUT_FORMAT),elf64-x86-64-freebsd))
cptofs$(EXESUF): LDLIBS += -largp
fs2tar$(EXESUF): LDLIBS += -largp
endif
fs2tar$(EXESUF): fs2tar-in.o liblkl.a
	$(QUIET_LINK)$(CC) -o $@ $(LDLIBS) -larchive $^

cptofs$(EXESUF): cptofs-in.o liblkl.a
	$(QUIET_LINK)$(CC) -o $@ $(LDLIBS) $^

cpfromfs$(EXESUF): cptofs$(EXESUF)
	$(Q)if ! [ -e $@ ]; then ln -s $< $@; fi

clean:
	$(Q)find . -name '*.o' -delete -o -name '\.*.cmd' -delete -o -name '\.*.d' -delete
	$(call QUIET_CLEAN, core-progs) rm -r -f include/lkl/ $(ALL_PROGRAMS) $(ALL_LIBRARIES)

$(TEST_TARGETS): all
	$(MAKE) -C tests $@

FORCE: ;
.PHONY: all clean $(TEST_TARGETS) FORCE
