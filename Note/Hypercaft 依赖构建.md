# Hypercaft 依赖构建

关注点：

- arceos/makefile
- arceos/scripts/make/qemu.mk

## arceos/makefile

只关注riscv-v HY=y部分，其他部分省略。

```makefile
# Arguments
ARCH ?= riscv64
SMP ?= 1
MODE ?= release
LOG ?= warn

A ?= apps/hv
APP ?= $(A)
APP_FEATURES ?=
DISK_IMG ?= disk.img

FS ?= n
NET ?= n
GRAPHIC ?= n
BUS ?= mmio
HV ?= y
# 架构相关
ACCEL ?= n
PLATFORM ?= qemu-virt-riscv
TARGET := riscv64gc-unknown-none-elf

#Paths
LD_SCRIPT := $(CURDIR)/modules/axhal/linker_$(PLATFORM).lds
OUT_ELF := $(OUT_DIR)/$(APP_NAME)_$(PLATFORM).elf
OUT_BIN := $(OUT_DIR)/$(APP_NAME)_$(PLATFORM).bin


# Build the target
build: $(OUT_DIR) $(OUT_BIN)
run: build justrun
justrun:
	$(call run_qemu) #run_qemu into qemu.mk

```

简单来说，这里就配置了下启动



## arceos/scripts/make/qemu.mk

```makefile
# QEMU arguments
QEMU := qemu-system-$(ARCH)
GUEST ?= linux
ROOTFS ?= apps/hv/guest/$(GUEST)/rootfs.img

GUEST_DTB ?= apps/hv/guest/$(GUEST)/$(GUEST).dtb
GUEST_BIN ?= apps/hv/guest/$(GUEST)/$(GUEST).bin
GUEST_BIOS ?=

qemu_args-riscv64 := \
  -machine virt \
  -bios default \
  -kernel $(OUT_BIN)
  
 
 qemu_args-y := \
	-m 3G -smp $(SMP) $(qemu_args-$(ARCH)) \
    -device loader,file=$(GUEST_DTB),addr=0x90000000,force-raw=on \
    -device loader,file=$(GUEST_BIN),addr=0x90200000,force-raw=on

# ifeq ($(GUEST), linux)
qemu_args-$(HV) += \
	-drive file=$(ROOTFS),format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -append "root=/dev/vda rw console=ttyS0" 
    
#==================== Target =====================    
define run_qemu
  @printf "    $(CYAN_C)Running$(END_C) $(QEMU) $(qemu_args-y) $(1)\n"
  @$(QEMU) $(qemu_args-y)
  
endef

```

guest Linux 以外设挂载的形式，放入qemu中。









