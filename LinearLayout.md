# 平均宽度

```java
public class AverageWidthLayout extends LinearLayout {

    public static final String TAG = AverageWidthLayout.class.getSimpleName();

    public AverageWidthLayout(Context context) {
        super(context);
        init();
    }

    public AverageWidthLayout(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public AverageWidthLayout(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        setOrientation(HORIZONTAL);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int width = MeasureSpec.getSize(widthMeasureSpec);

        int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            ViewGroup.LayoutParams layoutParams = child.getLayoutParams();
            layoutParams.width = width / childCount;
        }
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }

}
```

# 平均高度

```java
public class AverageHeightLayout extends LinearLayout {

    public AverageHeightLayout(Context context) {
        super(context);
        init();
    }

    public AverageHeightLayout(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public AverageHeightLayout(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        setOrientation(VERTICAL);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int height = MeasureSpec.getSize(heightMeasureSpec);

        int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            ViewGroup.LayoutParams layoutParams = child.getLayoutParams();
            layoutParams.height = height / childCount;
        }
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }
}
```

