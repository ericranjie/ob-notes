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

作者：[heaven](http://www.wowotech.net/author/532) 发布于：2020-1-7 14:25 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

我们继续沿着上一篇的以太网思路来继续分析，目的是为了学习以太网这块从应用层到底层的整块加载和匹配流程。  

如下是本人调试过程中的一点经验分享，以太网驱动架构毕竟涉及的东西太多，如下仅仅是针对加载流程和围绕这个问题产生的分析过程和驱动加载流程部分，并不涉及以太网协议层的数据流程分析。

【硬件环境】         Imx6ul

【Linux kernel版本】   Linux4.1.15

【以太网phy】        Realtek8201f

## 1.1 以太网初始化

fec_main.c   fec_probe

=>fec_enet_mii_init

=>of_get_child_by_name(pdev->dev.of_node, "mdio");

            of_mdiobus_register(fep->mii_bus, node);

                                        =>rc = of_mdiobus_register_phy(mdio, child, addr);

                                                => phy = get_phy_device(mdio, addr, is_c45);

                                                   rc = phy_device_register(phy);

[![图像 6.jpg](http://www.wowotech.net/content/uploadfile/202001/a6731578378528.jpg "点击查看原图")](http://www.wowotech.net/content/uploadfile/202001/a6731578378528.jpg)

搞驱动的都知道，probe是drvier的入口函数：

1. static int
2. fec_probe(struct platform_device *pdev)
3. {
4. 	struct fec_enet_private *fep;
5. 	struct fec_platform_data *pdata;
6. 	struct net_device *ndev;
7. 	int i, irq, ret = 0;
8. 	struct resource *r;
9. 	const struct of_device_id *of_id;
10. 	static int dev_id;
11. 	struct device_node *np = pdev->dev.of_node, *phy_node;
12. 	int num_tx_qs;
13. 	int num_rx_qs;

15. 	fec_enet_get_queue_num(pdev, &num_tx_qs, &num_rx_qs);

17. 	/* Init network device */
18. 	ndev = alloc_etherdev_mqs(sizeof(struct fec_enet_private),
19. 				  num_tx_qs, num_rx_qs);
20. 	if (!ndev)
21. 		return -ENOMEM;

23. 	SET_NETDEV_DEV(ndev, &pdev->dev);

25. 	/* setup board info structure */
26. 	fep = netdev_priv(ndev);

28. 	of_id = of_match_device(fec_dt_ids, &pdev->dev);
29. 	if (of_id)
30. 		pdev->id_entry = of_id->data;
31. 	fep->quirks = pdev->id_entry->driver_data;

33. 	fep->netdev = ndev;
34. 	fep->num_rx_queues = num_rx_qs;
35. 	fep->num_tx_queues = num_tx_qs;

37. #if !defined(CONFIG_M5272)
38. 	/* default enable pause frame auto negotiation */
39. 	if (fep->quirks & FEC_QUIRK_HAS_GBIT)
40. 		fep->pause_flag |= FEC_PAUSE_FLAG_AUTONEG;
41. #endif

43. 	/* Select default pin state */
44. 	pinctrl_pm_select_default_state(&pdev->dev);

46. 	r = platform_get_resource(pdev, IORESOURCE_MEM, 0);
47. 	fep->hwp = devm_ioremap_resource(&pdev->dev, r);
48. 	if (IS_ERR(fep->hwp)) {
49. 		ret = PTR_ERR(fep->hwp);
50. 		goto failed_ioremap;
51. 	}

53. 	fep->pdev = pdev;
54. 	fep->dev_id = dev_id++;

56. 	platform_set_drvdata(pdev, ndev);

58. 	fec_enet_of_parse_stop_mode(pdev);

60. 	if (of_get_property(np, "fsl,magic-packet", NULL))
61. 		fep->wol_flag |= FEC_WOL_HAS_MAGIC_PACKET;

63. 	phy_node = of_parse_phandle(np, "phy-handle", 0);
64. 	if (!phy_node && of_phy_is_fixed_link(np)) {
65. 		ret = of_phy_register_fixed_link(np);
66. 		if (ret < 0) {
67. 			dev_err(&pdev->dev,
68. 				"broken fixed-link specification\n");
69. 			goto failed_phy;
70. 		}
71. 		phy_node = of_node_get(np);
72. 	}
73. 	fep->phy_node = phy_node;

75. 	ret = of_get_phy_mode(pdev->dev.of_node);
76. 	if (ret < 0) {
77. 		pdata = dev_get_platdata(&pdev->dev);
78. 		if (pdata)
79. 			fep->phy_interface = pdata->phy;
80. 		else
81. 			fep->phy_interface = PHY_INTERFACE_MODE_MII;
82. 	} else {
83. 		fep->phy_interface = ret;
84. 	}

86. 	fep->clk_ipg = devm_clk_get(&pdev->dev, "ipg");
87. 	if (IS_ERR(fep->clk_ipg)) {
88. 		ret = PTR_ERR(fep->clk_ipg);
89. 		goto failed_clk;
90. 	}

92. 	fep->clk_ahb = devm_clk_get(&pdev->dev, "ahb");
93. 	if (IS_ERR(fep->clk_ahb)) {
94. 		ret = PTR_ERR(fep->clk_ahb);
95. 		goto failed_clk;
96. 	}

98. 	fep->itr_clk_rate = clk_get_rate(fep->clk_ahb);

100. 	/* enet_out is optional, depends on board */
101. 	fep->clk_enet_out = devm_clk_get(&pdev->dev, "enet_out");
102. 	if (IS_ERR(fep->clk_enet_out))
103. 		fep->clk_enet_out = NULL;

105. 	fep->ptp_clk_on = false;
106. 	mutex_init(&fep->ptp_clk_mutex);

108. 	/* clk_ref is optional, depends on board */
109. 	fep->clk_ref = devm_clk_get(&pdev->dev, "enet_clk_ref");
110. 	if (IS_ERR(fep->clk_ref))
111. 		fep->clk_ref = NULL;

113. 	fep->bufdesc_ex = fep->quirks & FEC_QUIRK_HAS_BUFDESC_EX;
114. 	fep->clk_ptp = devm_clk_get(&pdev->dev, "ptp");
115. 	if (IS_ERR(fep->clk_ptp)) {
116. 		fep->clk_ptp = NULL;
117. 		fep->bufdesc_ex = false;
118. 	}

120. 	pm_runtime_enable(&pdev->dev);
121. 	ret = fec_enet_clk_enable(ndev, true);
122. 	if (ret)
123. 		goto failed_clk;

125. 	fep->reg_phy = devm_regulator_get(&pdev->dev, "phy");
126. 	if (!IS_ERR(fep->reg_phy)) {
127. 		ret = regulator_enable(fep->reg_phy);
128. 		if (ret) {
129. 			dev_err(&pdev->dev,
130. 				"Failed to enable phy regulator: %d\n", ret);
131. 			goto failed_regulator;
132. 		}
133. 	} else {
134. 		fep->reg_phy = NULL;
135. 	}

137. 	fec_reset_phy(pdev);

139. 	if (fep->bufdesc_ex)
140. 		fec_ptp_init(pdev);

142. 	ret = fec_enet_init(ndev);
143. 	if (ret)
144. 		goto failed_init;

146. 	for (i = 0; i < FEC_IRQ_NUM; i++) {
147. 		irq = platform_get_irq(pdev, i);
148. 		if (irq < 0) {
149. 			if (i)
150. 				break;
151. 			ret = irq;
152. 			goto failed_irq;
153. 		}
154. 		ret = devm_request_irq(&pdev->dev, irq, fec_enet_interrupt,
155. 				       0, pdev->name, ndev);
156. 		if (ret)
157. 			goto failed_irq;

159. 		fep->irq[i] = irq;
160. 	}

162. 	ret = of_property_read_u32(np, "fsl,wakeup_irq", &irq);
163. 	if (!ret && irq < FEC_IRQ_NUM)
164. 		fep->wake_irq = fep->irq[irq];
165. 	else
166. 		fep->wake_irq = fep->irq[0];

168. 	init_completion(&fep->mdio_done);
169. 	ret = fec_enet_mii_init(pdev);
170. 	if (ret)
171. 		goto failed_mii_init;

173. 	/* Carrier starts down, phylib will bring it up */
174. 	netif_carrier_off(ndev);
175. 	fec_enet_clk_enable(ndev, false);
176. 	pinctrl_pm_select_sleep_state(&pdev->dev);

178. 	ret = register_netdev(ndev);
179. 	if (ret)
180. 		goto failed_register;

182. 	device_init_wakeup(&ndev->dev, fep->wol_flag &
183. 			   FEC_WOL_HAS_MAGIC_PACKET);

185. 	if (fep->bufdesc_ex && fep->ptp_clock)
186. 		netdev_info(ndev, "registered PHC device %d\n", fep->dev_id);

188. 	fep->rx_copybreak = COPYBREAK_DEFAULT;
189. 	INIT_WORK(&fep->tx_timeout_work, fec_enet_timeout_work);
190. 	return 0;

192. failed_register:
193. 	fec_enet_mii_remove(fep);
194. failed_mii_init:
195. failed_irq:
196. failed_init:
197. 	if (fep->reg_phy)
198. 		regulator_disable(fep->reg_phy);
199. failed_regulator:
200. 	fec_enet_clk_enable(ndev, false);
201. failed_clk:
202. failed_phy:
203. 	of_node_put(phy_node);
204. failed_ioremap:
205. 	free_netdev(ndev);

207. 	return ret;
208. }

  

这个 probe中主要做了哪些事情呢？以下我只写主要的一些，不是全部的。

struct net_device *ndev;这里对net_device进行初始化，分配内存

1. 	/* Init network device */
2. 	ndev = alloc_etherdev_mqs(sizeof(struct fec_enet_private),
3. 				  num_tx_qs, num_rx_qs);
4. 	if (!ndev)
5. 		return -ENOMEM;

  

接下来做如下动作，注释都很明显，我就不解释了，

struct fec_enet_private *fep;

2. struct fec_enet_private *fep;
    

5. 1. 	/* setup board info structure */
    
6. 	fep = netdev_priv(ndev);
    
7. 1. /**
    
8.  *	netdev_priv - access network device private data
    
9.  *	@dev: network device
    
10.  *
    
11.  * Get network device private data
    
12.  */
    
13. static inline void *netdev_priv(const struct net_device *dev)
    
14. {
    
15. 	return (char *)dev + ALIGN(sizeof(struct net_device), NETDEV_ALIGN);
    
16. }
    

  

获取时钟：

1. 	fep->clk_ipg = devm_clk_get(&pdev->dev, "ipg");
2. 	if (IS_ERR(fep->clk_ipg)) {
3. 		ret = PTR_ERR(fep->clk_ipg);
4. 		goto failed_clk;
5. 	}

7. 	fep->clk_ahb = devm_clk_get(&pdev->dev, "ahb");
8. 	if (IS_ERR(fep->clk_ahb)) {
9. 		ret = PTR_ERR(fep->clk_ahb);
10. 		goto failed_clk;
11. 	}

13. 	fep->itr_clk_rate = clk_get_rate(fep->clk_ahb);

15. 	/* enet_out is optional, depends on board */
16. 	fep->clk_enet_out = devm_clk_get(&pdev->dev, "enet_out");
17. 	if (IS_ERR(fep->clk_enet_out))
18. 		fep->clk_enet_out = NULL;

20. 	fep->ptp_clk_on = false;
21. 	mutex_init(&fep->ptp_clk_mutex);

23. 	/* clk_ref is optional, depends on board */
24. 	fep->clk_ref = devm_clk_get(&pdev->dev, "enet_clk_ref");
25. 	if (IS_ERR(fep->clk_ref))
26. 		fep->clk_ref = NULL;

28. 	fep->bufdesc_ex = fep->quirks & FEC_QUIRK_HAS_BUFDESC_EX;
29. 	fep->clk_ptp = devm_clk_get(&pdev->dev, "ptp");
30. 	if (IS_ERR(fep->clk_ptp)) {
31. 		fep->clk_ptp = NULL;
32. 		fep->bufdesc_ex = false;
33. 	}

35. 	pm_runtime_enable(&pdev->dev);
36. 	ret = fec_enet_clk_enable(ndev, true);
37. 	if (ret)
38. 		goto failed_clk;

  

使能clk，fec_enet_clk_enable(ndev, true);

复位phy，硬件复位，具体取决于不同phy的datasheet的reset时序，fec_reset_phy(pdev);

一些初始化动作，fec_enet_init(ndev);由于代码注释写的很好，上代码：

  

1.  /*
2.   * XXX:  We need to clean up on failure exits here.
3.   *
4.   */
5. static int fec_enet_init(struct net_device *ndev)
6. {
7. 	struct fec_enet_private *fep = netdev_priv(ndev);
8. 	struct fec_enet_priv_tx_q *txq;
9. 	struct fec_enet_priv_rx_q *rxq;
10. 	struct bufdesc *cbd_base;
11. 	dma_addr_t bd_dma;
12. 	int bd_size;
13. 	unsigned int i;

15. #if defined(CONFIG_ARM)
16. 	fep->rx_align = 0xf;
17. 	fep->tx_align = 0xf;
18. #else
19. 	fep->rx_align = 0x3;
20. 	fep->tx_align = 0x3;
21. #endif

23. 	fec_enet_alloc_queue(ndev);

25. 	if (fep->bufdesc_ex)
26. 		fep->bufdesc_size = sizeof(struct bufdesc_ex);
27. 	else
28. 		fep->bufdesc_size = sizeof(struct bufdesc);
29. 	bd_size = (fep->total_tx_ring_size + fep->total_rx_ring_size) *
30. 			fep->bufdesc_size;

32. 	/* Allocate memory for buffer descriptors. */
33. 	cbd_base = dma_alloc_coherent(NULL, bd_size, &bd_dma,
34. 				      GFP_KERNEL);
35. 	if (!cbd_base) {
36. 		return -ENOMEM;
37. 	}

39. 	memset(cbd_base, 0, bd_size);

41. 	/* Get the Ethernet address */
42. 	fec_get_mac(ndev);
43. 	/* make sure MAC we just acquired is programmed into the hw */
44. 	fec_set_mac_address(ndev, NULL);

46. 	/* Set receive and transmit descriptor base. */
47. 	for (i = 0; i < fep->num_rx_queues; i++) {
48. 		rxq = fep->rx_queue[i];
49. 		rxq->index = i;
50. 		rxq->rx_bd_base = (struct bufdesc *)cbd_base;
51. 		rxq->bd_dma = bd_dma;
52. 		if (fep->bufdesc_ex) {
53. 			bd_dma += sizeof(struct bufdesc_ex) * rxq->rx_ring_size;
54. 			cbd_base = (struct bufdesc *)
55. 				(((struct bufdesc_ex *)cbd_base) + rxq->rx_ring_size);
56. 		} else {
57. 			bd_dma += sizeof(struct bufdesc) * rxq->rx_ring_size;
58. 			cbd_base += rxq->rx_ring_size;
59. 		}
60. 	}

62. 	for (i = 0; i < fep->num_tx_queues; i++) {
63. 		txq = fep->tx_queue[i];
64. 		txq->index = i;
65. 		txq->tx_bd_base = (struct bufdesc *)cbd_base;
66. 		txq->bd_dma = bd_dma;
67. 		if (fep->bufdesc_ex) {
68. 			bd_dma += sizeof(struct bufdesc_ex) * txq->tx_ring_size;
69. 			cbd_base = (struct bufdesc *)
70. 			 (((struct bufdesc_ex *)cbd_base) + txq->tx_ring_size);
71. 		} else {
72. 			bd_dma += sizeof(struct bufdesc) * txq->tx_ring_size;
73. 			cbd_base += txq->tx_ring_size;
74. 		}
75. 	}

78. 	/* The FEC Ethernet specific entries in the device structure */
79. 	ndev->watchdog_timeo = TX_TIMEOUT;
80. 	ndev->netdev_ops = &fec_netdev_ops;
81. 	ndev->ethtool_ops = &fec_enet_ethtool_ops;

83. 	writel(FEC_RX_DISABLED_IMASK, fep->hwp + FEC_IMASK);
84. 	netif_napi_add(ndev, &fep->napi, fec_enet_rx_napi, NAPI_POLL_WEIGHT);

86. 	if (fep->quirks & FEC_QUIRK_HAS_VLAN)
87. 		/* enable hw VLAN support */
88. 		ndev->features |= NETIF_F_HW_VLAN_CTAG_RX;

90. 	if (fep->quirks & FEC_QUIRK_HAS_CSUM) {
91. 		ndev->gso_max_segs = FEC_MAX_TSO_SEGS;

93. 		/* enable hw accelerator */
94. 		ndev->features |= (NETIF_F_IP_CSUM | NETIF_F_IPV6_CSUM
95. 				| NETIF_F_RXCSUM | NETIF_F_SG | NETIF_F_TSO);
96. 		fep->csum_flags |= FLAG_RX_CSUM_ENABLED;
97. 	}

99. 	if (fep->quirks & FEC_QUIRK_HAS_AVB) {
100. 		fep->tx_align = 0;
101. 		fep->rx_align = 0x3f;
102. 	}

104. 	ndev->hw_features = ndev->features;

106. 	fec_restart(ndev);

108. 	return 0;
109. }

  

## 1.2 获取以太网mac地址

  

1. 	/* Get the Ethernet address */
2. 	fec_get_mac(ndev);
3. 	/* make sure MAC we just acquired is programmed into the hw */
4. 	fec_set_mac_address(ndev, NULL);

  

这里获取mac 地址的流程我要说一下，之前有讲过流程，我这里再提一下：

1）  模块化参数设置，如果没有跳到步骤2

1. static void fec_get_mac(struct net_device *ndev)
2. {
3. 	struct fec_enet_private *fep = netdev_priv(ndev);
4. 	struct fec_platform_data *pdata = dev_get_platdata(&fep->pdev->dev);
5. 	unsigned char *iap, tmpaddr[ETH_ALEN];

7. 	/*
8. 	 * try to get mac address in following order:
9. 	 *
10. 	 * 1) module parameter via kernel command line in form
11. 	 *    fec.macaddr=0x00,0x04,0x9f,0x01,0x30,0xe0
12. 	 */
13. 	iap = macaddr;

  

  

2）  device tree中设置，如果没有跳到步骤3；

  

1. 1. 	/*
    
2. 	 * 2) from device tree data
    
3. 	 */
    
4. 	if (!is_valid_ether_addr(iap)) {
    
5. 		struct device_node *np = fep->pdev->dev.of_node;
    
6. 		if (np) {
    
7. 			const char *mac = of_get_mac_address(np);
    
8. 			if (mac)
    
9. 				iap = (unsigned char *) mac;
    
10. 		}
    
11. 	}
    

  

3）  from flash / fuse / via platform data，如果没有跳到步骤4；

1. 	/*
2. 	 * 3) from flash or fuse (via platform data)
3. 	 */
4. 	if (!is_valid_ether_addr(iap)) {
5. #ifdef CONFIG_M5272
6. 		if (FEC_FLASHMAC)
7. 			iap = (unsigned char *)FEC_FLASHMAC;
8. #else
9. 		if (pdata)
10. 			iap = (unsigned char *)&pdata->mac;
11. #endif
12. 	}

  

4）  FEC mac registers set by bootloader===》即靠usb方式下载mac address ，如果没有跳到步骤5；

1. 	/*
2. 	 * 4) FEC mac registers set by bootloader
3. 	 */
4. 	if (!is_valid_ether_addr(iap)) {
5. 		*((__be32 *) &tmpaddr[0]) =
6. 			cpu_to_be32(readl(fep->hwp + FEC_ADDR_LOW));
7. 		*((__be16 *) &tmpaddr[4]) =
8. 			cpu_to_be16(readl(fep->hwp + FEC_ADDR_HIGH) >> 16);
9. 		iap = &tmpaddr[0];
10. 	}

  

5）靠kernel算一个随机数mac address出来，然后写入mac

1. 	/*
2. 	 * 5) random mac address
3. 	 */
4. 	if (!is_valid_ether_addr(iap)) {
5. 		/* Report it and use a random ethernet address instead */
6. 		netdev_err(ndev, "Invalid MAC address: %pM\n", iap);
7. 		eth_hw_addr_random(ndev);
8. 		netdev_info(ndev, "Using random MAC address: %pM\n",
9. 			    ndev->dev_addr);
10. 		return;
11. 	}

13. 	memcpy(ndev->dev_addr, iap, ETH_ALEN);

15. 	/* Adjust MAC if using macaddr */
16. 	if (iap == macaddr)
17. 		 ndev->dev_addr[ETH_ALEN-1] = macaddr[ETH_ALEN-1] + fep->dev_id;

  

所以最后一种方式就是kernel会自己算一个mac地址出来，我这里有个前提是这个是freescale（现在被nxp收购了）的控制器代码这样写的，我不确定其他厂商的控制器是否也是这样的流程，技术讲究严谨，所以这里不能一概而论。当然这个mac 地址也是可以用户自己在dts中进行自行配置的。

2. 1. /**
    
3.  * eth_hw_addr_random - Generate software assigned random Ethernet and
    
4.  * set device flag
    
5.  * @dev: pointer to net_device structure
    
6.  *
    
7.  * Generate a random Ethernet address (MAC) to be used by a net device
    
8.  * and set addr_assign_type so the state can be read by sysfs and be
    
9.  * used by userspace.
    
10.  */
    
11. static inline void eth_hw_addr_random(struct net_device *dev)
    
12. {
    
13. 	dev->addr_assign_type = NET_ADDR_RANDOM;
    
14. 	eth_random_addr(dev->dev_addr);
    
15. }
    
16. 1. /**
    
17.  * eth_random_addr - Generate software assigned random Ethernet address
    
18.  * @addr: Pointer to a six-byte array containing the Ethernet address
    
19.  *
    
20.  * Generate a random Ethernet address (MAC) that is not multicast
    
21.  * and has the local assigned bit set.
    
22.  */
    
23. static inline void eth_random_addr(u8 *addr)
    
24. {
    
25. 	get_random_bytes(addr, ETH_ALEN);
    
26. 	addr[0] &= 0xfe;	/* clear multicast bit */
    
27. 	addr[0] |= 0x02;	/* set local assignment bit (IEEE802) */
    
28. }
    

  

这里就是kernel随机数的接口了，会总随机池中获取一个随机数并返回。

  

1. /*
2.  * This function is the exported kernel interface.  It returns some
3.  * number of good random numbers, suitable for key generation, seeding
4.  * TCP sequence numbers, etc.  It does not rely on the hardware random
5.  * number generator.  For random bytes direct from the hardware RNG
6.  * (when available), use get_random_bytes_arch().
7.  */
8. void get_random_bytes(void *buf, int nbytes)
9. {
10. #if DEBUG_RANDOM_BOOT > 0
11. 	if (unlikely(nonblocking_pool.initialized == 0))
12. 		printk(KERN_NOTICE "random: %pF get_random_bytes called "
13. 		       "with %d bits of entropy available\n",
14. 		       (void *) _RET_IP_,
15. 		       nonblocking_pool.entropy_total);
16. #endif
17. 	trace_get_random_bytes(nbytes, _RET_IP_);
18. 	extract_entropy(&nonblocking_pool, buf, nbytes, 0, 0);
19. }
20. EXPORT_SYMBOL(get_random_bytes);

  

大家看到那些获取mac address的步骤中有这样的函数

is_valid_ether_addr，用来检测以太网地址是否正确的

  

1. /**
2.  * is_valid_ether_addr - Determine if the given Ethernet address is valid
3.  * @addr: Pointer to a six-byte array containing the Ethernet address
4.  *
5.  * Check that the Ethernet address (MAC) is not 00:00:00:00:00:00, is not
6.  * a multicast address, and is not FF:FF:FF:FF:FF:FF.
7.  *
8.  * Return true if the address is valid.
9.  *
10.  * Please note: addr must be aligned to u16.
11.  */
12. static inline bool is_valid_ether_addr(const u8 *addr)
13. {
14. 	/* FF:FF:FF:FF:FF:FF is a multicast address so we don't need to
15. 	 * explicitly check for it here. */
16. 	return !is_multicast_ether_addr(addr) && !is_zero_ether_addr(addr);
17. }

  

因此我们从代码中可以看出，内核认为全0或者是全FF的以太网地址是不正确的，

获取了mac 地址后，就会通过寄存器写入mac中

  

1. /* Set a MAC change in hardware. */
2. static int
3. fec_set_mac_address(struct net_device *ndev, void *p)
4. {
5. 	struct fec_enet_private *fep = netdev_priv(ndev);
6. 	struct sockaddr *addr = p;

8. 	if (addr) {
9. 		if (!is_valid_ether_addr(addr->sa_data))
10. 			return -EADDRNOTAVAIL;
11. 		memcpy(ndev->dev_addr, addr->sa_data, ndev->addr_len);
12. 	}

14. 	/* Add netif status check here to avoid system hang in below case:
15. 	 * ifconfig ethx down; ifconfig ethx hw ether xx:xx:xx:xx:xx:xx;
16. 	 * After ethx down, fec all clocks are gated off and then register
17. 	 * access causes system hang.
18. 	 */
19. 	if (!netif_running(ndev))
20. 		return 0;

22. 	writel(ndev->dev_addr[3] | (ndev->dev_addr[2] << 8) |
23. 		(ndev->dev_addr[1] << 16) | (ndev->dev_addr[0] << 24),
24. 		fep->hwp + FEC_ADDR_LOW);
25. 	writel((ndev->dev_addr[5] << 16) | (ndev->dev_addr[4] << 24),
26. 		fep->hwp + FEC_ADDR_HIGH);
27. 	return 0;
28. }

  

CONFIG_ARCH_MXC因为我们使用的是这个宏，因此：

  

1. #if defined(CONFIG_M523x) || defined(CONFIG_M527x) || defined(CONFIG_M528x) || \
2.     defined(CONFIG_M520x) || defined(CONFIG_M532x) || \
3.     defined(CONFIG_ARCH_MXC) || defined(CONFIG_SOC_IMX28)
4. /*
5.  *	Just figures, Motorola would have to change the offsets for
6.  *	registers in the same peripheral device on different models
7.  *	of the ColdFire!
8.  */
9. #define FEC_IEVENT		0x004 /* Interrupt event reg */
10. #define FEC_IMASK		0x008 /* Interrupt mask reg */
11. #define FEC_R_DES_ACTIVE_0	0x010 /* Receive descriptor reg */
12. #define FEC_X_DES_ACTIVE_0	0x014 /* Transmit descriptor reg */
13. #define FEC_ECNTRL		0x024 /* Ethernet control reg */
14. #define FEC_MII_DATA		0x040 /* MII manage frame reg */
15. #define FEC_MII_SPEED		0x044 /* MII speed control reg */
16. #define FEC_MIB_CTRLSTAT	0x064 /* MIB control/status reg */
17. #define FEC_R_CNTRL		0x084 /* Receive control reg */
18. #define FEC_X_CNTRL		0x0c4 /* Transmit Control reg */
19. #define FEC_ADDR_LOW		0x0e4 /* Low 32bits MAC address */
20. #define FEC_ADDR_HIGH		0x0e8 /* High 16bits MAC address */

  

否则：

  

1. #define FEC_ADDR_LOW		0x3c0 /* Low 32bits MAC address */
2. #define FEC_ADDR_HIGH		0x3c4 /* High 16bits MAC address */

  

设置buffer传输的基地址

1. 	/* Set receive and transmit descriptor base. */
2. 	for (i = 0; i < fep->num_rx_queues; i++) {
3. 		rxq = fep->rx_queue[i];
4. 		rxq->index = i;
5. 		rxq->rx_bd_base = (struct bufdesc *)cbd_base;
6. 		rxq->bd_dma = bd_dma;
7. 		if (fep->bufdesc_ex) {
8. 			bd_dma += sizeof(struct bufdesc_ex) * rxq->rx_ring_size;
9. 			cbd_base = (struct bufdesc *)
10. 				(((struct bufdesc_ex *)cbd_base) + rxq->rx_ring_size);
11. 		} else {
12. 			bd_dma += sizeof(struct bufdesc) * rxq->rx_ring_size;
13. 			cbd_base += rxq->rx_ring_size;
14. 		}
15. 	}

17. 	for (i = 0; i < fep->num_tx_queues; i++) {
18. 		txq = fep->tx_queue[i];
19. 		txq->index = i;
20. 		txq->tx_bd_base = (struct bufdesc *)cbd_base;
21. 		txq->bd_dma = bd_dma;
22. 		if (fep->bufdesc_ex) {
23. 			bd_dma += sizeof(struct bufdesc_ex) * txq->tx_ring_size;
24. 			cbd_base = (struct bufdesc *)
25. 			 (((struct bufdesc_ex *)cbd_base) + txq->tx_ring_size);
26. 		} else {
27. 			bd_dma += sizeof(struct bufdesc) * txq->tx_ring_size;
28. 			cbd_base += txq->tx_ring_size;
29. 		}
30. 	}

  

提供一些以太网控制器的操作接口，应用层调用socket通信最终的实现接口方式，并且提供开源工具ethtool工具的底层操作接口支持

  

1. 	/* The FEC Ethernet specific entries in the device structure */
2. 	ndev->watchdog_timeo = TX_TIMEOUT;
3. 	ndev->netdev_ops = &fec_netdev_ops;
4. 	ndev->ethtool_ops = &fec_enet_ethtool_ops;

1. static const struct net_device_ops fec_netdev_ops = {
2. 	.ndo_open		= fec_enet_open,
3. 	.ndo_stop		= fec_enet_close,
4. 	.ndo_start_xmit		= fec_enet_start_xmit,
5. 	.ndo_select_queue       = fec_enet_select_queue,
6. 	.ndo_set_rx_mode	= set_multicast_list,
7. 	.ndo_change_mtu		= eth_change_mtu,
8. 	.ndo_validate_addr	= eth_validate_addr,
9. 	.ndo_tx_timeout		= fec_timeout,
10. 	.ndo_set_mac_address	= fec_set_mac_address,
11. 	.ndo_do_ioctl		= fec_enet_ioctl,
12. #ifdef CONFIG_NET_POLL_CONTROLLER
13. 	.ndo_poll_controller	= fec_poll_controller,
14. #endif
15. 	.ndo_set_features	= fec_set_features,
16. };

1. static const struct ethtool_ops fec_enet_ethtool_ops = {
2. 	.get_settings		= fec_enet_get_settings,
3. 	.set_settings		= fec_enet_set_settings,
4. 	.get_drvinfo		= fec_enet_get_drvinfo,
5. 	.get_regs_len		= fec_enet_get_regs_len,
6. 	.get_regs		= fec_enet_get_regs,
7. 	.nway_reset		= fec_enet_nway_reset,
8. 	.get_link		= ethtool_op_get_link,
9. 	.get_coalesce		= fec_enet_get_coalesce,
10. 	.set_coalesce		= fec_enet_set_coalesce,
11. #ifndef CONFIG_M5272
12. 	.get_pauseparam		= fec_enet_get_pauseparam,
13. 	.set_pauseparam		= fec_enet_set_pauseparam,
14. 	.get_strings		= fec_enet_get_strings,
15. 	.get_ethtool_stats	= fec_enet_get_ethtool_stats,
16. 	.get_sset_count		= fec_enet_get_sset_count,
17. #endif
18. 	.get_ts_info		= fec_enet_get_ts_info,
19. 	.get_tunable		= fec_enet_get_tunable,
20. 	.set_tunable		= fec_enet_set_tunable,
21. 	.get_wol		= fec_enet_get_wol,
22. 	.set_wol		= fec_enet_set_wol,
23. };

  

  

所以有些人用ethtool工具发现不同平台可能不一样，同样的命令有些可能返回不同，或者功能不支持，就可以猜想一下可能是因为不同厂商的控制器驱动这里的实现问题导致，部分接口可能没有实现或者有bug等等，这些就要具体问题具体分析了，有些板子可能某些接口根本都没实现，自然ethtool的一些命令就无法正常使用了。

最后进行控制器对phy的复位动作

fec_restart(ndev);

  

1. /*
2.  * This function is called to start or restart the FEC during a link
3.  * change, transmit timeout, or to reconfigure the FEC.  The network
4.  * packet processing for this device must be stopped before this call.
5.  */
6. static void
7. fec_restart(struct net_device *ndev)
8. {
9. 	struct fec_enet_private *fep = netdev_priv(ndev);
10. 	u32 val;
11. 	u32 temp_mac[2];
12. 	u32 rcntl = OPT_FRAME_SIZE | 0x04;
13. 	u32 ecntl = FEC_ENET_ETHEREN; /* ETHEREN */

15. 	/* Whack a reset.  We should wait for this.
16. 	 * For i.MX6SX SOC, enet use AXI bus, we use disable MAC
17. 	 * instead of reset MAC itself.
18. 	 */
19. 	if (fep->quirks & FEC_QUIRK_HAS_AVB) {
20. 		writel(0, fep->hwp + FEC_ECNTRL);
21. 	} else {
22. 		writel(1, fep->hwp + FEC_ECNTRL);
23. 		udelay(10);
24. 	}

26. 	/*
27. 	 * enet-mac reset will reset mac address registers too,
28. 	 * so need to reconfigure it.
29. 	 */
30. 	if (fep->quirks & FEC_QUIRK_ENET_MAC) {
31. 		memcpy(&temp_mac, ndev->dev_addr, ETH_ALEN);
32. 		writel(cpu_to_be32(temp_mac[0]), fep->hwp + FEC_ADDR_LOW);
33. 		writel(cpu_to_be32(temp_mac[1]), fep->hwp + FEC_ADDR_HIGH);
34. 	}

36. 	/* Clear any outstanding interrupt. */
37. 	writel(0xffffffff, fep->hwp + FEC_IEVENT);

39. 	fec_enet_bd_init(ndev);

41. 	fec_enet_enable_ring(ndev);

43. 	/* Reset tx SKB buffers. */
44. 	fec_enet_reset_skb(ndev);

46. 	/* Enable MII mode */
47. 	if (fep->full_duplex == DUPLEX_FULL) {
48. 		/* FD enable */
49. 		writel(0x04, fep->hwp + FEC_X_CNTRL);
50. 	} else {
51. 		/* No Rcv on Xmit */
52. 		rcntl |= 0x02;
53. 		writel(0x0, fep->hwp + FEC_X_CNTRL);
54. 	}

56. 	/* Set MII speed */
57. 	writel(fep->phy_speed, fep->hwp + FEC_MII_SPEED);

59. #if !defined(CONFIG_M5272)
60. 	if (fep->quirks & FEC_QUIRK_HAS_RACC) {
61. 		/* set RX checksum */
62. 		val = readl(fep->hwp + FEC_RACC);
63. 		if (fep->csum_flags & FLAG_RX_CSUM_ENABLED)
64. 			val |= FEC_RACC_OPTIONS;
65. 		else
66. 			val &= ~FEC_RACC_OPTIONS;
67. 		writel(val, fep->hwp + FEC_RACC);
68. 	}
69. 	writel(PKT_MAXBUF_SIZE, fep->hwp + FEC_FTRL);
70. #endif

72. 	/*
73. 	 * The phy interface and speed need to get configured
74. 	 * differently on enet-mac.
75. 	 */
76. 	if (fep->quirks & FEC_QUIRK_ENET_MAC) {
77. 		/* Enable flow control and length check */
78. 		rcntl |= 0x40000000 | 0x00000020;

80. 		/* RGMII, RMII or MII */
81. 		if (fep->phy_interface == PHY_INTERFACE_MODE_RGMII ||
82. 		    fep->phy_interface == PHY_INTERFACE_MODE_RGMII_ID ||
83. 		    fep->phy_interface == PHY_INTERFACE_MODE_RGMII_RXID ||
84. 		    fep->phy_interface == PHY_INTERFACE_MODE_RGMII_TXID)
85. 			rcntl |= (1 << 6);
86. 		else if (fep->phy_interface == PHY_INTERFACE_MODE_RMII)
87. 			rcntl |= (1 << 8);
88. 		else
89. 			rcntl &= ~(1 << 8);

91. 		/* 1G, 100M or 10M */
92. 		if (fep->phy_dev) {
93. 			if (fep->phy_dev->speed == SPEED_1000)
94. 				ecntl |= (1 << 5);
95. 			else if (fep->phy_dev->speed == SPEED_100)
96. 				rcntl &= ~(1 << 9);
97. 			else
98. 				rcntl |= (1 << 9);
99. 		}
100. 	} else {
101. #ifdef FEC_MIIGSK_ENR
102. 		if (fep->quirks & FEC_QUIRK_USE_GASKET) {
103. 			u32 cfgr;
104. 			/* disable the gasket and wait */
105. 			writel(0, fep->hwp + FEC_MIIGSK_ENR);
106. 			while (readl(fep->hwp + FEC_MIIGSK_ENR) & 4)
107. 				udelay(1);

109. 			/*
110. 			 * configure the gasket:
111. 			 *   RMII, 50 MHz, no loopback, no echo
112. 			 *   MII, 25 MHz, no loopback, no echo
113. 			 */
114. 			cfgr = (fep->phy_interface == PHY_INTERFACE_MODE_RMII)
115. 				? BM_MIIGSK_CFGR_RMII : BM_MIIGSK_CFGR_MII;
116. 			if (fep->phy_dev && fep->phy_dev->speed == SPEED_10)
117. 				cfgr |= BM_MIIGSK_CFGR_FRCONT_10M;
118. 			writel(cfgr, fep->hwp + FEC_MIIGSK_CFGR);

120. 			/* re-enable the gasket */
121. 			writel(2, fep->hwp + FEC_MIIGSK_ENR);
122. 		}
123. #endif
124. 	}

126. #if !defined(CONFIG_M5272)
127. 	/* enable pause frame*/
128. 	if ((fep->pause_flag & FEC_PAUSE_FLAG_ENABLE) ||
129. 	    ((fep->pause_flag & FEC_PAUSE_FLAG_AUTONEG) &&
130. 	     fep->phy_dev && fep->phy_dev->pause)) {
131. 		rcntl |= FEC_ENET_FCE;

133. 		/* set FIFO threshold parameter to reduce overrun */
134. 		writel(FEC_ENET_RSEM_V, fep->hwp + FEC_R_FIFO_RSEM);
135. 		writel(FEC_ENET_RSFL_V, fep->hwp + FEC_R_FIFO_RSFL);
136. 		writel(FEC_ENET_RAEM_V, fep->hwp + FEC_R_FIFO_RAEM);
137. 		writel(FEC_ENET_RAFL_V, fep->hwp + FEC_R_FIFO_RAFL);

139. 		/* OPD */
140. 		writel(FEC_ENET_OPD_V, fep->hwp + FEC_OPD);
141. 	} else {
142. 		rcntl &= ~FEC_ENET_FCE;
143. 	}
144. #endif /* !defined(CONFIG_M5272) */

146. 	writel(rcntl, fep->hwp + FEC_R_CNTRL);

148. 	/* Setup multicast filter. */
149. 	set_multicast_list(ndev);
150. #ifndef CONFIG_M5272
151. 	writel(0, fep->hwp + FEC_HASH_TABLE_HIGH);
152. 	writel(0, fep->hwp + FEC_HASH_TABLE_LOW);
153. #endif

155. 	if (fep->quirks & FEC_QUIRK_ENET_MAC) {
156. 		/* enable ENET endian swap */
157. 		ecntl |= (1 << 8);
158. 		/* enable ENET store and forward mode */
159. 		writel(1 << 8, fep->hwp + FEC_X_WMRK);
160. 	}

162. 	if (fep->bufdesc_ex)
163. 		ecntl |= (1 << 4);

165. #ifndef CONFIG_M5272
166. 	/* Enable the MIB statistic event counters */
167. 	writel(0 << 31, fep->hwp + FEC_MIB_CTRLSTAT);
168. #endif

170. 	/* And last, enable the transmit and receive processing */
171. 	writel(ecntl, fep->hwp + FEC_ECNTRL);
172. 	fec_enet_active_rxring(ndev);

174. 	if (fep->bufdesc_ex)
175. 		fec_ptp_start_cyclecounter(ndev);

177. 	/* Enable interrupts we wish to service */
178. 	if (fep->link)
179. 		writel(FEC_DEFAULT_IMASK, fep->hwp + FEC_IMASK);
180. 	else
181. 		writel(FEC_ENET_MII, fep->hwp + FEC_IMASK);

183. 	/* Init the interrupt coalescing */
184. 	fec_enet_itr_coal_init(ndev);

186. }

  

流程如下：

1) Whack a reset.  We should wait for this. For i.MX6SX SOC, enet use AXI bus, we use disable MAC , instead of reset MAC itself.

2)enet-mac reset will reset mac address registers too, so need to reconfigure it.

3) Clear any outstanding interrupt.

4) Reset tx SKB buffers.

5) Enable MII mode

6) The phy interface and speed need to get configured,

7) configure the gasket: RMII, 50 MHz, no loopback, no echo; MII, 25 MHz, no loopback, no echo

8) Setup multicast filter.

9) And last, enable the transmit and receive processing

10) Enable interrupts we wish to service

11) Init the interrupt coalescing

fec_enet_init函数的流程到此结束

我们继续回归到fec_probe函数，然后是注册中断处理函数

  

  

1. 	for (i = 0; i < FEC_IRQ_NUM; i++) {
2. 		irq = platform_get_irq(pdev, i);
3. 		if (irq < 0) {
4. 			if (i)
5. 				break;
6. 			ret = irq;
7. 			goto failed_irq;
8. 		}
9. 		ret = devm_request_irq(&pdev->dev, irq, fec_enet_interrupt,
10. 				       0, pdev->name, ndev);
11. 		if (ret)
12. 			goto failed_irq;

14. 		fep->irq[i] = irq;
15. 	}

  

  

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

« [以太网驱动的流程浅析(五)-mii_bus初始化以及phy id的获取](http://www.wowotech.net/linux_kenrel/470.html) | [以太网驱动的流程浅析(二)-Ifconfig的详细代码流程](http://www.wowotech.net/linux_kenrel/466.html)»

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
    
    - [《奔跑吧，Linux内核》已经上架预售了](http://www.wowotech.net/tech_discuss/running_kernel.html)
    - [Linux kernel的中断子系统之（六）：ARM中断处理过程](http://www.wowotech.net/irq_subsystem/irq_handler.html)
    - [Linux设备模型(2)_Kobject](http://www.wowotech.net/device_model/kobject.html)
    - [Linux调度器：用户空间接口](http://www.wowotech.net/process_management/scheduler-API.html)
    - [Why Memory Barriers中文翻译（下）](http://www.wowotech.net/kernel_synchronization/why-memory-barrier-2.html)
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