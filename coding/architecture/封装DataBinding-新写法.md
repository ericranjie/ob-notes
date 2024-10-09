# 

互联网程序员

_2021年09月08日 10:02_

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/icPkicsUe9JFpMaYIO9CvOIaRp8PLBqkVcH3IBjwCtm2673vUstj4eedrvGKic04C2CCCB6e13J5RicymnbqaiawiaNw/640?wx_fmt=jpeg&wxfrom=13&tp=wxpic)

现在我们就可以开始做一些基础的封装工作，同时在app的`bulid.gradle`文件中开启`dataBinding`的使用

```
android {
```

## 基于DataBinding的封装

我们先创建一个简单的布局文件`activity_main.xml`。为了节约时间，同时我们也创建一个`fragment_main.xml`保持一样的布局。

```
<?xml version="1.0" encoding="utf-8"?>
```

我们在使用`DataBinding`初始化布局时候，我们通常喜欢使用下面几种方式， 在`Activity`中：

```
class MainActivity : AppCompatActivity() {
```

在`Fragment`中：

```
class HomeFragment:Fragment() {
```

这种情况下，每创建一个`activity`和`Fragment`都需要重写一遍。所以我们创建一个`BaseActivity`进行抽象，然后使用泛型传入我们需要的`ViewDataBinding`对象`VB`，再通过构造方法或者抽象方法获取`LayoutRes`资源

```
abstract class BaseActivity<VB : ViewDataBinding>(@LayoutRes resId:Int) : AppCompatActivity() {
```

这个时候是不是经过我们经过上面处理后，再使用的时候我们会方便很多。可能有些人封装过的人看到这里会想，你讲的这都是啥，这些我都会，没有一点创意。笔者想说：不要捉急，我们要讲的可不是上面的东西，毕竟做事情都需要前奏铺垫滴。

虽然经过上面的抽象以后，我们是减少了一些步骤。但是笔者还写觉得有些麻烦，因为我们还是需要手写的通过外部传一个`LayoutRes`资源才能进行使用。想要再次细化缩减代码，那我们就得看看`ActivityMainBinding`的实现。

我们在开启`DataBinding`的时候，通过使用`layout`的`activity_main.xml`布局，`DataBinding`在编译的时候会自动在我们的工程`app/build/generated/data_binding_base_class_source_out/packname/databinding`目录下为我们生成一个`ActivityMainBinding`类，我们看看它的实现：

```
public abstract class ActivityMainBinding extends ViewDataBinding {
```

我们可以看到在`ActivityMainBinding`中的有4个`inflate`方法，同时他们最后的都会直接使用我们的布局文件`activity_main.xml`进行加载。所以我们想在上面的基础上进一步的简化使用方式，我们就必须通过反射的机制，从拿到`ActivityMainBinding`中的`inflate`方法，使用相对应的`inflate`方法去加载我们的布局。代码如下：

```
inline fun <VB:ViewBinding> Any.getViewBinding(inflater: LayoutInflater):VB{
```

我们先定义个扩展方法，通过反射的方式，从我们的`Class`中拿到我们想要的泛型类`ViewBinding`，然后`invoke`调用`ViewBinding`的`inflate`方法。然后我们再创建一个接口用于`BaseActivity`子类进行UI初始化绑定操作。

```
interface BaseBinding<VB : ViewDataBinding> {
```

```
abstract class BaseActivity<VB : ViewDataBinding> : AppCompatActivity(), BaseBinding<VB> {
```

现在我们就可以继承`BaseActivity`实现我们的`Activity`：

```
class MainActivity : BaseActivity<ActivityMainBinding>() {
```

```
D/MainActivity: btn  :Hello World
```

现在我们的代码是不是变得简洁、清爽很多。这样我们不仅节省了编写大量重复代码的时间，同时也让我们代码的变得更加合理、美观。

和`Activity`有一些不同，因为`Fragment`创建布局的时候需要传入`ViewGroup`，所以我们稍微做一个变化。

```
inline fun <VB:ViewBinding> Any.getViewBinding(inflater: LayoutInflater, container: ViewGroup?):VB{
```

可能在某些环境下有些人在创建`Fragment`的时候需要把`attachToRoot`设置成`true`，这个时候自己扩展一个就好了，我们这里就不再演示。同理我们再抽象一个`BaseFragment`：

```
abstract class BaseFragment<VB : ViewDataBinding>: Fragment(),BaseBinding<VB> {
```

看到这里是不是心灵舒畅了很多，不仅代码量减少了，逼格也提升了许多，同时又增加了`XX技术群`里摸鱼吹水的时间。

当然我们也不能仅仅满足于此，在码代码的过程中还有一个大量重复工作的就是我们的`Adapter`,我们就拿使用到做多的`RecyclerView.Adapter`为例，假设我们创建一个最简单的`HomeAdapter`：

```
class HomeAdapter(private val data: List<String>? = null) : RecyclerView.Adapter<HomeAdapter.BindingViewHolder>() {
```

就这样一个最简单的`Adapter`，我们都需要些一堆啰嗦代码，如果再加上`item`的`click`事件的话，我们要做的 工作就变得更多。那么我们现在要解决这么一个问题的话，我们要处理哪里东西呢：

1. 统一`Adapter`的初始化工作。

1. 简化`onBindViewHolder`的使用。

1. 去掉每次都需要重复创建`ViewHolder`。

1. 统一我们设置`Item`的监听事件方式。

1. 统一`Adapter`的数据刷新。

首先我们需要修改一下我们之前定义的扩展`getViewBinding`,因为我们是不知道具体这个`getViewBinding`是用在哪个类上，这个类又定义了几个泛型。所以我们增加一个默认值为`0`的`position`参数代替之前写死的`0`，通过这种方式让调用者自行设定`VB:ViewBinding`所在的位置顺序：

```
inline fun <VB:ViewBinding> Any.getViewBinding(inflater: LayoutInflater,position:Int = 0):VB{
```

创建我们的`BaseAdapter`，先将完整代码贴出：

```
abstract class BaseAdapter<T, VB : ViewDataBinding> : RecyclerView.Adapter<BaseAdapter.BindViewHolder<VB>>() {
```

我们这里先忽略这个`BaseAdapter`的定义，现在我们将`HomeAdapter`修改一下就变成了：

```
class HomeAdapter : BaseAdapter<String, ItemHomeBinding>() {
```

我们在`Activity`中使用时：

```
class MainActivity : BaseActivity<ActivityMainBinding>() {
```

现在我们的`adapter`中的代码是不是变的超简洁，我相信现在即使让你再写`100`个`Adapter`也不害怕。

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

现在我们来一步一步的拆解`BaseAdapter`。我们看到`BaseAdapter`需要传入2个泛型，`T`是我们需要显示的实体的数据类型，`VB`是我们的布局绑定`ViewDataBinding`。

```
abstract class BaseAdapter<T, VB : ViewDataBinding> 
```

往下可以看到我们通过在`BaseAdapter`实现`onCreateViewHolder`来处理`Item`布局的初始化工作，我们这里调用`getViewBinding`的时候`position`传入的是`1`，正好对应我们`VB`所在的顺序：

```
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): BindViewHolder<VB> {
```

同时我们创建了一个内部类`BindViewHolder`来进行统一`ViewHolder`的创建工作。

```
class BindViewHolder<M : ViewDataBinding>(var binding: M) :
```

我们在初始化布局的同时又调用了一个空实现的`setListener`方法。为什么我们在这里采用`open`而不是采用`abstract`来定义。是因为我们不是每一个`Adapter`都需要设置`Item`的监听事件，因此我们把`setListener`只是作为一个可选的项来处理。

```
open fun VB.setListener() {}
```

!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

初始化完成以后，我们需要进行布局绑定，但是因为不同的`Adapter`的界面，需要处理的绑定是不一样的，所以我们在实现`onBindViewHolder`的同时，通过调用内部创建的抽象方法`VB.onBindViewHolder`将我们的绑定处理交由子类进行处理。

```
 override fun onBindViewHolder(holder: BindViewHolder<VB>, position: Int) {
```

同时将`VB.onBindViewHolder`参数转换为实际的数据`bean`和对应的位置`position`.

```
 abstract fun VB.onBindViewHolder(bean: T, position: Int)
```

到目前为止，我们在`BaseAdapter`中已经处理了：

1. 统一`Adapter`的初始化工作。

1. 简化`onBindViewHolder`的使用。

1. 去掉每次都需要重复创建`ViewHolder`。

1. 统一我们设置`Item`的监听事件方式。

现在我们就来看下是如何统一`Adapter`的数据刷新。可以看到我们在`BaseAdapter`创建了一个私有数据集合`mData`，在`mData`中存放的是我们需要显示的泛型`T`的数据类型。

```
private var mData: List<T> = mutableListOf()
```

同时我们增加了一个`setData`方法，在此方法中我们使用`DiffUtil`对我们的数据进行对比刷新。，如果对`DiffUtil`不太熟的可以查一下它的方法。

```
fun setData(data: List<T>?) {
```

这里我们需要注意一下，`DiffUtil.Callback`中的`areItemsTheSame`和`areContentsTheSame`2个对比数据的方法，实际上是通过我们在`BaseAdapter`中定义2个`open`方法`areItemContentsTheSame`,`areItemsTheSame`。

```
 protected open fun areItemContentsTheSame(oldItem: T, newItem: T, oldItemPosition: Int, newItemPosition: Int): Boolean {
```

为什么要这么去定义的呢。虽然在`BaseAdapter`中实现了这2个方法，因为我们不知道子类在实现的时候是否需要改变对比的方式。比如我在使用`areItemsTheSame`的时候，泛型`T`如果泛型T不是一个基本数据类型，通常只需要对比泛型`T`中的唯一`key`就可以。现在假设泛型`T`是一个数据实体类`User`:

```
  data class User(val id:Int,val name:String)
```

那我们在子类复写`areItemsTheSame`方法的时候，就可以在我们的实现的`apapter`如下使用：

```
  protected open fun areItemsTheSame(oldItem: User, newItem: User): Boolean {
```

细心的可能注意到我们有定义一个`getActualPosition`方法，为什么不是叫`getPosition`。这是因为在有些为了方便，我们在`onBindViewHolder`的时候把此时`position`保存下来，或者设置监听器的时候传入了`position`。

如果我们在之前的数据基础上插入或者减少几条数据的话，但是又因为我们使用了`DiffUtil`的方式去刷新，由于之前已存在`bean`的数据没变，只是位置变了，所以`onBindViewHolder`不会执行,这个时候我们直接使用`position`的时候会出现位置不对问题，或者是越界的问题。比如如下使用：

```
interface ItemClickListener<T> {
```

我们在`ItemClickListener`定义了2个方法，我们使用带有`position`的`onItemClick`方法来演示：

```
<?xml version="1.0" encoding="utf-8"?>
```

我们在XML中进行`Click`绑定,然后我们在`HomeAdapter`进行监听事件和数据设置

```
class HomeAdapter(private val listener: ItemClickListener<String>) : BaseAdapter<String, ItemHomeBinding>() {
```

接下来我们在`MainActivity`通过2次设置数据在点击查看日志

```
class MainActivity : BaseActivity<ActivityMainBinding>() {
```

```

```

所以我们需要在使用`position`的时候，最好是通过`getActualPosition`来获取真实的位置，我们修改一下`itemClickListener`中的日志输出。

```
private val itemClickListener = object : ItemClickListener<String> {
```

这个时候我们再重复上面操作的时候，就可以看到`position`的位置就是它目前所处的真实位置。

```
D/onItemClick: data:c   position:0
```

到此为止，我们对于这个`BaseAdapter<T, VB : ViewDataBinding>`的抽象原理，以及使用方式有了大概的了解。

**需要注意的是**：_为了方便简单演示，我们这里假设是，没有在xml中直接使用`Databinding`进行绑定。因为有些复杂逻辑我们是没有办法简单的在xml中进行绑定的。_

很显然我们的工作并没有到此结束，因为我们的`adapter`在常用的场景中还有多布局的情况，那我们又应该如何处理呢。

这个其实很好办。因为我们是多布局，那么就意味着我们需要把`onCreateViewHolder`中的一部分工作暴露给子类处理，所以我们需要在上面`BaseAdapter`的基础上做一些修改。照例上代码：

```
abstract class BaseMultiTypeAdapter<T> : RecyclerView.Adapter<BaseMultiTypeAdapter.MultiTypeViewHolder>() {
```

通过上面的代码可以看到，我们没有在`BaseMultiTypeAdapter`中定义泛型`VB :ViewDataBinding`，因为我们是多布局，如果都写在类的定义中明显是不合适的，我们也不知道在具体实现需要有多少个布局。

所以我们`onCreateViewHolder`初始化布局的时候调用了一个抽象的`onCreateMultiViewHolder`方法，这个方法交由我们具体业务实现类去实现。同时我们对`onBindViewHolder`进行修改，增加了一个`holder`参数供外部使用。我们先数据实体类型

```
sealed class Person(open val id :Int, open val name:String)
```

和我们需要实现的Adapter业务类，：

```
class SecondAdapter: BaseMultiTypeAdapter<Person>() {
```

运行一下就可以看到我们想要的结果：

```
D/ItemTeacherBinding: item : Teacher(id=1, name=Person, subject=语文)   position : 0
```

经过上面的处理以后，我们在创建`Activiy`、`Fragment`、`Adapter`的时候减少了大量的代码。同时也节省了码这些重复垃圾代码的时间，起码让你们的工作效率起码提升`10`个百分点，是不是感觉到自己无形中又变帅了许多。

经过以上封装处理以后，我们是不是也可以对`Dialog`，`PopWindow`、动态初始化`View`进行处理呢。那还等什么，赶紧去实现吧。毕竟授人以鱼，不如授人以渔。

原创不易。如果您喜欢这篇文章，您可以动动小手点赞收藏!\[图片\](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)。

作者：一个被摄影耽误的程序猿\
链接：https://juejin.cn/post/6957608813809795108

**版权申明：内容来源网络，版权归原创者所有。除非无法确认，都会标明作者及出处，如有侵权烦请告知，我们会立即删除并致歉。谢谢!**

![](http://mmbiz.qpic.cn/mmbiz_png/icPkicsUe9JFqHElszMhzVDlACDw553ybumOsbFMIeO1W6CM6ygXtWGWAyAIRdpM1qQEg9Brf58BhsnR6GAlKemA/300?wx_fmt=png&wxfrom=19)

**互联网程序员**

互联网程序员的家园，包括但不限于Java，Python，架构，大数据，云计算，数据分析，机器学习，人工智能，5G，物联网，车联网，区块链等技术分享！

54篇原创内容

公众号

阅读 706

​

写留言

**留言 2**

- 淸泽

  2021年9月8日

  赞

  之前都封装过，奈何大家不习惯这种写法，索性去掉封装自己用了

- 野生奥特曼

  2021年9月8日

  赞

  这个想法6啊，学起来

已无更多数据

[](javacript:;)

![](http://mmbiz.qpic.cn/mmbiz_png/icPkicsUe9JFqHElszMhzVDlACDw553ybumOsbFMIeO1W6CM6ygXtWGWAyAIRdpM1qQEg9Brf58BhsnR6GAlKemA/300?wx_fmt=png&wxfrom=18)

互联网程序员

1分享2

2

写留言

**留言 2**

- 淸泽

  2021年9月8日

  赞

  之前都封装过，奈何大家不习惯这种写法，索性去掉封装自己用了

- 野生奥特曼

  2021年9月8日

  赞

  这个想法6啊，学起来

已无更多数据
