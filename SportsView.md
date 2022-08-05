```java
 // 垂直居中绘制字体
        canvas.drawText((int) mProgress + "%", centerX, centerY - (mPaint.ascent() + mPaint.descent()) / 2, mPaint);
```

ObjectAnimator

```java
  public float getProgress() {
        return mProgress;
    }

    public void setProgress(float progress) {
        mProgress = progress;
        invalidate();
    }

```



SportsView

```java
public class SportsView extends View {

    private float mProgress = 0;
    private float mRadius = dpToPixel(80);
    private RectF mArcRectF = new RectF();
    private Paint mPaint = new Paint();

    public SportsView(Context context) {
        super(context);
    }

    {
        mPaint.setAntiAlias(true);
        mPaint.setTextSize(dpToPixel(40));
        mPaint.setTextAlign(Paint.Align.CENTER);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        float centerX = getWidth() / 2;
        float centerY = getHeight() / 2;

        mPaint.setColor(Color.parseColor("#E91E63"));
        mPaint.setStrokeCap(Paint.Cap.ROUND);
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setStrokeWidth(dpToPixel(15));
        mArcRectF.set(centerX - mRadius, centerY - mRadius, centerX + mRadius, centerY + mRadius);
        canvas.drawArc(mArcRectF, 135, mProgress * 2.7f, false, mPaint);
        mPaint.setColor(Color.WHITE);
        mPaint.setStyle(Paint.Style.FILL);
        // mPaint.setTextAlign(Paint.Align.CENTER);// 水平居中绘制
        // 垂直居中绘制字体
        canvas.drawText((int) mProgress + "%", centerX, centerY - (mPaint.ascent() + mPaint.descent()) / 2, mPaint);
    }

    public float getProgress() {
        return mProgress;
    }

    public void setProgress(float progress) {
        mProgress = progress;
        invalidate();
    }

    public static float dpToPixel(float dp) {
        DisplayMetrics metrics = Resources.getSystem().getDisplayMetrics();
        return dp * metrics.density;
    }

}

```

**使用**

```java
        button.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                ObjectAnimator animator = ObjectAnimator.ofFloat(sportsView, "progress", 0, 65);
                animator.setDuration(1000);
                animator.setInterpolator(new FastOutSlowInInterpolator());
                animator.start();
            }
        });
```