```kotlin
import android.content.Context
import android.graphics.Canvas
import android.graphics.Path
import android.graphics.RectF
import android.util.AttributeSet
import android.widget.FrameLayout
import com.zhiboshow.phonelive.ext.dp

class BottomCornerLayout @JvmOverloads constructor(
        context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : FrameLayout(context, attrs, defStyleAttr) {

    private val path = Path()
    private val leftTop = 0f.dp
    private val rightTop = 0f.dp
    private val leftBottom = 4f.dp
    private val rightBottom = 4f.dp
    private val radius = floatArrayOf(
            leftTop, leftTop,
            rightTop, rightTop,
            rightBottom, rightBottom,
            leftBottom, leftBottom
    )
    private val rectF = RectF()

    override fun dispatchDraw(canvas: Canvas?) {
        rectF.set(0.toFloat(), 0.toFloat(), width.toFloat(), height.toFloat())
        path.addRoundRect(rectF, radius, Path.Direction.CCW)
        canvas?.clipPath(path)
        super.dispatchDraw(canvas)
    }
}
```

