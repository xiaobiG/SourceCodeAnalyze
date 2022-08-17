#### 事件分发机制

- 一个 MotionEvent 产生后，按 Activity -> Window -> decorView -> View 顺序传递，View 传递过程就是事件分发，主要依赖三个方法:
- dispatchTouchEvent：用于分发事件，只要接受到点击事件就会被调用，返回结果表示是否消耗了当前事件
- onInterceptTouchEvent：用于判断是否拦截事件，当 ViewGroup 确定要拦截事件后，该事件序列都不会再触发调用此 ViewGroup 的 onIntercept
- onTouchEvent：用于处理事件，返回结果表示是否处理了当前事件，未处理则传递给父容器处理
- 细节： 一个事件序列只能被一个 View 拦截且消耗 View 没有 onIntercept 方法，直接调用 onTouchEvent 处理 OnTouchListener 优先级比 OnTouchEvent 高，onClickListener 优先级最低 requestDisallowInterceptTouchEvent 可以屏蔽父容器 onIntercet 方法的调用