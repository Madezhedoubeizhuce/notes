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

可以看到这里

### 数据展示

### 动画实现

### Header/Footer实现

### 空数据实现

### DataBinding实现

### 局部刷新实现

## 多布局实现

## 带头部列表

## 可收缩列表

## 加载更多




