# invalidate

默认情况下，View的clipChildren属性为true，即每个View绘制区域不能超出其父View的范围。如果设置一个页面根布局的clipChildren属性为false，则子View可以超出父View的绘制区域。

当一个View触发invalidate，且没有播放动画、没有触发layout的情况下：

- 对于全不透明的View，其自身会设置标志位`PFLAG_DIRTY`，其父View会设置标志位`PFLAG_DIRTY_OPAQUE`。在`draw(canvas)`方法中，只有这个View自身重绘。
- 对于可能有透明区域的View，其自身和父View都会设置标志位`PFLAG_DIRTY`。
- clipChildren为true时，脏区会被转换成ViewRoot中的Rect，刷新时层层向下判断，当View与脏区有重叠则重绘。如果一个View超出父View范围且与脏区重叠，但其父View不与脏区重叠，这个子View不会重绘。
- clipChildren为false时，`ViewGroup.invalidateChildInParent()`中会把脏区扩大到自身整个区域，于是与这个区域重叠的所有View都会重绘。

# requestLayout