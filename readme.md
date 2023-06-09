# attest: a language-agnostic test runner

## summary
`attest` is a program which detects and runs tests. At its core, it assumes
tests come in triples: input file, executable, and output file: the executable
can be reused between multiple tests. A test passes if it successfully exits and
its output is close (compared by `numdiff`) to the provided output file.

`attest` was developed for IBAMR and contains a variety of features (using
`numdiff` for numerical tolerances, running programs with MPI) which are
convenient for testing scientific programs.

## key features
- language-agnostic: tests are simply executables, input files, and output files
- independent of the build system: relies on some external tool to actually
  compile tests
- runs in parallel (e.g., `./attest -j24`)
- understands MPI
- understands restart files (in the style of IBAMR)
- Checks for both successful execution (the `RUN` stage) and successful output
  comparison (the `DIFF` stage)

## motivation
- *Assertions are not enough*: for scientific computing workflows, its typically
  useful to print a convergence rate, or to just dump a bunch of known-correct
  values to a log file. Many tests start as example problems computing some
  known quantity (like lift and drag). In these cases, its easier to specify an
  output file than to manually write assertions validating correct values.
- *Native support for MPI*: if a test needs 8 processors, the test runner needs
  to be aware of that and schedule tasks accordingly.
- *one executable, many tests*: when writing IBAMR's test suite I wanted to
  store output for several dozen IB kernels: each test needs to do the same
  thing with just one change. It is vastly more efficient to compile one test
  and modify its run-time behavior with an input file than to compile dozens of
  tests with slight tweaks in their behavior. In particular, since tests are
  very quick to execute, we can vastly improve the overall performance of the
  test-suite by limiting the total number of executables that need to be
  compiled since that (for C++) tends to be the more expensive step.

## describing tests
`attest` assumes that a project's build system can create tests in a directory
named `tests/`, as well as create (or link) in the input and output files.

Specific instructions for executing tests are encoded in the input files between
periods. `attest` supports the following options:
- `.mpirun=X` executes the test via `mpiexec -n X`
- `.expect_error=True` will let a test's `RUN` stage as successful if it returns
  a nonzero exit code (the test still needs to pass the `DIFF` stage in the
  normal way).
- `.restart=Y` will execute a test and then execute it again with the additional
  command-line arguments `./restart Y`, i.e., for picking the `Y`th restart file
  found in directory `./restart`.
  
Additional documentation on tests can be added between periods - `attest` will
just ignore them. An example input file name is
`explicit_ex4_2d.scratch_hier.restart=25.mpirun=4.input`: `attest` will use 4
MPI processes, execute the restart as described above, and will ignore the
string `.scratch_hier.`.

## command-line arguments
For a complete listing run `./attest --help`. A few useful options:
- `--verbose`: print some details regarding why a test failed (like the first
  few lines of `numdiff` output)
- `-jX`: run `X` executables in parallel. This is aware of MPI (e.g., if a test
  specifies `mpirun=8`, then `-j8` will run that test alone, or if two tests
  specify `mpirun=4` they will be run concurrently)

## history
`attest` is strongly influenced by `ctest` and more-or-less resembles the way
deal.II calls `ctest`. Its name was originally short for "autotools test
runner", as it was written to support tests compiled and linked by an
autotools-based build system.

## integration with CMake
It is usually better to symbolically link input and output files. Since `CMake`'s
symlink function is extremely slow it is better to do this with a shell script:
```
# Replacement for cmake -E create_symlink that is much faster.
#
# $1 is the source test directory
# $2 is the build test directory

INPUT_DIR="$1"
OUTPUT_DIR="$2"

# We have to avoid using loops or other such things to work with weird file
# names (e.g., paths that include spaces)
find "$INPUT_DIR" \( -name '*.input' -o -name '*.output' \) -exec ln -f -s {} "$OUTPUT_DIR" \;
```

You should create a file `tests/attest.conf.in`:
```
# attest default configuration file. Generated by the build system.
# If you make any changes to this file they will be overwritten when
# you rerun ./configure or cmake.
[attest]
jobs = 1
keep_work_directories = False
mpiexec = @MPIEXEC_EXECUTABLE@
numdiff = @NUMDIFF_EXECUTABLE@
show_only = False
test_directory = /tests/
test_timeout = 600
include_regex = .*
exclude_regex = ^$
verbose = False
```

This specifies the default arguments for `attest`, including the installed
locations of `mpiexec` and `numdiff`. Set up this file and find numdiff as
```cmake
ADD_CUSTOM_COMMAND(TARGET tests
  POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_SOURCE_DIR}/attest ${CMAKE_BINARY_DIR}/attest)

# Set up the input and output files. Since the input and output files aren't
# really used by the build system we use a shell script to find them every time
# 'make test' is run rather than evaluating the glob when cmake generates the
# build system.
FOREACH(_dir ${TEST_DIRECTORIES})
  ADD_CUSTOM_COMMAND(TARGET "tests-${_dir}"
    POST_BUILD
    COMMAND bash ${CMAKE_SOURCE_DIR}/tests/link-test-files.sh
    ${CMAKE_SOURCE_DIR}/tests/${_dir} ${CMAKE_BINARY_DIR}/tests/${_dir}
    VERBATIM)
ENDFOREACH()
# Find numdiff, if possible (we only need it for tests so its not essential that
# we find it now)
FIND_PROGRAM(NUMDIFF_EXECUTABLE NAMES numdiff HINTS ${NUMDIFF_ROOT} PATH_SUFFIXES bin)

IF ("${NUMDIFF_EXECUTABLE}" STREQUAL "NUMDIFF_EXECUTABLE-NOTFOUND")
  MESSAGE(WARNING "\
The configuration script was not able to locate numdiff. If you want to run \
the test suite you will need to either edit attest.conf, specify the path to \
numdiff to attest, or rerun CMake with the argument NUMDIFF_ROOT specifying \
numdiff's root installation directory.")
  # clear the value so that attest.conf doesn't contain an invalid path
  SET(NUMDIFF_EXECUTABLE "")
ENDIF()

# Set up the default attest configuration file:
CONFIGURE_FILE(${CMAKE_SOURCE_DIR}/tests/attest.conf.in
  ${CMAKE_BINARY_DIR}/attest.conf)
```

## integration with autotools
You'll need to add the default configuration file `attest.conf.in` to
`AC_CONFIG_FILES()` to get it set up correctly: the same template file works for
both CMake and autotools. You can then add the following to the top-level
`Makefile.am`:
```makefile
MPIEXEC = @MPIEXEC@
NUMDIFF = @NUMDIFF@
# attest uses parameters set in attest.conf; said parameters are overridden by
# command line arguments. Use configuration info to set up the path to mpirun:
attest.conf: $(abs_builddir)/tests/attest.conf
	ln -f -s ./tests/attest.conf $(abs_builddir)

tests: lib attest.conf
	ln -f -s $(top_srcdir)/attest $(abs_builddir)
	@cd $@ && $(MAKE) $(AM_MAKEFLAGS) $@
.PHONY: tests
```
You'll need to somehow set up `numdiff` and `mpirun` so that they are known to
the build system. To set up input and output files you'll need a test
`Makefile.am` that looks something like this (taken from IBAMR):
```makefile
EXTRA_PROGRAMS = adv_diff_01_3d adv_diff_02_2d

adv_diff_01_3d_CXXFLAGS = $(AM_CXXFLAGS) -DNDIM=3
adv_diff_01_3d_LDADD = $(IBAMR_LDFLAGS) $(IBAMR3d_LIBS) $(IBAMR_LIBS)
adv_diff_01_3d_SOURCES = adv_diff_01.cpp

adv_diff_02_2d_CXXFLAGS = $(AM_CXXFLAGS) -DNDIM=2
adv_diff_02_2d_LDADD = $(IBAMR_LDFLAGS) $(IBAMR2d_LIBS) $(IBAMR_LIBS)
adv_diff_02_2d_SOURCES = adv_diff_02.cpp

tests: $(EXTRA_PROGRAMS)
	if test "$(top_srcdir)" != "$(top_builddir)" ; then \
	  ln -f -s $(srcdir)/*input $(PWD) ; \
	  ln -f -s $(srcdir)/*output $(PWD) ; \
	fi ;
.PHONY: tests
```
