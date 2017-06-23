# android事件分发机制

android Touch事件分发与处理涉及到两个基础类View和ViewGroup，其中View是所有视图类的超类，ViewGroup是视图类的集合类，可以存放ViewGroup和View作为其孩子视图类，事件传递是由外往内传递

ViewGroup的相关事件有三个：onInterceptTouchEvent、dispatchTouchEvent、onTouchEvent。View的相关事件只有两个：dispatchTouchEvent、onTouchEvent。

结构模型概念：ViewGroup和View组成了一棵树形结构，最顶层为Activity的DecorView（本质是ViewGroup），下面有若干的ViewGroup节点，每个节点之下又有若干的ViewGroup节点或者View节点，依次类推。

![ViewGroup和View组成了一棵树形结构](/img/android_ontouch_1.jpg "ViewGroup和View组成了一棵树形结构")

当一个Touch事件(触摸事件为例)到达根节点，即Acitivty的ViewGroup时，它会依次下发，下发的过程是调用子View(ViewGroup)的dispatchTouchEvent方法实现的。简单来说，就是ViewGroup遍历它包含着的子View，调用每个View的dispatchTouchEvent方法，而当子View为ViewGroup时，又会通过调用ViwGroup的dispatchTouchEvent方法继续调用其内部的View的dispatchTouchEvent方法。上述例子中的消息下发顺序是这样的：①-②-⑤-⑥-⑦-③-④。dispatchTouchEvent方法只负责事件的分发，它拥有boolean类型的返回值，当返回为true时，顺序下发会中断。在上述例子中如果⑤的dispatchTouchEvent返回结果为true，那么⑥-⑦-③-④将都接收不到本次Touch事件。

源码分析流程：

当一个Touch事件(触摸事件为例)到达根节点，即Acitivty的DecorView时，它会执行dispatchTouchEvent方法

```
DecorView.java

@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```

执行Window.Callback接口在Activity类下实现的

```
Activity.java

public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

getWindow()返回Window类，[PhoneWindow](https://github.com/android/platform_frameworks_base/blob/c29fff50322599f53feadf9cf87df9956c9ac44e/core/java/com/android/internal/policy/PhoneWindow.java#LC1807)为Window实现类，它执行DecorView类下的superDispatchTouchEvent()方法

```
DecorView.java

public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

最终执行到ViewGroup类的[dispatchTouchEvent](https://github.com/android/platform_frameworks_base/blob/c29fff50322599f53feadf9cf87df9956c9ac44e/core/java/android/view/ViewGroup.java#L2143)方法

```
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    // 1. 用于测试目，直接忽略
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }
 
    boolean handled = false;
    // 2. 未被其他窗口遮盖  
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
 
        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            // 3. 清理触摸操作的所有痕迹，即派发取消操作
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }
 
        // 检测是否拦截Touch Event  
        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            // 是否允许拦截，可以通过requestDisallowInterceptTouchEvent方法设置
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                // 允许拦截
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }
 
        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;
 
        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        // 如果Touch Event没有取消并且没有被拦截，才会考虑是否向其子视图派发  
        if (!canceled && !intercepted) {
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;
 
                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);
 
                final int childrenCount = mChildrenCount;
                if (childrenCount != 0) {
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final View[] children = mChildren;
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
 
                    // 遍历当前ViewGroup的所有子视图
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final View child = children[i];                     
                        // 4 与5
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            // 满足以上条件，没必要接收Touch Event
                            continue;
                        }
 
                        // 如果是在子视图上触摸，经过以上过滤条件，只有当前手指正下方的子视图才会获取到此事件  
                        // 子视图是否在TouchTarget中  
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }
 
                        resetCancelNextUpFlag(child);
                        // 6. 执行触摸操作（会传递到最终视图的onTouchEvent方法）
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            mLastTouchDownIndex = i;
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // 会改变mFirstTouchTarget的值  
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                    }
                }
 
                // 没有发现可以接收事件的子视图
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }
 
        // 子视图未消耗Touch Event 或者 被拦截未向子视图派发Touch Event  
        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            // 由当前ViewGroup处理Touch Event  
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                    || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }
 
        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }
 
    // 1. 用于测试目，直接忽略
    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```
