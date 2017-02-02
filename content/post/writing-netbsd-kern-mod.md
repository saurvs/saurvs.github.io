+++
date = "2017-02-02"
title = "Writing a NetBSD kernel module"
+++

Kernel modules are object files used to extend an operating system's
kernel functionality at *run time*.

In this post, we’ll look at implementing a simple character device driver as a
kernel module in NetBSD.
Once it is loaded, userspace processes will be able to `write` an arbitrary byte string to
the device,
and on every successive `read` expect a cryptographically-secure pseudorandom permutation of
the original byte string.

Before we begin, compiling a kernel module requires the NetBSD source code to live in `/usr/src`.
[This](https://www.netbsd.org/docs/guide/en/chap-fetch.html) explains how to
get that.

Usually, most userspace interfaces to character or block devices are through
special files that live in `/dev`. We'll create one such special file through
the command

```C
$ mknod /dev/rperm c 420 0
```

The `c` indicates that this file is an interface
to a character device, `420` indicates this device's *major number*, and `0`
indicates this device's *minor number*. The major number is used by the kernel
to uniquely identify each device, and the minor number is usually used
internally by device drivers but we won't be bothering with it.

Our device driver will specifically implement the `open`, `read`, `write`, and `close` I/O methods. To
register our implementations of those methods with the kernel, we first
prototype them in way that makes the compiler happy using the
`dev_type_*` set of macros, and then put them into a `struct cdevsw`.

```C
dev_type_open(rperm_open);
dev_type_close(rperm_close);
dev_type_write(rperm_write);
dev_type_read(rperm_read);

static struct cdevsw rperm_cdevsw = {
    .d_open = rperm_open,
    .d_close = rperm_close,
    .d_read = rperm_read,
    .d_write = rperm_write,

    .d_ioctl = noioctl,
    .d_stop = nostop,
    .d_tty = notty,
    .d_poll = nopoll,
    .d_mmap = nommap,
    .d_kqfilter = nokqfilter,
    .d_discard = nodiscard,
    .d_flag = D_OTHER
};
```


As we can see, there are plenty of functions we won't be implementing.
`devsw` stands for *device switch*.

Every kernel module is required to define it's metadata through the C macro
`MODULE(class, name, required)`. Since our module is a device driver, named
`rperm`, and won't require another module being pre-loaded, we write

```C
MODULE(MODULE_CLASS_DRIVER, rperm, NULL);
```

Every module is also required to implement a `MODNAME_modcmd` function, which the kernel
calls to report important module-related events, like when the module loads
or unloads. This is where we'll register our `struct cdevsw`.

```C
#define CMAJOR 420

static int
rperm_modcmd(modcmd_t cmd, void *args)
{
    devmajor_t bmajor, cmajor;

    bmajor = -1;
    cmajor = CMAJOR;
    switch(cmd) {
        case MODULE_CMD_INIT:
            devsw_attach("rperm", NULL, &bmajor, &rperm_cdevsw, &cmajor);
	    break;
        case MODULE_CMD_FINI:
            devsw_detach(NULL, &rperm_cdevsw);
            break;
        default:
            break;
    }
    return 0;
}
```

The `NULL` argument to the `devsw_*` functions is for a block device switch structure,
which we aren't bothered with. Similarly for `bmajor`, but the kernel ends up
assigning an unused block device number for our driver anyway.

Now we turn to actually implementing the four device I/O methods.

On every `write`, we need to store the byte string somewhere. We use a static
structure for that.

```C
static struct rperm_softc {
    char *buf;
    int buf_len;
} sc;
```

`sc.buf` will end up pointing to a location in the kernel's heap that contains
the byte string. `sc` and `softc` stand for software context,
which is just a convention followed in the NetBSD kernel for naming static structures in
device driver code.

`open` is a *required* implementation, as it is always the first
syscall in Unix I/O. But, there is nothing meaningful for us to do there, so we
simply write a stub.

```C
int
rperm_open(dev_t self, int flag, int mod, struct lwp *l)
{
    return 0;
}
```

In `write`, we allocate enough memory in the kernel's heap to store the
byte string, and then transfer the byte string from userspace to kernelspace.

```C
int
rperm_write(dev_t self, struct uio *uio, int flags)
{
    if (sc.buf)
	kmem_free(sc.buf, sc.buf_len);
    sc.buf_len = uio->uio_iov->iov_len;
    sc.buf = (char *)kmem_alloc(sc.buf_len, KM_SLEEP);
    uiomove(sc.buf, sc.buf_len, uio);
    return 0;
}
```

First, let's discuss the allocations.

`kmem_alloc` is similar to userspace `malloc`,
in that it allocates some number of bytes of memory in the heap. Interestingly, this memory
is *wired*, which means that during physical memory pressure, it is *not* paged
out to a swap disk like userspace memory is. The `KM_SLEEP` flag to `kmem_alloc` tells the kernel that
the current kernel thread should sleep until enough physical memory is avaiable for the request, if it
already isn't, as opposed to `kmem_alloc` simply returning `NULL` in such a situation. Hence,
our allocation request never fails, and we don't have to test for `sc.buf == NULL`.

`kmem_free` is similar to userspace `free`, except for a second argument that has to be the number of bytes allocated using `kmem_alloc`.

Next, we come to the transfer of the byte string from userspace to kernelspace.
Generally, memory to be transfered, in either direction, comes in one or more
non-contiguous chunks of memory (think scatter-gather I/O) along with some additional
state variables like the amount of data remaining to be transfered in the current
session, an offset into a block device, and some flags. All that information
is encapulated in a `struct uio` data type. And `uiomove` performs the actual transfer
by using that information. For example, here `uio->uio_rw` is set to `UIO_WRITE`,
telling `uiomove` that data
from `uio` should be transfered to `sc.buf`. `uiomove` also ends up updating
`uio->uio_resid`, which is the total number of bytes left to transfer to `uio`.

Next we come to `read`.

```C
int
rperm_read(dev_t self, struct uio *uio, int flags)
{
    if (sc.buf == NULL || uio->uio_resid < sc.buf_len)
	return EINVAL;

    char c;
    uint32_t i, n, r;

    for (i = 0; i < sc.buf_len-1; i++) {
	r = rand_n(i, sc.buf_len);
	c = sc.buf[r];
	sc.buf[r] = sc.buf[i];
	sc.buf[i] = c;
    }
    uiomove(sc.buf, sc.buf_len, uio);
    return 0;
}
```

We first check if there is enough space in `uio` to transfer a permuted byte
string, then use the [Fisher–Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)
to permute the original byte string using a random number generated by `rand_n`,
and then copy the permuted string back to userspace. In this case, `uio->uio_rw`
would be set to `UIO_READ`, telling `uiomove` that data
from `sc.buf` should be transfered to `uio`.

The `rand_n` function, which we need to implement, returns a random integer `n` uniformly distributed
over the range `[low, high)`.

```C
#define R32MAX 4294967295

uint32_t rand_n(uint32_t low, uint32_t high) {
    uint32_t limit, diff, r;

    diff = high - low;
    limit = diff * (R32MAX/diff);
    do {
	r = cprng_strong32();
    } while (r > limit);
    return (r % diff) + low;
}
```

For a source of randomness, we use `cprng_strong32`. The `cprng_*` family of
functions supply cryptographically secure pseudorandom
bytes (in this case, `4`) to callers within the kernel.

Once we have it, we transform the range of our random number from `[0, 2^32)` to
`[low, high)` by an iterative test that discards those values of `r` that are
larger than the largest multiple of `r` less than `2^32`, as using those numbers
would result in numbers in a certain subrange of `[low, high)` being *more*
likely to occur than those not.

In `close`, we free `sc.buf` if it was allocated before.

```C
int
rperm_close(dev_t self, int flag, int mod, struct lwp *l)
{
    if (sc.buf != NULL) {
	kmem_free(sc.buf, sc.buf_len);
	sc.buf = NULL;
    }
    return 0;
}
```

Lastly, we write a three line `Makefile` to build our module.

```Makefile
KMOD=   rperm
SRCS=   rperm.c

.include <bsd.kmodule.mk>
```

Now we compile and load the module.

```
$ make
$ modload ./rperm.kmod
```

The `./` *has* to be present for `modload`.

Unfortunately, we can't simply do

```
$ echo 'bloop' > /dev/rperm
$ cat /dev/rperm
```

as that would end up copying the `\0` along with the rest of the string into the driver, and the driver
would end up shuffling the `\0` as well, which we don't want.

So we settle for a simple test program.

```C
#define BUF_LEN  80
#define N_ITER   20

int main() {
    char buf[BUF_LEN] = "Hello NetBSD!";
    int i, fd, str_len;

    str_len = strlen(buf);
    fd = open("/dev/rperm", O_RDWR);
    write(fd, buf, str_len);
    for (i = 0; i < N_ITER; i++) {
	read(fd, buf, BUF_LEN);
	printf("%s\n", buf);
    }
    close(fd);
    return 0;
}
```

```C
$ ./test
NDe eBlS!tHol
Hl!elDSNeo Bt
N!llD eSoHetB
...
tHelS!BlNeo D
DBeSlt!HloNe
llSetB!oeNHD
```

All of the above code is available in it's entirety on [github](https://github.com/saurvs/rperm-netbsd).
More examples of kernel modules can be found in the NetBSD source tree
at `src/sys/modules/examples/`.
