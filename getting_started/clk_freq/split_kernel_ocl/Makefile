# Points to Utility Directory
COMMON_REPO = ../../../
ABS_COMMON_REPO = $(shell readlink -f $(COMMON_REPO))

include ./utils.mk
# Run Target:
#   hw  - Compile for hardware
#   sw_emu/hw_emu - Compile for software/hardware emulation
# FPGA Board Platform (Default ~ vcu1525)

TARGETS := hw
TARGET := $(TARGETS)
DEVICES := xilinx_vcu1525_dynamic
DEVICE := $(DEVICES)
XCLBIN := ./xclbin
DSA := $(call device2sandsa, $(DEVICE))

CXX := $(XILINX_SDX)/bin/xcpp
XOCC := $(XILINX_SDX)/bin/xocc

CXXFLAGS := $(opencl_CXXFLAGS) -Wall -O0 -g -std=c++14
LDFLAGS := $(opencl_LDFLAGS)

HOST_SRCS = src/host.cpp

# Host compiler global settings
CXXFLAGS = -I $(XILINX_SDX)/runtime/include/1_2/ -I/$(XILINX_SDX)/Vivado_HLS/include/ -O0 -g -Wall -fmessage-length=0 -std=c++14
LDFLAGS = -lOpenCL -lpthread -lrt -lstdc++ -L$(XILINX_SDX)/runtime/lib/x86_64

# Kernel compiler global settings
CLFLAGS = -t $(TARGET) --platform $(DEVICE) --save-temps 
CLFLAGS += --xp "param:compiler.xclDataflowFifoDepth=16"


EXECUTABLE = host

EMCONFIG_DIR = $(XCLBIN)/$(DSA)

BINARY_CONTAINERS += $(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xclbin
BINARY_CONTAINER_sketch_GOOD_OBJS += $(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xo
BINARY_CONTAINERS += $(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xclbin
BINARY_CONTAINER_sketch_BAD_OBJS += $(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xo

#Include Libraries
include $(ABS_COMMON_REPO)/libs/opencl/opencl.mk
include $(ABS_COMMON_REPO)/libs/xcl2/xcl2.mk
include $(ABS_COMMON_REPO)/libs/bitmap/bitmap.mk
CXXFLAGS += $(xcl2_CXXFLAGS) $(bitmap_CXXFLAGS)
LDFLAGS += $(xcl2_LDFLAGS) $(bitmap_LDFLAGS)
HOST_SRCS += $(xcl2_SRCS) $(bitmap_SRCS)

CP = cp -rf
DATA = ./data

.PHONY: all clean cleanall docs emconfig
all: $(EXECUTABLE) $(BINARY_CONTAINERS) emconfig

.PHONY: exe
exe: $(EXECUTABLE)

# Building kernel
$(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xo: ./src/sketch.cl
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) -c -k process_image -I'$(<D)' -o'$@' '$<'
$(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xo: ./src/sketch_bad.cl
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) -c -k process_image -I'$(<D)' -o'$@' '$<'

$(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xclbin: $(BINARY_CONTAINER_sketch_GOOD_OBJS)
	$(XOCC) $(CLFLAGS) -l $(LDCLFLAGS) --nk process_image:1 -o'$@' $(+)
$(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xclbin: $(BINARY_CONTAINER_sketch_BAD_OBJS)
	$(XOCC) $(CLFLAGS) -l $(LDCLFLAGS) --nk process_image:1 -o'$@' $(+)

# Building Host
$(EXECUTABLE): $(HOST_SRCS) $(HOST_HDRS)
	mkdir -p $(XCLBIN)
	$(CXX) $(CXXFLAGS) $(HOST_SRCS) $(HOST_HDRS) -o '$@' $(LDFLAGS)

emconfig:$(EMCONFIG_DIR)/emconfig.json
$(EMCONFIG_DIR)/emconfig.json:
	emconfigutil --platform $(DEVICE) --od $(EMCONFIG_DIR)

check: all
ifeq ($(TARGET),$(filter $(TARGET),sw_emu hw_emu))
	$(CP) $(EMCONFIG_DIR)/emconfig.json .
	XCL_EMULATION_MODE=$(TARGET) ./$(EXECUTABLE) ./data/car_128x128.bmp
else
	 ./$(EXECUTABLE) ./data/car_128x128.bmp
endif
	sdx_analyze profile -i sdaccel_profile_summary.csv -f html

# Cleaning stuff
clean:
	-$(RMDIR) $(EXECUTABLE) $(XCLBIN)/{*sw_emu*,*hw_emu*} 
	-$(RMDIR) sdaccel_* TempConfig system_estimate.xtxt *.rpt
	-$(RMDIR) src/*.ll _xocc_* .Xil emconfig.json dltmp* xmltmp* *.log *.jou *.wcfg *.wdb

cleanall: clean
	-$(RMDIR) $(XCLBIN)
	-$(RMDIR) ./_x
	-$(RMDIR) ./output.bmp 
.PHONY: help

help::
	$(ECHO) "Makefile Usage:"
	$(ECHO) "  make all TARGET=<sw_emu/hw_emu/hw> DEVICE=<FPGA platform>"
	$(ECHO) "      Command to generate the design for specified Target and Device."
	$(ECHO) ""
	$(ECHO) "  make clean "
	$(ECHO) "      Command to remove the generated non-hardware files."
	$(ECHO) ""
	$(ECHO) "  make cleanall"
	$(ECHO) "      Command to remove all the generated files."
	$(ECHO) ""
	$(ECHO) "  make check TARGET=<sw_emu/hw_emu/hw> DEVICE=<FPGA platform>"
	$(ECHO) "      Command to run application in emulation."
	$(ECHO) ""

docs: README.md

README.md: description.json
	$(ABS_COMMON_REPO)/utility/readme_gen/readme_gen.py description.json

