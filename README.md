# bc, but only for building the Linux Kernel

The Linux kernel requires a the GNU `bc` tool to be able to compile. From what
I can see, the only thing it actually needs this for is to generate the file
`include/generated/timeconst.h` according to the script in
`kernel/time/timeconst.bc`. (`bc` is used for tests, however.)

This is a problem for bootstrapping a Linux kernel, because it requires yet
another tool that you have to build, using other tools, ad nauseum, and for
such a trivial task!

I had the clever idea to write a fake `bc` that does nothing other than generate
this file, thereby trimming down the dependencies needed to build the Linux
kernel by one.

This tool builds with:

- glibc
- (probably) musl libc
- nolibc (Part of the Linux kernel code in `tools/include/nolibc`)
- no standard library at all

## Building

### glibc build

```bash
gcc -o bc ./bc4linux.c 
```

### nolibc build

```bash
gcc -o bc -static -nostdlib -include PATH_TO_LINUX/tools/include/nolibc/nolibc.h ./bc4linux.c
```

### No standard library build

Warning: this only works on Linux on x86-64.

```bash
gcc -o bc -DNO_STD_LIB ./bc4linux.c
```

## Using

Pipe a decimal number (the value of `CONFIG_HZ`) into the command like so:

```bash
echo '250' | ./bc
```

Any command line arguments are just ignored.

## Output

The output of the real `bc` looks like:

```text
#ifndef KERNEL_TIMECONST_H
#define KERNEL_TIMECONST_H

#include <linux/param.h>
#include <linux/types.h>

#if HZ != 250
#error "include/generated/timeconst.h has the wrong HZ value!"
#endif

#define HZ_TO_MSEC_MUL32        U64_C(0x80000000)
#define HZ_TO_MSEC_ADJ32        U64_C(0x0)
#define HZ_TO_MSEC_SHR32        29
#define MSEC_TO_HZ_MUL32        U64_C(0x80000000)
#define MSEC_TO_HZ_ADJ32        U64_C(0x180000000)
#define MSEC_TO_HZ_SHR32        33
#define HZ_TO_MSEC_NUM          4
#define HZ_TO_MSEC_DEN          1
#define MSEC_TO_HZ_NUM          1
#define MSEC_TO_HZ_DEN          4

#define HZ_TO_USEC_MUL32        U64_C(0xFA000000)
#define HZ_TO_USEC_ADJ32        U64_C(0x0)
#define HZ_TO_USEC_SHR32        20
#define USEC_TO_HZ_MUL32        U64_C(0x83126E98)
#define USEC_TO_HZ_ADJ32        U64_C(0x7FF7CED9168)
#define USEC_TO_HZ_SHR32        43
#define HZ_TO_USEC_NUM          4000
#define HZ_TO_USEC_DEN          1
#define USEC_TO_HZ_NUM          1
#define USEC_TO_HZ_DEN          4000
#define HZ_TO_NSEC_NUM          4000000
#define HZ_TO_NSEC_DEN          1
#define NSEC_TO_HZ_NUM          1
#define NSEC_TO_HZ_DEN          4000000
```

And the output of this command looks like:

```text
#ifndef KERNEL_TIMECONST_H
#define KERNEL_TIMECONST_H

#include <linux/param.h>
#include <linux/types.h>

#define HZ_TO_MSEC_MUL32        U64_C(2147483648)
#define HZ_TO_MSEC_ADJ32        U64_C(0)
#define HZ_TO_MSEC_SHR32        29
#define MSEC_TO_HZ_MUL32        U64_C(2147483648)
#define MSEC_TO_HZ_ADJ32        U64_C(6442450944)
#define MSEC_TO_HZ_SHR32        33
#define HZ_TO_MSEC_NUM          4
#define HZ_TO_MSEC_DEN          1
#define MSEC_TO_HZ_NUM          1
#define MSEC_TO_HZ_DEN          4

#define HZ_TO_USEC_MUL32        U64_C(4194304000)
#define HZ_TO_USEC_ADJ32        U64_C(0)
#define HZ_TO_USEC_SHR32        20
#define USEC_TO_HZ_MUL32        U64_C(2199023256)
#define USEC_TO_HZ_ADJ32        U64_C(8793893998952)
#define USEC_TO_HZ_SHR32        43
#define HZ_TO_USEC_NUM          4000
#define HZ_TO_USEC_DEN          1
#define USEC_TO_HZ_NUM          1
#define USEC_TO_HZ_DEN          4000
#define HZ_TO_NSEC_NUM          4000000
#define HZ_TO_NSEC_DEN          1
#define NSEC_TO_HZ_NUM          1
#define NSEC_TO_HZ_DEN          4000000
```

You might notice that it outputs number in base-10 / decimal where the original
outputs them in hexadecimal. I don't think that should matter. All the numbers
match up.

## Quality

I am not sure if this will work for all numbers. I don't know what to expect
for values of `CONFIG_HZ`, and I really don't understand what this file is doing
to be honest. But the logic was fairly easy to port to C, at least. This is to
say: I DO NOT GUARANTEE THIS TO BE BUG FREE!
