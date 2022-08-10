LayoutInflater也是通过Context获取，它也是系统服务的一种，被注册在ContextImpl的map里，然后通过LAYOUT_INFLATER_SERVICE来获取。

```java
layoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

LayoutInflater是一个抽象类，它的实现类是**PhoneLayoutInflater**。LayoutInflater会采用**深度优先遍历**自顶向下遍历View树，根据View的全路径名利用反射获取构造器 从而构建View的实例。

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)
```

它主要有两个方面的作用：

- 当attachToRoot == true且root ！= null时，新解析出来的View会被add到root中去，然后将root作为结果返回。
- 当attachToRoot == false且root ！= null时，新解析的View会直接作为结果返回，而且root会为新解析的View生成LayoutParams并设置到该View中去。
- 当attachToRoot == false且root == null时，新解析的View会直接作为结果返回。

注意第二条和第三条是由区别的，你可以去写个例子试一下，当root为null时，新解析出来的View没有LayoutParams参数，这时候你设置的layout_width和layout_height是不生效的。

> Activity的setContentView()方法，实际上调用的PhoneWindow的setContentView()方法。它调用的时候将Activity的顶级DecorView（FrameLayout） 作为root传了进去，mLayoutInflater.inflate(layoutResID,  mContentParent)实际调用的是inflate(resource, root, root !=  null)，所以在调用Activity的setContentView()方法时 可以将解析出的View添加到顶级DecorView中，我们设置的layout_width和layout_height参数也可以生效。

具体代码如下：

```java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

# inflate()

```java
public abstract class LayoutInflater {

    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }
        
        //获取xml资源解析器XmlResourceParser
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);//解析View
        } finally {
            parser.close();
        }
    }
}
```

可以发现在该方法里，主要完成了两件事情：

1. 获取xml资源解析器XmlResourceParser。
2. 解析View

我们先来看看XmlResourceParser是如何获取的。

从上面的序列图可以看出，调用了Resources的getLayout(resource)去获取对应的XmlResourceParser。getLayout(resource)又去调用了Resources的loadXmlResourceParser() 方法来完成XmlResourceParser的加载，如下所示：

```java
public class Resources {
    
     XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type)
             throws NotFoundException {
         final TypedValue value = obtainTempTypedValue();
         try {
             final ResourcesImpl impl = mResourcesImpl;
             //1. 获取xml布局资源，并保存在TypedValue中。
             impl.getValue(id, value, true);
             if (value.type == TypedValue.TYPE_STRING) {
                 //2. 加载对应的loadXmlResourceParser解析器。
                 return impl.loadXmlResourceParser(value.string.toString(), id,
                         value.assetCookie, type);
             }
             throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)
                     + " type #0x" + Integer.toHexString(value.type) + " is not valid");
         } finally {
             releaseTempTypedValue(value);
         }
     }   
}
```

