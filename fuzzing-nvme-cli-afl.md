# fuzzing-nvme-cli-afl

Steps:
1. Build AFLplusplus following the instructions in AFLplusplus/docks/INSTALL.md
	a. Install dependencies
	b. run `make distrib`
2. Add AFL's `argv-fuzz-inl.h` to nvme-cli by creating a symlink.
3. Add `#include "argv-fuzz-inl.h"` to the end of the includes in `nvme.c` of nvme-cli.
4. Add `AFL_INIT_ARGV();` to the beginning of `main()` in `nvme.c` of nvme-cli.
	a. This replaces the argv array with input from stdin.
5. Run the `build_with_afl.sh` script in nvme-cli:
```bash
#!/bin/bash

AFL_ROOT_DIR="$HOME/Projects/AFLplusplus"
BUILD_DIR="build"
export CC="$AFL_ROOT_DIR/afl-clang-fast"
export CXX="$AFL_ROOT_DIR/afl-clang-fast++"
export LD="$AFL_ROOT_DIR/afl-clang-fast"
meson "$BUILD_DIR"
ninja -C "$BUILD_DIR"
```
6. Create fuzz script:
```bash
#!/bin/bash

AFL_FUZZ="$HOME/Projects/AFLplusplus/afl-fuzz"
NVME_CLI="$HOME/Projects/nvme-cli/build/nvme"

IN_DIR="$HOME/Projects/fuzz_nvme_cli/inputs"
OUT_DIR="$HOME/Projects/fuzz_nvme_cli/outputs"

$AFL_FUZZ -i "$IN_DIR" -o "$OUT_DIR" -Q -- "$NVME_CLI" @@
```
7. Configure `core_pattern` to ensure core dump notifications are not sent to an external utility:
	```text
	[-] Hmm, your system is configured to send core dump notifications to an
		external utility. This will cause issues: there will be an extended delay
		between stumbling upon a crash and having this information relayed to the
		fuzzer via the standard waitpid() API.
		If you're just testing, set 'AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1'.

		To avoid having crashes misinterpreted as timeouts, please log in as root
		and temporarily modify /proc/sys/kernel/core_pattern, like so:

		echo core >/proc/sys/kernel/core_pattern

	[-] PROGRAM ABORT : Pipe at the beginning of 'core_pattern'
			Location : check_crash_handling(), src/afl-fuzz-init.c:2201
	```
	a. The original value of core_pattern is: `|/usr/share/apport/apport -p%p -s%s -c%c -d%d -P%P -u%u -g%g -- %E`
8. Modify cpu frequency scaling:
	```text
	[-] Whoops, your system uses on-demand CPU frequency scaling, adjusted
    between 781 and 3515 MHz. Unfortunately, the scaling algorithm in the
    kernel is imperfect and can miss the short-lived processes spawned by
    afl-fuzz. To keep things moving, run these commands as root:

    cd /sys/devices/system/cpu
    echo performance | tee cpu*/cpufreq/scaling_governor

    You can later go back to the original state by replacing 'performance'
    with 'ondemand' or 'powersave'. If you don't want to change the settings,
    set AFL_SKIP_CPUFREQ to make afl-fuzz skip this check - but expect some
    performance drop.

	[-] PROGRAM ABORT : Suboptimal CPU scaling governor
         Location : check_cpu_governor(), src/afl-fuzz-init.c:2310
	```
	a. The original value is `powersave`
9. Create an initial test case:
	```bash
	# -e enables escaped characters with backslash
	# -n disables automatic newlines
	echo -en "nvme\0x00" > 1.testcase
	```
10. AFLplusplus fails with a FATAL Error because nvme-cli dynamically loads several libraries. See [FAQ](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/FAQ.md).
11. The recommended solution from the FAQ is to use `strace` to identify the libraries that it loads and add them to the `AFL_PRELOAD` environment variable:
	```bash
	NVME_CLI_BUILD_DIR="$HOME/Projects/nvme-cli/build"
	AFL_PRELOAD="$NVME_CLI_BUILD_DIR/subprojects/libnvme/src/libnvme.so.1"
	AFL_PRELOAD="$AFL_PRELOAD:$NVME_CLI_BUILD_DIR/subprojects/libnvme/src/libnvme-mi.so.1"
	AFL_PRELOAD="$AFL_PRELOAD:/lib/x86_64-linux-gnu/libuuid.so.1"
	AFL_PRELOAD="$AFL_PRELOAD:/lib/x86_64-linux-gnu/libjson-c.so"
	AFL_PRELOAD="$AFL_PRELOAD:/lib/x86_64-linux-gnu/libz.so.1"
	AFL_PRELOAD="$AFL_PRELOAD:/lib/x86_64-linux-gnu/libc.so.6"
	AFL_PRELOAD="$AFL_PRELOAD:$NVME_CLI_BUILD_DIR/subprojects/libnvme/src/../../openssl-1.1.1l/libcrypto.so"
	```
12. However, this did not solve the problem. Perhaps I missed a library?
13. I ended up setting `AFL_IGNORE_PROBLEMS=1` to see if fuzzing worked despite this.
