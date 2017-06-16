

从用户态是如何陷入到内核态的呢？

以下面这个man bind的示例开始。

```java
#include <sys/socket.h>
#include <sys/un.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#define MY_SOCK_PATH "/somepath"
#define LISTEN_BACKLOG 50

#define handle_error(msg) \
    do { perror(msg); exit(EXIT_FAILURE); } while (0)

int
main(int argc, char *argv[])
{
    int sfd, cfd;
    struct sockaddr_un my_addr, peer_addr;
    socklen_t peer_addr_size;

    sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sfd == -1)
        handle_error("socket");

    memset(&my_addr, 0, sizeof(struct sockaddr_un));
                        /* Clear structure */
    my_addr.sun_family = AF_UNIX;
    strncpy(my_addr.sun_path, MY_SOCK_PATH,
            sizeof(my_addr.sun_path) - 1);

    if (bind(sfd, (struct sockaddr *) &my_addr,
            sizeof(struct sockaddr_un)) == -1)
        handle_error("bind");

   if (listen(sfd, LISTEN_BACKLOG) == -1)
       handle_error("listen");
```


用户态调用了bind函数。其声明在`<sys/socket.h>`。

```java
/* Give the socket FD the local address ADDR (which is LEN bytes long).  */
extern int bind (int __fd, __CONST_SOCKADDR_ARG __addr, socklen_t __len)
     __THROW;
```

其定义在`glibc-2.23/sysdeps/unix/sysv/linux/bind.c`。

```java
int
__bind (int fd, __CONST_SOCKADDR_ARG addr, socklen_t len)
{
#ifdef __ASSUME_BIND_SYSCALL
  return INLINE_SYSCALL (bind, 3, fd, addr.__sockaddr__, len);
#else
  return SOCKETCALL (bind, fd, addr.__sockaddr__, len, 0, 0, 0);
#endif
}
weak_alias (__bind, bind)
```

从4.3版本内核开始，bind会直接走socketcalls。所以上述bind走INLINE_SYSCALL。

```java
/* Direct socketcalls available with kernel 4.3.  */
#if __LINUX_KERNEL_VERSION >= 0x040300
# define __ASSUME_RECVMMSG_SYSCALL           1
# define __ASSUME_SENDMMSG_SYSCALL           1
# define __ASSUME_SOCKET_SYSCALL             1
# define __ASSUME_SOCKETPAIR_SYSCALL         1
# define __ASSUME_BIND_SYSCALL               1
```

INLINE_SYSCALL定义在`glibc-2.23/sysdeps/unix/sysv/linux/x86_64/sysdep.h`

```assembly
# define INTERNAL_SYSCALL_NCS(name, err, nr, args...) \
  ({									      \
    unsigned long int resultvar;					      \
    LOAD_ARGS_##nr (args)						      \
    LOAD_REGS_##nr							      \
    asm volatile (							      \
    "syscall\n\t"							      \
    : "=a" (resultvar)							      \
    : "0" (name) ASM_ARGS_##nr : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);   \
    (long int) resultvar; })
# undef INTERNAL_SYSCALL
# define INTERNAL_SYSCALL(name, err, nr, args...) \
  INTERNAL_SYSCALL_NCS (__NR_##name, err, nr, ##args)

# undef INLINE_SYSCALL
# define INLINE_SYSCALL(name, nr, args...) \
  ({									      \
    unsigned long int resultvar = INTERNAL_SYSCALL (name, , nr, args);	      \
    if (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (resultvar, )))	      \
      {									      \
	__set_errno (INTERNAL_SYSCALL_ERRNO (resultvar, ));		      \
	resultvar = (unsigned long int) -1;				      \
      }									      \
    (long int) resultvar; })
```


> 64-bit x86 uses syscall instead of interrupt 0x80. The result value will be in %rax


跟x86使用0x80中断去系统调用不一样，x86-64使用syscall。

!!TODO这里的syscall的汇编没有太看懂。

syscall的name为`__NR_##name`，在本例中即为`__NR_bind`。其定义在内核的`linux-4.1-rc2/include/uapi/asm-generic/unistd.h`。（内核从3.5版本开始，将各组件纯用户态的头文件抽出来放到uapi中。）

```java
/* net/socket.c */
#define __NR_socket 198
__SYSCALL(__NR_socket, sys_socket)
#define __NR_socketpair 199
__SYSCALL(__NR_socketpair, sys_socketpair)
#define __NR_bind 200
__SYSCALL(__NR_bind, sys_bind)
#define __NR_listen 201
__SYSCALL(__NR_listen, sys_listen)
#define __NR_accept 202
__SYSCALL(__NR_accept, sys_accept)
#define __NR_connect 203
__SYSCALL(__NR_connect, sys_connect)
```

内核通过__NR_bind知道本次syscall的是谁。

至此，用户态是怎么到内核的流程梳理完了。下面再来看看内核是怎么整的。

---



x86-64架构的内核syscall的入口在`linux-4.1-rc2/arch/x86/kernel/entry_64.S`。



```java

/*
 * 64bit SYSCALL instruction entry. Up to 6 arguments in registers.
 *
 * 64bit SYSCALL saves rip to rcx, clears rflags.RF, then saves rflags to r11,
 * then loads new ss, cs, and rip from previously programmed MSRs.
 * rflags gets masked by a value from another MSR (so CLD and CLAC
 * are not needed). SYSCALL does not save anything on the stack
 * and does not change rsp.
 *
 * Registers on entry:
 * rax  system call number
 * rcx  return address
 * r11  saved rflags (note: r11 is callee-clobbered register in C ABI)
 * rdi  arg0
 * rsi  arg1
 * rdx  arg2
 * r10  arg3 (needs to be moved to rcx to conform to C ABI)
 * r8   arg4
 * r9   arg5
 * (note: r12-r15,rbp,rbx are callee-preserved in C ABI)
 *
 * Only called from user space.
 *
 * When user can change pt_regs->foo always force IRET. That is because
 * it deals with uncanonical addresses better. SYSRET has trouble
 * with them due to bugs in both AMD and Intel CPUs.
 */

ENTRY(system_call)
..
    movq %r10,%rcx
    call *sys_call_table(,%rax,8)
    movq %rax,RAX(%rsp)

```

rax中存的就是这次syscall的num，即__NR_bind。

sys_call_table在`linux-4.1-rc2/arch/x86/kernel/syscall_64.c`定义。

```java
extern void sys_ni_syscall(void);

asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	/*
	 * Smells like a compiler bug -- it doesn't work
	 * when the & below is removed.
	 */
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
#include <asm/syscalls_64.h>
};
```

TODO：但什么时候，将，syscall number对应上sys_xxx的呢？

sys_ni_syscall在`linux-4.1-rc2/kernel/sys_ni.c`定义。sys_bind等符号也是在这个文件里定义的。


```
asmlinkage long sys_ni_syscall(void)
{
	return -ENOSYS;
}
..
cond_syscall(sys_socketpair);
cond_syscall(sys_bind);
cond_syscall(sys_listen);
cond_syscall(sys_accept);
cond_syscall(sys_accept4);
cond_syscall(sys_connect);

```

cond_syscall在`linux-4.1-rc2/include/linux/linkage.h`中定义。

```c
#ifndef cond_syscall
#define cond_syscall(x)	asm(				\
	".weak " VMLINUX_SYMBOL_STR(x) "\n\t"		\
	".set  " VMLINUX_SYMBOL_STR(x) ","		\
		 VMLINUX_SYMBOL_STR(sys_ni_syscall))
#endif
```

cond_syscall中的`.weak func() .set func_backup()` 的意思是，如果func()不存在，则调用func_backup。具体到bind，即表示如果sys_bind不存在，则调用sys_ni_syscall。

这里会导出符号。但不知道什么时候对应上的呢？

回来看前面提到的`linux-4.1-rc2/include/uapi/asm-generic/unistd.h`

```java
/* net/socket.c */
#define __NR_socket 198
__SYSCALL(__NR_socket, sys_socket)
#define __NR_socketpair 199
__SYSCALL(__NR_socketpair, sys_socketpair)
#define __NR_bind 200
__SYSCALL(__NR_bind, sys_bind)
#define __NR_listen 201
__SYSCALL(__NR_listen, sys_listen)
#define __NR_accept 202
__SYSCALL(__NR_accept, sys_accept)
#define __NR_connect 203
__SYSCALL(__NR_connect, sys_connect)
```

TODO：感觉是在这里定义的，但__SYSCALL是空的呀。

以bind为例。代码在`net/socket.c`中。

```java
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err, fput_needed;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
		err = move_addr_to_kernel(umyaddr, addrlen, &address);
		if (err >= 0) {
			err = security_socket_bind(sock,
						   (struct sockaddr *)&address,
						   addrlen);
			if (!err)
				err = sock->ops->bind(sock,
						      (struct sockaddr *)
						      &address, addrlen);
		}
		fput_light(sock->file, fput_needed);
	}
	return err;
}
```

SYSCALL_DEFINE3是怎么展开成```sys_bind(int fd, struct sockaddr __user * umyaddr, int addrlen)```的呢？

```java
/*
 * __MAP - apply a macro to syscall arguments
 * __MAP(n, m, t1, a1, t2, a2, ..., tn, an) will expand to
 *    m(t1, a1), m(t2, a2), ..., m(tn, an)
 */
#define __MAP0(m,...)
#define __MAP1(m,t,a) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
..
#define __MAP(n,...) __MAP##n(__VA_ARGS__)

#define __SC_DECL(t, a)	t a

#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
#define __SYSCALL_DEFINEx(x, name, ...)					\
	asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))	\
		__attribute__((alias(__stringify(SyS##name))));		\
	static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));	\
	asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
	asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));	\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
		return ret;						\
	}								\
	static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

这步还挺复杂的。简单来说就是，sys_bind是SyS_bind的别名，在SyS_bind里调用SYSC_bind。所以，将上述代码展开以后，在内核里真正定义的函数是SYSC_Bind。
据说之所以这么做是为了避免64位CPU在接受32位值时，可能有安全问题。用long型字段来接受就可以保证。（待补充）

再往下就是linux协议栈里bind的实现了。

---
