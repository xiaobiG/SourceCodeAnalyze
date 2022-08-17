#### Window 、 WindowManager、WMS、SurfaceFlinger

- **Window**：抽象概念不是实际存在的，而是以 View 的形式存在，通过 PhoneWindow 实现
- **WindowManager**：外界访问 Window 的入口，内部与 WMS 交互是个 IPC 过程
- **WindowManagerService**：管理窗口 Surface 的布局和次序，作为系统级服务单独运行在一个进程
- **SurfaceFlinger**：将 WMS 维护的窗口按一定次序混合后显示到屏幕上



# Window类型

一般Window有三种类型：应用Window、子Window和系统Window。

应用类的Window对应着一个Activity。子Window是不能单独存在的，他需要在特定的父Window之中，比如常见的Dialog就是一个子Window。系统Window需要声明特殊的权限才能创建，比如Toast跟系统状态栏等。





# Window的分层

Window是分层的，每个Window都有对应的z-ordered，层级大的会覆盖在层级小的Window的上面，这和HTML中的z-index的概念是完全一致的。在三类Window中，应用Window的层级范围是1~99，子Window的层级范围是1000～1999，系统Window的层级范围是2000～2999，这些层级范围对应着WindowManager.LayoutParams的type参数。

如果想要Window位于所有Window的最顶层，那么采用较大的层级即可。

很显然系统Window的层级是最大的，而且系统层级有很多值，一般我们可以选用`TYPE_SYSTEM_OVERLAY`或者`TYPE_SYSTEM_ERROR`，如果采用`TYPE_SYSTEM_ERROR`，只需要为type参数指定这个层级即可：`mLayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR`；

同时声明权限：

`<uses-permissionandroid: name= "android.permission .SYSTEM_ALERT_WINDOW"/>`。因为系统类型的Window是需要检查权限的，如果不在AndroidManifest 中使用相应的权限，那么创建Window的时候就会报错。



# WindowManager

WindowManager的功能比较简单，常用的就是三个方法：addView、updateViewLayout和removeView，这三个方法都定义在ViewManager中，WindowManager继承了ViewManager。

```java
public interface ViewManager
{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

WindowManager操作Window其实就是在操作里面的View。

通过这些方法，我们可以实现诸如随意拖拽位置的Window等效果。

WindowManager的具体实现类是WindowManagerImp：

```java
// wm是WindowManagerImp实例对象
WindowManager wm = (WindowManager) context.getSystemService(WINDOW_SERVICE);

public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    private final Window mParentWindow;

    private IBinder mDefaultToken;

    public WindowManagerImpl(Context context) {
        this(context, null);
    }

    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

    
    public void addView( View view,  ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    
    public void updateViewLayout( View view,  ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

    
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
}
```

WindowManagerImp也没有做什么，它把3个方法的操作都委托给了WindowManagerGlobal单例类

# Window的内部机制

Window是一个抽象类，每个Window都对应一个View跟一个ViewRootImpl，Window跟View是通过ViewRootImpl建立联系的，因此Window并不实际存在，它是以View的形式存在的。从WindowManager的定义跟主要方法也能看出，**View是Window存在的实体**。