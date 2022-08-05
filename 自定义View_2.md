# 改变颜色透明度


```java
    private static int changeAlpha(int color, int alpha) {
        int red = Color.red(color);
        int green = Color.green(color);
        int blue = Color.blue(color);
        return Color.argb(alpha, red, green, blue);
    }

```

# 颜色定义

```
int color;
            switch (i % 3) {
                case 0:
                    color = 0xffcc6666;
                    break;
                case 1:
                    color = 0xffcccc66;
                    break;
                case 2:
                default:
                    color = 0xff6666cc;
                    break;
            }
```

# 获取颜色的方法

```kotlin
ContextCompat.getColor(context, R.color.colorPrimary)
```

```kotlin
Color.parseColor("#FA5858")
```

# 默认宽高处理

```java
    private int defaultWidth = 200;
    private int defaultHeight = 35;
    
     @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(resolveSize(defaultWidth, widthMeasureSpec),
                resolveSize(defaultHeight, heightMeasureSpec));
    }
```

# 尺寸单位转换

```
 private static final Resources sResource = Resources.getSystem();

    public static float dp2px(float dp) {
        DisplayMetrics dm = sResource.getDisplayMetrics();
        return TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dp, dm);
    }
```

TypedValue.java

```java
// 将其他尺寸单位（例如dp，sp）转换为像素单位px
public static float applyDimension(int unit, float value,
                                   DisplayMetrics metrics)
    {
        switch (unit) {
        case COMPLEX_UNIT_PX:
            return value;
        case COMPLEX_UNIT_DIP:
            return value * metrics.density;
        case COMPLEX_UNIT_SP:
            return value * metrics.scaledDensity;
        case COMPLEX_UNIT_PT:
            return value * metrics.xdpi * (1.0f/72);
        case COMPLEX_UNIT_IN:
            return value * metrics.xdpi;
        case COMPLEX_UNIT_MM:
            return value * metrics.xdpi * (1.0f/25.4f);
        }
        return 0;
    }

```

# 从View中获取Bitmap

```java
public static Bitmap loadBitmapFromView(View v) {
        Bitmap b = Bitmap.createBitmap(v.getWidth(), v.getHeight(),
                Bitmap.Config.ARGB_8888);
        Canvas c = new Canvas(b);
        v.layout(v.getLeft(), v.getTop(), v.getRight(), v.getBottom());
        v.draw(c);
        return b;
    }
```

# 将 Bitmap 保存到 File

```java
bitmap.compress(Bitmap.CompressFormat.JPEG, quality, outputStream);
```

```java
File appDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES);
String fileName = System.currentTimeMillis() + ".jpg";
File file = new File(appDir, fileName);
```



# overScrollMode

首先我们先来了解一下与边缘阴影相关的overScrollMode属性。overScrollMode有三个可以说设置的值：

1、always：无论滑动布局的内容是否可以滑动，只要滑动事件超出边界，都会显示边缘阴影。

2、ifContentScrolls：默认值。只有内容可以滑动，并且滑动事件超出边界时，才会显示边缘阴影。

3、never：不显示边缘阴影。

判断内容是否可以滑动的条件是内容的高度是否大于容器的显示高度。比如RecyclerView的item不满一屏时，item不能滑动，但是在always模式下，只要用户的手指滑动屏幕，就会显示边缘阴影。overScrollMode决定了布局是否需要绘制边缘阴影，阴影的绘制则又具体的布局来实现。



# 使用

内部类或外部类都可用下面方法使用：

```xml
 <view xmlns:android="http://schemas.android.com/apk/res/android"
        class="com.example.android.notepad.NoteEditor$LinedEditText"
        android:id="@+id/note"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="@android:color/transparent"
        android:padding="5dp"
        android:scrollbars="vertical"
        android:fadingEdge="vertical"
        android:gravity="top"
        android:textSize="22sp"
        android:capitalize="sentences"
    />
```

外部类可用下面方式使用:

```xml
    <com.example.android.notepad.LinedEditText
      id="@+id/note"
      ... />
```

