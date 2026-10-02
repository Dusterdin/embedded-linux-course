# Lab 02 report

- Student name: Nazar Humenuk 
- GitHub username: dusterdin

Replace each placeholder with your own command output. Run HOST build commands
from this lab directory and BBB module commands from `/home/debian/labs/lab02`.
Keep the complete `vermagic`, not only its release prefix.

## Prerequisites and module metadata

### BBB: `uname -r`

```text
6.12.96-bone64
```

### HOST: `cat ~/bbb-workspace/kernel/bb-kernel/KERNEL/include/config/kernel.release`

```text
6.12.96-bone64
```

### HOST: `file lab02_hello.ko`

```text
lab02_hello.ko: ELF 32-bit LSB relocatable, ARM, EABI5 version 1 (SYSV), BuildID[sha1]=771e59fc38442847816baa3ddd9050f604c9f650, not stripped
```

### HOST: `modinfo lab02_hello.ko | grep vermagic`

```text
vermagic: 6.12.96-bone64 preempt mod_unload ARMv7 thumb2 p2v8
```

Explain why the full kernel releases must match and whether yours match:

 The kernel version must match because a module is compiled against the exact data structures and symbol versions of one specific kernel build; a mismatch can cause insmod to reject the module or crash the kernel. In our case they match: both are 6.12.96-bone64.

## Load, inspect, and unload on BBB

### `sudo insmod ./lab02_hello.ko name=Student` and `sudo dmesg | tail -20`

```text
$ sudo insmod ./lab02_hello.ko name=Student
$ echo $?
0
$ sudo dmesg | tail -20
[ 1336.008359] lab02_hello: loading out-of-tree module taints kernel.
[ 1336.016318] hello_module: Hello, Student, from kernel space on BBB!
```

### `lsmod | grep lab02_hello`

```text
lab02_hello            12288  0
```

### `sudo rmmod lab02_hello` and `sudo dmesg | tail -20`

```text
$ sudo rmmod lab02_hello
$ sudo dmesg | tail -20
[ 1723.847902] hello_module: Goodbye from kernel space on BBB!
```

## Parameter tests on BBB

After implementing the student tasks, rebuild, check metadata, and copy the
module again. Unload after every successful load before testing another case.
For each case record the command, result, and relevant `dmesg` output. Capture
`echo $?` immediately after `insmod` to record its exit status.

### Valid name: `name=YourName`

```text
$ sudo insmod ./lab02_hello.ko name=Nazar_Humenuk
$ echo $?
0
$ sudo dmesg | tail -5
[ 5197.381956] hello_module: Hello, Nazar_Humenuk, from kernel space on BBB! (1/1)
$ sudo rmmod lab02_hello
```

### Invalid empty name: `name=` (must return `-EINVAL`)

```text
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=
echo $?
insmod: ERROR: could not insert module ./lab02_hello.ko: Invalid parameters
1
```

### Valid count: test `count=1` and `count=10`

```text
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=Naz count=1
debian@BeagleBone:~/labs/lab02$ echo $?
0
debian@BeagleBone:~/labs/lab02$ sudo rmmod lab02_hello
debian@BeagleBone:~/labs/lab02$ sudo insmod ./lab02_hello.ko name=Naz count=10
debian@BeagleBone:~/labs/lab02$ echo $?
0
debian@BeagleBone:~/labs/lab02$ sudo dmesg | tail -20
[  381.588722] hello_module: Hello, Naz, from kernel space on BBB! (1/1)
[  433.551672] hello_module: Goodbye from kernel space on BBB!
[  437.259401] hello_module: Hello, Naz, from kernel space on BBB! (1/10)
[  437.266187] hello_module: Hello, Naz, from kernel space on BBB! (2/10)
[  437.275340] hello_module: Hello, Naz, from kernel space on BBB! (3/10)
[  437.282746] hello_module: Hello, Naz, from kernel space on BBB! (4/10)
[  437.290642] hello_module: Hello, Naz, from kernel space on BBB! (5/10)
[  437.297926] hello_module: Hello, Naz, from kernel space on BBB! (6/10)
[  437.305029] hello_module: Hello, Naz, from kernel space on BBB! (7/10)
[  437.312769] hello_module: Hello, Naz, from kernel space on BBB! (8/10)
[  437.320043] hello_module: Hello, Naz, from kernel space on BBB! (9/10)
[  437.327170] hello_module: Hello, Naz, from kernel space on BBB! (10/10)
```

### Invalid count: test `count=0` and `count=11` (must return `-EINVAL`)

```text
$ sudo insmod ./lab02_hello.ko name=Naz count=0
echo $?
insmod: ERROR: could not insert module ./lab02_hello.ko: Invalid parameters
1
$ sudo insmod ./lab02_hello.ko name=Naz count=11
echo $?
insmod: ERROR: could not insert module ./lab02_hello.ko: Invalid parameters
1
```

## Questions

1. Why does a kernel module not have `main()`?

   A kernel module is not a standalone program run once — it is code
   loaded into an already-running kernel. Instead of a single entry
   point called at startup like `main()`, it registers callback
   functions (here `hello_init` and `hello_exit`) that the kernel
   itself calls at the moments the module is loaded (`insmod`) and
   unloaded (`rmmod`).

2. What does `module_init()` do?

   module_init() tells the kernel which function to run as the module's entry point. For a loadable module (.ko), it is called when insmod inserts the module. For a module built directly into the kernel, the same macro instead registers the function pointer in the kernel's .initcall section, so it gets called automatically during boot, in the standard initcall sequence, without any explicit insmod step.

3. What does `vermagic` show?

   It shows the exact kernel release the module was compiled against,
   together with build-relevant configuration flags (preemption model,
   module-unload support, CPU architecture variant, instruction set,
   page offset). The kernel compares this string against its own
   running configuration before allowing the module to load.

4. Why can `insmod` fail with `invalid module format`?

   This happens when the module's `vermagic` (or other ABI-relevant
   build settings) does not match the currently running kernel — for
   example, if the module was compiled against a different kernel
   version, commit, or configuration than the one actually running on
   the board.

5. What does the `name` parameter do, and what do its permissions `0444` mean?

   `name` is a module parameter that can be set when the module is
   loaded (`insmod lab02_hello.ko name=...`) and is used once inside
   `hello_init()` to print a greeting. Its permissions `0444` make the
   corresponding file in `/sys/module/lab02_hello/parameters/name`
   read-only for everyone (owner, group, others): the current value can
   be inspected after loading, but it cannot be changed at runtime,
   since it is only read once during initialization.

Submit `src/hello.c`, `Makefile`, and this completed `report.md`. Do not commit
`.ko`, `.o`, `.mod`, `.mod.c`, `.cmd`, `Module.symvers`, or `modules.order`.
