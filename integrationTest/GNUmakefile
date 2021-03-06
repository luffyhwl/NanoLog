########
## This make file is adapted from the benchmark GNUmakefile.
## It compiles a test suite of tricky syntax with the preprocessor
#######

####
## Library Options [Required]
####

# [Required] Specifies the path to the NanoLog's Library directory
NANOLOG_DIR=..

####
# User Section
####
USER_SRCS=main.cc SimpleTestObject.cc folder/main.cc

# [Required] Macro MUST used to identify user *.o files
USER_OBJS=$(USER_SRCS:.cc=.o)

# -O3 should always be passed for high performance
CXXFLAGS= -std=c++0x -O3
CXX=g++

all: run-test run-empty-test
	@printf "\r\n\033[0;32mIntegration Tests completed without error!\033[0m\r\n"

# [Required] run-cxx will compile the user C++ source file into an object file using
# the NanoLog system. See documentation in the Library Compilation section.
%.o: %.cc
	$(call run-cxx, $@, $<, $(CXXFLAGS))

testApp: $(USER_OBJS) libNanoLog.a
	$(CXX) -o testApp $(CXXFLAGS) $(USER_OBJS) -L. -lNanoLog $(LIBRARY_LIBS)


run-test: testApp
	@rm -f /tmp/testLog
	./testApp > /dev/null
	@chmod 666 /tmp/testLog

	@# Test for string special case
	@test -n "$$(find /tmp/testLog -size -100000c)" || \
		(printf "\r\n\033[0;31mError: /tmp/testLog is very large, suggesting a failure of the 'Special case string precision' test in main.cc\033[0m" \
		&& echo "" && exit 1)

	@# Run the decompressor and cut out the timestamps before the ':'
	./decompressor /tmp/testLog | cut -d':' -f5- > output.txt
	diff -w expectedOutput.txt output.txt

	@rm -f /tmp/testLog

run-empty-test: testApp
	@touch emptyFile
	@printf "Error: Could not read initial checkpoint, the compressed log may be corrupted.\r\n" > expected.txt
	@printf "Unable to open file emptyFile\r\n" >> expected.txt

	./decompressor emptyFile > output.txt
	diff -w expected.txt output.txt
	@rm -f output.txt expected.txt emptyFile

clean:
	@rm -f *.o testApp folder/*.o /tmp/testLog output.txt emptyFile

####
# Library Compilation (copy verbatim)
####

RUNTIME_DIR=$(NANOLOG_DIR)/runtime
PREPROC_DIR=$(NANOLOG_DIR)/preprocessor

# run-cxx:
# Compile a user C++ source file to an object file using the NanoLog system.
# The first parameter $(1) should be the output filename (*.o)
# The second parameter $(2) should be the input filename (*.cc)
# The optional third parameter $(3) is any additional options compiler options.
define run-cxx
	$(CXX) -E -I $(RUNTIME_DIR) $(2) -o $(2).i $(3)
	@mkdir -p generated
	python $(PREPROC_DIR)/parser.py --mapOutput="generated/$(2).map" $(2).i
	$(CXX) -I $(RUNTIME_DIR) -c -o $(1) $(2).ii $(3)
	@rm -f $(2).i $(2).ii generated/BufferStuffer.h
endef

RUNTIME_CXX_FLAGS= -std=c++0x -O3 -g
LIBRARY_LIBS=-lrt -pthread

RUNTIME_DEPS=$(wildcard $(RUNTIME_DIR)/*.h)
RUNTIME_CC=$(RUNTIME_DIR)/Cycles.cc $(RUNTIME_DIR)/NanoLog.cc \
						  $(RUNTIME_DIR)/Util.cc $(RUNTIME_DIR)/Log.cc
RUNTIME_OBJS=$(RUNTIME_CC:.cc=.o)

COMWARNS := -Wall -Wformat=2 -Wextra \
           -Wwrite-strings -Wno-unused-parameter -Wmissing-format-attribute
CWARNS   := $(COMWARNS) -Wmissing-prototypes -Wmissing-declarations -Wshadow \
		-Wbad-function-cast
CXXWARNS := $(COMWARNS) -Wno-non-template-friend -Woverloaded-virtual \
		-Wcast-qual -Wcast-align -Wconversion
ifeq ($(COMPILER),gnu)
CXXWARNS += -Weffc++
endif

generated/BufferStuffer.o: $(USER_OBJS)
	mkdir -p generated
	python $(PREPROC_DIR)/parser.py --combinedOutput="generated/BufferStuffer.cc" $(shell find generated -type f -name "*.map" -printf ' "%h/%f" ')
	$(CXX) $(RUNTIME_CXX_FLAGS) $(CXXWARNS) -c -o $@ generated/BufferStuffer.cc -I $(RUNTIME_DIR) -Igenerated

$(RUNTIME_DIR)/%.o: $(RUNTIME_DIR)/%.cc generated/BufferStuffer.o
	$(CXX) $(RUNTIME_CXX_FLAGS) $(CXXWARNS) -c -o $@ $< -I $(RUNTIME_DIR) -Igenerated -Werror $(LIBRARY_LIBS)

libNanoLog.a: $(RUNTIME_OBJS) $(RUNTIME_DEPS) $(RUNTIME_DIR)/Cycles.o decompressor
	ar -cr libNanoLog.a $(RUNTIME_OBJS) generated/BufferStuffer.o

decompressor: generated/BufferStuffer.o $(RUNTIME_OBJS) $(RUNTIME_DIR)/LogDecompressor.cc
	$(CXX) $(RUNTIME_CXX_FLAGS) $(CXXWARNS) $(RUNTIME_OBJS) generated/BufferStuffer.o $(RUNTIME_DIR)/LogDecompressor.cc -o decompressor -I$(RUNTIME_DIR) -Igenerated -Werror $(LIBRARY_LIBS)

clean-all: clean
	@rm -f libNanoLog.a $(RUNTIME_OBJS) *.pyc decompressor
	@rm -rf generated
