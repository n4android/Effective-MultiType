## 前言

在我开发我的 **[TimeMachine](https://github.com/drakeet/TimeMachine)** 时，我有一个聊天页面，于是我设计了我的消息类型池系统，它是完全解耦的，于是我能够轻松将它抽离出来分享，并给它取名为 **MultiType**.

从前，我们写一个复杂的、多种 item view 类型的列表视图时，经常要做一堆繁琐的工作，而且很容易导致代码堆积严重：我们需要覆写 `RecyclerView.Adapter` 的 `getItemViewType` 方法，并罗列一些 `type` 整型常量，而且 `ViewHolder` 转型也比较麻烦。一旦我们需要新增一些新的 item view types ，就得去修改 `Adapter` 代码，步骤繁多，侵入较强。

现在好了，我们有了 **MultiType**，简单来说，**MultiType** 就是一个多类型列表视图的中间层分发框架，它本是为 IM 视图开发的，想想 IM 的消息类型可能有多少种而且新增频繁，而 **MultiType** 完成能够胜任，并使得随时可拓展新的类型进入列表当中。它内建了 `类型 - View` 的复用池系统，支持 `RecyclerView`、复用，代码模块化开发，清晰而灵活。

## 目录

- [MultiType 的特性](#multitype-的特性)
- [总览](#总览)
- [MultiType 基础用法](#multitype-基础用法)
- [高级用法](#高级用法)
  - [使用 MultiTypeTemplates 插件自动生成代码](#使用-multitypetemplates-插件自动生成代码)
  - [使用 全局类型池](#使用-全局类型池)
  - [一个类型对应多个 ViewProvider](#一个类型对应多个-viewprovider)
  - [与 provider 通讯](#与-provider-通讯)
  - [支持 Google AutoValue](#支持-google-autovalue)
  - [对 class 进行二级分发](#对-class-进行二级分发)
  - [MultiType 与下拉刷新、加载更多、HeaderView、FooterView、Diff](#multitype-与下拉刷新加载更多headerviewfooterviewdiff)
- [更多示例](#更多示例)
  - drakeet/about-page
  - 线性和网格布局混排
  - drakeet/TimeMachine
  - 类似 Bilibili iOS 端首页
- [设计思想](#设计思想)
- [引用文献](#引用文献)

## MultiType 的特性

- 周到，支持 局部类型池 和 全局类型池，并支持二者共用，当出现冲突时，以局部的为准
- 灵活，几乎所有的部件(类)都可被替换、可继承定制，留够了丰富的可覆写的接口
- 轻盈，整个类库只有 11 个类文件，`aar` 或 `jar` 包大小只有 10KB
- 纯粹，只负责本分事情，专注多类型的列表试图类型分发
- 高效，没有性能损失，内存友好，最大限度发挥 `RecyclerView` 的复用性
- 可读，代码清晰干净、设计精巧，极力避免复杂化，可读性很好，为拓展和自行解决问题提供了基础

## 总览

![](http://ww3.sinaimg.cn/large/86e2ff85gw1f93likz705j21kq14eqak.jpg)

## MultiType 基础用法

话不多说，我们先看看基础用法，使用 **MultiType** 一般情况下只要引入 + 三个步骤，后面还有使用插件生成代码方式，步骤将更加简化：

#### 引入

在你的 `build.gradle`:

```java
dependencies {
    compile 'me.drakeet.multitype:multitype:2.2.0-beta1'
}
```
### 使用

**Step 1**. 创建一个 `class implements Item`，它将是你的数据类型或 Java bean/model，示例：

```java
public class Category implements Item {

    @NonNull public String text;

    public Category(@NonNull final String text) {
        this.text = text;
    }
}
```

**Step 2**. 创建一个 `class` 继承 `ItemViewProvider`，示例：

```java
public class CategoryViewProvider
    extends ItemViewProvider<Category, CategoryViewProvider.ViewHolder> {

    @NonNull @Override
    protected ViewHolder onCreateViewHolder(
        @NonNull LayoutInflater inflater, @NonNull ViewGroup parent) {
        View root = inflater.inflate(R.layout.item_category, parent, false);
        return new ViewHolder(root);
    }

    @Override
    protected void onBindViewHolder(
        @NonNull ViewHolder holder, @NonNull Category category) {
        holder.category.setText(category.text);
    }

    static class ViewHolder extends RecyclerView.ViewHolder {

        @NonNull private final TextView category;

        ViewHolder(@NonNull View itemView) {
            super(itemView);
            this.category = (TextView) itemView.findViewById(R.id.category);
        }
    }
}
```

**Step 3**. 好了，你不必再创建新的类文件了，在 `Activity` 中加入 `RecyclerView` 和 `List` 并注册你都类型即可，示例：

```java
public class MainActivity extends AppCompatActivity {

    private MultiTypeAdapter adapter;

    /* Items 等价于 ArrayList<Item> */
    private Items items;

    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.list);

        items = new Items();
        adapter = new MultiTypeAdapter(items);

        /* 注册类型和 View 的对应关系 */
        adapter.register(Category.class, new CategoryViewProvider());
        adapter.register(Song.class, new SongViewProvider());

        /* 模拟加载数据，也可以稍后再加载，然后使用
           adapter.notifyDataSetChanged() 刷新列表 */
        for (int i = 0; i < 20; i++) {
            items.add(new Category("Songs"));
            items.add(new Song("小艾大人", R.drawable.avatar_dakeet));
            items.add(new Song("许岑", R.drawable.avatar_cen));
        }

        recyclerView.setAdapter(adapter);
    }
}
```

大功告成！这就是 **MultiType** 的基础用法了，简单、符合直觉。其中 `onCreateViewHolder` 和 `onBindViewHolder` 方法名沿袭了使用 `RecyclerView` 的习惯，令人一目了然，减少了新人的学习成本。



## 高级用法

### 使用 MultiTypeTemplates 插件自动生成代码

上面我们介绍了通过 3 个步骤完成 **MultiType** 的初次接入使用，实际上这个过程可以更加简化，**MultiType** 提供了 Android Studio 插件来自动生成代码：**MultiTypeTemplates**，源码也是开源的，https://github.com/drakeet/MultiTypeTemplates ，不仅提供了一键生成 `Item` 和 `ItemViewProvider`，而且是一个很好的利用代码模版自动生成代码的示例，其中使用到了官方提供的代码模版 API，也用到了 drakeet 自己发明的更加灵活修改模版内容的方法，有兴趣做这方面插件的可以看看。

话说回来，安装和使用 **MultiTypeTemplates** 非常简单：

**Step 1** 打开 Android Studio 的`设置` -> `Plugin` -> `Browse repositories`，搜索 `MultiTypeTemplates` 即可获得下载安装：

![](http://ww4.sinaimg.cn/large/86e2ff85gw1f935l0kwilj21kw0t3akm.jpg)

**Step 2** 右键点击你的 package，选择 `New` -> `MultiType Item`，然后输入你的 `Item` 名字，它就会自动生成 `Item` and `ItemViewProvider` 文件和代码。

比如你输入的是 "Category"，它就会自动生成 `Category.java` 和 `CategoryViewProvider.java`.

特别方便，相信你会很喜欢它。未来这个插件也将会支持自动生成布局文件，这是目前欠缺的。但其实 AS 在这方面已经很方便了，对布局 `R.layout.item_category` 使用 `alt + enter` 即可自动生成布局文件。

----

### 使用 全局类型池

**MultiType** 支持 局部类型池 和 全局类型池，并支持二者共用，当出现冲突时，以局部的为准。使用局部类型池就是如上面的示例，调用 `adapter.register()` 即可。而使用全局类型池也是很容易的，
**MultiType** 提供了一个内置的 `GlobalMultiTypePool` 作为全局类型池来存储类型和 view 关系，使用如下：

在你使用你的全局类型之前，任意位置注册类型即可，通过调用 `GlobalMultiTypePool.register()` 静态方法完成注册。推荐统一在 `Application` 初始化便进行注册，这样代码便于寻找和阅读。

之后回到你的 `Activity`，调用 `adapter.applyGlobalMultiTypePool()` 方法应用你注册过的全局类型即可。

`GlobalMultiTypePool` 提供了让一些普适性的类型能够全局共用的功能，但同时使用全局类型池不当也会带来问题，这也是没有全然采用全局类型池但原因。问题在于全局类型池是静态的，如果你在 `Activity` 中注册全局类型，并传入带 `Activity` 引用的变量进去，就可能造成内存泄露。举个例子，如下是一个很常见的场景，我们把一个点击回调传递给 `provider`，并注册进了全局类型池：

```java
public class LeakActivity extends Activity {

    private MultiTypeAdapter adapter;
    private Items items;

    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.list);
        items = new Items();
        adapter = new MultiTypeAdapter(items);

        OnClickListener listener = new OnClickListener() {
            @Override
            public void onClick(View v) {
                // ...
            }
        }

        /* 在 applyGlobalMultiTypePool 之前注册全局 */
        GlobalMultiTypePool.register(Post.class, new PostViewProvider(listener));
        adapter.applyGlobalMultiTypePool(); // <- 使全局的类型加入到局部中来

        recyclerView.setAdapter(adapter);
    }
}
```

由于 匿名内部类 或 非静态内部类，都会默认持有 外部类 的引用，比如这里的 `OnClickListener` 匿名类对象会持有 `LeakActivity.this`，当 `listener` 传递给 `new PostViewProvider()` 构造函数的时候，`GlobalMultiTypePool` 内置的静态类型池将长久持有 `provider - listener - LeakActivity.this` 引用链，若没有及时释放，将引起内存泄露。因此，在使用全局类型池的时候，最好不要给 `provider` 传递回调对象或者外部引用，否则就应该做手动释放。除此之外，全局类型池没有什么其他问题了，类型池都只会持有 `class` 和非常轻薄的 `provider` 对象，我做过一个试验，就算拥有上万个类型和 `provider`，内存占用也是很少而且索引速度特别快，在主线程连续注册一万个类型也花费不过 10 毫秒的时间，何况一般一个应用根本不可能有这么多类型，完全不用担心这方面的问题。

另外一个特性是，不管是全局类型池还是局部类型池，都支持重复注册类型，当发现重复时，之后注册的会把之前注册的类型覆盖掉，因此对于全局类型池，最好不要重复注册，以免影响到其他地方。

----

### 一个类型对应多个 `ViewProvider`

**MultiType** 支持一个类型对应多个 `ViewProvider`，但仅限于在不同的列表中。比如你在 `adapter1` 中注册了 `Post.class` 对应 `SinglePostViewProvider`，在另一个 `adapter2` 中注册了 `Post.class` 对应 `PostDetailViewProvider`，这便是一对多的场景，但只要在不同的局部类型池中，无论如何都不会相互干扰，都是允许的。

而对于在同一个列表中一对多的问题，首先这种场景非常少见，再者不管支不支持一对多，开发者都要去判断哪个时候运用哪个 `ViewProvider`，这是逃不掉的，不然程序就无所适从了。因此，**MultiType** 不去特别解决这个问题，但如果要实现同一个列表中一对多，只要空继承你的类型，然后把它视为新的类型，注册到你的类型池中即可。

----

### 与 `provider` 通讯

`provider` 对象可以接受外部类型、回调函数，只要在使用之前，传递进去即可，例如：

```java
OnClickListener listener = new OnClickListener() {
    @Override
    public void onClick(View v) {
        // ...
    }
}
adapter.register(Post.class, new PostViewProvider(xxx, listener));
```

但话说回来，对于点击事件，能够不依赖 `provider` 外部的内容的话，最好就在 `provider` 内部完成，`provider` 内部能够拿到 View 也能够拿到数据，大部分情况下，完全有能力不依赖外部独立完成逻辑。这样能够使代码更加模块化，便于解耦，例如：

```java
public class SquareViewProvider extends ItemViewProvider<Square, SquareViewProvider.ViewHolder> {

    @NonNull @Override
    protected ViewHolder onCreateViewHolder(
        @NonNull LayoutInflater inflater, @NonNull ViewGroup parent) {
        View root = inflater.inflate(R.layout.item_square, parent, false);
        return new ViewHolder(root);
    }

    @Override
    protected void onBindViewHolder(@NonNull ViewHolder holder, @NonNull Square square) {
        holder.square = square;
        holder.squareView.setText(valueOf(square.number));
        holder.squareView.setSelected(square.isSelected);
    }

    public class ViewHolder extends RecyclerView.ViewHolder {

        private TextView squareView;
        private Square square;

        ViewHolder(final View itemView) {
            super(itemView);
            squareView = (TextView) itemView.findViewById(R.id.square);
            itemView.setOnClickListener(new View.OnClickListener() {
                @Override public void onClick(View v) {
                    itemView.setSelected(square.isSelected = !square.isSelected);
                }
            });
        }
    }
}
```

----

### 支持 Google AutoValue

**MultiType** 支持 Google [AutoValue](https://github.com/google/auto/tree/master/value)，同时支持映射子类到同一 view provider 了，规则是：如果子类有注册，就用注册的映射关系；如果子类没注册，则该子类对象使用注册过的父类映射关系。

![](http://ww2.sinaimg.cn/large/86e2ff85gw1f93i8wgoubj21ee0nmdnk.jpg)

### 对 class 进行二级分发

在我的 **TimeMachine** 中，我的消息数据结构是 `Message` - `MessageContent`，简单说就是，我的 `message` 对象们都是一样的 `Message.class`，但 `message` 包含的 `content` 对象不一样，我需要根据 `content` 来分发数据到 `ItemViewProvider`，但我加入 `Items` List 中的数据是 `Message.class`，因此，如果什么也不做，它们会被视为同一类型。对于这种场景，我们可以继承 `MultiTypeAdapter` 并覆写 `onFlattenClass(@NonNull Item message)` 方法进行二级分发，我的 `MessageAdapter` 为例：

```java
public class MessageAdapter extends MultiTypeAdapter {

    public MessageAdapter(@NonNull List<Message> messages) {
        super(messages);
    }


    @NonNull @Override public Class onFlattenClass(@NonNull Item message) {
        return ((Message) message).content.getClass();
    }
}
```

是不是十分简单？这样以后，我就可以直接将我的 `MessageContent.class` 注册进类型池，而将包含了不同 content 的 `Message` 对象 add 进 `Items` List，`MessageAdapter` 会自动取出 `message` 的 `content` 对象，并以它为基准定位 `ItemViewProvider`.

----

### MultiType 与下拉刷新、加载更多、HeaderView、FooterView、Diff

**MultiType** 设计从始至终，我都极力避免它往复杂化方向发展，一开始我的设计宗旨就是它应该是一个非常纯粹的、专一的项目，而非各种乱七八糟的功能都要囊括进来的多合一组件库，因此整个过程我都特别克制，期间有许多人给我发过加入一些新的无关特性的 Pull Request，大多被拒绝了。

对于很多人关心的 下拉刷新、加载更多、HeaderView、FooterView、Diff 这些功能特性，其实都不应该是 **MultiType** 都范畴，**MultiType** 的分内之事是做类型、事件与 View 的分发、连接工作，其余所有的这些需求，都是可以在 **MultiType** 外面完成，或者通过继承进行自行封装和拓展，而作为一个基础、公共类库，我想它是不应该包含这些内容的。

但很多新手可能并不习惯一码归一码，不习惯代码模块化，因此在此我有必要对这几个点简单示范下如何在 **MultiType** 之外去实现。

- **下拉刷新：**

  对于下拉刷新，`Android` 官方提供了 `support.v4` `SwipeRefreshLayout`，在 `Activity` 层面，可以拿到 `SwipeRefreshLayout` 并 `setOnRefreshListener`.

- **加载更多：**

  `RecyclerView` 提供了 `addOnScrollListener` 滚动位置变换监听，要实现加载更多，只要监听并检测列表是否滚动到底部即可，有多种方式，鉴于 `LayoutManager` 本应该只做布局相关的事务，因此我们推荐在直接在 `OnScrollListener` 层面进行判断。提供一个简单版 `OnScrollListener` 继承类：

  ```java
  public abstract class OnLoadMoreListener extends RecyclerView.OnScrollListener {

      private LinearLayoutManager layoutManager;
      private int itemCount, lastPosition, lastItemCount;

      public abstract void onLoadMore();

      @Override
      public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
          if (recyclerView.getLayoutManager() instanceof LinearLayoutManager) {
              layoutManager = (LinearLayoutManager) recyclerView.getLayoutManager();

              itemCount = layoutManager.getItemCount();
              lastPosition = layoutManager.findLastCompletelyVisibleItemPosition();
          } else {
              Log.e("OnLoadMoreListener", "The OnLoadMoreListener only support LinearLayoutManager");
              return;
          }

          if (lastItemCount != itemCount && lastPosition == itemCount - 1) {
              lastItemCount = itemCount;
              this.onLoadMore();
          }
      }
  }
  ```

- **获取数据后做 Diff 更新：**

  可以在 `Activity` 中进行 Diff，或者继承 `MultiTypeAdapter` 提供接收数据方法，在方法中进行 Diff. **MultiType** 不提供内置 Diff 方案，不然需要依赖 v4 包，并且这也不应该属于它的范畴。

- **HeaderView、FooterView**

  **MultiType** 其实本身就支持 `HeaderView`、`FooterView`，只要创建一个 `Header.class` - `HeaderViewProvider` 和 `Footer.class` - `FooterViewProvider` 即可，然后把 `new Header()` 添加到 `items` 第一个位置，把 `new Footer` 添加到 `items` 最后一个位置即可。需要注意的是，如果使用了 Footer View，在底部插入数据的时候，需要添加到 `最后位置 - 1`，即倒二个位置，或者把 `Footer` remove 掉，再添加数据，最后再插入一个新的 `Footer`.


## 更多示例

**MultiType** 的开源项目提供了许多的 sample (示例) 程序：

- [drakeet/about-page](https://github.com/drakeet/about-page)

  一个 material design 的关于页面，包含了多种 Item，并且非常易于使用。

  ![](http://ww2.sinaimg.cn/large/86e2ff85gw1f93gq2tevbj21700pcjyp.jpg)

- [线性和网格布局混排](https://github.com/drakeet/MultiType/tree/master/sample/src/main/java/me/drakeet/multitype/sample/grid)

  使用 `MultiType` 和 `GridLayoutManager` 实现网格和线性混合布局，简单、直观。

  <img src="https://github.com/drakeet/MultiType/raw/master/art/screenshot-multigrid.png" width=270 height=486/>

- [drakeet/TimeMachine](https://github.com/drakeet/TimeMachine)

  TimeMachine 使用了 **MultiType** 来创建一个复杂的聊天页面，页面和需求虽然复杂，但使用 **MultiType** 却轻松简单。

  <img src="https://github.com/drakeet/TimeMachine/raw/master/art/ts2.jpg" width="270" height="486"/>

- [类似 Bilibili iOS 端首页](https://github.com/drakeet/MultiType/tree/master/sample/src/main/java/me/drakeet/multitype/sample/bilibili)

  使用 `MultiType` 轻轻松松实现类似 Bilibili iOS 端首页复杂的多类型列表视图，却能保持很好的复用性能和清晰的代码。

  <img src="https://github.com/drakeet/MultiType/raw/master/art/screenshot-bilibili.png" width="270" height="486"/>

## 设计思想

**MultiType** 设计伊初，我就给它定了几个原则：

- 要简单，便于他人阅读代码。

  因此我极力去避免将它复杂化，比如引入 item id 机制，比如加入许多不相干的内容，比如使用 apt + 注解完成类型和 View 自动绑定、自动注册，再比如，使用反射。这些我都是拒绝的。我想写人人可读的代码，使用简单的方式，去实现复杂的需求。过多不相干、没必要的代码，将会使项目变得令人晕头转向，难以阅读，遇到需要定制、解决问题的时候，无从下手。

- 要灵活，便于拓展和适应各种需求

  很多人会得意地告诉我，他们把 **MultiType** 源码精简成三个类，甚至一个类，以为代码越少就是越好，这我也是不能赞同的，**MultiType** 考虑得比他们更远，这是一个提供给大众使用的类库，过度的精简只会使得灵活性大幅失去。**它或许不是使用起来最简单的，但绝对是使用起来最灵活的。** 在我看来，灵活性的优先级大于简单性。因此，**MultiType** 各个组件都是以接口进行连接，这意味着它所有的角色、组件都可以被替换，或者被拓展和继承。如果你觉得它使用起来还不够简单，完全可以通过继承来封装出更具体符合你使用需求的方法，它已经暴露了足够丰富、周到的接口以供继承。我们不应该直接去修改源码，这会导致一旦你后续发现你的精简版满足不了你的需求时候，已经没有回头路了。

- 要直观，使用起来能令项目代码更清晰、模块化

  **MultiType** 提供的 `ItemViewProvider` 沿袭了 `RecyclerView Adapter` 的接口命名，使用起来更加舒适，符合习惯和直觉。另外，`ItemViewProvider` 需要提供了 类型 泛型，虽然略微有点儿麻烦，但这样带来的好处却是很多的，指定泛型之后，我们不需要再自己做强制转型，而且代码能够显式表明 `ItemViewProvider` 和 `Item class` 的对应关系。遵循了简单可依赖的原则。另外，现在我们有 MultiTypeTemplates 插件来自动生成代码，这个过程变得更加顺滑简单。

## 引用文献

- 《Android 内存泄漏案例和解析》https://drakeet.me/android-leaks
- 《Android 复杂的多类型列表视图新写法：MultiType》https://drakeet.me/multitype
