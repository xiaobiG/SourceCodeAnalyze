# 封装权限处理



# 封装数据保存

```
setRetainInstance(boolean retain)
```

表示当 Activity 重新创建的时候， fragment 实例是否会被重新创建（比如横竖屏切换），设置为 true，表示 configuration change 的时候，fragment 实例不会背重新创建，这样，有一个好处，即 configuration 变化的时候，我们不需要再做额外的处理。



# Example

```kotlin
  override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setStyle(STYLE_NO_TITLE, R.style.Theme_Holo_Light_Dialog_MinWidth)
    }

    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog {
        val dialog = super.onCreateDialog(savedInstanceState)
        dialog.setCanceledOnTouchOutside(true)
        return dialog
    }

```

