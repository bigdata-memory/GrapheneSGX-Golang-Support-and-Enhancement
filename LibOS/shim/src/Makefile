RUNTIME_DIR = $(CURDIR)/../../../Runtime

include ../../../Makefile.configs
include ../../../Pal/src/Makefile.Host

OMIT_FRAME_POINTER = no

CFLAGS	= -Wall -fPIC -std=c11 -Winline -Wwrite-strings \
	  -fmerge-all-constants -Wstrict-prototypes \
	  -Werror=implicit-function-declaration \
	  $(cc-option, -Wnull-dereference) \
	  -fno-stack-protector -fno-builtin -Wno-inline \
	  -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx \
	  -I../include -I../../../Pal/lib -I../../../Pal/include/pal

EXTRAFLAGS = -Wextra

CFLAGS += $(EXTRAFLAGS)

ifeq ($(OMIT_FRAME_POINTER),yes)
CFLAGS += -DOMIT_FRAME_POINTER=1
else
CFLAGS += -fno-omit-frame-pointer -DOMIT_FRAME_POINTER=0
endif
ASFLAGS	= -Wa,--noexecstack -x assembler-with-cpp -I../include

LDFLAGS	= -shared -nostdlib -z combreloc -z relro -z now -z defs \
	  -dynamic-link=libpal.so \
	  -rpath-link=$(abspath $(RUNTIME_DIR))
ARFLAGS	=

ifeq ($(WERROR),1)
CFLAGS += -Werror
endif

files_to_build = libsysdb.a libsysdb.so libsysdb_debug.so
files_to_install = $(addprefix $(RUNTIME_DIR)/,$(files_to_build))

defs	= -DIN_SHIM
CFLAGS += $(defs)
ASFLAGS += $(defs)
fs	= chroot str pipe socket proc dev
ipcns	= pid sysv
objs	= $(addprefix bookkeep/shim_,handle vma thread signal) \
	  $(patsubst %.c,%,$(wildcard utils/*.c)) \
	  $(addprefix fs/shim_,dcache namei fs_hash fs) \
	  $(patsubst %.c,%,$(foreach f,$(fs),$(wildcard fs/$(f)/*.c))) \
	  $(addprefix ipc/shim_,ipc ipc_helper ipc_child) \
	  $(addprefix ipc/shim_ipc_,$(ipcns)) \
	  elf/shim_rtld \
	  $(addprefix shim_,init table syscalls checkpoint malloc \
	  async parser debug object) syscallas start \
	  $(patsubst %.c,%,$(wildcard sys/*.c)) \
	  vdso/vdso-data
graphene_lib = .lib/graphene-lib.a
pal_lib = $(RUNTIME_DIR)/libpal-$(PAL_HOST).so
headers = ../include/*.h ../../../Pal/lib/*.h ../../../Pal/include/pal/*.h

all: $(files_to_build) $(files_to_install)

ifeq ($(DEBUG),1)
CC += -gdwarf-2 -g3
CFLAGS += -DDEBUG
ASFLAGS += -DDEBUG
endif
export DEBUG

ifeq ($(PROFILING), 1)
CFLAGS += -DPROFILE
endif

$(files_to_install): $(RUNTIME_DIR)/%: %
	$(call cmd,ln_sf)

ifeq ($(findstring x86_64,$(SYS))$(findstring linux,$(SYS)),x86_64linux)
LDFLAGS-libsysdb.so += --version-script shim.map -T shim.lds
libsysdb.so: $(addsuffix .o,$(objs)) $(filter %.map %.lds,$(LDFLAGS-$@)) \
	     $(graphene_lib) $(pal_lib)
	$(call cmd,ld_so_o)

LDFLAGS-libsysdb_debug.so += --version-script shim-debug.map -T shim.lds
libsysdb_debug.so: $(addsuffix .o,$(objs)) \
		   $(filter %.map %.lds,$(LDFLAGS-$@)) \
		   $(graphene_lib) $(pal_lib)
	$(call cmd,ld_so_o)

.lib/host_endian.h: ../../../Pal/src/host/$(PAL_HOST)/host_endian.h
	@mkdir -p .lib
	$(call cmd,ln_sf)

sys/shim_sched.o: CFLAGS += -I../../../Pal/src/host/$(PAL_HOST)/ -I../../../Pal/include

.PHONY: $(graphene_lib)
$(graphene_lib): .lib/host_endian.h
	$(MAKE) -C ../../../Pal/lib target=$(abspath .lib)/

libsysdb.a: $(addsuffix .o,$(objs))
	$(call cmd,ar_a_o)
else
.IGNORE: $(shim_target)
$(shim_target):
endif

$(addsuffix .o,$(addprefix ipc/shim_ipc_,$(ipcns))): $(wildcard ipc/*.h)
elf/shim_rtld.o: $(wildcard elf/*.h)


%.o: %.c $(headers)
	$(call cmd,cc_o_c)

%.i: %.c $(headers)
	$(call cmd,cpp_i_c)

%.s: %.c $(headers)
	$(call cmd,cc_s_c)

%.o: %.S $(headers)
	$(call cmd,as_o_S)

%.s: %.S $(headers)
	$(call cmd,cpp_s_S)

LDFLAGS_vdso = -nostdlib -shared  \
               --hash-style=both --build-id -Bsymbolic \
               -m elf_x86_64 -soname linux-vdso.so.1 \
               --no-undefined \
               -z max-page-size=4096 -z common-page-size=4096
vdso/vdso.so.dbg: vdso/vdso.lds vdso/vdso.o
	@echo [ $@ ]
	$(LD) -o $@ -T $(filter %.lds, $^) $(LDFLAGS_vdso) $(filter %.o, $^)

vdso/vdso.so: vdso/vdso.so.dbg
	@echo [ $@ ]
	objcopy -S $^ $@

vdso/vdso-data.c: vdso/vdso.so
	@echo [ $@ ]
	xxd -i $^ | sed -e 's/vdso_vdso_/vdso_/' -e 's/\[\]/[4096]/' -e 's/= {/__attribute__((aligned(4096))) = {/' > $@

CLEAN_FILES += \
	vdso/vdso.o \
	vdso/vdso.so.dbg \
	vdso/vdso.so \
	vdso/vdso-data.c

syscallas.S shim_checkpoint.c: asm-offsets.h

include ../../../Makefile.rules

clean:
	rm -rf $(addsuffix .o,$(objs)) $(shim_target) $(files_to_build) .lib $(CLEAN_FILES)
