# Android ListView不响应OnItemClickListener解决办法

开发中很常见的一个问题，项目中的listview不仅仅是简单的文字，常常需要自己定义listview，自己的Adapter去继承BaseAdapter，在adapter中按照需求进行编写，问题就出现了，可能会发生点击每一个item的时候没有反应，无法获取的焦点。原因多半是由于在你自己定义的Item中存在诸如ImageButton，Button，CheckBox等子控件(也可以说是Button或者Checkable的子类控件)，此时这些子控件会将焦点获取到，所以常常当点击item时变化的是子控件，item本身的点击没有响应。

​    这时候就可以使用descendantFocusability来解决：

#### android:descendantFocusability

Defines the relationship between the ViewGroup and its descendants when looking for a View to take focus.

Must be one of the following constant values.

 

该属性是当一个为view获取焦点时，定义viewGroup和其子控件两者之间的关系。

属性的值有三种：

​        beforeDescendants：viewgroup会优先其子类控件而获取到焦点

​        afterDescendants：viewgroup只有当其子类控件不需要获取焦点时才获取焦点

​        blocksDescendants：viewgroup会覆盖子类控件而直接获得焦点

 

通常我们用到的是第三种，即在Item布局的根布局加上**android:descendantFocusability=”blocksDescendants”**的属性，还有就是在要添加事件的控件上添加**android:focusable="false"**