---
title: 知识总结之 Material Design库常用控件总结
date: 2017-06-29 20:55:12
categories: android 
tags: Material Design
---

Google从安卓5.0开始提出了Meterial Design设计，各种被模仿学习，开始有超越IOS的劲头。同时，谷歌官方还为安卓开发者提供了design库，可以很容易实现meterial设计华丽丽效果，公司项目很少能够直接用这些控件的，这几天业余时间学习下用在自己小项目中，这里整理一篇这些控件细节，为日后参考。

![Material Design](http://wx4.sinaimg.cn/mw690/70e618a5ly1fh1whznzjnj20w80hcq3s.jpg)

<!--more-->

# Material Design库常用控件使用总结

## 一、CoordinatorLayout

CoordinatorLayout的主要功能一句话概括：作为顶层GroupView来调度或协调子布局。

官方解释：
> CoordinatorLayout is a super-powered FrameLayout.
CoordinatorLayout is intended for two primary use cases:
As a top-level application decor or chrome layout
As a container for a specific interaction with one or more child views


主要搭配子布局有，AppBarLayout等App顶部位置的View，及Content位置的view，还有一些装饰性引导控件如FloatingAcitionButton， 可产生各种炫酷的效果。

CoordinatorLayout的强大主要原因是**Behavior**类，CoordinatorLayout自己并不控制View，所有的控制权都在Behavior。

```
public class CoordinatorLayout extends ViewGroup implements NestedScrollingParent
```

看代码发现，CoordinatorLayout其实就是个自定义的ViewGroup，实现了一个**NestedScrollingParent**接口。NestedScrollingParent接口主要是解决联合滑动问题，也就是解决android系统事件传递中事件被一个控件开始处理，这个事件的后续事件将不会被其他控件收到的缺陷。
这里引用[网络上](http://mobile.51cto.com/android-524405.htm)的一张安卓事件传递图，个人认为比较齐全：

![事件传递](dispatch.png)

### Behavior深入了解
控件处了自己可以默认有自己的Behavior，还可以在xml中动态引用。

```
app:layout_behavior="xxx.Behavior"
```

查代码看看CoordinatorLayout 什么时候用到了behavior。

```
          public static class LayoutParams extends ViewGroup.MarginLayoutParams {
       		 /**
       	      * A {@link Behavior} that the child view should obey.
        	 */
        	Behavior mBehavior;

        	boolean mBehaviorResolved = false;
        	......
        }
        
		private void prepareChildren() {
        	mDependencySortedChildren.clear();
       		 mChildDag.clear();

       		 for (int i = 0, count = getChildCount(); i < count; i++) {
            final View view = getChildAt(i);

            final LayoutParams lp = getResolvedLayoutParams(view);
            lp.findAnchorView(this, view);

            mChildDag.addNode(view);

            // Now iterate again over the other children, adding any dependencies to the graph
            for (int j = 0; j < count; j++) {
                if (j == i) {
                    continue;
                }
                final View other = getChildAt(j);
                final LayoutParams otherLp = getResolvedLayoutParams(other);
                if (otherLp.dependsOn(this, other, view)) {
                    if (!mChildDag.contains(other)) {
                        // Make sure that the other node is added
                        mChildDag.addNode(other);
                    }
                    // Now add the dependency to the graph
                    mChildDag.addEdge(view, other);
                }
            }
        }
        
        LayoutParams getResolvedLayoutParams(View child) {
          final LayoutParams result = (LayoutParams) child.getLayoutParams();
          if (!result.mBehaviorResolved) {
            Class<?> childClass = child.getClass();
            DefaultBehavior defaultBehavior = null;
            while (childClass != null &&
                    (defaultBehavior = childClass.getAnnotation(DefaultBehavior.class)) == null) {
                childClass = childClass.getSuperclass();
            }
            if (defaultBehavior != null) {
                try {
                    result.setBehavior(defaultBehavior.value().newInstance());
                } catch (Exception e) {
                    Log.e(TAG, "Default behavior class " + defaultBehavior.value().getName() +
                            " could not be instantiated. Did you forget a default constructor?", e);
                }
            }
            result.mBehaviorResolved = true;
        }
        return result;
    }
```

Behavior是CoordinatorLayout.LayoutParams的成员变量，那么也就是说只有当你的Behavior设置给CoordinatorLayout的直接子View才会有效果

依赖关系在*dependsOn（）*方法中建立联系。

```
 /**
         * Check if an associated child view depends on another child view of the CoordinatorLayout.
         *
         * @param parent the parent CoordinatorLayout
         * @param child the child to check
         * @param dependency the proposed dependency to check
         * @return true if child depends on dependency
         */
        boolean dependsOn(CoordinatorLayout parent, View child, View dependency) {
            return dependency == mAnchorDirectChild
                    || shouldDodge(dependency, ViewCompat.getLayoutDirection(parent))
                    || (mBehavior != null && mBehavior.layoutDependsOn(parent, child, dependency));
        }
```

CoordinatorLayout与各个子View建立关系依赖链后，怎么实施控制的呢？ 下面具体分析下Behavior起了什么作用。看下部分源码如下：

```
    /**
     * Interaction behavior plugin for child views of {@link CoordinatorLayout}.
     *
     * <p>A Behavior implements one or more interactions that a user can take on a child view.
     * These interactions may include drags, swipes, flings, or any other gestures.</p>
     *
     * @param <V> The View type that this Behavior operates on
     */
    public static abstract class Behavior<V extends View> {


        /**
         * Called when the Behavior has been attached to a LayoutParams instance.
         *
         * <p>This will be called after the LayoutParams has been instantiated and can be
         * modified.</p>
         *
         * @param params the LayoutParams instance that this Behavior has been attached to
         */
        public void onAttachedToLayoutParams(@NonNull CoordinatorLayout.LayoutParams params) {
        }

        /**
         * Called when the Behavior has been detached from its holding LayoutParams instance.
         *
         * <p>This will only be called if the Behavior has been explicitly removed from the
         * LayoutParams instance via {@link LayoutParams#setBehavior(Behavior)}. It will not be
         * called if the associated view is removed from the CoordinatorLayout or similar.</p>
         */
        public void onDetachedFromLayoutParams() {
        }

        /**
         * Respond to CoordinatorLayout touch events before they are dispatched to child views.
         *
         * <p>Behaviors can use this to monitor inbound touch events until one decides to
         * intercept the rest of the event stream to take an action on its associated child view.
         * This method will return false until it detects the proper intercept conditions, then
         * return true once those conditions have occurred.</p>
         *
         * <p>Once a Behavior intercepts touch events, the rest of the event stream will
         * be sent to the {@link #onTouchEvent} method.</p>
         *
         * <p>The default implementation of this method always returns false.</p>
         *
         * @param parent the parent view currently receiving this touch event
         * @param child the child view associated with this Behavior
         * @param ev the MotionEvent describing the touch event being processed
         * @return true if this Behavior would like to intercept and take over the event stream.
         *         The default always returns false.
         */
        public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
            return false;
        }

        /**
         * Respond to CoordinatorLayout touch events after this Behavior has started
         * {@link #onInterceptTouchEvent intercepting} them.
         *
         * <p>Behaviors may intercept touch events in order to help the CoordinatorLayout
         * manipulate its child views. For example, a Behavior may allow a user to drag a
         * UI pane open or closed. This method should perform actual mutations of view
         * layout state.</p>
         *
         * @param parent the parent view currently receiving this touch event
         * @param child the child view associated with this Behavior
         * @param ev the MotionEvent describing the touch event being processed
         * @return true if this Behavior handled this touch event and would like to continue
         *         receiving events in this stream. The default always returns false.
         */
        public boolean onTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
            return false;
        }

        /**
         * Supply a scrim color that will be painted behind the associated child view.
         *
         * <p>A scrim may be used to indicate that the other elements beneath it are not currently
         * interactive or actionable, drawing user focus and attention to the views above the scrim.
         * </p>
         *
         * <p>The default implementation returns {@link Color#BLACK}.</p>
         *
         * @param parent the parent view of the given child
         * @param child the child view above the scrim
         * @return the desired scrim color in 0xAARRGGBB format. The default return value is
         *         {@link Color#BLACK}.
         * @see #getScrimOpacity(CoordinatorLayout, android.view.View)
         */
        @ColorInt
        public int getScrimColor(CoordinatorLayout parent, V child) {
            return Color.BLACK;
        }

        /**
         * Determine the current opacity of the scrim behind a given child view
         *
         * <p>A scrim may be used to indicate that the other elements beneath it are not currently
         * interactive or actionable, drawing user focus and attention to the views above the scrim.
         * </p>
         *
         * <p>The default implementation returns 0.0f.</p>
         *
         * @param parent the parent view of the given child
         * @param child the child view above the scrim
         * @return the desired scrim opacity from 0.0f to 1.0f. The default return value is 0.0f.
         */
        @FloatRange(from = 0, to = 1)
        public float getScrimOpacity(CoordinatorLayout parent, V child) {
            return 0.f;
        }

      
        public boolean blocksInteractionBelow(CoordinatorLayout parent, V child) {
            return getScrimOpacity(parent, child) > 0.f;
        }

        /**
         * 一定条件下返回true即可确立依赖关系。子类重写该方法，并实现自己依赖逻辑。
         * 该方法会被Behavior的LayoutParamas的dependsOn方法调用
         */
        public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) {
            return false;
        }

        /**
        *当我们的dependency发生改变的时候，这个方法会调用，而我们在onDependentViewChanged方法里做出相应的改变，就能做出我们想要的交互效果了
        *当我们改变了child的size或者position的时候我们需要返回true，差不多可以理解为 当我们的dependency发生了改变，同样的，child也需要发生改变，这个时候我们需要返回true。
        * 它会被dispatchOnDependentViewChanged方法调用
          */
        public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {
            return false;
        }

        /**
         */
        public void onDependentViewRemoved(CoordinatorLayout parent, V child, View dependency) {
        }


        /**
         * Called when the parent CoordinatorLayout is about to measure the given child view.
         *
         * <p>This method can be used to perform custom or modified measurement of a child view
         * in place of the default child measurement behavior. The Behavior's implementation
         * can delegate to the standard CoordinatorLayout measurement behavior by calling
         * {@link CoordinatorLayout#onMeasureChild(android.view.View, int, int, int, int)
         * parent.onMeasureChild}.</p>
         *
         * @param parent the parent CoordinatorLayout
         * @param child the child to measure
         * @param parentWidthMeasureSpec the width requirements for this view
         * @param widthUsed extra space that has been used up by the parent
         *        horizontally (possibly by other children of the parent)
         * @param parentHeightMeasureSpec the height requirements for this view
         * @param heightUsed extra space that has been used up by the parent
         *        vertically (possibly by other children of the parent)
         * @return true if the Behavior measured the child view, false if the CoordinatorLayout
         *         should perform its default measurement
         */
        public boolean onMeasureChild(CoordinatorLayout parent, V child,
                int parentWidthMeasureSpec, int widthUsed,
                int parentHeightMeasureSpec, int heightUsed) {
            return false;
        }

        /**
         * Called when the parent CoordinatorLayout is about the lay out the given child view.
         *
         * <p>This method can be used to perform custom or modified layout of a child view
         * in place of the default child layout behavior. The Behavior's implementation can
         * delegate to the standard CoordinatorLayout measurement behavior by calling
         * {@link CoordinatorLayout#onLayoutChild(android.view.View, int)
         * parent.onLayoutChild}.</p>
         *
         * <p>If a Behavior implements
         * {@link #onDependentViewChanged(CoordinatorLayout, android.view.View, android.view.View)}
         * to change the position of a view in response to a dependent view changing, it
         * should also implement <code>onLayoutChild</code> in such a way that respects those
         * dependent views. <code>onLayoutChild</code> will always be called for a dependent view
         * <em>after</em> its dependency has been laid out.</p>
         *
         * @param parent the parent CoordinatorLayout
         * @param child child view to lay out
         * @param layoutDirection the resolved layout direction for the CoordinatorLayout, such as
         *                        {@link ViewCompat#LAYOUT_DIRECTION_LTR} or
         *                        {@link ViewCompat#LAYOUT_DIRECTION_RTL}.
         * @return true if the Behavior performed layout of the child view, false to request
         *         default layout behavior
         */
        public boolean onLayoutChild(CoordinatorLayout parent, V child, int layoutDirection) {
            return false;
        }

        // Utility methods for accessing child-specific, behavior-modifiable properties.


        /**
         * Called when a descendant of the CoordinatorLayout attempts to initiate a nested scroll.
         *
         * <p>Any Behavior associated with any direct child of the CoordinatorLayout may respond
         * to this event and return true to indicate that the CoordinatorLayout should act as
         * a nested scrolling parent for this scroll. Only Behaviors that return true from
         * this method will receive subsequent nested scroll events.</p>
         *
         * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
         *                          associated with
         * @param child the child view of the CoordinatorLayout this Behavior is associated with
         * @param directTargetChild the child view of the CoordinatorLayout that either is or
         *                          contains the target of the nested scroll operation
         * @param target the descendant view of the CoordinatorLayout initiating the nested scroll
         * @param nestedScrollAxes the axes that this nested scroll applies to. See
         *                         {@link ViewCompat#SCROLL_AXIS_HORIZONTAL},
         *                         {@link ViewCompat#SCROLL_AXIS_VERTICAL}
         * @return true if the Behavior wishes to accept this nested scroll
         *
         * @see NestedScrollingParent#onStartNestedScroll(View, View, int)
         *联合滑动的方法监听
         */
        public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout,
                V child, View directTargetChild, View target, int nestedScrollAxes) {
            return false;
        }

        /**
         * Called when a nested scroll has been accepted by the CoordinatorLayout.
         *
         * <p>Any Behavior associated with any direct child of the CoordinatorLayout may elect
         * to accept the nested scroll as part of {@link #onStartNestedScroll}. Each Behavior
         * that returned true will receive subsequent nested scroll events for that nested scroll.
         * </p>
         *
         * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
         *                          associated with
         * @param child the child view of the CoordinatorLayout this Behavior is associated with
         * @param directTargetChild the child view of the CoordinatorLayout that either is or
         *                          contains the target of the nested scroll operation
         * @param target the descendant view of the CoordinatorLayout initiating the nested scroll
         * @param nestedScrollAxes the axes that this nested scroll applies to. See
         *                         {@link ViewCompat#SCROLL_AXIS_HORIZONTAL},
         *                         {@link ViewCompat#SCROLL_AXIS_VERTICAL}
         *
         * @see NestedScrollingParent#onNestedScrollAccepted(View, View, int)
         */
        public void onNestedScrollAccepted(CoordinatorLayout coordinatorLayout, V child,
                View directTargetChild, View target, int nestedScrollAxes) {
            // Do nothing
        }

        /**
         * Called when a nested scroll has ended.
         *
         * <p>Any Behavior associated with any direct child of the CoordinatorLayout may elect
         * to accept the nested scroll as part of {@link #onStartNestedScroll}. Each Behavior
         * that returned true will receive subsequent nested scroll events for that nested scroll.
         * </p>
         *
         * <p><code>onStopNestedScroll</code> marks the end of a single nested scroll event
         * sequence. This is a good place to clean up any state related to the nested scroll.
         * </p>
         *
         * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
         *                          associated with
         * @param child the child view of the CoordinatorLayout this Behavior is associated with
         * @param target the descendant view of the CoordinatorLayout that initiated
         *               the nested scroll
         *
         * @see NestedScrollingParent#onStopNestedScroll(View)
         */
        public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target) {
            // Do nothing
        }

        /**
         * Called when a nested scroll in progress has updated and the target has scrolled or
         * attempted to scroll.
         *
         * <p>Any Behavior associated with the direct child of the CoordinatorLayout may elect
         * to accept the nested scroll as part of {@link #onStartNestedScroll}. Each Behavior
         * that returned true will receive subsequent nested scroll events for that nested scroll.
         * </p>
         *
         * <p><code>onNestedScroll</code> is called each time the nested scroll is updated by the
         * nested scrolling child, with both consumed and unconsumed components of the scroll
         * supplied in pixels. <em>Each Behavior responding to the nested scroll will receive the
         * same values.</em>
         * </p>
         *
         * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
         *                          associated with
         * @param child the child view of the CoordinatorLayout this Behavior is associated with
         * @param target the descendant view of the CoordinatorLayout performing the nested scroll
         * @param dxConsumed horizontal pixels consumed by the target's own scrolling operation
         * @param dyConsumed vertical pixels consumed by the target's own scrolling operation
         * @param dxUnconsumed horizontal pixels not consumed by the target's own scrolling
         *                     operation, but requested by the user
         * @param dyUnconsumed vertical pixels not consumed by the target's own scrolling operation,
         *                     but requested by the user
         *
         * @see NestedScrollingParent#onNestedScroll(View, int, int, int, int)
         */
        public void onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target,
                int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
            // Do nothing
        }

        /**
         * Called when a nested scroll in progress is about to update, before the target has
         * consumed any of the scrolled distance.
         *
         * <p>Any Behavior associated with the direct child of the CoordinatorLayout may elect
         * to accept the nested scroll as part of {@link #onStartNestedScroll}. Each Behavior
         * that returned true will receive subsequent nested scroll events for that nested scroll.
         * </p>
         *
         * <p><code>onNestedPreScroll</code> is called each time the nested scroll is updated
         * by the nested scrolling child, before the nested scrolling child has consumed the scroll
         * distance itself. <em>Each Behavior responding to the nested scroll will receive the
         * same values.</em> The CoordinatorLayout will report as consumed the maximum number
         * of pixels in either direction that any Behavior responding to the nested scroll reported
         * as consumed.</p>
         *
         * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
         *                          associated with
         * @param child the child view of the CoordinatorLayout this Behavior is associated with
         * @param target the descendant view of the CoordinatorLayout performing the nested scroll
         * @param dx the raw horizontal number of pixels that the user attempted to scroll
         * @param dy the raw vertical number of pixels that the user attempted to scroll
         * @param consumed out parameter. consumed[0] should be set to the distance of dx that
         *                 was consumed, consumed[1] should be set to the distance of dy that
         *                 was consumed
         *
         * @see NestedScrollingParent#onNestedPreScroll(View, int, int, int[])
         */
        public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target,
                int dx, int dy, int[] consumed) {
            // Do nothing
        }

        /**
         * Called when a nested scrolling child is starting a fling or an action that would
         * be a fling.
         *
         * <p>Any Behavior associated with the direct child of the CoordinatorLayout may elect
         * to accept the nested scroll as part of {@link #onStartNestedScroll}. Each Behavior
         * that returned true will receive subsequent nested scroll events for that nested scroll.
         * </p>
         *
         * <p><code>onNestedFling</code> is called when the current nested scrolling child view
         * detects the proper conditions for a fling. It reports if the child itself consumed
         * the fling. If it did not, the child is expected to show some sort of overscroll
         * indication. This method should return true if it consumes the fling, so that a child
         * that did not itself take an action in response can choose not to show an overfling
         * indication.</p>
         *
         * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
         *                          associated with
         * @param child the child view of the CoordinatorLayout this Behavior is associated with
         * @param target the descendant view of the CoordinatorLayout performing the nested scroll
         * @param velocityX horizontal velocity of the attempted fling
         * @param velocityY vertical velocity of the attempted fling
         * @param consumed true if the nested child view consumed the fling
         * @return true if the Behavior consumed the fling
         *
         * @see NestedScrollingParent#onNestedFling(View, float, float, boolean)
         */
        public boolean onNestedFling(CoordinatorLayout coordinatorLayout, V child, View target,
                float velocityX, float velocityY, boolean consumed) {
            return false;
        }

        /**
         * Called when a nested scrolling child is about to start a fling.
         *
         * <p>Any Behavior associated with the direct child of the CoordinatorLayout may elect
         * to accept the nested scroll as part of {@link #onStartNestedScroll}. Each Behavior
         * that returned true will receive subsequent nested scroll events for that nested scroll.
         * </p>
         *
         * <p><code>onNestedPreFling</code> is called when the current nested scrolling child view
         * detects the proper conditions for a fling, but it has not acted on it yet. A
         * Behavior can return true to indicate that it consumed the fling. If at least one
         * Behavior returns true, the fling should not be acted upon by the child.</p>
         *
         * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
         *                          associated with
         * @param child the child view of the CoordinatorLayout this Behavior is associated with
         * @param target the descendant view of the CoordinatorLayout performing the nested scroll
         * @param velocityX horizontal velocity of the attempted fling
         * @param velocityY vertical velocity of the attempted fling
         * @return true if the Behavior consumed the fling
         *
         * @see NestedScrollingParent#onNestedPreFling(View, float, float)
         */
        public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, V child, View target,
                float velocityX, float velocityY) {
            return false;
        }

        /**
         * Called when the window insets have changed.
         *
         * <p>Any Behavior associated with the direct child of the CoordinatorLayout may elect
         * to handle the window inset change on behalf of it's associated view.
         * </p>
         *
         * @param coordinatorLayout the CoordinatorLayout parent of the view this Behavior is
         *                          associated with
         * @param child the child view of the CoordinatorLayout this Behavior is associated with
         * @param insets the new window insets.
         *
         * @return The insets supplied, minus any insets that were consumed
         */
        @NonNull
        public WindowInsetsCompat onApplyWindowInsets(CoordinatorLayout coordinatorLayout,
                V child, WindowInsetsCompat insets) {
            return insets;
        }
    }
```

### 为什么Beavior可以解决之前安卓事件传递缺陷？

普通版本的ViewGroup一直存在的事件传递缺陷，那继承了ViewGroup的新控件CoordinatorLayout怎么解决该问题的，主要还是把相关事物优先交给了Beavior处理。

View的绘制三大步骤：Measure，Layout，Draw。那么我们分别看下重写的父类方法*onMeasure*, *onLayout*, *onDraw*.及事件传递相关方法*onInterceptTouchEvent*， *onTouchEvent*.

```
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
       boolean handled = false;
       boolean cancelSuper = false;
       MotionEvent cancelEvent = null;
       final int action = MotionEventCompat.getActionMasked(ev);
       // mBehaviorTouchView不为null（代表之前有behavior处理了down事件） 或者 performIntercept返回true 那么事件就交给mBehaviorTouchView
       if (mBehaviorTouchView != null || (cancelSuper = performIntercept(ev, TYPE_ON_TOUCH))) {
        // Safe since performIntercept guarantees that
        // mBehaviorTouchView != null if it returns true
        final LayoutParams lp = (LayoutParams) mBehaviorTouchView.getLayoutParams();
        final Behavior b = lp.getBehavior();
        if (b != null) {
            // 交给 behavior去处理事件  
            handled = b.onTouchEvent(this, mBehaviorTouchView, ev);
        }
     }
    // Keep the super implementation correct
    // 省略调用默认实现 up&cancel的时候重置状态
    ......
    return handled;
}
```
可以看到，其实这两个方法做的事情并不多，其实都交给performIntercept方法去做处理了！

```
    // type 标记是intercept还是touch
    private boolean performIntercept(MotionEvent ev, final int type) {
     final int action = MotionEventCompat.getActionMasked(ev);
     final List<View> topmostChildList = mTempList1;
    //按Z轴排序 原因很简单 让最上面的View先处理事件  
     getTopSortedChildren(topmostChildList);
    // Let topmost child views inspect first 
     final int childCount = topmostChildList.size();
     for (int i = 0; i < childCount; i++) {
        final View child = topmostChildList.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        final Behavior b = lp.getBehavior();
        //当前事件已经被某个behavior拦截了（or newBlock），并且事件不为down，那么就发送一个 取消事件 给所有在拦截的behavior之后的behavior
        if ((intercepted || newBlock) && action != MotionEvent.ACTION_DOWN) {
            // Cancel all behaviors beneath the one that intercepted.
            // If the event is "down" then we don't have anything to cancel yet.
            if (b != null) {
                if (cancelEvent == null) {
                    final long now = SystemClock.uptimeMillis();
                    cancelEvent = MotionEvent.obtain(now, now,
                            MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                }
                switch (type) {
                    case TYPE_ON_INTERCEPT:
                        b.onInterceptTouchEvent(this, child, cancelEvent);
                        break;
                    case TYPE_ON_TOUCH:
                        b.onTouchEvent(this, child, cancelEvent);
                        break;
                }
            }
            continue;
        }
        // 如果还没有被拦截，那么继续询问每个Behavior 是否要处理该事件
        if (!intercepted && b != null) {
            switch (type) {
                case TYPE_ON_INTERCEPT:
                    intercepted = b.onInterceptTouchEvent(this, child, ev);
                    break;
                case TYPE_ON_TOUCH:
                    intercepted = b.onTouchEvent(this, child, ev);
                    break;
            }
            //如果有behavior处理了当前的事件，那么把它赋值给mBehaviorTouchView,它其实跟ViewGroup源码中的mFirstTouchTarget作用是一样的
            if (intercepted) {
                mBehaviorTouchView = child;
            }
        }
        // Don't keep going if we're not allowing interaction below this.
        // Setting newBlock will make sure we cancel the rest of the behaviors.
        // 是否拦截一切在它之后的交互 好暴力-0-
        final boolean wasBlocking = lp.didBlockInteraction();
        final boolean isBlocking = lp.isBlockingInteractionBelow(this, child);
        newBlock = isBlocking && !wasBlocking;
        if (isBlocking && !newBlock) {
            // Stop here since we don't have anything more to cancel - we already did
            // when the behavior first started blocking things below this point.
            break;
        }
    }
    topmostChildList.clear();
    return intercepted;
   }
```
通过分析源码, 事件传递过程中，Behavior起到了一个高层次的拦截能力，可以更近每个子view自定义的实现逻辑，来统筹处理相关事件。

## 二、AppBarLayout
