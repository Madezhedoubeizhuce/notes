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

BaseQuickAdapter是这个框架的核心类，其他的多布局等adapter也都是继承自这个类的，包含创建绑定ViewHolder，添加header、footer，设置动画等逻辑。在分析BaseQuickAdapter工作流程之前，需要先介绍一下ViewHolder类。

### BaseViewHolder

BaseViewHolder是ViewHolder的基类，提供了一些基本操控item中child view的方法，比如设置文字内容、设置图片或背景等。

```java
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

可以看出，BaseViewHolder提供了getView、getViewOrNull两个方法根据id获取item中的子view，提供了一系列set方法，用以设置子view的属性。

### ViewHolder创建



### 数据展示

### 点击事件绑定

### 动画实现

### Header/Footer实现

### 空数据实现

### DataBinding实现

### 局部刷新实现

## 多布局实现

## 带头部列表

## 可收缩列表

## 加载更多




