# 获取宽高的时机

```java
final Button btnTest = findViewById(R.id.btn_test);
Log.d(TAG, "width: " + btnTest.getWidth() + " height: " + btnTest.getHeight());
btnTest.post(new Runnable() {
	@Override
	public void run() {
		Log.d(TAG, "on post width: " + btnTest.getWidth() + " height: " + btnTest.getHeight());
	}
});
```

> 2020-04-27 11:22:44.916 4347-4347/com.ganxiao.glidetest D/xxxxxxxxxxxxxxxxx: width: 0 height: 0
> 2020-04-27 11:22:44.960 4347-4347/com.ganxiao.glidetest D/xxxxxxxxxxxxxxxxx: on post width: 132 height: 72

# 将Image对象保存为File

```java
    private static class ImageSaver implements Runnable {

        private final Image mImage;

        private final File mFile;

        ImageSaver(Image image, File file) {
            mImage = image;
            mFile = file;
        }

        @Override
        public void run() {
            ByteBuffer buffer = mImage.getPlanes()[0].getBuffer();
            byte[] bytes = new byte[buffer.remaining()];
            buffer.get(bytes);
            FileOutputStream output = null;
            try {
                output = new FileOutputStream(mFile);
                output.write(bytes);
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                mImage.close();
                if (null != output) {
                    try {
                        output.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

    }
```

# 将View装换为Bitmap

If you are using [Android-KTX](https://developer.android.com/kotlin/ktx), use [toBitmap](https://developer.android.com/reference/kotlin/androidx/core/view/package-summary#tobitmap).

```kotlin
val bitmap = view.toBitmap()
```

or without `Android-KTX`.

```kotlin
view.isDrawingCacheEnabled = true
val bitmap = view.drawingCache
view.isDrawingCacheEnabled = false
```

If you happens to change Layout Width/Height programatically before converting the view to bitmap.

```kotlin
view.updateLayoutParams {
    this.width = newWidth
}

view.doOnLayout {
    val bitmap = view.toBitmap()
}
```

# view 保存为图片

```
view.setDrawingCacheEnabled(true);
Bitmap b = view.getDrawingCache();
b.compress(CompressFormat.JPEG, 95, new FileOutputStream("/some/location/image.jpg"));
```

# canvas.save

```java
						int saveCount = canvas.save(Canvas.MATRIX_SAVE_FLAG);
                        try {
                            canvas.translate(0, -yoff);
                            if (mTranslator != null) {
                                mTranslator.translateCanvas(canvas);
                            }
                            canvas.setScreenDensity(scalingRequired
                                    ? DisplayMetrics.DENSITY_DEVICE : 0);
                            mView.draw(canvas);
                            if (Config.DEBUG && ViewDebug.consistencyCheckEnabled) {
                                mView.dispatchConsistencyCheck(ViewDebug.CONSISTENCY_DRAWING);
                            }
                        } finally {
                            canvas.restoreToCount(saveCount);
                        }
```

# FrameLayout中的测量方法

1. 计算了子控件和前景的最大宽度和高度；
2. padding；
3. visiblity 为 Gone 的 view 不会被计算；
4. 通过 measureChildWithMargins 方法计算 margin；
5. 静态方法 resolveSize；

```java
public class FrameLayout extends ViewGroup {
    
       @Override
       protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
           final int count = getChildCount();
   
           int maxHeight = 0;
           int maxWidth = 0;
   
           // Find rightmost and bottommost child
           for (int i = 0; i < count; i++) {
               final View child = getChildAt(i);
               if (mMeasureAllChildren || child.getVisibility() != GONE) {
                   measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                   maxWidth = Math.max(maxWidth, child.getMeasuredWidth());
                   maxHeight = Math.max(maxHeight, child.getMeasuredHeight());
               }
           }
   
           // Account for padding too
           maxWidth += mPaddingLeft + mPaddingRight + mForegroundPaddingLeft + mForegroundPaddingRight;
           maxHeight += mPaddingTop + mPaddingBottom + mForegroundPaddingTop + mForegroundPaddingBottom;
   
           // Check against our minimum height and width
           maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
           maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
   
           // Check against our foreground's minimum height and width
           final Drawable drawable = getForeground();
           if (drawable != null) {
               maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
               maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
           }
   
           setMeasuredDimension(resolveSize(maxWidth, widthMeasureSpec),
                   resolveSize(maxHeight, heightMeasureSpec));
       }
       
      public static int resolveSize(int size, int measureSpec) {
           int result = size;
           int specMode = MeasureSpec.getMode(measureSpec);
           int specSize =  MeasureSpec.getSize(measureSpec);
           switch (specMode) {
           case MeasureSpec.UNSPECIFIED:
               result = size;
               break;
           case MeasureSpec.AT_MOST:
               result = Math.min(size, specSize);
               break;
           case MeasureSpec.EXACTLY:
               result = specSize;
               break;
           }
           return result;
       }
}
```

# FrameLayout的onLayout方法

1. ViewGroup在onLayout()中会调用自己的所有子View的layout()方法，把它们的尺寸和位置传给它们，让它们完成自我的内部布局。
2. layout()方法用来确定View本身的位置，onLayout()方法用来确定子元素的位置。

```java
public class FrameLayout extends ViewGroup {
    
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
            final int count = getChildCount();
    
            final int parentLeft = mPaddingLeft + mForegroundPaddingLeft;
            final int parentRight = right - left - mPaddingRight - mForegroundPaddingRight;
    
            final int parentTop = mPaddingTop + mForegroundPaddingTop;
            final int parentBottom = bottom - top - mPaddingBottom - mForegroundPaddingBottom;
    
            mForegroundBoundsChanged = true;
            
            for (int i = 0; i < count; i++) {
                final View child = getChildAt(i);
                if (child.getVisibility() != GONE) {
                    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    
                    final int width = child.getMeasuredWidth();
                    final int height = child.getMeasuredHeight();
    
                    int childLeft = parentLeft;
                    int childTop = parentTop;
    
                    final int gravity = lp.gravity;
    
                    if (gravity != -1) {
                        final int horizontalGravity = gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
                        final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
    
                        switch (horizontalGravity) {
                            case Gravity.LEFT:
                                childLeft = parentLeft + lp.leftMargin;
                                break;
                            case Gravity.CENTER_HORIZONTAL:
                                childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                                        lp.leftMargin - lp.rightMargin;
                                break;
                            case Gravity.RIGHT:
                                childLeft = parentRight - width - lp.rightMargin;
                                break;
                            default:
                                childLeft = parentLeft + lp.leftMargin;
                        }
    
                        switch (verticalGravity) {
                            case Gravity.TOP:
                                childTop = parentTop + lp.topMargin;
                                break;
                            case Gravity.CENTER_VERTICAL:
                                childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                                        lp.topMargin - lp.bottomMargin;
                                break;
                            case Gravity.BOTTOM:
                                childTop = parentBottom - height - lp.bottomMargin;
                                break;
                            default:
                                childTop = parentTop + lp.topMargin;
                        }
                    }
    
                    child.layout(childLeft, childTop, childLeft + width, childTop + height);
                }
            }
        }
}
```

