# [蜗窝科技](http://www.wowotech.net/)

### 慢下来，享受技术。

[![](http://www.wowotech.net/content/uploadfile/201401/top-1389777175.jpg)](http://www.wowotech.net/)

- [博客](http://www.wowotech.net/)
- [项目](http://www.wowotech.net/sort/project)
- [关于蜗窝](http://www.wowotech.net/about.html)
- [联系我们](http://www.wowotech.net/contact_us.html)
- [支持与合作](http://www.wowotech.net/support_us.html)
- [登录](http://www.wowotech.net/admin)

﻿

## 

作者：[heaven](http://www.wowotech.net/author/532) 发布于：2020-1-7 14:42 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

我们继续沿着上一篇的以太网思路来继续分析，目的是为了学习以太网这块从应用层到底层的整块加载和匹配流程。  

如下是本人调试过程中的一点经验分享，以太网驱动架构毕竟涉及的东西太多，如下仅仅是针对加载流程和围绕这个问题产生的分析过程和驱动加载流程部分，并不涉及以太网协议层的数据流程分析。

【硬件环境】         Imx6ul

【Linux kernel版本】   Linux4.1.15

【以太网phy】        Realtek8201f

  

## 1.1 mii_bus初始化以及phy id的获取

然后进行mii的一些初始化fec_enet_mii_init(pdev);

主要是对struct mii_bus这里的成员进行初始化

  

1. /*
2.  * The Bus class for PHYs.  Devices which provide access to
3.  * PHYs should register using this structure
4.  */
5. struct mii_bus {
6. 	const char *name;
7. 	char id[MII_BUS_ID_SIZE];
8. 	void *priv;
9. 	int (*read)(struct mii_bus *bus, int phy_id, int regnum);
10. 	int (*write)(struct mii_bus *bus, int phy_id, int regnum, u16 val);
11. 	int (*reset)(struct mii_bus *bus);

13. 	/*
14. 	 * A lock to ensure that only one thing can read/write
15. 	 * the MDIO bus at a time
16. 	 */
17. 	struct mutex mdio_lock;

19. 	struct device *parent;
20. 	enum {
21. 		MDIOBUS_ALLOCATED = 1,
22. 		MDIOBUS_REGISTERED,
23. 		MDIOBUS_UNREGISTERED,
24. 		MDIOBUS_RELEASED,
25. 	} state;
26. 	struct device dev;

28. 	/* list of all PHYs on bus */
29. 	struct phy_device *phy_map[PHY_MAX_ADDR];

31. 	/* PHY addresses to be ignored when probing */
32. 	u32 phy_mask;

34. 	/*
35. 	 * Pointer to an array of interrupts, each PHY's
36. 	 * interrupt at the index matching its address
37. 	 */
38. 	int *irq;
39. };
40. #define to_mii_bus(d) container_of(d, struct mii_bus, dev)

1. 	fep->mii_bus = mdiobus_alloc();
2. 	if (fep->mii_bus == NULL) {
3. 		err = -ENOMEM;
4. 		goto err_out;
5. 	}

7. 	fep->mii_bus->name = "fec_enet_mii_bus";
8. 	fep->mii_bus->read = fec_enet_mdio_read;
9. 	fep->mii_bus->write = fec_enet_mdio_write;
10. 	snprintf(fep->mii_bus->id, MII_BUS_ID_SIZE, "%s-%x",
11. 		pdev->name, fep->dev_id + 1);
12. 	fep->mii_bus->priv = fep;
13. 	fep->mii_bus->parent = &pdev->dev;

15. 	fep->mii_bus->irq = kmalloc(sizeof(int) * PHY_MAX_ADDR, GFP_KERNEL);
16. 	if (!fep->mii_bus->irq) {
17. 		err = -ENOMEM;
18. 		goto err_out_free_mdiobus;
19. 	}

21. 	for (i = 0; i < PHY_MAX_ADDR; i++)
22. 		fep->mii_bus->irq[i] = PHY_POLL;

  

  

并且会做注册mdiobus的工作

  

  

1. 	node = of_get_child_by_name(pdev->dev.of_node, "mdio");
2. 	if (node) {
3. 		err = of_mdiobus_register(fep->mii_bus, node);
4. 		of_node_put(node);
5. 	} else {
6. 		err = mdiobus_register(fep->mii_bus);
7. 	}

  

因为我们系统是使用device tree，因此会执行of_mdiobus_register

  

1. /**
2.  * of_mdiobus_register - Register mii_bus and create PHYs from the device tree
3.  * @mdio: pointer to mii_bus structure
4.  * @np: pointer to device_node of MDIO bus.
5.  *
6.  * This function registers the mii_bus structure and registers a phy_device
7.  * for each child node of @np.
8.  */
9. int of_mdiobus_register(struct mii_bus *mdio, struct device_node *np)
10. {
11. 	struct device_node *child;
12. 	const __be32 *paddr;
13. 	bool scanphys = false;
14. 	int addr, rc, i;

16. 	/* Mask out all PHYs from auto probing.  Instead the PHYs listed in
17. 	 * the device tree are populated after the bus has been registered */
18. 	mdio->phy_mask = ~0;

20. 	/* Clear all the IRQ properties */
21. 	if (mdio->irq)
22. 		for (i=0; i<PHY_MAX_ADDR; i++)
23. 			mdio->irq[i] = PHY_POLL;

25. 	mdio->dev.of_node = np;

27. 	/* Register the MDIO bus */
28. 	rc = mdiobus_register(mdio);
29. 	if (rc)
30. 		return rc;

32. 	/* Loop over the child nodes and register a phy_device for each one */
33. 	for_each_available_child_of_node(np, child) {
34. 		addr = of_mdio_parse_addr(&mdio->dev, child);
35. 		if (addr < 0) {
36. 			scanphys = true;
37. 			continue;
38. 		}

40. 		rc = of_mdiobus_register_phy(mdio, child, addr);
41. 		if (rc)
42. 			continue;
43. 	}

45. 	if (!scanphys)
46. 		return 0;

48. 	/* auto scan for PHYs with empty reg property */
49. 	for_each_available_child_of_node(np, child) {
50. 		/* Skip PHYs with reg property set */
51. 		paddr = of_get_property(child, "reg", NULL);
52. 		if (paddr)
53. 			continue;

55. 		for (addr = 0; addr < PHY_MAX_ADDR; addr++) {
56. 			/* skip already registered PHYs */
57. 			if (mdio->phy_map[addr])
58. 				continue;

60. 			/* be noisy to encourage people to set reg property */
61. 			dev_info(&mdio->dev, "scan phy %s at address %i\n",
62. 				 child->name, addr);

64. 			rc = of_mdiobus_register_phy(mdio, child, addr);
65. 			if (rc)
66. 				continue;
67. 		}
68. 	}

70. 	return 0;
71. }
72. EXPORT_SYMBOL(of_mdiobus_register);

  

进行midobus_register

  

1. 	/* Register the MDIO bus */
2. 	rc = mdiobus_register(mdio);
3. 	if (rc)
4. 		return rc;

6. 	/* Loop over the child nodes and register a phy_device for each one */
7. 	for_each_available_child_of_node(np, child) {
8. 		addr = of_mdio_parse_addr(&mdio->dev, child);
9. 		if (addr < 0) {
10. 			scanphys = true;
11. 			continue;
12. 		}

14. 		rc = of_mdiobus_register_phy(mdio, child, addr);
15. 		if (rc)
16. 			continue;
17. 	}

  

  

由于设备树代码是这样的：

  

  

1. 	mdio {
2. 		#address-cells = <1>;
3. 		#size-cells = <0>;

5. 		ethphy0: ethernet-phy@1 {
6. 			compatible = "ethernet-phy-ieee802.3-c22";
7. 			reg = <0>;
8. 		};
9. 	};

如下路径：drivers/of/of_mdio.c

  

1. static int of_mdiobus_register_phy(struct mii_bus *mdio, struct device_node *child,
2. 				   u32 addr)
3. {
4. 	struct phy_device *phy;
5. 	bool is_c45;
6. 	int rc;
7. 	u32 phy_id;

9. 	is_c45 = of_device_is_compatible(child,
10. 					 "ethernet-phy-ieee802.3-c45");

12. 	if (!is_c45 && !of_get_phy_id(child, &phy_id))
13. 		phy = phy_device_create(mdio, addr, phy_id, 0, NULL);
14. 	else
15. 		phy = get_phy_device(mdio, addr, is_c45);
16. 	if (!phy || IS_ERR(phy))
17. 		return 1;

19. 	rc = irq_of_parse_and_map(child, 0);
20. 	if (rc > 0) {
21. 		phy->irq = rc;
22. 		if (mdio->irq)
23. 			mdio->irq[addr] = rc;
24. 	} else {
25. 		if (mdio->irq)
26. 			phy->irq = mdio->irq[addr];
27. 	}

29. 	/* Associate the OF node with the device structure so it
30. 	 * can be looked up later */
31. 	of_node_get(child);
32. 	phy->dev.of_node = child;

34. 	/* All data is now stored in the phy struct;
35. 	 * register it */
36. 	rc = phy_device_register(phy);
37. 	if (rc) {
38. 		phy_device_free(phy);
39. 		of_node_put(child);
40. 		return 1;
41. 	}

43. 	dev_dbg(&mdio->dev, "registered phy %s at address %i\n",
44. 		child->name, addr);

46. 	return 0;
47. }

因此我们是走get_phy_device这个函数：

所以我说内核代码写的好，就是注释和函数名基本就是意思了，获取phy device，

1. /**
2.  * get_phy_device - reads the specified PHY device and returns its @phy_device
3.  *		    struct
4.  * @bus: the target MII bus
5.  * @addr: PHY address on the MII bus
6.  * @is_c45: If true the PHY uses the 802.3 clause 45 protocol
7.  *
8.  * Description: Reads the ID registers of the PHY at @addr on the
9.  *   @bus, then allocates and returns the phy_device to represent it.
10.  */
11. struct phy_device *get_phy_device(struct mii_bus *bus, int addr, bool is_c45)
12. {
13. 	struct phy_c45_device_ids c45_ids = {0};
14. 	u32 phy_id = 0;
15. 	int r;

17. 	r = get_phy_id(bus, addr, &phy_id, is_c45, &c45_ids);
18. 	if (r)
19. 		return ERR_PTR(r);

21. 	/* If the phy_id is mostly Fs, there is no device there */
22. 	if ((phy_id & 0x1fffffff) == 0x1fffffff)
23. 		return NULL;

25. 	return phy_device_create(bus, addr, phy_id, is_c45, &c45_ids);
26. }
27. EXPORT_SYMBOL(get_phy_device);

1. /**
2.  * get_phy_id - reads the specified addr for its ID.
3.  * @bus: the target MII bus
4.  * @addr: PHY address on the MII bus
5.  * @phy_id: where to store the ID retrieved.
6.  * @is_c45: If true the PHY uses the 802.3 clause 45 protocol
7.  * @c45_ids: where to store the c45 ID information.
8.  *
9.  * Description: In the case of a 802.3-c22 PHY, reads the ID registers
10.  *   of the PHY at @addr on the @bus, stores it in @phy_id and returns
11.  *   zero on success.
12.  *
13.  *   In the case of a 802.3-c45 PHY, get_phy_c45_ids() is invoked, and
14.  *   its return value is in turn returned.
15.  *
16.  */
17. static int get_phy_id(struct mii_bus *bus, int addr, u32 *phy_id,
18. 		      bool is_c45, struct phy_c45_device_ids *c45_ids)
19. {
20. 	int phy_reg;

22. 	if (is_c45)
23. 		return get_phy_c45_ids(bus, addr, phy_id, c45_ids);

25. 	/* Grab the bits from PHYIR1, and put them in the upper half */
26. 	phy_reg = mdiobus_read(bus, addr, MII_PHYSID1);
27. 	if (phy_reg < 0)
28. 		return -EIO;

30. 	*phy_id = (phy_reg & 0xffff) << 16;

32. 	/* Grab the bits from PHYIR2, and put them in the lower half */
33. 	phy_reg = mdiobus_read(bus, addr, MII_PHYSID2);
34. 	if (phy_reg < 0)
35. 		return -EIO;

37. 	*phy_id |= (phy_reg & 0xffff);

39. 	return 0;
40. }

最关键的函数就是它，也就是本文的核心，这里是从寄存器中通过mdiobus的read方法来从phy中获取phy id，但是这里并没有获取到phy_id ，这寄存器都是以太网的通用寄存器

  

#define MII_PHYSID1 0x02 /* PHYS ID 1                   */  
#define MII_PHYSID2 0x03 /* PHYS ID 2                   */

  

既然没有从寄存器中获取到phy_id，因此phy_device_create也不会在mii bus数据结构中创建phy_device，

那么应用层在进行socket的时候，回调了open函数 fec_enet_open，这个函数中的fec_enet_mii_probe就不会从of_phy_connect中获取到phy_device，因此就会出现-19的错误。那么获取不到phy_id的根本原因就是因为reset的时序没满足datasheet的要求，具体原因分析请见之前第一篇以太网分析的《标题2 原因分析》

## 1.2 Realtek phy的内核配置

那这是获取不到phy id的过程，那么正常的获取phy id的流程又是怎样的呢？

我们可以看到这样的log：

  

fec 2188000.ethernet eth0: Freescale FEC PHY driver [RTL8201F Fast Ethernet] (mii_bus:phy_addr=2188000.ethernet:00, irq=-1)

  

那这里又是怎样匹配的呢？

make kernel_menuconfig中我们需要选中realtek这款phy

[![图像 8.jpg](http://www.wowotech.net/content/uploadfile/202001/07cd1578380351.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/07cd1578380351.jpg)

  
[![图像 9.jpg](http://www.wowotech.net/content/uploadfile/202001/thum-738b1578380351.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/738b1578380351.jpg)

  

[![图像 10.jpg](http://www.wowotech.net/content/uploadfile/202001/thum-11971578380351.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/11971578380351.jpg)

选中Realtek PHYs，这样realtek.c就可以编译到kernel了

  

[![图像 11.jpg](http://www.wowotech.net/content/uploadfile/202001/thum-4bae1578380351.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/4bae1578380351.jpg)

  

代码路径：drivers/net/phy/realtek.c

  

1. static struct phy_driver realtek_drvs[] = {
2. 	{
3. 		.phy_id         = 0x00008201,
4. 		.name           = "RTL8201CP Ethernet",
5. 		.phy_id_mask    = 0x0000ffff,
6. 		.features       = PHY_BASIC_FEATURES,
7. 		.flags          = PHY_HAS_INTERRUPT,
8. 		.config_aneg    = &genphy_config_aneg,
9. 		.read_status    = &genphy_read_status,
10. 		.driver         = { .owner = THIS_MODULE,},
11. 	},{
12. 		.phy_id		= 0x001cc816,
13. 		.name		= "RTL8201F Fast Ethernet",
14. 		.phy_id_mask	= 0x001fffff,
15. 		.features	= PHY_BASIC_FEATURES,
16. 		.flags		= PHY_HAS_INTERRUPT,
17. 		.config_init	= rtl8201f_config_init,
18. 		.config_aneg	= &genphy_config_aneg,
19. 		.read_status	= &genphy_read_status,
20. 		.ack_interrupt	= &rtl8201f_ack_interrupt,
21. 		.config_intr	= &rtl8201f_config_intr,
22. 		.set_wol		= rtl8201f_set_wol,
23. 		.get_wol		= rtl8201f_get_wol,
24. 		.suspend	= rtl8201f_suspend,
25. 		.resume		= rtl8201f_resume,
26. 		.driver		= { .owner = THIS_MODULE,},
27. 	},{
28. 		.phy_id		= 0x001cc912,
29. 		.name		= "RTL8211B Gigabit Ethernet",
30. 		.phy_id_mask	= 0x001fffff,
31. 		.features	= PHY_GBIT_FEATURES,
32. 		.flags		= PHY_HAS_INTERRUPT,
33. 		.config_aneg	= &genphy_config_aneg,
34. 		.read_status	= &genphy_read_status,
35. 		.ack_interrupt	= &rtl821x_ack_interrupt,
36. 		.config_intr	= &rtl8211b_config_intr,
37. 		.driver		= { .owner = THIS_MODULE,},
38. 	},{
39. 		.phy_id		= 0x001cc915,
40. 		.name		= "RTL8211E Gigabit Ethernet",
41. 		.phy_id_mask	= 0x001fffff,
42. 		.features	= PHY_GBIT_FEATURES,
43. 		.flags		= PHY_HAS_INTERRUPT,
44. 		.config_aneg	= &genphy_config_aneg,
45. 		.read_status	= &genphy_read_status,
46. 		.ack_interrupt	= &rtl821x_ack_interrupt,
47. 		.config_intr	= &rtl8211e_config_intr,
48. 		.suspend	= genphy_suspend,
49. 		.resume		= genphy_resume,
50. 		.driver		= { .owner = THIS_MODULE,},
51. 	},
52. };

54. module_phy_driver(realtek_drvs);

56. static struct mdio_device_id __maybe_unused realtek_tbl[] = {
57. 	{ 0x001cc816, 0x001fffff },
58. 	{ 0x001cc912, 0x001fffff },
59. 	{ 0x001cc915, 0x001fffff },
60. 	{ }
61. };

phy_id                = 0x001cc816我们需要把这个phy id填入

module_phy_driver(realtek_drvs);

  

  

1. /**
2.  * module_phy_driver() - Helper macro for registering PHY drivers
3.  * @__phy_drivers: array of PHY drivers to register
4.  *
5.  * Helper macro for PHY drivers which do not do anything special in module
6.  * init/exit. Each module may only use this macro once, and calling it
7.  * replaces module_init() and module_exit().
8.  */
9. #define phy_module_driver(__phy_drivers, __count)			\
10. static int __init phy_module_init(void)					\
11. {									\
12. 	return phy_drivers_register(__phy_drivers, __count);		\
13. }									\
14. module_init(phy_module_init);						\
15. static void __exit phy_module_exit(void)				\
16. {									\
17. 	phy_drivers_unregister(__phy_drivers, __count);			\
18. }									\
19. module_exit(phy_module_exit)

21. #define module_phy_driver(__phy_drivers)				\
22. 	phy_module_driver(__phy_drivers, ARRAY_SIZE(__phy_drivers))

  

  

这里会将这个phy_drvier注册进去

  

  

1. /**
2.  * phy_probe - probe and init a PHY device
3.  * @dev: device to probe and init
4.  *
5.  * Description: Take care of setting up the phy_device structure,
6.  *   set the state to READY (the driver's init function should
7.  *   set it to STARTING if needed).
8.  */
9. static int phy_probe(struct device *dev)
10. {
11. 	struct phy_device *phydev = to_phy_device(dev);
12. 	struct device_driver *drv = phydev->dev.driver;
13. 	struct phy_driver *phydrv = to_phy_driver(drv);
14. 	int err = 0;

16. 	phydev->drv = phydrv;

  

  

然后在这里把phy_device与phy_drvier关联了起来，再由phy_driver_register注册

  

  

1. int phy_drivers_register(struct phy_driver *new_driver, int n)
2. {
3. 	int i, ret = 0;

5. 	for (i = 0; i < n; i++) {
6. 		ret = phy_driver_register(new_driver + i);
7. 		if (ret) {
8. 			while (i-- > 0)
9. 				phy_driver_unregister(new_driver + i);
10. 			break;
11. 		}
12. 	}
13. 	return ret;
14. }
15. EXPORT_SYMBOL(phy_drivers_register);

  

  

Freescale的以太网控制器驱动fec_main.c中

static int fec_enet_mii_probe(struct net_device *ndev)

  

[![图像 12.jpg](http://www.wowotech.net/content/uploadfile/202001/11101578380721.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/11101578380721.jpg)

  

## 1.3 以太网流程总图

最后汇总一个图给大家：

[![图像 13.jpg](http://www.wowotech.net/content/uploadfile/202001/6a901578380836.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/6a901578380836.jpg)

[![图像 14.jpg](http://www.wowotech.net/content/uploadfile/202001/19f81578380837.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/19f81578380837.jpg)

[![图像 15.jpg](http://www.wowotech.net/content/uploadfile/202001/a1481578380837.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/a1481578380837.jpg)

  

以上五篇文章就是我个人的记录，如有表达不精准的地方或者理解不到位的地方可以多多交流指出，共同进步

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

« [F2FS技术拆解](http://www.wowotech.net/filesystem/f2fs.html) | [以太网驱动的流程浅析(四)-以太网驱动probe流程](http://www.wowotech.net/linux_kenrel/469.html)»

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
    
    - [fixmap addresses原理](http://www.wowotech.net/memory_management/440.html)
    - [Linux电源管理(11)_Runtime PM之功能描述](http://www.wowotech.net/pm_subsystem/rpm_overview.html)
    - [Concurrency Managed Workqueue之（三）：创建workqueue代码分析](http://www.wowotech.net/irq_subsystem/alloc_workqueue.html)
    - [蓝牙协议分析(3)_蓝牙低功耗(BLE)协议栈介绍](http://www.wowotech.net/bluetooth/ble_stack_overview.html)
    - [KASAN实现原理](http://www.wowotech.net/memory_management/424.html)
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