# LoadingView

```kotlin
import android.animation.ObjectAnimator
import android.animation.ValueAnimator
import android.content.Context
import android.content.res.Resources
import android.graphics.Canvas
import android.graphics.Color
import android.graphics.Paint
import android.util.AttributeSet
import android.util.TypedValue
import android.view.View
import android.view.animation.LinearInterpolator
import kotlin.math.sin

class LoadingView @JvmOverloads constructor(
        context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    private val paint = genPaint()

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        setMeasuredDimension(resolveSize(200, widthMeasureSpec),
                resolveSize(200, heightMeasureSpec))
    }

    override fun onDraw(canvas: Canvas?) {
        paint.color = color
        paint.style = Paint.Style.STROKE
        paint.strokeWidth = strokeWidth
        paint.strokeCap = Paint.Cap.ROUND

        val startX = strokeWidth / 2f
        val gap = width / 4f
        drawLine(canvas, startX, core(progress - 4))
        drawLine(canvas, gap, core(progress - 2))
        drawLine(canvas, gap * 2, core(progress))
        drawLine(canvas, gap * 3, core(progress + 2))
        drawLine(canvas, gap * 4-startX, core(progress + 4))

    }

    private fun core(x: Float): Float {
        return 0.2f + ((sin(x) + 1f) / 2f * 0.7f)
    }

    private fun drawLine(canvas: Canvas?, x: Float, progress: Float) {
        val curLen = height * progress
        canvas?.drawLine(
                x, height / 2f - curLen / 2f,
                x, height / 2f + curLen / 2f,
                paint
        )
    }

    private var color = Color.parseColor("#1E90FF")
    private var strokeWidth = 3f.dp

    private var progress = 0f
        set(value) {
            field = value
            invalidate()
        }

    private val animate = ObjectAnimator.ofFloat(
            this, "progress", 0f, 2 * Math.PI.toFloat()
    ).apply {
        interpolator = LinearInterpolator()
        duration = 1000
        repeatCount = ValueAnimator.INFINITE
        repeatMode = ValueAnimator.RESTART
    }

    override fun onAttachedToWindow() {
        super.onAttachedToWindow()
        animate.start()
    }

    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        animate.cancel()
    }

}

fun genPaint(): Paint {
    return Paint(Paint.ANTI_ALIAS_FLAG xor Paint.DITHER_FLAG)
}

val Float.dp
    get() = TypedValue.applyDimension(
            TypedValue.COMPLEX_UNIT_DIP,
            this,
            Resources.getSystem().displayMetrics
    )
```