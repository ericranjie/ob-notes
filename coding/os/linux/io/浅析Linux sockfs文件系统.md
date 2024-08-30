
Linux内核之旅

 _2021年12月02日 22:28_

Editor's Note

本文作者为Linux内核之旅社区研究生团队—董旭

The following article is from 技术简说 Author 董旭


![](http://wx.qlogo.cn/mmhead/Q3auHgzwzM5b9FrINsvZl3ribRAQicibVs74cJ9zMmiagjpZREkxAeAFuQ/0)

**技术简说**.

主要分享Linux内核、内核网络知识，欢迎大家关注！

](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664610414&idx=1&sn=81b5f6d60b6a479d82e573392dd83170&chksm=f04d978bc73a1e9de3bdd459ae00de143d5fc53f205a8530fe0f71a9664d5297d6bcc4838655&mpshare=1&scene=24&srcid=1203A6MDynStm9BXW9nxHKgt&sharer_sharetime=1638461219630&sharer_shareid=5fb9813bfe9ffc983435bfc8d8c5e9ca&key=daf9bdc5abc4e8d0d0fca404309e8f74e8683dbe7810841a589b95649b9bd6b0a8f323fcf115a6c1dabeb2892276736a77fc17c8418b969ea11a288cd665ae0c75c4e7f40c45e6a9a84192fb1143be020e5ceeaa24efb474132615e537be0128fbd4609cde4829ea35edfebd24cd71e3c2191eb6026588bc359f25af8ba71cbb&ascene=14&uin=MTEwNTU1MjgwMw%3D%3D&devicetype=iMac+MacBookAir10%2C1+OSX+OSX+14.6.1+build(23G93)&version=13080710&nettype=WIFI&lang=en&session_us=gh_3e85a6de261f&countrycode=CN&fontScale=100&exportkey=n_ChQIAhIQZhJ3l9xtzJ5Fyzh%2BUToBaxKUAgIE97dBBAEAAAAAAMDCMG7s854AAAAOpnltbLcz9gKNyK89dVj0yWa3vv8WG6Z9pVJBNadIBiVEJNhNwkmZnouuELKuNrvOIEz%2FToAZMCj71258L2VF02beQ1XX%2BQhA0izOd4VOXo%2F2JR7Ep7PYHORRDWmciyHp3%2FNVoXANvdi2oQXJIevWR2W7ZTtJdBP0XxzwVOYQmOkCs5GhWVNpvXKBqFp3zvj8ELlwRXy6vE6h6pBrDXe2SfciO6lz7fe5XTsfSmzv23%2FEKBDbwTLvMhrfHpTdg%2FbR2NqgRtuhZsrHdDnaC%2FoFX71vw%2BugMCkk6NY2xCxjxNYxPatyXEk2Ovbxue4f%2F%2Bxxqz2U6YeNyaLGL3YVkw%3D%3D&acctmode=0&pass_ticket=86p9ryWCa%2BKH%2BVCPHc5wrVimsPhCsoCesu%2BxdkUE9anguaRojus0W45KPdaYlIHa&wx_header=0#)

  

浅析Linux sockfs文件系统


![](http://mmbiz.qpic.cn/mmbiz_png/EjWxxIM2EQI0YHzCpIYYwO0iaqh08EGCibYEjLqIqYm2CXPzmicQTkxqF453q1d9RcSicSLGjjCNyBsjDXdx8oDhcA/300?wx_fmt=png&wxfrom=19)

**技术简说**

主要分享Linux内核、内核网络知识，欢迎大家关注！

35篇原创内容

公众号

本文主要对Linux网络文件系统的注册与挂载过程进行分析

**一、简介**

Linux中"万物皆文件"，socket在Linux中对应的文件系统叫Sockfs,每创建一个socket,就在sockfs中创建了一个特殊的文件，同时创建了sockfs文件系统中的inode，该inode唯一标识当前socket的通信。

本文的重点放在sockfs文件系统的注册和挂载流程上，以后会对socket的底层来龙去脉进行详细地分析与记录。

**二、三个核心结构体**

**1、struct file_system_type**

file_system_type结构体代表Linux内核的各种文件系统，每一种文件系统必须要有自己的file_system_type结构，用于描述具体的文件系统的类型，如ext4对应的ext4_fs_type,struct file_system_type结构体如所示：

```
struct file_system_type { const char *name;  //文件系统的名字 int fs_flags;#define FS_REQUIRES_DEV  1 #define FS_BINARY_MOUNTDATA 2#define FS_HAS_SUBTYPE  4#define FS_USERNS_MOUNT  8 /* Can be mounted by userns root */#define FS_RENAME_DOES_D_MOVE 32768 /* FS will handle d_move() during rename() internally. */ struct dentry *(*mount) (struct file_system_type *, int,         const char *, void *);  //挂载此文件系统时使用的回调函数 void (*kill_sb) (struct super_block *); //释放超级块函数指针 struct module *owner;//指向实现这个文件系统的模块，通常为THIS_MODULE宏 struct file_system_type * next;//指向文件系统类型链表的下一个文件系统类型 struct hlist_head fs_supers; struct lock_class_key s_lock_key; struct lock_class_key s_umount_key; struct lock_class_key s_vfs_rename_key; struct lock_class_key s_writers_key[SB_FREEZE_LEVELS]; struct lock_class_key i_lock_key; struct lock_class_key i_mutex_key; struct lock_class_key i_mutex_dir_key;};
```

如struct file_system_type * next结构体成员，所有文件系统的file_system_type结构形成一个链表，在fs/filesystem.c中有一个全局的文件系统变量  

```
/* fs/filesystem.c*/static struct file_system_type *file_systems;
```

在Linux内核中sock_fs_type结构定义代表了sockfs的网络文件系统，如下所示：

```
static struct file_system_type sock_fs_type = { .name =  "sockfs", .mount = sockfs_mount, .kill_sb = kill_anon_super,};
```

**2、struct vfsmount与struct mount**

每当一个文件系统被安装时，就会有一个vfsmount结构和mount被创建，mount代表该文件系统的一个安装实例,比较旧的内核版本中mount和vfsmount的成员都在vfsmount里，现在Linux内核将vfsmount改作mount结构体，并将mount中mnt_root, mnt_sb, mnt_flags成员移到vfsmount结构体中了。这样使得vfsmount的内容更加精简，在很多情况下只需要传递vfsmount而已。struct vfsmount如下：

```
struct vfsmount { struct dentry *mnt_root; //指向这个文件系统的根的dentry struct super_block *mnt_sb; // 指向这个文件系统的超级块对象 int mnt_flags; // 此文件系统的挂载标志}
```

对于每一个mount的文件系统都有一个vfsmount结构来表示，sockfs安装时的vfsmount定义如下所示：

```
static struct vfsmount *sock_mnt __read_mostly;
```

struct mount如下：

```
struct mount {        struct hlist_node mnt_hash;    /* 用于链接到全局已挂载文件系统的链表 */        struct mount *mnt_parent;      /* 指向此文件系统的挂载点所属的文件系统，即父文件系统 */         struct dentry *mnt_mountpoint; /* 指向此文件系统的挂载点的dentry */        struct vfsmount mnt;           /* 指向此文件系统的vfsmount实例 */        union {                struct rcu_head mnt_rcu;                struct llist_node mnt_llist;        };#ifdef CONFIG_SMP        struct mnt_pcp __percpu *mnt_pcp;#else        int mnt_count;        int mnt_writers;#endif        struct list_head mnt_mounts;    /* 挂载在此文件系统下的所有子文件系统的链表的表头，下面的节点都是mnt_child */        struct list_head mnt_child;     /* 链接到被此文件系统所挂的父文件系统的mnt_mounts上 */        struct list_head mnt_instance;  /* 链接到sb->s_mounts上的一个mount实例 */        const char *mnt_devname;        /* 设备名，如/dev/sdb1 */        struct list_head mnt_list;      /* 链接到进程namespace中已挂载文件系统中，表头为mnt_namespace的list域 */         struct list_head mnt_expire;    /* 链接到一些文件系统专有的过期链表，如NFS, CIFS等 */        struct list_head mnt_share;     /* 链接到共享挂载的循环链表中 */        struct list_head mnt_slave_list;/* 此文件系统的slave mount链表的表头 */        struct list_head mnt_slave;     /* 连接到master文件系统的mnt_slave_list */        struct mount *mnt_master;       /* 指向此文件系统的master文件系统，slave is on master->mnt_slave_list */        struct mnt_namespace *mnt_ns;   /* 指向包含这个文件系统的进程的name space */        struct mountpoint *mnt_mp;      /* where is it mounted */        struct hlist_node mnt_mp_list;  /* list mounts with the same mountpoint */        struct list_head mnt_umounting; /* list entry for umount propagation */#ifdef CONFIG_FSNOTIFY        struct fsnotify_mark_connector __rcu *mnt_fsnotify_marks;        __u32 mnt_fsnotify_mask;#endif        int mnt_id;                     /* mount identifier */        int mnt_group_id;               /* peer group identifier */        int mnt_expiry_mark;            /* true if marked for expiry */        struct hlist_head mnt_pins;        struct fs_pin mnt_umount;        struct dentry *mnt_ex_mountpoint;}
```

**三、sockfs文件系统的注册**

Linux内核初始化时，执行sock_init()函数登记sockfs,sock_init()函数如下：

```
static int __init sock_init(void){ ...... err = register_filesystem(&sock_fs_type);//注册网络文件系统 ...... sock_mnt = kern_mount(&sock_fs_type);//安装网络文件系统  ......}
```

注册函数：

```
 int register_filesystem(struct file_system_type * fs){ int res = 0; struct file_system_type ** p; BUG_ON(strchr(fs->name, '.')); if (fs->next)  return -EBUSY; write_lock(&file_systems_lock); p = find_filesystem(fs->name, strlen(fs->name));  //查找是否存在 if (*p)  res = -EBUSY; else  *p = fs; //将filesystem静态变量指向fs write_unlock(&file_systems_lock); return res;}
```

注册函数中的find函数如下,for循环一开始的file_systems变量就是上面说的注册文件系统使用到的全局变量指针，strncmp去比较file_system_type的第一项name(文件系统名)是否和将要注册的文件系统名字相同，如果相同返回的P就是指向同名file_system_type结构的指针，如果没找到则指向NULL。

```
static struct file_system_type **find_filesystem(const char *name, unsigned len){ struct file_system_type **p; for (p = &file_systems; *p; p = &(*p)->next)  if (strncmp((*p)->name, name, len) == 0 &&      !(*p)->name[len])   break; return p;}
```

在返回register_filesystem函数后，判断返回值，如果找到重复的则返回EBUSY错误，如果没找到重复的，就把当前要注册的文件系统挂到尾端file_system_type的next指针上，串联进链表，至此一个文件系统模块就注册好了。

四、sockfs文件系统的安装

在上面的sock_init()函数中的sock_mnt = kern_mount(&sock_fs_type)开始进行安装。kern_mount函数主要用于那些没有实体介质的文件系统，该函数主要是获取文件系统的super_block对象与根目录的inode与dentry对象，并将这些对象加入到系统链表。kern_mount宏如下所示：

```
#define kern_mount(type) kern_mount_data(type, NULL)
```

kern_mount_data如下：

```
struct vfsmount *kern_mount_data(struct file_system_type *type, void *data){ struct vfsmount *mnt; mnt = vfs_kern_mount(type, SB_KERNMOUNT, type->name, data); if (!IS_ERR(mnt)) {  /*   * it is a longterm mount, don't release mnt until   * we unmount before file sys is unregistered  */  real_mount(mnt)->mnt_ns = MNT_NS_INTERNAL; } return mnt;}
```

调用：vfs_kern_mount

```
struct vfsmount *vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data){ struct mount *mnt; struct dentry *root; if (!type)  return ERR_PTR(-ENODEV); mnt = alloc_vfsmnt(name);//分配一个mount对象，并对其进行部分初始化 if (!mnt)  return ERR_PTR(-ENOMEM); if (flags & SB_KERNMOUNT)  mnt->mnt.mnt_flags = MNT_INTERNAL; root = mount_fs(type, flags, name, data);//获取该文件系统的根目录的dentry,同时也获取super_block if (IS_ERR(root)) {  mnt_free_id(mnt);  free_vfsmnt(mnt);  return ERR_CAST(root); }//对mnt对象与root进行绑定 mnt->mnt.mnt_root = root; mnt->mnt.mnt_sb = root->d_sb; mnt->mnt_mountpoint = mnt->mnt.mnt_root; mnt->mnt_parent = mnt; lock_mount_hash(); list_add_tail(&mnt->mnt_instance, &root->d_sb->s_mounts);//将mnt添加到root->d_sb->s_mounts链表中  unlock_mount_hash(); return &mnt->mnt;}
```

vfs_kern_mount函数调用mount_fs获取该文件系统的根目录的dentry,同时也获取super_block，具体实现如下：

```
struct dentry *mount_fs(struct file_system_type *type, int flags, const char *name, void *data){ struct dentry *root; struct super_block *sb; char *secdata = NULL; int error = -ENOMEM; if (data && !(type->fs_flags & FS_BINARY_MOUNTDATA)) {//在kern_mount调用中data为NULL，所以该if判断为假  secdata = alloc_secdata();  if (!secdata)   goto out;  error = security_sb_copy_data(data, secdata);  if (error)   goto out_free_secdata; } root = type->mount(type, flags, name, data);//调用file_system_type中的 mount方法 if (IS_ERR(root)) {  error = PTR_ERR(root);  goto out_free_secdata; } sb = root->d_sb; BUG_ON(!sb); WARN_ON(!sb->s_bdi); sb->s_flags |= SB_BORN; error = security_sb_kern_mount(sb, flags, secdata);......}
```

其中type->mount()继续调用了sockfs的回调函数sockfs_mount

`static struct dentry *sockfs_mount(struct file_system_type *fs_type,       int flags, const char *dev_name, void *data)   {    return mount_pseudo_xattr(fs_type, "socket:", &sockfs_ops,         sockfs_xattr_handlers,         &sockfs_dentry_operations, SOCKFS_MAGIC);   }   `

`struct dentry *mount_pseudo_xattr(struct file_system_type *fs_type, char *name,    const struct super_operations *ops, const struct xattr_handler **xattr,    const struct dentry_operations *dops, unsigned long magic)   {    struct super_block *s;    struct dentry *dentry;    struct inode *root;    struct qstr d_name = QSTR_INIT(name, strlen(name));       s = sget_userns(fs_type, NULL, set_anon_super, SB_KERNMOUNT|SB_NOUSER,      &init_user_ns, NULL);    if (IS_ERR(s))     return ERR_CAST(s);       s->s_maxbytes = MAX_LFS_FILESIZE;    s->s_blocksize = PAGE_SIZE;    s->s_blocksize_bits = PAGE_SHIFT;    s->s_magic = magic;    s->s_op = ops ? ops : &simple_super_operations;    s->s_xattr = xattr;    s->s_time_gran = 1;    root = new_inode(s);    if (!root)     goto Enomem;    /*     * since this is the first inode, make it number 1. New inodes created     * after this must take care not to collide with it (by passing     * max_reserved of 1 to iunique).     */    root->i_ino = 1;    root->i_mode = S_IFDIR | S_IRUSR | S_IWUSR;    root->i_atime = root->i_mtime = root->i_ctime = current_time(root);    dentry = __d_alloc(s, &d_name);    if (!dentry) {     iput(root);     goto Enomem;    }    d_instantiate(dentry, root);    s->s_root = dentry;    s->s_d_op = dops;    s->s_flags |= SB_ACTIVE;    return dget(s->s_root);      Enomem:    deactivate_locked_super(s);    return ERR_PTR(-ENOMEM);   }   `

以上函数进行超级块、根root、根dentry相关的创建及初始化操作，其中上面的s->s_d_op =dops就说指向了sockfs_ops结构体，也就是该sockfs文件系统的struct super_block的函数操作集指向了sockfs_ops。

`static const struct super_operations sockfs_ops = {    .alloc_inode = sock_alloc_inode,    .destroy_inode = sock_destroy_inode,    .statfs  = simple_statfs,   };   `

该函数表对sockfs文件系统的节点和目录提供了具体的操作函数，后面涉及到的sockfs文件系统的重要操作均会到该函数表中查找到对应的操作函数，例如Linux内核在创建socket节点时会查找sockfs_ops的alloc_inode函数， 从而调用sock_alloc_indode函数完成socket以及inode节点的创建。

  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

END

![](http://mmbiz.qpic.cn/mmbiz_png/EjWxxIM2EQI0YHzCpIYYwO0iaqh08EGCibYEjLqIqYm2CXPzmicQTkxqF453q1d9RcSicSLGjjCNyBsjDXdx8oDhcA/300?wx_fmt=png&wxfrom=19)

**技术简说**

主要分享Linux内核、内核网络知识，欢迎大家关注！

35篇原创内容

公众号

点个关注，一起学技术！  

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

您的点赞和关注是我最大的动力

Reads 3569

​

Comment

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/SeWfibBcBT0EibtIWVNvshnuWMN1AoJw3poFIsbpaIVyZibCCqwBUR21rcDfrQgoqYzaYNdS14IIXzvmzvibdDa5Rw/300?wx_fmt=png&wxfrom=18)

Linux内核之旅

1818

Comment

Comment

**Comment**

暂无留言