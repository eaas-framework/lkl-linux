# Do not use make's built-in rules
# (this improves performance and avoids hard-to-debug behaviour);
.SUFFIXES:
MAKEFLAGS += -r

ifeq ($(V),1)
  Q =
else
  Q = @
endif

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

# forward all config-related targets to the main Kernel makefile
$(KCONFIG_AUTOCONFIG) $(KCONFIG_AUTOHEADER):
	$(Q)$(MAKE) -C $(srctree) ARCH=lkl $@

%config:
	$(Q)$(MAKE) -C $(srctree) ARCH=lkl $@

# include kernel config
include $(KCONFIG_AUTOCONFIG)

export CFLAGS += -Iinclude -Wall -g -O2 -Wextra -Wno-unused-parameter \
	  -Wno-missing-field-initializers -fno-strict-aliasing

export OUTPUT_FORMAT=$(shell $(LD) -r -print-output-format)

TEST_TARGETS=test valgrind gdb

ifneq (,$(filter $(OUTPUT_FORMAT),elf64-x86-64 elf32-i386 elf64-x86-64-freebsd elf32-littlearm ))
CFLAGS += -fPIC
LDFLAGS += -lpthread -lrt
else ifeq ($(OUTPUT_FORMAT),pe-i386)
KOPT="KALLSYMS_EXTRA_PASS=1"
LDFLAGS += -lws2_32
else
$(error Unrecognized platform: $(OUTPUT_FORMAT))
endif


ALL_PROGRAMS := lklfuse cptofs cpfromfs fs2tar
ALL_LIBRARIES := liblkl.a liblkl.so liblkl-hijack.so

all: $(ALL_PROGRAMS) $(ALL_LIBRARIES)

%-in.o: lib/lkl.o
	$(Q)$(MAKE) $(build)=$*

lib/lkl.o:
	$(Q)$(MAKE) -C ../.. ARCH=lkl defconfig
	$(Q)$(MAKE) -C ../.. ARCH=lkl $(KOPT) install INSTALL_PATH=$(PWD)

liblkl.a: lkl-in.o lib/lkl.o
	$(Q)$(AR) -rc $@ $^

liblkl.so: lkl-in.o lib/lkl.o
	$(Q)$(CC) -o $@ -shared $^

liblkl-hijack.so: hijack-in.o liblkl.a
	$(Q)$(CC) -shared -nodefaultlibs $^ -ldl $(LDFLAGS) -o $@

lklfuse: lklfuse-in.o liblkl.a
	$(Q)$(CC) -o $@ $(LDFLAGS) -lfuse $^

ifneq (,$(filter $(OUTPUT_FORMAT),elf64-x86-64-freebsd))
cptofs: LDFLAGS += -largp
fs2tar: LDFLAGS += -largp
endif
fs2tar: fs2tar-in.o liblkl.a
	$(Q)$(CC) -o $@ $(LDFLAGS) -larchive $^

cptofs: cptofs-in.o liblkl.a
	$(Q)$(CC) -o $@ $(LDFLAGS) $^

cpfromfs: cptofs
	$(Q)if ! [ -e $@ ]; then ln -s $< $@; fi

clean:
	$(Q)find . -name '*.o' -delete -o -name '\.*.cmd' -delete -o -name '\.*.d' -delete
	$(Q)-rm -r -f include/lkl/ $(ALL_PROGRAMS) $(ALL_LIBRARIES)

$(TEST_TARGETS): all
	$(MAKE) -C tests $@

