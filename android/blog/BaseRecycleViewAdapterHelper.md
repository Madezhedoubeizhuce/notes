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

### ViewHolder创建

### item点击事件

### ViewHolder绑定及UI更新

