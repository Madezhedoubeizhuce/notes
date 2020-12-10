# BaseRecycleViewAdapterHelper解析

BaseRecycleViewAdapterHelper是一个强大并且灵活的RecyclerViewAdapter，使用这个adapter能够快速使用RecycleView生成各种形式的列表，包括添加Header、Footer、空布局、多布局、折叠/展开等。这里来介绍一下他的实现原理

## 基本使用

在学习一个框架原理之前肯定要先学会使用这个框架，BaseRecycleViewAdapterHelper有一个基类Adapter：*BaseQuickAdapter*，我们只需要继承这个Adapter并重写convert()方法就可以实现一个RecycleView的Adapter。

如下：

```kotlin
class FilterAdapter(data: List<FilterItem>) :
    BaseQuickAdapter<FilterItem, BaseViewHolder>(R.layout.item_filter, data) {
    override fun convert(helper: BaseViewHolder?, item: FilterItem?) {
        helper?.apply {
            setText(R.id.tvTitle, item?.title ?: "")
            setText(R.id.tvValue, item?.value ?: "")
            setGone(R.id.ivEnter, item?.showIcon == true)
            addOnClickListener(R.id.rlFilterItem)
        }
    }
}
```

这里需要继承BaseQuickAdapter，然后声明数据类型和ViewHolder类型，同时指明item的布局文件，最后再重写convert方法。这里接收一个BaseViewHolder实例，这个类中提供了一些常用的控件操作方法，比如setText、setImageResource等。在convert的实现中，调用BaseViewHolder相关函数给布局中的UI设值。

总体来说，这样子比自己完全重头实现一个RecyclerView.Adapter会方便很多，接下去就要看一下其核心实现了。

## 核心实现

RecyclerView的adapter都是通过ViewHolder来复用item布局的，此库提供了一个BaseViewHolder基类，他继承自RecyclerView.ViewHolder，BaseViewHolder提供了一些常用的方法。这里先来看一下BaseViewHolder：

### BaseViewHolder

```kotlin
import android.graphics.Bitmap
import android.graphics.drawable.Drawable
import android.util.SparseArray
import android.view.View
import android.widget.ImageView
import android.widget.TextView
import androidx.annotation.*
import androidx.databinding.DataBindingUtil
import androidx.databinding.ViewDataBinding
import androidx.recyclerview.widget.RecyclerView

/**
 * ViewHolder 基类
 */
@Keep
open class BaseViewHolder(view: View) : RecyclerView.ViewHolder(view) {
    /**
     * Views indexed with their IDs
     */
    private val views: SparseArray<View> = SparseArray()

    /**
     * 如果使用了 DataBinding 绑定 View，可调用此方法获取 [ViewDataBinding]
     *
     * Deprecated, Please use [BaseDataBindingHolder]
     *
     * @return B?
     */
    @Deprecated("Please use BaseDataBindingHolder class", ReplaceWith("DataBindingUtil.getBinding(itemView)", "androidx.databinding.DataBindingUtil"))
    open fun <B : ViewDataBinding> getBinding(): B? = DataBindingUtil.getBinding(itemView)


    fun <T : View> getView(@IdRes viewId: Int): T {
        val view = getViewOrNull<T>(viewId)
        checkNotNull(view) { "No view found with id $viewId" }
        return view
    }

    @Suppress("UNCHECKED_CAST")
    fun <T : View> getViewOrNull(@IdRes viewId: Int): T? {
        val view = views.get(viewId)
        if (view == null) {
            itemView.findViewById<T>(viewId)?.let {
                views.put(viewId, it)
                return it
            }
        }
        return view as? T
    }

    fun <T : View> Int.findView(): T? {
        return itemView.findViewById(this)
    }

    open fun setText(@IdRes viewId: Int, value: CharSequence?): BaseViewHolder {
        getView<TextView>(viewId).text = value
        return this
    }

    open fun setText(@IdRes viewId: Int, @StringRes strId: Int): BaseViewHolder? {
        getView<TextView>(viewId).setText(strId)
        return this
    }

    open fun setTextColor(@IdRes viewId: Int, @ColorInt color: Int): BaseViewHolder {
        getView<TextView>(viewId).setTextColor(color)
        return this
    }

    open fun setTextColorRes(@IdRes viewId: Int, @ColorRes colorRes: Int):
    	BaseViewHolder {
        getView<TextView>(viewId).setTextColor(itemView.resources.getColor(colorRes))
        return this
    }

    open fun setImageResource(@IdRes viewId: Int, @DrawableRes imageResId: Int): 	
    	BaseViewHolder {
        getView<ImageView>(viewId).setImageResource(imageResId)
        return this
    }

    open fun setImageDrawable(@IdRes viewId: Int, drawable: Drawable?): BaseViewHolder {
        getView<ImageView>(viewId).setImageDrawable(drawable)
        return this
    }

    open fun setImageBitmap(@IdRes viewId: Int, bitmap: Bitmap?): BaseViewHolder {
        getView<ImageView>(viewId).setImageBitmap(bitmap)
        return this
    }

    open fun setBackgroundColor(@IdRes viewId: Int, @ColorInt color: Int): 
    	BaseViewHolder {
        getView<View>(viewId).setBackgroundColor(color)
        return this
    }

    open fun setBackgroundResource(@IdRes viewId: Int, @DrawableRes backgroundRes: Int): BaseViewHolder {
        getView<View>(viewId).setBackgroundResource(backgroundRes)
        return this
    }

    open fun setVisible(@IdRes viewId: Int, isVisible: Boolean): BaseViewHolder {
        val view = getView<View>(viewId)
        view.visibility = if (isVisible) View.VISIBLE else View.INVISIBLE
        return this
    }

    open fun setGone(@IdRes viewId: Int, isGone: Boolean): BaseViewHolder {
        val view = getView<View>(viewId)
        view.visibility = if (isGone) View.GONE else View.VISIBLE
        return this
    }
    
    open fun setEnabled(@IdRes viewId: Int, isEnabled: Boolean): BaseViewHolder {
        getView<View>(viewId).isEnabled = isEnabled
        return this
    }
}
```

BaseViewHolder提供了getView、getViewOrNull两个方法根据id获取item中的子view，setText方法用于设置TextView的内容，setTextColor用于设置文本框字体颜色，setImageResource、setImageDrawable、setImageBitmap用于设置ImageView图片，setBackgroundColor、setBackgroundResource用于设置View的背景，setVisible、setGone设置View的显示及隐藏，setEnabled用于启用或禁用View。

这些都是一般情况下比较常用作列表item项的布局的，如果有更复杂的布局需要，可以自定义VIewHolder继承BaseViewHolder进行扩展。

### ViewHolder创建

在使用RecycleView时，都要继承RecyclerView.Adapter来实现item项的显示，而在实现RecyclerView.Adapter时，必须重写onCreateViewHolder()、onBindViewHolder()和getItemCount()这三个方法。onCreateViewHolder方法用于创建ViewHolder实例，onBindViewHolder用来更新RecyclerView的item的数据内容，会在每个item被滚动到屏幕内的时候被回调。此adapter库提供了一个BaseQuickAdapter类，可以通过继承此adapter快速使用RecycleView。

BaseQuickAdapter是这个adapter库的核心类，它继承RecyclerView.Adapter，实现了这个库的核心功能，包括创建、绑定ViewHolder，item点击事件监听，添加header、footer，设置动画等功能。此库中所有的adapter都继承于BaseQuickAdapter。

首先，分析一下BaseQuickAdapter中ViewHolder的创建流程：

```kotlin
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val baseViewHolder: VH
        when (viewType) {
            LOAD_MORE_VIEW -> {
                val view = mLoadMoreModule!!.loadMoreView.getRootView(parent)
                baseViewHolder = createBaseViewHolder(view)
                mLoadMoreModule!!.setupViewHolder(baseViewHolder)
            }
            HEADER_VIEW -> {
                val headerLayoutVp: ViewParent? = mHeaderLayout.parent
                if (headerLayoutVp is ViewGroup) {
                    headerLayoutVp.removeView(mHeaderLayout)
                }

                baseViewHolder = createBaseViewHolder(mHeaderLayout)
            }
            EMPTY_VIEW -> {
                val emptyLayoutVp: ViewParent? = mEmptyLayout.parent
                if (emptyLayoutVp is ViewGroup) {
                    emptyLayoutVp.removeView(mEmptyLayout)
                }

                baseViewHolder = createBaseViewHolder(mEmptyLayout)
            }
            FOOTER_VIEW -> {
                val footerLayoutVp: ViewParent? = mFooterLayout.parent
                if (footerLayoutVp is ViewGroup) {
                    footerLayoutVp.removeView(mFooterLayout)
                }

                baseViewHolder = createBaseViewHolder(mFooterLayout)
            }
            else -> {
                val viewHolder = onCreateDefViewHolder(parent, viewType)
                bindViewClickListener(viewHolder, viewType)
                mDraggableModule?.initView(viewHolder)
                onItemViewHolderCreated(viewHolder, viewType)
                baseViewHolder = viewHolder
            }
        }

        return baseViewHolder
    }
```

onCreateViewHolder中又多重分支，每个分支都代表不同类型的item布局，这里暂时先不分析其他类型的item，直接把焦点放到else分支上。

else分支第一行就调用了onCreateDefViewHolder(parent, viewType)方法，可以看出是这个方法创建了对应的viewHolder，那么来看下他的实现：

```kotlin
/**
     * Override this method and return your ViewHolder.
     * 重写此方法，返回你的ViewHolder。
     */
protected open fun onCreateDefViewHolder(parent: ViewGroup, viewType: Int): VH {
    return createBaseViewHolder(parent, layoutResId)
}
```

这里可以在自己的adapter中重写此方法创建自定义的ViewHolder，默认调用了createBaseViewHolder(parent, layoutResId)方法来创建默认的ViewHolder，这里来看下默认的实现：

```kotlin
protected open fun createBaseViewHolder(parent: ViewGroup, 
                                        @LayoutRes layoutResId: Int): VH {
    return createBaseViewHolder(parent.getItemView(layoutResId))
}
```

这里通过parent.getItemView(layoutResId)来创建item的布局，这里是通过kotlin的扩展函数给ViewGroup添加了一个getItemView的扩展函数：

```kotlin
/**
 * 扩展方法，用于获取View
 * @receiver ViewGroup parent
 * @param layoutResId Int
 * @return View
 */
fun ViewGroup.getItemView(@LayoutRes layoutResId: Int): View {
    return LayoutInflater.from(this.context).inflate(layoutResId, this, false)
}
```

这里就是通过inflater创建item的view实例。

再回到createBaseViewHolder(parent, layoutResId)方法中，它内部又通过createBaseViewHolder(view: View)创建ViewHolder实例：

```kotlin
protected open fun createBaseViewHolder(view: View): VH {
    var temp: Class<*>? = javaClass
    var z: Class<*>? = null
    while (z == null && null != temp) {
        z = getInstancedGenericKClass(temp)
        temp = temp.superclass
    }
    // 泛型擦除会导致z为null
    val vh: VH? = if (z == null) {
        BaseViewHolder(view) as VH
    } else {
        createBaseGenericKInstance(z, view)
    }
    return vh ?: BaseViewHolder(view) as VH
}

/**
  * get generic parameter VH
  *
  * @param z
  * @return
  */
private fun getInstancedGenericKClass(z: Class<*>): Class<*>? {
    try {
        val type = z.genericSuperclass
        if (type is ParameterizedType) {
            val types = type.actualTypeArguments
            for (temp in types) {
                if (temp is Class<*>) {
                    if (BaseViewHolder::class.java.isAssignableFrom(temp)) {
                        return temp
                    }
                } else if (temp is ParameterizedType) {
                    val rawType = temp.rawType
                    if (rawType is Class<*> && 
                        	BaseViewHolder::class.java.isAssignableFrom(rawType)) {
                        return rawType
                    }
                }
            }
        }
    } catch (e: java.lang.reflect.GenericSignatureFormatError) {
        e.printStackTrace()
    } catch (e: TypeNotPresentException) {
        e.printStackTrace()
    } catch (e: java.lang.reflect.MalformedParameterizedTypeException) {
        e.printStackTrace()
    }
    return null
}

/**
  * try to create Generic VH instance
  *
  * @param z
  * @param view
  * @return
  */
@Suppress("UNCHECKED_CAST")
private fun createBaseGenericKInstance(z: Class<*>, view: View): VH? {
    try {
        val constructor: Constructor<*>
        // inner and unstatic class
        return if (z.isMemberClass && !Modifier.isStatic(z.modifiers)) {
            constructor = z.getDeclaredConstructor(javaClass, View::class.java)
            constructor.isAccessible = true
            constructor.newInstance(this, view) as VH
        } else {
            constructor = z.getDeclaredConstructor(View::class.java)
            constructor.isAccessible = true
            constructor.newInstance(view) as VH
        }
    } catch (e: NoSuchMethodException) {
        e.printStackTrace()
    } catch (e: IllegalAccessException) {
        e.printStackTrace()
    } catch (e: InstantiationException) {
        e.printStackTrace()
    } catch (e: InvocationTargetException) {
        e.printStackTrace()
    }

    return null
}
```

getInstancedGenericKClass()方法先通过反射获得子adapter中的泛型参数列表，然后找到类型为BaseViewHolder子类的Class对象并将其返回。如果因为泛型擦除而获取不到BaseViewHolder的Class对象，直接创建BaseViewHolder，否则通过反射创建对应ViewHolder 的实例。

### 点击事件绑定

RecyclerView没有ListView默认的item点击事件监听，一般都要自己在adapter中添加，BaseQuickAdapter类提供了OnItemClickListener、OnItemLongClickListener、OnItemChildClickListener以及OnItemChildLongClickListener，用来监听RecyclerView每个item及内部child的点击事件，先来看下基本的使用：

```kotlin
public class ItemClickAdapter extends BaseMultiItemQuickAdapter<ClickEntity, BaseViewHolder> implements OnItemClickListener, OnItemChildClickListener {

    public ItemClickAdapter(List<ClickEntity> data) {
        super(data);
        ......

        addChildClickViewIds(R.id.btn,
                R.id.iv_num_reduce, R.id.iv_num_add,
                R.id.item_click);

        addChildLongClickViewIds(R.id.iv_num_reduce, R.id.iv_num_add,
                R.id.btn);
    }


    @Override
    protected void convert(@NonNull final BaseViewHolder helper, 
                           final ClickEntity item) {
        ......
    }
}

adapter.setOnItemClickListener(new OnItemClickListener() {
    @Override
    public void onItemClick(@NonNull BaseQuickAdapter<?, ?> adapter, 
                            @NonNull View view, int position) {
        Tips.show("onItemClick " + position);
    }
});
adapter.setOnItemLongClickListener(new OnItemLongClickListener() {
    @Override
    public boolean onItemLongClick(@NonNull BaseQuickAdapter adapter, 
                                   @NonNull View view, int position) {
        Tips.show("onItemLongClick " + position);
        return true;
    }
});
adapter.setOnItemChildClickListener(new OnItemChildClickListener() {
    @Override
    public void onItemChildClick(@NonNull BaseQuickAdapter adapter, 
                                 @NonNull View view, int position) {
        Tips.show("onItemChildClick " + position);
    }
});
adapter.setOnItemChildLongClickListener(new OnItemChildLongClickListener() {
    @Override
    public boolean onItemChildLongClick(@NonNull BaseQuickAdapter adapter, 
                                        @NonNull View view, int position) {
        Tips.show("onItemChildLongClick " + position);
        return true;
    }
});
```

可以看出，在设置item的点击监听时，直接调用adapter.setOnItemClickListener()及adapter.setOnItemLongClickListener()方法即可实现监听。但是在设置item中具体子view监听时，则需要先调用adapter的addChildClickViewIds()或addChildLongClickViewIds()方法将要添加点击事件对应view的ID添加到adapter中，然后调用setOnItemChildClickListener、setOnItemChildLongClickListener添加对应的监听。

需要注意的一点是，addChildClickViewIds和addChildLongClickViewIds两个方法不能在convert方法中调用，在convert中调用，点击事件不会生效，详细原因接下来的分析中就会讲到。

在初步介绍了如何添加click监听之后，就要开始分析BaseQuickAdapter如何实现的item监听，这里先回到BaseQuickAdapter的onCreateViewHolder方法中：

```kotlin
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val baseViewHolder: VH
        when (viewType) {
            ......
            else -> {
                val viewHolder = onCreateDefViewHolder(parent, viewType)
                bindViewClickListener(viewHolder, viewType)
                mDraggableModule?.initView(viewHolder)
                onItemViewHolderCreated(viewHolder, viewType)
                baseViewHolder = viewHolder
            }
        }

        return baseViewHolder
    }
```

可以看到这里调用了bindViewClickListener()方法，在其中实现了点击事件的绑定：

```kotlin
    /**
     * 绑定 item 点击事件
     * @param viewHolder VH
     */
    protected open fun bindViewClickListener(viewHolder: VH, viewType: Int) {
        mOnItemClickListener?.let {
            viewHolder.itemView.setOnClickListener { v ->
                var position = viewHolder.adapterPosition
                if (position == RecyclerView.NO_POSITION) {
                    return@setOnClickListener
                }
                position -= headerLayoutCount
                setOnItemClick(v, position)
            }
        }
        mOnItemLongClickListener?.let {
            viewHolder.itemView.setOnLongClickListener { v ->
                var position = viewHolder.adapterPosition
                if (position == RecyclerView.NO_POSITION) {
                    return@setOnLongClickListener false
                }
                position -= headerLayoutCount
                setOnItemLongClick(v, position)
            }
        }

        mOnItemChildClickListener?.let {
            for (id in getChildClickViewIds()) {
                viewHolder.itemView.findViewById<View>(id)?.let { childView ->
                    if (!childView.isClickable) {
                        childView.isClickable = true
                    }
                    childView.setOnClickListener { v ->
                        var position = viewHolder.adapterPosition
                        if (position == RecyclerView.NO_POSITION) {
                            return@setOnClickListener
                        }
                        position -= headerLayoutCount
                        setOnItemChildClick(v, position)
                    }
                }
            }
        }
        mOnItemChildLongClickListener?.let {
            for (id in getChildLongClickViewIds()) {
                viewHolder.itemView.findViewById<View>(id)?.let { childView ->
                    if (!childView.isLongClickable) {
                        childView.isLongClickable = true
                    }
                    childView.setOnLongClickListener { v ->
                        var position = viewHolder.adapterPosition
                        if (position == RecyclerView.NO_POSITION) {
                            return@setOnLongClickListener false
                        }
                        position -= headerLayoutCount
                        setOnItemChildLongClick(v, position)
                    }
                }
            }
        }
    }
```

bindViewClickListener中，只有当对应的监听器，诸如mOnItemClickListener非空时才会绑定item的点击事件，否则就不会进行绑定，这些监听器都是用户需要手工调用setOnItemClickListener等接口进行设置的。因此，这里说明了在adapter创建ViewHolder之前要先设置监听，ViewHolder已经创建之后设置的监听都无法生效。

BaseQuickAdapter提供了下面这些设置的点击事件监听的方法：

```kotlin
override fun setOnItemClickListener(listener: OnItemClickListener?) {
    this.mOnItemClickListener = listener
}

override fun setOnItemLongClickListener(listener: OnItemLongClickListener?) {
    this.mOnItemLongClickListener = listener
}

override fun setOnItemChildClickListener(listener: OnItemChildClickListener?) {
    this.mOnItemChildClickListener = listener
}

override fun setOnItemChildLongClickListener(listener: OnItemChildLongClickListener?) {
    this.mOnItemChildLongClickListener = listener
}
```

bindViewClickListener中分为四部分，分别是item点击事件绑定、item长按事件绑定、item子view点击事件绑定和item子view长按事件绑定，先来看一下item点击和长按事件绑定流程：

```kotlin 
        mOnItemClickListener?.let {
            viewHolder.itemView.setOnClickListener { v ->
                var position = viewHolder.adapterPosition
                if (position == RecyclerView.NO_POSITION) {
                    return@setOnClickListener
                }
                position -= headerLayoutCount
                setOnItemClick(v, position)
            }
        }
```

通过给viewHolder.itemView设置clickListener来监听item的点击事件，viewHolder.itemView就是列表子项的根布局，可以查看一下ViewHolder的构造器。然后通过setOnItemClick方法回调用户设置的监听。

```kotlin
    /**
     * override this method if you want to override click event logic
     *
     * 如果你想重新实现 item 点击事件逻辑，请重写此方法
     * @param v
     * @param position
     */
    protected open fun setOnItemClick(v: View, position: Int) {
        mOnItemClickListener?.onItemClick(this, v, position)
    }
```

item长按事件的流程都差不多，这里就不赘述了。下面来看下item中的子view的点击事件绑定：

```kotlin
        mOnItemChildClickListener?.let {
            for (id in getChildClickViewIds()) {
                viewHolder.itemView.findViewById<View>(id)?.let { childView ->
                    if (!childView.isClickable) {
                        childView.isClickable = true
                    }
                    childView.setOnClickListener { v ->
                        var position = viewHolder.adapterPosition
                        if (position == RecyclerView.NO_POSITION) {
                            return@setOnClickListener
                        }
                        position -= headerLayoutCount
                        setOnItemChildClick(v, position)
                    }
                }
            }
        }
```

首先，通过getChildClickViewIds()方法获取到所有要添加点击监听的view对应的ID，然后循环遍历这些ID，并且使用viewHolder.itemView.findViewById()找到对应的view，如果找到的view默认是不可点击的（如TextView），则将其设为可点击的，然后为期添加点击事件，并且通过setOnItemChildClick()将事件回调给用户。

这里其他逻辑都比较简单，重点来看一下getChildClickViewIds()的实现：

```kotlin
    /**
     * 用于保存需要设置点击事件的 item
     */
    private val childClickViewIds = LinkedHashSet<Int>()

    fun getChildClickViewIds(): LinkedHashSet<Int> {
        return childClickViewIds
    }

    /**
     * 设置需要点击事件的子view
     * @param viewIds IntArray
     */
    fun addChildClickViewIds(@IdRes vararg viewIds: Int) {
        for (viewId in viewIds) {
            childClickViewIds.add(viewId)
        }
    }
```

getChildClickViewIds()只是返回了childClickViewIds列表，childClickViewIds列表又是在addChildClickViewIds()中添加数据的。因此，如前面的demo所示，在添加item中子view点击监听时，要先调用addChildClickViewIds()将要监听的view id添加到adapter中，然后在调用setOnItemChildClickListener添加点击监听。

上诉分析的绑定过程都是在onCreateViewHolder中执行，所以addChildClickViewIds和setOnItemChildClickListener都需要在这之前调用，否则设置的监听就不生效。因为convert是在onBindViewHolder中执行，这也是前面说的为什么不能再convert中调用addChildClickViewIds的原因。

itemChild的长按事件和点击事件的逻辑基本也都一致，这里也就不再赘述了。

### 数据展示

前面分析了ViewHolder的创建以及item的点击事件的绑定过程，接下去就要来分析一下item的的数据展示了。这里主要在onBindViewHolder方法中实现：

```kotlin
override fun onBindViewHolder(holder: VH, position: Int) {
    ......
    when (holder.itemViewType) {
        LOAD_MORE_VIEW -> {
            ......
        }
        HEADER_VIEW, EMPTY_VIEW, FOOTER_VIEW -> return
        else -> convert(holder, getItem(position - headerLayoutCount))
    }
}
```

这里先跳过其他分支，直接看else分支，可以看出，它调用了convert()方法，这也是开头例子中唯一重写的方法，我们再convert方法中根据对应的数据来更新item的界面。

### 动画实现

BaseQuickAdapter提供了setAnimationWithDefault和setAdapterAnimation两个方法，setAnimationWithDefault用以设置默认的动画，而当默认动画不满足需求时，可以通过setAdapterAnimation方法设置自定义的动画效果，此方法接受一个BaseAnimation的参数，自定义动画需继承此类。并需要调用setAnimationEnable(true)方法将动画打开。一个简单的使用如下：

```java
AnimationAdapter mAnimationAdapter = new AnimationAdapter();
mAnimationAdapter.setAnimationEnable(true);
mAnimationAdapter.setAnimationWithDefault(BaseQuickAdapter.AnimationType.AlphaIn);
```

BaseQuickAdapter默认提供五种动画，枚举值分别如下：

```kotlin
/**
  * 内置默认动画类型
  */
enum class AnimationType {
    AlphaIn, ScaleIn, SlideInBottom, SlideInLeft, SlideInRight
}
```

自定义动画的使用如下：

```java
/**
 * 自定义动画2
 */
public class CustomAnimation2 implements BaseAnimation {
    @NotNull
    @Override
    public Animator[] animators(@NotNull View view) {
        Animator translationX =
                ObjectAnimator.ofFloat(view, "translationX", -view.getRootView().getWidth(), 0f);

        translationX.setDuration(800);
        translationX.setInterpolator(new MyInterpolator2());

        return new Animator[]{translationX};
    }

    class MyInterpolator2 implements Interpolator {
        @Override
        public float getInterpolation(float input) {
            float factor = 0.7f;
            return (float) (pow(2.0, -10.0 * input) * sin((input - factor / 4) * (2 * PI) / factor) + 1);
        }
    }
}

AnimationAdapter mAnimationAdapter = new AnimationAdapter();
mAnimationAdapter.setAnimationEnable(true);
mAnimationAdapter.setAdapterAnimation(new CustomAnimation2());
```

下面来分析一下动画的实现，先来看一下setAdapterAnimation和setAnimationWithDefault的实现：

```kotlin
    /**
     * 设置自定义动画
     */
    var adapterAnimation: BaseAnimation? = null
        set(value) {
            animationEnable = true
            field = value
        }

    /**
     * 使用内置默认动画设置
     * @param animationType AnimationType
     */
    fun setAnimationWithDefault(animationType: AnimationType) {
        adapterAnimation = when (animationType) {
            AnimationType.AlphaIn -> AlphaInAnimation()
            AnimationType.ScaleIn -> ScaleInAnimation()
            AnimationType.SlideInBottom -> SlideInBottomAnimation()
            AnimationType.SlideInLeft -> SlideInLeftAnimation()
            AnimationType.SlideInRight -> SlideInRightAnimation()
        }
    }
```

setAdapterAnimation起始就是kotlin中的adapterAnimation属性，而setAnimationWithDefault就是根据传入的动画类型创建对应的实例赋值给adapterAnimation。我们这里先挑一个默认的动画效果看一下，就先看AlphaInAnimation类吧：

```kotlin
class AlphaInAnimation @JvmOverloads constructor(private val mFrom: Float = DEFAULT_ALPHA_FROM) : BaseAnimation {
    override fun animators(view: View): Array<Animator> {
        val animator = ObjectAnimator.ofFloat(view, "alpha", mFrom, 1f)
        animator.duration = 300L
        animator.interpolator = LinearInterpolator()
        return arrayOf(animator)
    }

    companion object {
        private const val DEFAULT_ALPHA_FROM = 0f
    }

}
```

这是一个淡入的动画，它重写了animators(view: View)方法，在这个方法中创建了一个淡入的动画，最后把她包装成一个数组返回了，这里看一下BaseAnimation接口：

```kotlin
interface BaseAnimation {
    fun animators(view: View): Array<Animator>
}
```

BaseAnimation这个接口只定义了animators，它接收一个需要产生动画的的view，然后返回一个动画数组，用以表示在这个view上要做的动画。

那么接下去要分析的就是这些添加的动画是如何应用在RecyclerView的元素上的，这里就要了解onViewAttachedToWindow(VH holder)方法了：

```java
        /**
         * Called when a view created by this adapter has been attached to a window.
         *
         * <p>This can be used as a reasonable signal that the view is about to be seen
         * by the user. If the adapter previously freed any resources in
         * {@link #onViewDetachedFromWindow(RecyclerView.ViewHolder) 
         * onViewDetachedFromWindow}
         * those resources should be restored here.</p>
         *
         * @param holder Holder of the view being attached
         */
        public void onViewAttachedToWindow(@NonNull VH holder) {
        }
```

onViewAttachedToWindow会在adapter创建的view被关联到当前window上是被调用，也就是对应的item显示时会调用此方法，所以可以在这里给对应的view添加动画。

BaseQuickAdapter的onViewAttachedToWindow如下：

```kotlin
    /**
     * Called when a view created by this holder has been attached to a window.
     * simple to solve item will layout using all
     * [setFullSpan]
     *
     * @param holder
     */
    override fun onViewAttachedToWindow(holder: VH) {
        super.onViewAttachedToWindow(holder)
        val type = holder.itemViewType
        if (isFixedViewType(type)) {
            setFullSpan(holder)
        } else {
            addAnimation(holder)
        }
    }
```

如果当前view不是fixed type时，就会添加动画。什么是fixed type呢，其实就是header、footer、加载更多这些非数据的item。这里重点是添加动画，所以我们接下来分析addAnimation(holder)方法：

```kotlin
    /**
     * add animation when you want to show time
     *
     * @param holder
     */
    private fun addAnimation(holder: RecyclerView.ViewHolder) {
        if (animationEnable) {
            if (!isAnimationFirstOnly || holder.layoutPosition > mLastPosition) {
                val animation: BaseAnimation = adapterAnimation?.let {
                    it
                } ?: AlphaInAnimation()
                animation.animators(holder.itemView).forEach {
                    startAnim(it, holder.layoutPosition)
                }
                mLastPosition = holder.layoutPosition
            }
        }
    }
```

首先判断是否要开启动画效果，如果开启，就取出之前设置的的动画数组开始绘制动画。这里要注意里面那层判断条件：

```kotlin
!isAnimationFirstOnly || holder.layoutPosition > mLastPosition
```

isAnimationFirstOnly表示动画是否仅第一次执行，默认为false，此时肯定会执行if内的逻辑，即不管是向上滑动和向下滑动时都绘制动画，而如果为true则表示，只有第一次往下滑动时才会加载动画，而向上滑动或者回到顶部再次下滑都不再加载动画。

那这个效果是如何实现的，那就是第二个判断条件的作用了，`holder.layoutPosition > mLastPosition` holder.layoutPosition表示当前显示item的位置，mLastPosition则表示上次绘制动画的位置。

mLastPosition初始值为-1，列表第一次下滑时，当列表数据依次被加载时，holder.layoutPosition从1开始递增，第一个item被加载时，mLastPosition为-1，此时`holder.layoutPosition > mLastPosition` 为true，则绘制动画，同时将mLastPosition设置为当前item的位置，这样再加载下一个item时，`holder.layoutPosition > mLastPosition`仍为true。而上滑时，holder.layoutPosition在不断递减，mLastPosition确没有更新，故此时不会进入if内。然后再次下滑时，一直到mLastPosition记录的位置之前，都不会进入if内，也就实现了只有第一次下拉才会加载动画。

### Header/Footer/空数据实现

BaseQuickAdapter提供addHeaderView、setHeaderView、addFooterView和setFooterView来添加Header和Footer。

```kotlin
@JvmOverloads
fun addHeaderView(view: View, index: Int = -1, orientation: Int = LinearLayout.VERTICAL): Int {
    if (!this::mHeaderLayout.isInitialized) {
        mHeaderLayout = LinearLayout(view.context)
        mHeaderLayout.orientation = orientation
        mHeaderLayout.layoutParams = if (orientation == LinearLayout.VERTICAL) {
            RecyclerView.LayoutParams(MATCH_PARENT, WRAP_CONTENT)
        } else {
            RecyclerView.LayoutParams(WRAP_CONTENT, MATCH_PARENT)
        }
    }

    val childCount = mHeaderLayout.childCount
    var mIndex = index
    if (index < 0 || index > childCount) {
        mIndex = childCount
    }
    mHeaderLayout.addView(view, mIndex)
    if (mHeaderLayout.childCount == 1) {
        val position = headerViewPosition
        if (position != -1) {
            notifyItemInserted(position)
        }
    }
    return mIndex
}

@JvmOverloads
fun setHeaderView(view: View, index: Int = 0, orientation: Int = LinearLayout.VERTICAL): Int {
    return if (!this::mHeaderLayout.isInitialized || mHeaderLayout.childCount <= index) {
        addHeaderView(view, index, orientation)
    } else {
        mHeaderLayout.removeViewAt(index)
        mHeaderLayout.addView(view, index)
        index
    }
}
```

setHeaderView内部也是通过addHeaderView添加Header的，所以重点分析addHeaderView。

addHeaderView中，如果mHeaderLayout没有初始化过，则创建新的LinearLayout，然后调用mHeaderLayout.addView添加header。

然后调用notifyItemInserted方法通知adapter插入了一条数据，插入的位置是0。接下去又回到了onCreateViewHolder中:

```kotlin
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val baseViewHolder: VH
        when (viewType) {
            ......
            HEADER_VIEW -> {
                val headerLayoutVp: ViewParent? = mHeaderLayout.parent
                if (headerLayoutVp is ViewGroup) {
                    headerLayoutVp.removeView(mHeaderLayout)
                }

                baseViewHolder = createBaseViewHolder(mHeaderLayout)
            }
            ......
        }

        return baseViewHolder
    }
```

如果当前header已经有父节点，则需要将它从父节点删除，然后再重新添加进去。而其添加到adapter中的逻辑就是createBaseViewHolder方法，之前分析过，这里就不再赘述了。

Footer和空数据的添加过程和Header的过程是差不多的，这里也不再进行分析了。

### 局部刷新实现

有时候，我们列表中可能只有部分数据需要更新，比如文字，而图片什么的并没有改变，如果整个item都刷新的话会浪费很多资源，所以这时候就需要局部刷新了。局部刷新是通过重载方法`onBindViewHolder(holder: VH, position: Int, payloads: MutableList<Any>)`实现的，此重载函数多了一个payloads参数，下面介绍一下payloads:

这个 payloads 参数是一个 List 对象，该对象不是 null 但可能是 空的。通过 Adapter 的 notifyXXX 函数的带有 payload 参数的函数可以设置 payload 对象，例如通知一个条目数据变化的函数：

```java
public final void notifyItemChanged(int position, Object payload) {
    mObservable.notifyItemRangeChanged(position, 1, payload);
}
```

下面来看看这个 payload 是干什么的。

如果没有 payload ， 当调用 notifyItemChanged 的时候， RecyclerView 会通过回调 onBindViewHolder(holder, position) 来更新当前数据变化的 View，但是对于比较复杂的条目，里面有很多个不同的控件，比如有图片、文字、CheckBox 等，用户点击一下当前条目（比如 喜欢一个微博信息），需要把喜欢的状态高亮。 也就是说，当前一个微博条目中只有一个喜欢状态的变化，但是需要重新在 onBindViewHolder(holder, position) 中设置所有View 的内容。对于每个 View ，当设置其内容的时候，都会触发 View 的重新布局和计算位置，这样至少一个 View 状态变化了 最终导致整个条目都需要重新布局一遍。

如果通过 payload 来告诉系统这个微博消息只有喜欢状态变化了，这样在调用 onBindViewHolder(VH holder, int position, List

需要注意的是，当 payloads 为 空的时候，说明是该条目的整个数据都变化了， 需要更新所有的数据，所以你可以在当 payloads 为 空 的时候调用不带 payloads 参数的函数。

BaseQuickAdapter中重写了这个函数：

```kotlin
    override fun onBindViewHolder(holder: VH, position: Int, payloads: MutableList<Any>) {
        if (payloads.isEmpty()) {
            onBindViewHolder(holder, position)
            return
        }
        //Add up fetch logic, almost like load more, but simpler.
        mUpFetchModule?.autoUpFetch(position)
        //Do not move position, need to change before LoadMoreView binding
        mLoadMoreModule?.autoLoadMore(position)
        when (holder.itemViewType) {
            LOAD_MORE_VIEW -> {
                mLoadMoreModule?.let {
                    it.loadMoreView.convert(holder, position, it.loadMoreStatus)
                }
            }
            HEADER_VIEW, EMPTY_VIEW, FOOTER_VIEW -> return
            else -> convert(holder, getItem(position - headerLayoutCount), payloads)
        }
    }
```

这里先略过其他逻辑，直接跳到else分支中，在这里调用了`convert(holder: VH, item: T, payloads: List<Any>)` 函数，因此，在需要实现局部刷新的列表中需要重写对应的convert方法。

## 多布局实现



## 带头部列表



## 可收缩列表



## 加载更多






