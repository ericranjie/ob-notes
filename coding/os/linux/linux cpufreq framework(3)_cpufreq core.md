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

作者：[wowo](http://www.wowotech.net/author/2 "runangaozhong@163.com") 发布于：2015-7-30 20:58 分类：[电源管理子系统](http://www.wowotech.net/sort/pm_subsystem)

#### 1. 前言

前文（[Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)）从平台驱动工程师的角度，简单的介绍了编写一个cpufreq driver的大概步骤。但要更深入理解、更灵活的使用，必须理解其内部的实现逻辑。

因此，本文将从cpufreq framework core的角度，对cpufreq framework的内部实现做一个简单的分析。

#### 2. 提供给用户空间的接口

cpufreq core的内部实现，比较繁杂，因此我们需要一个分析的突破口，而cpufreq framework提供的API是一个不错的选择。因为API抽象了cpufreq的功能，而所有的cpufreq core、cpufreq driver等模块，都是为这些功能服务的。

由“[linux cpufreq framework(1)_概述](http://www.wowotech.net/pm_subsystem/cpufreq_overview.html)”的介绍可知，cpufreq framework通过sysfs向用户空间提供接口，具体如下：

> /sys/devices/system/cpu/cpu0/cpufreq/  
> |-- affected_cpus  
> |-- cpuinfo_cur_freq  
> |-- cpuinfo_max_freq  
> |-- cpuinfo_min_freq  
> |-- cpuinfo_transition_latency  
> |-- related_cpus  
> |-- scaling_available_frequencies  
> |-- scaling_available_governors  
> |-- scaling_cur_freq  
> |-- scaling_driver  
> |-- scaling_governor  
> |-- scaling_max_freq  
> |-- scaling_min_freq  
> |-- scaling_setspeed  
> `—stats  
>     |-- time_in_state  
>     |-- total_trans  
>     `-- trans_table

1）“cpufreq”目录

在kernel的模型中，cpufreq被抽象为cpu device的一个interface，因此它位于对应的cpu目录（/sys/devices/system/cpu/cpuX）下面。

有些平台，所有cpu core的频率和电压时统一控制的，即改变某个core上的频率，其它core同样受影响。此时只需要实现其中一个core（通常为cpu0）的cpufreq即可，其它core的cpufreq直接是cpu0的符号链接。因此，使用这些API时，随便进入某一个cpu下面的cpufreq目录即可。

而另一些些平台，不同core可以单独控制，这时不同cpu目录下的cpufreq就不一样了。

到底某一个cpufreq可以控制多少cpu core呢？可以通过cpufreq/affected_cpus和cpufreq/related_cpus两个文件查看，其中的区别是：affected_cpus表示该cpufreq影响到哪些cpu core（没有显示处于offline状态的cpu），related_cpus则包括了online+offline的所有core。

2）cpuinfo相关的信息

由“[Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)”的描述可知，cpufreq driver初始化时，会根据frequency table等信息，填充struct cpufreq_policy变量中的struct cpufreq_cpuinfo变量，该变量保存了CPU调频有关的固定信息（不可以在运行过程中修改，主要包括：最大频率（cpuinfo_max_freq）、最小频率（cpuinfo_min_freq）、频率转换延迟（cpuinfo_transition_latency ）。

另外，通过cpuinfo_cur_freq ，可以获取cpu core的当前频率（真实的、cpu的当前运行频率，会通过cpufreq_driver->get回调读取）。

当前，这四个“cpuinfo_”开头的sysfs API，都是只读的。

3）scaling_available_frequencies，获取当前可以配置的频率列表，从frequency table直接读取。readonly。

4） scaling_driver，当前加载的cpufreq driver名称，readonly。

5）scaling_available_governors和scaling_governor，系统中可用的governor列表，以及当前使用的governor。governor相关的内容会在后续详细描述。readonly。

6）scaling_cur_freq，从cpufreq core或者governor的角度，看到的当前频率，和cpuinfo_cur_freq的意义不相同，后面的分析中根据实例在描述。readonly。

7）scaling_max_freq、scaling_min_freq和scaling_setspeed

scaling_max_freq和scaling_min_freq表示调频策略所允许的最大和最小频率，对于可以自动调整频率的cpu，修改它们，就是最终的频率调整。

对不能自动调整频率的cpu，则需要通过其它方式，主动的设置cpu频率，这些都是由具体的governor完成。其中有一个特例：

如果使用的governor是“userspace” governor，则可以通过scaling_setspeed节点，直接修改cpu频率。

#### 3. 频率调整的步骤

开始分析之前，我们先以“userspace” governor为例，介绍一下频率调整的步骤。“userspace”governor是所有governor中最简单的一个，同时又是驱动工程师比较常用的一个，借助它，可以从用户空间修改cpu的频率，操作方法如下（为了简单，以shell脚本的形式给出）：

> cd /sys/devices/system/cpu/cpu0/cpufreq/
> 
> cat cpuinfo_max_freq; cat cpuinfo_min_freq            #获取“物理”上的频率范围 
> 
> cat scaling_available_frequencies                          #获取可用的频率列表  
> cat scaling_available_governors                             #获取可用的governors  
> cat scaling_governor                                             #当前的governor  
> cat cpuinfo_cur_freq; cat scaling_cur_freq              #获取当前的频率信息，可以比较一下是否不同
> 
> cat scaling_max_freq; cat scaling_min_freq           #获取当前调频策略所限定的频率范围
> 
> #假设CPU不可以自动调整频率  
> echo userspace > scaling_governor                      #governor切换为userspace
> 
> #如果需要切换的频率值在scaling_available_frequencies内，且在cpuinfo_max_freq/cpuinfo_min_freq的范围内。
> 
> #如果需要切换的频率不在scaling_max_freq/scaling_min_freq的范围内，修改这两个值  
> echo xxx > scaling_max_freq; echo xxx > scaling_min_freq  
> 
> #最后，设置频率值  
> echo xxx > scaling_setspeed  

#### 4. 内部逻辑分析

**4.1 初始化**

基于linux设备模型的思想，kernel会使用device和driver两个实体，抽象设备及其驱动，当这两个实体同时存在时，则执行driver的初始化接口（即driver开始运行）。cpufreq也不例外，基本遵循了上面的思路。但由于cpufreq是一类比较特殊的设备（它只是cpu device的一个功能，本身并不以任何形式存在），在实现上，就有点迂回。

首先，driver的抽象比较容易理解，就是我们在“[Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)”中描述的struct cpufreq_driver结构。那device呢？先看看下面的图片：

注1：该图片不包括CPU hotplug的情况，hotplug时，会走两外一个流程，大概原理类似，本文就不详细介绍这种情况了。

[![cpufreq init flow](http://www.wowotech.net/content/uploadfile/201507/0de6805816b4e770c67f13c65b26134220150730125809.gif "cpufreq init flow")](http://www.wowotech.net/content/uploadfile/201507/b4fe5ff46aa4f9c3ec019dbc7c3538a120150730125808.gif)1）cpufreq_interface

前面讲过，cpufreq driver注册时，会调用subsys_interface_register接口，注册一个subsystem interface，该interface的定义如下：

  1: /* drivers/cpufreq/cpufreq.c */

  2: static struct subsys_interface cpufreq_interface = {

  3: 	.name		= "cpufreq",

  4: 	.subsys		= &cpu_subsys,

  5: 	.add_dev	= cpufreq_add_dev,

  6: 	.remove_dev	= cpufreq_remove_dev,

  7: };

> 该interface的subsys是“cpu_subsys”，就是cpu bus（struct bus_type cpu_subsys），提供了add_dev和remove_dev两个回调函数。

由“[Linux设备模型(6)_Bus](http://www.wowotech.net/linux_kenrel/bus.html)”中的描述可知，当bus下有设备probe的时候（此处为cpu device的probe），会调用其下所有interface的add_dev回调函数。物理意义是什么呢？

> cpufreq是cpu device的一个功能，当cpu device开始枚举时，当然要创建代表该功能（cpufreq）的设备。而这个设备的具体形态，只有该功能相关的代码（cpufreq core）知道，因此就只能交给它了。

2）__cpufreq_add_dev

由上面图片所示的流程可知，cpufreq设备的添加，最终是由__cpufreq_add_dev接口完成的，该接口的主要功能，是创建一个代表该cpufreq的设备（struct cpufreq_policy类型的变量），并以它为参数，调用cpufreq_driver的init接口（可参考“[Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)”），对它进行初始化。如下：

  1: static int __cpufreq_add_dev(struct device *dev, struct subsys_interface *sif)

  2: {

  3: 	unsigned int j, cpu = dev->id;

  4: 	int ret = -ENOMEM;

  5: 	struct cpufreq_policy *policy;

  6: 	unsigned long flags;

  7: 	bool recover_policy = cpufreq_suspended;

  8: #ifdef CONFIG_HOTPLUG_CPU

  9: 	struct cpufreq_policy *tpolicy;

 10: #endif

 11: 

 12: 	if (cpu_is_offline(cpu))

 13: 		return 0;

 14: 

 15: 	pr_debug("adding CPU %u\n", cpu);

 16: 

 17: #ifdef CONFIG_SMP

 18: 	/* check whether a different CPU already registered this

 19: 	 * CPU because it is in the same boat. */

 20: 	policy = cpufreq_cpu_get(cpu);

 21: 	if (unlikely(policy)) {

 22: 		cpufreq_cpu_put(policy);

 23: 		return 0;

 24: 	}

 25: #endif

 26: 

 27: 	if (!down_read_trylock(&cpufreq_rwsem))

 28: 		return 0;

 29: 

 30: #ifdef CONFIG_HOTPLUG_CPU

 31: 	/* Check if this cpu was hot-unplugged earlier and has siblings */

 32: 	read_lock_irqsave(&cpufreq_driver_lock, flags);

 33: 	list_for_each_entry(tpolicy, &cpufreq_policy_list, policy_list) {

 34: 		if (cpumask_test_cpu(cpu, tpolicy->related_cpus)) {

 35: 			read_unlock_irqrestore(&cpufreq_driver_lock, flags);

 36: 			ret = cpufreq_add_policy_cpu(tpolicy, cpu, dev);

 37: 			up_read(&cpufreq_rwsem);

 38: 			return ret;

 39: 		}

 40: 	}

 41: 	read_unlock_irqrestore(&cpufreq_driver_lock, flags);

 42: #endif

 43: 

 44: 	/*

 45: 	 * Restore the saved policy when doing light-weight init and fall back

 46: 	 * to the full init if that fails.

 47: 	 */

 48: 	policy = recover_policy ? cpufreq_policy_restore(cpu) : NULL;

 49: 	if (!policy) {

 50: 		recover_policy = false;

 51: 		policy = cpufreq_policy_alloc();

 52: 		if (!policy)

 53: 			goto nomem_out;

 54: 	}

 55: 

 56: 	/*

 57: 	 * In the resume path, since we restore a saved policy, the assignment

 58: 	 * to policy->cpu is like an update of the existing policy, rather than

 59: 	 * the creation of a brand new one. So we need to perform this update

 60: 	 * by invoking update_policy_cpu().

 61: 	 */

 62: 	if (recover_policy && cpu != policy->cpu)

 63: 		WARN_ON(update_policy_cpu(policy, cpu, dev));

 64: 	else

 65: 		policy->cpu = cpu;

 66: 

 67: 	cpumask_copy(policy->cpus, cpumask_of(cpu));

 68: 

 69: 	init_completion(&policy->kobj_unregister);

 70: 	INIT_WORK(&policy->update, handle_update);

 71: 

 72: 	/* call driver. From then on the cpufreq must be able

 73: 	 * to accept all calls to ->verify and ->setpolicy for this CPU

 74: 	 */

 75: 	ret = cpufreq_driver->init(policy);

 76: 	if (ret) {

 77: 		pr_debug("initialization failed\n");

 78: 		goto err_set_policy_cpu;

 79: 	}

 80: 

 81: 	/* related cpus should atleast have policy->cpus */

 82: 	cpumask_or(policy->related_cpus, policy->related_cpus, policy->cpus);

 83: 

 84: 	/*

 85: 	 * affected cpus must always be the one, which are online. We aren't

 86: 	 * managing offline cpus here.

 87: 	 */

 88: 	cpumask_and(policy->cpus, policy->cpus, cpu_online_mask);

 89: 

 90: 	if (!recover_policy) {

 91: 		policy->user_policy.min = policy->min;

 92: 		policy->user_policy.max = policy->max;

 93: 	}

 94: 

 95: 	down_write(&policy->rwsem);

 96: 	write_lock_irqsave(&cpufreq_driver_lock, flags);

 97: 	for_each_cpu(j, policy->cpus)

 98: 		per_cpu(cpufreq_cpu_data, j) = policy;

 99: 	write_unlock_irqrestore(&cpufreq_driver_lock, flags);

100: 

101: 	if (cpufreq_driver->get && !cpufreq_driver->setpolicy) {

102: 		policy->cur = cpufreq_driver->get(policy->cpu);

103: 		if (!policy->cur) {

104: 			pr_err("%s: ->get() failed\n", __func__);

105: 			goto err_get_freq;

106: 		}

107: 	}

108: 

109: 	/*

110: 	 * Sometimes boot loaders set CPU frequency to a value outside of

111: 	 * frequency table present with cpufreq core. In such cases CPU might be

112: 	 * unstable if it has to run on that frequency for long duration of time

113: 	 * and so its better to set it to a frequency which is specified in

114: 	 * freq-table. This also makes cpufreq stats inconsistent as

115: 	 * cpufreq-stats would fail to register because current frequency of CPU

116: 	 * isn't found in freq-table.

117: 	 *

118: 	 * Because we don't want this change to effect boot process badly, we go

119: 	 * for the next freq which is >= policy->cur ('cur' must be set by now,

120: 	 * otherwise we will end up setting freq to lowest of the table as 'cur'

121: 	 * is initialized to zero).

122: 	 *

123: 	 * We are passing target-freq as "policy->cur - 1" otherwise

124: 	 * __cpufreq_driver_target() would simply fail, as policy->cur will be

125: 	 * equal to target-freq.

126: 	 */

127: 	if ((cpufreq_driver->flags & CPUFREQ_NEED_INITIAL_FREQ_CHECK)

128: 	    && has_target()) {

129: 		/* Are we running at unknown frequency ? */

130: 		ret = cpufreq_frequency_table_get_index(policy, policy->cur);

131: 		if (ret == -EINVAL) {

132: 			/* Warn user and fix it */

133: 			pr_warn("%s: CPU%d: Running at unlisted freq: %u KHz\n",

134: 				__func__, policy->cpu, policy->cur);

135: 			ret = __cpufreq_driver_target(policy, policy->cur - 1,

136: 				CPUFREQ_RELATION_L);

137: 

138: 			/*

139: 			 * Reaching here after boot in a few seconds may not

140: 			 * mean that system will remain stable at "unknown"

141: 			 * frequency for longer duration. Hence, a BUG_ON().

142: 			 */

143: 			BUG_ON(ret);

144: 			pr_warn("%s: CPU%d: Unlisted initial frequency changed to: %u KHz\n",

145: 				__func__, policy->cpu, policy->cur);

146: 		}

147: 	}

148: 

149: 	blocking_notifier_call_chain(&cpufreq_policy_notifier_list,

150: 				     CPUFREQ_START, policy);

151: 

152: 	if (!recover_policy) {

153: 		ret = cpufreq_add_dev_interface(policy, dev);

154: 		if (ret)

155: 			goto err_out_unregister;

156: 		blocking_notifier_call_chain(&cpufreq_policy_notifier_list,

157: 				CPUFREQ_CREATE_POLICY, policy);

158: 	}

159: 

160: 	write_lock_irqsave(&cpufreq_driver_lock, flags);

161: 	list_add(&policy->policy_list, &cpufreq_policy_list);

162: 	write_unlock_irqrestore(&cpufreq_driver_lock, flags);

163: 

164: 	cpufreq_init_policy(policy);

165: 

166: 	if (!recover_policy) {

167: 		policy->user_policy.policy = policy->policy;

168: 		policy->user_policy.governor = policy->governor;

169: 	}

170: 	up_write(&policy->rwsem);

171: 

172: 	kobject_uevent(&policy->kobj, KOBJ_ADD);

173: 	up_read(&cpufreq_rwsem);

174: 

175: 	pr_debug("initialization complete\n");

176: 

177: 	return 0;

178: 

179: err_out_unregister:

180: err_get_freq:

181: 	write_lock_irqsave(&cpufreq_driver_lock, flags);

182: 	for_each_cpu(j, policy->cpus)

183: 		per_cpu(cpufreq_cpu_data, j) = NULL;

184: 	write_unlock_irqrestore(&cpufreq_driver_lock, flags);

185: 

186: 	up_write(&policy->rwsem);

187: 

188: 	if (cpufreq_driver->exit)

189: 		cpufreq_driver->exit(policy);

190: err_set_policy_cpu:

191: 	if (recover_policy) {

192: 		/* Do not leave stale fallback data behind. */

193: 		per_cpu(cpufreq_cpu_data_fallback, cpu) = NULL;

194: 		cpufreq_policy_put_kobj(policy);

195: 	}

196: 	cpufreq_policy_free(policy);

197: 

198: nomem_out:

199: 	up_read(&cpufreq_rwsem);

200: 

201: 	return ret;

202: }

> 12~13行：如果cpu处于offline状态，则直接返回。
> 
> 17~25行：对于SMP系统，可能存在所有的CPU core，使用相同的cpufreq policy的情况。此时，当primary CPU枚举时，调用__cpufreq_add_dev接口创建cpufreq policy时，会同时将该policy提供给其它CPU使用。当其他CPU枚举时，它需要判断是否已经有人代劳了，如果有，则直接返回。这就是这几行代码的逻辑。
> 
> 30~40行：这几行是处理具有hotplug功能的CPU的。有前面的逻辑可知，如果系统中多个CPU共用一个cpufreq policy，primary CPU在枚举时，会帮忙添加其它CPU的cpufreq policy。但这有一个条件：它只帮忙处理那些处于online状态的CPU，而那些offline状态的CPU，则需要在online的时候，自行判断，并将自身的cpufreq policy添加到系统。
> 
> 注2：有关上面的两段实现，后面会专门用一个章节介绍。
> 
> 44~54行：分配cpufreq policy。根据是否是suspend & resume的过程，会有不同的处理方式，我们先不去纠结这些细节。
> 
> 55~65行：将当前cpu保存在policy的cpu字段。
> 
> 67行：初始化policy->cpus变量，由“[Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)”可知，该变量是一个cpumask类型的变量，记录该policy可以控制哪些online的cpu（毫无疑问，至少会包含当前的cpu）。
> 
> 75行：调用driver的init接口，相当于设备模型中的driver probe。此时cpufreq deice（policy）和cpufreq driver已经成功会和，driver开始执行。具体行为，请参考“[Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)”。
> 
> 82行：初始化policy->related_cpus，至少包含所有的online CPUs（policy->cpus，由cpufreq_driver->init初始化）。
> 
> 88行：根据当前CPU online状态，将policy->cpus中处于offline状态的CPU剔除。
> 
> 97~98行：初始化所有其它共用cpufreq policy的、处于online状态的CPU（policy->cpus）的cpufreq_cpu_data变量，表明它们也都已经有了cpufreq policy（回到17~25行的逻辑，其中的判断，就是依据该变量）。
> 
> 109~147行：如果cpufreq driver定义了CPUFREQ_NEED_INITIAL_FREQ_CHECK flag，则要求cpufreq core在启动时检查当前频率（policy->cur）是否在frequency table范围内，如果不在，调用__cpufreq_driver_target重新设置频率。
> 
> 152~158行：对于不是suspend&resume的场景，调用cpufreq_add_dev_interface接口，在该cpu的目录下，创建cpufreq的sysfs目录，以及相应的attribute文件（可参考本文第2章的内容），并为其它共用policy的、处于online状态的cpu，创建指向该cpufreq目录的符号链接。
> 
> 161行：将新创建的policy，挂到一个全局链表中（cpufreq_policy_list）。
> 
> 164行：调用cpufreq_init_policy，为该policy分配一个governor，并调用cpufreq_set_policy接口，为该cpu设置一个默认的cpufreq policy。这一段的逻辑有点纠结，我们后面再细讲。
> 
> 166~-169：更新policy->user_policy，后面再细讲。

3）SMP、Hotplug等场景下，多个CPU共用cpufreq policy的情况总结

前面多次提到，在SMP系统中，多个CPU core可能会由相同的调频机制（其实大多数平台都是这样的）控制，也就是说，所有CPU core的频率和电压，是同时调节的。这种情况下，只需要创建一个cpufreq policy即可，涉及到的代码逻辑包括：

> a）primary CPU枚举时，__cpufreq_add_dev会调用cpufreq driver的init接口（cpufreq_driver->init），driver需要根据当前的系统情况，设置policy->cpus，告诉cpufreq core哪些CPU共用同一个cpufreq policy。
> 
> b）primary CPU的__cpufreq_add_dev继续执行，初始化policy->related_cpus，并将policy->cpus中处于offline状态的CPU剔除。具体可参考上面的代码分析。
> 
> c）primary CPU的__cpufreq_add_dev继续执行，创建sysfs接口，同时为policy->cpus中的其它CPU创建相应的符号链接。
> 
> d）secondary CPUs枚举，执行__cpufreq_add_dev，判断primary CPU已经代劳之后，直接退出。
> 
> e）对于hotplugable的CPU，hotplug in时，由于primary CPU没有帮忙创建sysfs的符号链接，或者hotplug out的时候符号链接被删除，因此需要重新创建。

4.2 频率调整

cpufreq framework的频率调整逻辑，总结如下：

> 通过调整policy（struct cpufreq_policy），确定CPU频率调整的一个大方向，主要是由min_freq和max_freq组成的频率范围；
> 
> 通过cpufreq governor，确定最终的频率值。
> 
> 下面结合代码，做进一步的阐述。

1）cpufreq_set_policy

cpufreq_set_policy用来设置一个新的cpufreq policy，调用的时机包括：

> a）初始化时（__cpufreq_add_dev->cpufreq_init_policy->cpufreq_set_policy），将cpufreq_driver->init时提供的基础policy，设置生效。
> 
> b）修改scaling_max_freq或scaling_min_freq时（store_one->cpufreq_set_policy），将用户空间设置的新的频率范围，设置生效。
> 
> c）修改cpufreq governor时（scaling_governor->store_scaling_governor->cpufreq_set_policy），更新governor。

来看一下cpufreq_set_policy都做了什么事情。

  1: static int cpufreq_set_policy(struct cpufreq_policy *policy,

  2: 				struct cpufreq_policy *new_policy)

  3: {

  4: 	struct cpufreq_governor *old_gov;

  5: 	int ret;

  6: 

  7: 	pr_debug("setting new policy for CPU %u: %u - %u kHz\n",

  8: 		 new_policy->cpu, new_policy->min, new_policy->max);

  9: 

 10: 	memcpy(&new_policy->cpuinfo, &policy->cpuinfo, sizeof(policy->cpuinfo));

 11: 

 12: 	if (new_policy->min > policy->max || new_policy->max < policy->min)

 13: 		return -EINVAL;

 14: 

 15: 	/* verify the cpu speed can be set within this limit */

 16: 	ret = cpufreq_driver->verify(new_policy);

 17: 	if (ret)

 18: 		return ret;

 19: 

 20: 	/* adjust if necessary - all reasons */

 21: 	blocking_notifier_call_chain(&cpufreq_policy_notifier_list,

 22: 			CPUFREQ_ADJUST, new_policy);

 23: 

 24: 	/* adjust if necessary - hardware incompatibility*/

 25: 	blocking_notifier_call_chain(&cpufreq_policy_notifier_list,

 26: 			CPUFREQ_INCOMPATIBLE, new_policy);

 27: 

 28: 	/*

 29: 	 * verify the cpu speed can be set within this limit, which might be

 30: 	 * different to the first one

 31: 	 */

 32: 	ret = cpufreq_driver->verify(new_policy);

 33: 	if (ret)

 34: 		return ret;

 35: 

 36: 	/* notification of the new policy */

 37: 	blocking_notifier_call_chain(&cpufreq_policy_notifier_list,

 38: 			CPUFREQ_NOTIFY, new_policy);

 39: 

 40: 	policy->min = new_policy->min;

 41: 	policy->max = new_policy->max;

 42: 

 43: 	pr_debug("new min and max freqs are %u - %u kHz\n",

 44: 		 policy->min, policy->max);

 45: 

 46: 	if (cpufreq_driver->setpolicy) {

 47: 		policy->policy = new_policy->policy;

 48: 		pr_debug("setting range\n");

 49: 		return cpufreq_driver->setpolicy(new_policy);

 50: 	}

 51: 

 52: 	if (new_policy->governor == policy->governor)

 53: 		goto out;

 54: 

 55: 	pr_debug("governor switch\n");

 56: 

 57: 	/* save old, working values */

 58: 	old_gov = policy->governor;

 59: 	/* end old governor */

 60: 	if (old_gov) {

 61: 		__cpufreq_governor(policy, CPUFREQ_GOV_STOP);

 62: 		up_write(&policy->rwsem);

 63: 		__cpufreq_governor(policy, CPUFREQ_GOV_POLICY_EXIT);

 64: 		down_write(&policy->rwsem);

 65: 	}

 66: 

 67: 	/* start new governor */

 68: 	policy->governor = new_policy->governor;

 69: 	if (!__cpufreq_governor(policy, CPUFREQ_GOV_POLICY_INIT)) {

 70: 		if (!__cpufreq_governor(policy, CPUFREQ_GOV_START))

 71: 			goto out;

 72: 

 73: 		up_write(&policy->rwsem);

 74: 		__cpufreq_governor(policy, CPUFREQ_GOV_POLICY_EXIT);

 75: 		down_write(&policy->rwsem);

 76: 	}

 77: 

 78: 	/* new governor failed, so re-start old one */

 79: 	pr_debug("starting governor %s failed\n", policy->governor->name);

 80: 	if (old_gov) {

 81: 		policy->governor = old_gov;

 82: 		__cpufreq_governor(policy, CPUFREQ_GOV_POLICY_INIT);

 83: 		__cpufreq_governor(policy, CPUFREQ_GOV_START);

 84: 	}

 85: 

 86: 	return -EINVAL;

 87: 

 88:  out:

 89: 	pr_debug("governor: change or update limits\n");

 90: 	return __cpufreq_governor(policy, CPUFREQ_GOV_LIMITS);

 91: }

> 15~18行：调用driver的verify接口，判断新的policy是否有效。
> 
> 20~26行：notifier接口用于policy的调整，如果有其它模块对新的policy不满意，可以通过这样的机制修正，具体不再描述，感兴趣的读者可以自行研究。
> 
> 28~34行：修正后，再交给driver进行verify。
> 
> 46~50行：如果driver提供了setpolicy回调（回忆一下“[Linux cpufreq framework(2)_cpufreq driver](http://www.wowotech.net/pm_subsystem/cpufreq_driver.html)”），则说明硬件有能力根据policy所指定的范围，自行调节频率，其它机制就不需要了，调用该回调，将新的policy配置给硬件后，退出。
> 
> 否则（后面的代码），则需要由governor执行具体的频率调整动作，调用governor的接口，切换governor。有关cpufreq governor的描述，请参考下一篇文章。

2）scaling_setspeed

上面提到了，policy只规定了频率调整的一个范围，如果driver不支持setpolicy操作，则需要由cpufreq governor确定具体的频率值，并调用driver的target或者target_index接口，修改CPU的频率值。

有关cpufreq governor的介绍，请参考后续的文章（“linux cpufreq framework(4)_cpufreq governor”）。不过这其中有一个例外，就是当governor为“userspace”时（参考第3章的描述），可以直接通过scaling_setspeed文件，从用户空间修改频率值，代码如下：

  1: static ssize_t store_scaling_setspeed(struct cpufreq_policy *policy,

  2:                                         const char *buf, size_t count)

  3: {

  4:         unsigned int freq = 0;

  5:         unsigned int ret;

  6: 

  7:         if (!policy->governor || !policy->governor->store_setspeed)

  8:                 return -EINVAL;

  9: 

 10:         ret = sscanf(buf, "%u", &freq);

 11:         if (ret != 1)

 12:                 return -EINVAL;

 13: 

 14:         policy->governor->store_setspeed(policy, freq);

 15: 

 16:         return count;

 17: }

> 由此可以猜到，“userspace” governor具有store_setspeed函数，该函数应该可以直接修改频率值。留到（“linux cpufreq framework(4)_cpufreq governor”）再欣赏吧。

_原创文章，转发请注明出处。蜗窝科技_，[www.wowotech.net](http://www.wowotech.net/pm_subsystem/cpufreq_core.html)。

标签: [Linux](http://www.wowotech.net/tag/Linux) [core](http://www.wowotech.net/tag/core) [cpufreq](http://www.wowotech.net/tag/cpufreq)

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [Concurrency Managed Workqueue之（二）：CMWQ概述](http://www.wowotech.net/irq_subsystem/cmwq-intro.html) | [总结Linux kernel开发中0的巧妙使用](http://www.wowotech.net/201.html)»

**评论：**

**zgq**  
2018-10-09 20:09

cpufre 有可能调整到clock source的时钟吗?如果被调整到？会对系统时间有影响。这个内核有考虑到吗？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-6972)

**[linuxer](http://www.wowotech.net/)**  
2018-12-05 17:15

@zgq：基本上不会，当然这和底层硬件相关，如果你设计的SOC上，cpu clock domain和HW counter的clock domain是同一个（或者是cpu clock domain的sub domain），那么调整CPU就会影响到clock source的时钟。当然，任何一个思维正常的IC公司都应该不会设计这样的SOC吧。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-7069)

**jiffy**  
2018-04-21 10:21

@wowo 你好，请问下：发起频率改变的请求一定是应用层（如android层）触发的吗？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-6687)

**[wowo](http://www.wowotech.net/)**  
2018-04-25 20:25

@jiffy：是的，只有应用才知道该用什么样的策略调频。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-6700)

**wanghyunqian**  
2019-02-20 11:01

@wowo：发起频率改变的请求不一定应用层吧？这需要根据调频的governor来决定吧！比如说在governor是ondemand的时候，改变频率是根据负载情况来改变频率的呀？

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-7193)

**[wowo](http://www.wowotech.net/)**  
2019-02-20 21:16

@wanghyunqian：其实jiffy同学的问题和我的回答有点搭不上，给jiffy同学一个迟到道歉吧！！  
对jiffy的问题来说，你是对的，具体用什么频率，不一定都是应用层直接干预。  
但从根本上讲，用什么策略调频，只有应用才知道。

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-7195)

**lamaboy**  
2016-08-13 20:17

看了楼主的分享，感觉国内的对 Linux 理解会因你而上了个档次

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-4397)

**[wowo](http://www.wowotech.net/)**  
2016-08-14 10:48

@lamaboy：谢谢鼓励，大家一起努力～～

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-4398)

**mdong**  
2016-07-14 19:59

这篇文章:beautiful, nice and good. thanks.

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-4263)

**[wowo](http://www.wowotech.net/)**  
2016-07-14 21:41

@mdong：多谢夸奖:-)

[回复](http://www.wowotech.net/pm_subsystem/cpufreq_core.html#comment-4264)

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
    
    - Shiina  
        [一个电路（circuit）中，由于是回路，所以用电势差的概念...](http://www.wowotech.net/basic_subject/voltage.html#8926)
    - Shiina  
        [其中比较关键的点是相对位置概念和点电荷的静电势能计算。](http://www.wowotech.net/basic_subject/voltage.html#8925)
    - leelockhey  
        [你这是哪个内核版本](http://www.wowotech.net/pm_subsystem/generic_pm_architecture.html#8924)
    - ja  
        [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
    - 元神高手  
        [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
    - 十七  
        [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
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
    
    - [kernel启动优化](http://www.wowotech.net/180.html)
    - [作業系統之前的程式 for rpi2 (1) - mmu (0) : 位址轉換](http://www.wowotech.net/linux_kenrel/address-translation.html)
    - [Linux电源管理(1)_整体架构](http://www.wowotech.net/pm_subsystem/pm_architecture.html)
    - [蜗窝微信群问题整理](http://www.wowotech.net/tech_discuss/question_set_1.html)
    - [玩转BLE(3)_使用微信蓝牙精简协议伪造记步数据](http://www.wowotech.net/bluetooth/weixin_ble_1.html)
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