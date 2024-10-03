
作者：[heaven](http://www.wowotech.net/author/532) 发布于：2019-12-27 16:53 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

我们继续上一节的内容往下分析代码


【硬件环境】         Imx6ul
【Linux kernel版本】   Linux4.1.15
【以太网phy】        Realtek8201f

## 1.1 Ifconfig的详细代码流程
ret_fast_syscall ===》这是返回系统调用的syscall，大家可以看注释，saving r0，back into the SVC stack
arch/arm/kernel/entry-common.S

```cpp
1. /*
2.  * This is the fast syscall return path.  We do as little as
3.  * possible here, and this includes saving r0 back into the SVC
4.  * stack.
5.  */
6. ret_fast_syscall:
7.  UNWIND(.fnstart	)
8.  UNWIND(.cantunwind	)
9. 	disable_irq				@ disable interrupts
10. 	ldr	r1, [tsk, #TI_FLAGS]		@ re-check for syscall tracing
11. 	tst	r1, #_TIF_SYSCALL_WORK
12. 	bne	__sys_trace_return
13. 	tst	r1, #_TIF_WORK_MASK
14. 	bne	fast_work_pending
15. 	asm_trace_hardirqs_on

17. 	/* perform architecture specific actions before user return */
18. 	arch_ret_to_user r1, lr
19. 	ct_user_enter

21. 	restore_user_regs fast = 1, offset = S_OFF
22.  UNWIND(.fnend		)
```
  
fs/ioctl.c
```cpp
1. SYSCALL_DEFINE3(ioctl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
2. {
3. 	int error;
4. 	struct fd f = fdget(fd);

6. 	if (!f.file)
7. 		return -EBADF;
8. 	error = security_file_ioctl(f.file, cmd, arg);
9. 	if (!error)
10. 		error = do_vfs_ioctl(f.file, fd, cmd, arg);
11. 	fdput(f);
12. 	return error;
13. }
```

```cpp
1. /*
2.  * When you add any new common ioctls to the switches above and below
3.  * please update compat_sys_ioctl() too.
4.  *
5.  * do_vfs_ioctl() is not for drivers and not intended to be EXPORT_SYMBOL()'d.
6.  * It's just a simple helper for sys_ioctl and compat_sys_ioctl.
7.  */
8. int do_vfs_ioctl(struct file *filp, unsigned int fd, unsigned int cmd,
9. 	     unsigned long arg)
10. {
11. 	int error = 0;
12. 	int __user *argp = (int __user *)arg;
13. 	struct inode *inode = file_inode(filp);

15. 	switch (cmd) {
16. 	case FIOCLEX:
17. 		set_close_on_exec(fd, 1);
18. 		break;

20. 	case FIONCLEX:
21. 		set_close_on_exec(fd, 0);
22. 		break;

24. 	case FIONBIO:
25. 		error = ioctl_fionbio(filp, argp);
26. 		break;

28. 	case FIOASYNC:
29. 		error = ioctl_fioasync(fd, filp, argp);
30. 		break;

32. 	case FIOQSIZE:
33. 		if (S_ISDIR(inode->i_mode) || S_ISREG(inode->i_mode) ||
34. 		    S_ISLNK(inode->i_mode)) {
35. 			loff_t res = inode_get_bytes(inode);
36. 			error = copy_to_user(argp, &res, sizeof(res)) ?
37. 					-EFAULT : 0;
38. 		} else
39. 			error = -ENOTTY;
40. 		break;

42. 	case FIFREEZE:
43. 		error = ioctl_fsfreeze(filp);
44. 		break;

46. 	case FITHAW:
47. 		error = ioctl_fsthaw(filp);
48. 		break;

50. 	case FS_IOC_FIEMAP:
51. 		return ioctl_fiemap(filp, arg);

53. 	case FIGETBSZ:
54. 		return put_user(inode->i_sb->s_blocksize, argp);

56. 	default:
57. 		if (S_ISREG(inode->i_mode))
58. 			error = file_ioctl(filp, cmd, arg);
59. 		else
60. 			error = vfs_ioctl(filp, cmd, arg);
61. 		break;
62. 	}
63. 	return error;
64. }
```

通过dump信息，我们知道是调用了vfs_ioctl，

继续看vfs_ioctl：
```cpp
1. /**
2.  * vfs_ioctl - call filesystem specific ioctl methods
3.  * @filp:	open file to invoke ioctl method on
4.  * @cmd:	ioctl command to execute
5.  * @arg:	command-specific argument for ioctl
6.  *
7.  * Invokes filesystem specific ->unlocked_ioctl, if one exists; otherwise
8.  * returns -ENOTTY.
9.  *
10.  * Returns 0 on success, -errno on error.
11.  */
12. static long vfs_ioctl(struct file *filp, unsigned int cmd,
13. 		      unsigned long arg)
14. {
15. 	int error = -ENOTTY;
1.     printk("zbh %s:%s(%d) file system name:%s \r\n", 
1.         __FILE__, __func__, __LINE__, filp->f_path.dentry->d_sb->s_type->name);
1. 	if (!filp->f_op->unlocked_ioctl)
2. 		goto out;

4. 	error = filp->f_op->unlocked_ioctl(filp, cmd, arg);
5. 	if (error == -ENOIOCTLCMD)
6. 		error = -ENOTTY;
7.  out:
8. 	return error;
9. }
```
  
打印的目的是告诉大家一个查看文件系统类型的方法。
这个是属于vfs下的sockfs文件系统
到了这里我们要找到unlocked_ioctl的回调函数是哪个

方法一：
![[Pasted image 20241003145826.png]] 
方法二：

因为我们在kernel的dump信息里面知道是调用了sock_ioctl，所以我们直接去找这个函数就好了，net/socket.c

```cpp
1. /*
2.  *	Socket files have a set of 'special' operations as well as the generic file ones. These don't appear
3.  *	in the operation structures but are done directly via the socketcall() multiplexor.
4.  */

6. static const struct file_operations socket_file_ops = {
7. 	.owner =	THIS_MODULE,
8. 	.llseek =	no_llseek,
9. 	.read_iter =	sock_read_iter,
10. 	.write_iter =	sock_write_iter,
11. 	.poll =		sock_poll,
12. 	.unlocked_ioctl = sock_ioctl,
13. #ifdef CONFIG_COMPAT
14. 	.compat_ioctl = compat_sock_ioctl,
15. #endif
16. 	.mmap =		sock_mmap,
17. 	.release =	sock_close,
18. 	.fasync =	sock_fasync,
19. 	.sendpage =	sock_sendpage,
20. 	.splice_write = generic_splice_sendpage,
21. 	.splice_read =	sock_splice_read,
22. };
```

注册是在如下地方注册的：
使用函数
1. sock_alloc_file
```cpp
1. /*
2.  *	Obtains the first available file descriptor and sets it up for use.
3.  *
4.  *	These functions create file structures and maps them to fd space
5.  *	of the current process. On success it returns file descriptor
6.  *	and file struct implicitly stored in sock->file.
7.  *	Note that another thread may close file descriptor before we return
8.  *	from this function. We use the fact that now we do not refer
9.  *	to socket after mapping. If one day we will need it, this
10.  *	function will increment ref. count on file by 1.
11.  *
12.  *	In any case returned fd MAY BE not valid!
13.  *	This race condition is unavoidable
14.  *	with shared fd spaces, we cannot solve it inside kernel,
15.  *	but we take care of internal coherence yet.
16.  */

18. struct file *sock_alloc_file(struct socket *sock, int flags, const char *dname)
19. {
20. 	struct qstr name = { .name = "" };
21. 	struct path path;
22. 	struct file *file;

24. 	if (dname) {
25. 		name.name = dname;
26. 		name.len = strlen(name.name);
27. 	} else if (sock->sk) {
28. 		name.name = sock->sk->sk_prot_creator->name;
29. 		name.len = strlen(name.name);
30. 	}
31. 	path.dentry = d_alloc_pseudo(sock_mnt->mnt_sb, &name);
32. 	if (unlikely(!path.dentry))
33. 		return ERR_PTR(-ENOMEM);
34. 	path.mnt = mntget(sock_mnt);

36. 	d_instantiate(path.dentry, SOCK_INODE(sock));

38. 	file = alloc_file(&path, FMODE_READ | FMODE_WRITE,
39. 		  &socket_file_ops);
40. 	if (unlikely(IS_ERR(file))) {
41. 		/* drop dentry, keep inode */
42. 		ihold(d_inode(path.dentry));
43. 		path_put(&path);
44. 		return file;
45. 	}

47. 	sock->file = file;
48. 	file->f_flags = O_RDWR | (flags & O_NONBLOCK);
49. 	file->private_data = sock;
50. 	return file;
51. }
52. EXPORT_SYMBOL(sock_alloc_file);
```
从socket到下的流程是这样
系统调用socket
```cpp
1. SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
2. {
3. 	int retval;
4. 	struct socket *sock;
5. 	int flags;

7. 	/* Check the SOCK_* constants for consistency.  */
8. 	BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
9. 	BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
10. 	BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
11. 	BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);

13. 	flags = type & ~SOCK_TYPE_MASK;
14. 	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
15. 		return -EINVAL;
16. 	type &= SOCK_TYPE_MASK;

18. 	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
19. 		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

21. 	retval = sock_create(family, type, protocol, &sock);
22. 	if (retval < 0)
23. 		goto out;

25. 	retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
26. 	if (retval < 0)
27. 		goto out_release;

29. out:
30. 	/* It may be already another descriptor 8) Not kernel problem. */
31. 	return retval;

33. out_release:
34. 	sock_release(sock);
35. 	return retval;
36. }

1. 在sock_map_fd中调用了sock_alloc_file

4. 1. static int sock_map_fd(struct socket *sock, int flags)
5. {
6. 	struct file *newfile;
7. 	int fd = get_unused_fd_flags(flags);
8. 	if (unlikely(fd < 0))
9. 		return fd;
11. 	newfile = sock_alloc_file(sock, flags, NULL);
12. 	if (likely(!IS_ERR(newfile))) {
13. 		fd_install(fd, newfile);
14. 		return fd;
15. 	}

17. 	put_unused_fd(fd);
18. 	return PTR_ERR(newfile);
19. }
```    

看下sock_ioctl代码：
```cpp
1. /*
2.  *	With an ioctl, arg may well be a user mode pointer, but we don't know
3.  *	what to do with it - that's up to the protocol still.
4.  */

6. static long sock_ioctl(struct file *file, unsigned cmd, unsigned long arg)
7. {
8. 	struct socket *sock;
9. 	struct sock *sk;
10. 	void __user *argp = (void __user *)arg;
11. 	int pid, err;
12. 	struct net *net;

14. 	sock = file->private_data;
15. 	sk = sock->sk;
16. 	net = sock_net(sk);
17. 	if (cmd >= SIOCDEVPRIVATE && cmd <= (SIOCDEVPRIVATE + 15)) {
18. 		err = dev_ioctl(net, cmd, argp);
19. 	} else
20. #ifdef CONFIG_WEXT_CORE
21. 	if (cmd >= SIOCIWFIRST && cmd <= SIOCIWLAST) {
22. 		err = dev_ioctl(net, cmd, argp);
23. 	} else
24. #endif
25. 		switch (cmd) {
26. 		case FIOSETOWN:
27. 		case SIOCSPGRP:
28. 			err = -EFAULT;
29. 			if (get_user(pid, (int __user *)argp))
30. 				break;
31. 			f_setown(sock->file, pid, 1);
32. 			err = 0;
33. 			break;
34. 		case FIOGETOWN:
35. 		case SIOCGPGRP:
36. 			err = put_user(f_getown(sock->file),
37. 				       (int __user *)argp);
38. 			break;
39. 		case SIOCGIFBR:
40. 		case SIOCSIFBR:
41. 		case SIOCBRADDBR:
42. 		case SIOCBRDELBR:
43. 			err = -ENOPKG;
44. 			if (!br_ioctl_hook)
45. 				request_module("bridge");

47. 			mutex_lock(&br_ioctl_mutex);
48. 			if (br_ioctl_hook)
49. 				err = br_ioctl_hook(net, cmd, argp);
50. 			mutex_unlock(&br_ioctl_mutex);
51. 			break;
52. 		case SIOCGIFVLAN:
53. 		case SIOCSIFVLAN:
54. 			err = -ENOPKG;
55. 			if (!vlan_ioctl_hook)
56. 				request_module("8021q");

58. 			mutex_lock(&vlan_ioctl_mutex);
59. 			if (vlan_ioctl_hook)
60. 				err = vlan_ioctl_hook(net, argp);
61. 			mutex_unlock(&vlan_ioctl_mutex);
62. 			break;
63. 		case SIOCADDDLCI:
64. 		case SIOCDELDLCI:
65. 			err = -ENOPKG;
66. 			if (!dlci_ioctl_hook)
67. 				request_module("dlci");

69. 			mutex_lock(&dlci_ioctl_mutex);
70. 			if (dlci_ioctl_hook)
71. 				err = dlci_ioctl_hook(cmd, argp);
72. 			mutex_unlock(&dlci_ioctl_mutex);
73. 			break;
74. 		default:
75. 			err = sock_do_ioctl(net, sock, cmd, arg);
76. 			break;
77. 		}
78. 	return err;
79. }
```

最后执行sock_do_ioctl
```cpp
1. static long sock_do_ioctl(struct net *net, struct socket *sock,
2. 				 unsigned int cmd, unsigned long arg)
3. {
4. 	int err;
5. 	void __user *argp = (void __user *)arg;

7. 	err = sock->ops->ioctl(sock, cmd, arg);

9. 	/*
10. 	 * If this ioctl is unknown try to hand it down
11. 	 * to the NIC driver.
12. 	 */
13. 	if (err == -ENOIOCTLCMD)
14. 		err = dev_ioctl(net, cmd, argp);

16. 	return err;
17. }
```
  
我们可以得出err是-19，
这里的sock->ops->ioctl回调的是inet_ioctl, 路径：net/ipv4/af_inet.c
```cpp
1. /*
2.  *	ioctl() calls you can issue on an INET socket. Most of these are
3.  *	device configuration and stuff and very rarely used. Some ioctls
4.  *	pass on to the socket itself.
5.  *
6.  *	NOTE: I like the idea of a module for the config stuff. ie ifconfig
7.  *	loads the devconfigure module does its configuring and unloads it.
8.  *	There's a good 20K of config code hanging around the kernel.
9.  */

11. int inet_ioctl(struct socket *sock, unsigned int cmd, unsigned long arg)
12. {
13. 	struct sock *sk = sock->sk;
14. 	int err = 0;
15. 	struct net *net = sock_net(sk);

17. 	switch (cmd) {
18. 	case SIOCGSTAMP:
19. 		err = sock_get_timestamp(sk, (struct timeval __user *)arg);
20. 		break;
21. 	case SIOCGSTAMPNS:
22. 		err = sock_get_timestampns(sk, (struct timespec __user *)arg);
23. 		break;
24. 	case SIOCADDRT:
25. 	case SIOCDELRT:
26. 	case SIOCRTMSG:
27. 		err = ip_rt_ioctl(net, cmd, (void __user *)arg);
28. 		break;
29. 	case SIOCDARP:
30. 	case SIOCGARP:
31. 	case SIOCSARP:
32. 		err = arp_ioctl(net, cmd, (void __user *)arg);
33. 		break;
34. 	case SIOCGIFADDR:
35. 	case SIOCSIFADDR:
36. 	case SIOCGIFBRDADDR:
37. 	case SIOCSIFBRDADDR:
38. 	case SIOCGIFNETMASK:
39. 	case SIOCSIFNETMASK:
40. 	case SIOCGIFDSTADDR:
41. 	case SIOCSIFDSTADDR:
42. 	case SIOCSIFPFLAGS:
43. 	case SIOCGIFPFLAGS:
44. 	case SIOCSIFFLAGS:
45. 		err = devinet_ioctl(net, cmd, (void __user *)arg);
46. 		break;
47. 	default:
48. 		if (sk->sk_prot->ioctl)
49. 			err = sk->sk_prot->ioctl(sk, cmd, arg);
50. 		else
51. 			err = -ENOIOCTLCMD;
52. 		break;
53. 	}
54. 	return err;
55. }
56. EXPORT_SYMBOL(inet_ioctl);
```
我们看到这个代码，和ifconfig出问题的那个宏SIOCSIFFLAGS一样
```cpp
case SIOCSIFFLAGS:  
err = devinet_ioctl(net, cmd, (void __user *)arg);  
break;
```

调到了devinet_ioctl，路径：net/ipv4/devinet.c
这个函数太长，我就黏贴部分代码
```cpp
1. int devinet_ioctl(struct net *net, unsigned int cmd, void __user *arg)
2. {
3. 	struct ifreq ifr;
4. 	struct sockaddr_in sin_orig;
5. 	struct sockaddr_in *sin = (struct sockaddr_in *)&ifr.ifr_addr;
6. 	struct in_device *in_dev;
7. 	struct in_ifaddr **ifap = NULL;
8. 	struct in_ifaddr *ifa = NULL;
9. 	struct net_device *dev;
10. ...

12. 	case SIOCSIFFLAGS:
13. 		ret = -EPERM;
14. 		if (!ns_capable(net->user_ns, CAP_NET_ADMIN))
15. 			goto out;
16. 		break;

18. ...

20. 	case SIOCSIFFLAGS:
21. 		if (colon) {
22. 			ret = -EADDRNOTAVAIL;
23. 			if (!ifa)
24. 				break;
25. 			ret = 0;
26. 			if (!(ifr.ifr_flags & IFF_UP))
27. 				inet_del_ifa(in_dev, ifap, 1);
28. 			break;
29. 		}
30. 		ret = dev_change_flags(dev, ifr.ifr_flags);
31. 		break;
```
  
继续跟踪dev_change_flags，路径：net/core/dev.c

```cpp
1. /**
2.  *	dev_change_flags - change device settings
3.  *	@dev: device
4.  *	@flags: device state flags
5.  *
6.  *	Change settings on device based state flags. The flags are
7.  *	in the userspace exported format.
8.  */
9. int dev_change_flags(struct net_device *dev, unsigned int flags)
10. {
11. 	int ret;
12. 	unsigned int changes, old_flags = dev->flags, old_gflags = dev->gflags;

14. 	ret = __dev_change_flags(dev, flags);
15. 	if (ret < 0)
16. 		return ret;

18. 	changes = (old_flags ^ dev->flags) | (old_gflags ^ dev->gflags);
19. 	__dev_notify_flags(dev, old_flags, changes);
20. 	return ret;
21. }
22. EXPORT_SYMBOL(dev_change_flags);

1. int __dev_change_flags(struct net_device *dev, unsigned int flags)
2. {
3. 	unsigned int old_flags = dev->flags;
4. 	int ret;

6. 	ASSERT_RTNL();

8. 	/*
9. 	 *	Set the flags on our device.
10. 	 */

12. 	dev->flags = (flags & (IFF_DEBUG | IFF_NOTRAILERS | IFF_NOARP |
13. 			       IFF_DYNAMIC | IFF_MULTICAST | IFF_PORTSEL |
14. 			       IFF_AUTOMEDIA)) |
15. 		     (dev->flags & (IFF_UP | IFF_VOLATILE | IFF_PROMISC |
16. 				    IFF_ALLMULTI));

18. 	/*
19. 	 *	Load in the correct multicast list now the flags have changed.
20. 	 */

22. 	if ((old_flags ^ flags) & IFF_MULTICAST)
23. 		dev_change_rx_flags(dev, IFF_MULTICAST);

25. 	dev_set_rx_mode(dev);

27. 	/*
28. 	 *	Have we downed the interface. We handle IFF_UP ourselves
29. 	 *	according to user attempts to set it, rather than blindly
30. 	 *	setting it.
31. 	 */

33. 	ret = 0;
34. 	if ((old_flags ^ flags) & IFF_UP)
35. 		ret = ((old_flags & IFF_UP) ? __dev_close : __dev_open)(dev);

37. 	if ((flags ^ dev->gflags) & IFF_PROMISC) {
38. 		int inc = (flags & IFF_PROMISC) ? 1 : -1;
39. 		unsigned int old_flags = dev->flags;

41. 		dev->gflags ^= IFF_PROMISC;

43. 		if (__dev_set_promiscuity(dev, inc, false) >= 0)
44. 			if (dev->flags != old_flags)
45. 				dev_set_rx_mode(dev);
46. 	}

48. 	/* NOTE: order of synchronization of IFF_PROMISC and IFF_ALLMULTI
49. 	   is important. Some (broken) drivers set IFF_PROMISC, when
50. 	   IFF_ALLMULTI is requested not asking us and not reporting.
51. 	 */
52. 	if ((flags ^ dev->gflags) & IFF_ALLMULTI) {
53. 		int inc = (flags & IFF_ALLMULTI) ? 1 : -1;

55. 		dev->gflags ^= IFF_ALLMULTI;
56. 		__dev_set_allmulti(dev, inc, false);
57. 	}

59. 	return ret;
60. }
```

我们看这里
```cpp
1. 	/*
2. 	 *	Have we downed the interface. We handle IFF_UP ourselves
3. 	 *	according to user attempts to set it, rather than blindly
4. 	 *	setting it.
5. 	 */

7. 	ret = 0;
8. 	if ((old_flags ^ flags) & IFF_UP)
9. 		ret = ((old_flags & IFF_UP) ? __dev_close : __dev_open)(dev);
```
  
到这里大家有印象了吧？__dev_open最终回调的是控制器驱动fec_main.c中的那个

fec_enet_open，大家还记得我们要分析什么吧？那个-19的错误就是这个open里面返回的

我们继续看这个最底层实现。

[http://stackoverflow.com/questions/5308090/set-ip-address-using-siocsifaddr-ioctl](https://community.nxp.com/external-link.jspa?url=http%3A%2F%2Fstackoverflow.com%2Fquestions%2F5308090%2Fset-ip-address-using-siocsifaddr-ioctl)

[http://www.ibm.com/support/knowledgecenter/ssw_aix_72/com.ibm.aix.commtrf2/ioctl_socket_control_operations.htm](https://community.nxp.com/external-link.jspa?url=http%3A%2F%2Fwww.ibm.com%2Fsupport%2Fknowledgecenter%2Fssw_aix_72%2Fcom.ibm.aix.commtrf2%2Fioctl_socket_control_operations.htm)

[https://lkml.org/lkml/2017/2/3/396](https://community.nxp.com/external-link.jspa?url=https%3A%2F%2Flkml.org%2Flkml%2F2017%2F2%2F3%2F396)

linux PHY驱动

[http://www.latelee.org/programming-under-linux/linux-phy-driver.html](http://www.latelee.org/programming-under-linux/linux-phy-driver.html)

Linux PHY几个状态的跟踪

[http://www.latelee.org/programming-under-linux/linux-phy-state.html](http://www.latelee.org/programming-under-linux/linux-phy-state.html)

第十六章PHY -基于Linux3.10

[https://blog.csdn.net/shichaog/article/details/44682931](https://blog.csdn.net/shichaog/article/details/44682931)

  

  

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [以太网驱动的流程浅析(四)-以太网驱动probe流程](http://www.wowotech.net/linux_kenrel/469.html) | [以太网驱动的流程浅析(三)-ifconfig的-19错误最底层分析](http://www.wowotech.net/linux_kenrel/465.html)»

**评论：**

**kangkang**  
2020-07-21 13:49

这分析的也太浅了。。根本没有发文章的必要，流水账可以记录到自己的博客里面

[回复](http://www.wowotech.net/linux_kenrel/466.html#comment-8058)

**btrace**  
2024-02-16 15:59

@kangkang：你来个深入的，否则就shutup

[回复](http://www.wowotech.net/linux_kenrel/466.html#comment-8866)

**发表评论：**

 昵称

 邮件地址 (选填)

 个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php) 

- ### 站内搜索
    
       
     蜗窝站内  互联网
    
- ### 功能
    
    [留言板  
    ](http://www.wowotech.net/message_board.html)[评论列表  
    ](http://www.wowotech.net/?plugin=commentlist)[支持者列表  
    ](http://www.wowotech.net/support_list)
- ### 最新评论
    
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
    - lw  
        [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
    - 肥饶  
        [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
    - orange  
        [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)
- ### 文章分类
    
    - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
        - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
        - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
        - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
        - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
        - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
        - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
        - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
        - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
        - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
        - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
        - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
        - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
    - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
    - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
    - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
    - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
        - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
        - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
        - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
        - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
    - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
    - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
    - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
        - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)
- ### 随机文章
    
    - [蓝牙协议中LQ和RSSI的原理及应用场景](http://www.wowotech.net/bluetooth/lqi_rssi.html)
    - [Atomic operation in aarch64](http://www.wowotech.net/armv8a_arch/492.html)
    - [ARM64的启动过程之（四）：打开MMU](http://www.wowotech.net/armv8a_arch/turn-on-mmu.html)
    - [perfbook memory barrier（14.2章节）的中文翻译（上）](http://www.wowotech.net/kernel_synchronization/memory-barrier-1.html)
    - [Linux电源管理(2)_Generic PM之基本概念和软件架构](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html)
- ### 文章存档
    
    - [2024年2月(1)](http://www.wowotech.net/record/202402)
    - [2023年5月(1)](http://www.wowotech.net/record/202305)
    - [2022年10月(1)](http://www.wowotech.net/record/202210)
    - [2022年8月(1)](http://www.wowotech.net/record/202208)
    - [2022年6月(1)](http://www.wowotech.net/record/202206)
    - [2022年5月(1)](http://www.wowotech.net/record/202205)
    - [2022年4月(2)](http://www.wowotech.net/record/202204)
    - [2022年2月(2)](http://www.wowotech.net/record/202202)
    - [2021年12月(1)](http://www.wowotech.net/record/202112)
    - [2021年11月(5)](http://www.wowotech.net/record/202111)
    - [2021年7月(1)](http://www.wowotech.net/record/202107)
    - [2021年6月(1)](http://www.wowotech.net/record/202106)
    - [2021年5月(3)](http://www.wowotech.net/record/202105)
    - [2020年3月(3)](http://www.wowotech.net/record/202003)
    - [2020年2月(2)](http://www.wowotech.net/record/202002)
    - [2020年1月(3)](http://www.wowotech.net/record/202001)
    - [2019年12月(3)](http://www.wowotech.net/record/201912)
    - [2019年5月(4)](http://www.wowotech.net/record/201905)
    - [2019年3月(1)](http://www.wowotech.net/record/201903)
    - [2019年1月(3)](http://www.wowotech.net/record/201901)
    - [2018年12月(2)](http://www.wowotech.net/record/201812)
    - [2018年11月(1)](http://www.wowotech.net/record/201811)
    - [2018年10月(2)](http://www.wowotech.net/record/201810)
    - [2018年8月(1)](http://www.wowotech.net/record/201808)
    - [2018年6月(1)](http://www.wowotech.net/record/201806)
    - [2018年5月(1)](http://www.wowotech.net/record/201805)
    - [2018年4月(7)](http://www.wowotech.net/record/201804)
    - [2018年2月(4)](http://www.wowotech.net/record/201802)
    - [2018年1月(5)](http://www.wowotech.net/record/201801)
    - [2017年12月(2)](http://www.wowotech.net/record/201712)
    - [2017年11月(2)](http://www.wowotech.net/record/201711)
    - [2017年10月(1)](http://www.wowotech.net/record/201710)
    - [2017年9月(5)](http://www.wowotech.net/record/201709)
    - [2017年8月(4)](http://www.wowotech.net/record/201708)
    - [2017年7月(4)](http://www.wowotech.net/record/201707)
    - [2017年6月(3)](http://www.wowotech.net/record/201706)
    - [2017年5月(3)](http://www.wowotech.net/record/201705)
    - [2017年4月(1)](http://www.wowotech.net/record/201704)
    - [2017年3月(8)](http://www.wowotech.net/record/201703)
    - [2017年2月(6)](http://www.wowotech.net/record/201702)
    - [2017年1月(5)](http://www.wowotech.net/record/201701)
    - [2016年12月(6)](http://www.wowotech.net/record/201612)
    - [2016年11月(11)](http://www.wowotech.net/record/201611)
    - [2016年10月(9)](http://www.wowotech.net/record/201610)
    - [2016年9月(6)](http://www.wowotech.net/record/201609)
    - [2016年8月(9)](http://www.wowotech.net/record/201608)
    - [2016年7月(5)](http://www.wowotech.net/record/201607)
    - [2016年6月(8)](http://www.wowotech.net/record/201606)
    - [2016年5月(8)](http://www.wowotech.net/record/201605)
    - [2016年4月(7)](http://www.wowotech.net/record/201604)
    - [2016年3月(5)](http://www.wowotech.net/record/201603)
    - [2016年2月(5)](http://www.wowotech.net/record/201602)
    - [2016年1月(6)](http://www.wowotech.net/record/201601)
    - [2015年12月(6)](http://www.wowotech.net/record/201512)
    - [2015年11月(9)](http://www.wowotech.net/record/201511)
    - [2015年10月(9)](http://www.wowotech.net/record/201510)
    - [2015年9月(4)](http://www.wowotech.net/record/201509)
    - [2015年8月(3)](http://www.wowotech.net/record/201508)
    - [2015年7月(7)](http://www.wowotech.net/record/201507)
    - [2015年6月(3)](http://www.wowotech.net/record/201506)
    - [2015年5月(6)](http://www.wowotech.net/record/201505)
    - [2015年4月(9)](http://www.wowotech.net/record/201504)
    - [2015年3月(9)](http://www.wowotech.net/record/201503)
    - [2015年2月(6)](http://www.wowotech.net/record/201502)
    - [2015年1月(6)](http://www.wowotech.net/record/201501)
    - [2014年12月(17)](http://www.wowotech.net/record/201412)
    - [2014年11月(8)](http://www.wowotech.net/record/201411)
    - [2014年10月(9)](http://www.wowotech.net/record/201410)
    - [2014年9月(7)](http://www.wowotech.net/record/201409)
    - [2014年8月(12)](http://www.wowotech.net/record/201408)
    - [2014年7月(6)](http://www.wowotech.net/record/201407)
    - [2014年6月(6)](http://www.wowotech.net/record/201406)
    - [2014年5月(9)](http://www.wowotech.net/record/201405)
    - [2014年4月(9)](http://www.wowotech.net/record/201404)
    - [2014年3月(7)](http://www.wowotech.net/record/201403)
    - [2014年2月(3)](http://www.wowotech.net/record/201402)
    - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")