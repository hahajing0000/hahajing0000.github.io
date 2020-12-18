---
title: 事件分发
date: 2019-08-08
tags: [Android基础,事件]
categories: Android基础
toc: true
---

# 事件分发机制

## Android事件冒泡

> Activity -> PhoneWindow$DecorView -> ViewGroup ->View

​	Android的事件机制如果从头开始说是Android获取到用户触屏操作后利用socket进行跨进程通信，使用InputDispatcher将事件发送给APP进程使用android的消息机制将事件发送给主线程，由ActivityThread的Looper去取出消息进行处理。

<!--more-->



## 从Activity  ViewGroup View三个角度来分析事件分发机制



### Activity事件分发

Activity事件分发的起始位置在dispatchTouchEvent

```java
/**
     * Called to process touch screen events.  You can override this to
     * intercept all touch screen events before they are dispatched to the
     * window.  Be sure to call this implementation for touch screen events
     * that should be handled normally.
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            //系统空实现 一般是处理屏保
            onUserInteraction();
        }
        //后续事件分发处理流程
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        //如果superDispatchTouchEvent返回false 那么当前事件就由activity自己消费
        return onTouchEvent(ev);
    }

```

有Android View绘制原理中我们得知getWindow其实就是PhoneWindow的实例。

superDispatchTouchEvent方法其实就是PhoneWindow下的对应方法，接下来我们看一下该方法

PhoneWindow.java

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```
mDecor即DecorView，DecorView是PhoneWindow的内部类，进而我们来看一下DecorView的superDispatchTouchEvent方法

PhoneWindow$DecorView

```java
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```
> private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {}
>
> public class FrameLayout extends ViewGroup {}

我们发现DecorView中的该方法调用了super中的dispatchTouchEvent，DecorView的super是FrameLayout，也就是调用了Framelayout的dispatchTouchEvent方法，然后改方法具体实现其实是在FrameLayout的父类也就是ViewGroup中。

### ViewGroup事件分发

ViewGroup.java



```java
 @Override
 public boolean dispatchTouchEvent(MotionEvent ev) {
       ...

        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            //获取FLAG_DISALLOW_INTERCEPT标记为的值，该标记为可用由requestDisallowInterceptTouchEvent来设置 一般用于事件冲突内部拦截法
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                //触发onInterceptTouchEvent拦截方法
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


        ...
        if (!canceled && !intercepted) {
		...
                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    //处理如果子控件有叠加的情况，处理用户点击该由那个子控件来响应，如果设置了elevation translationZ 就由这两个属性来控制，如果没设置就按xml里面谁后加载谁来处理
                    final ArrayList<View> preorderedList = buildOrderedChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = customOrder
                                ? getChildDrawingOrder(childrenCount, i) : i;
                        final View child = (preorderedList == null)
                                ? children[childIndex] : preorderedList.get(childIndex);

                        // If there is a view that has accessibility focus we want it
                        // to get the event first and if not handled we will perform a
                        // normal dispatch. We may do a double iteration but this is
                        // safer given the timeframe.
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }

                        //isTransformedTouchPointInView 来处理用户点击位置是否在当前子控件范围内
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        ...
                        //分发事件给子控件
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                    ...
                }

   				...
            }
        }
		...
    return handled;
    }
```

dispatchTransformedTouchEvent方法

```java
/**
     * Transforms a motion event into the coordinate space of a particular child view,
          * filters out irrelevant pointer ids, and overrides its action if necessary.
          * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
*/
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;    
	// Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            //子控件为空就交由父控件处理
            handled = super.dispatchTouchEvent(event);
        } else {
            //将当前事件分发给子控件
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
```
### View的事件分发

View.java

```java
   /**

   		* Pass the touch screen motion event down to the target view, or this
      	view if it is the target.
        *
        * @param event The motion event to be dispatched.
        @return True if the event was handled by the view, false otherwise.
   */
public boolean dispatchTouchEvent(MotionEvent event) {
  ...


    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        //判断如果View的onTouchListener被设置过并且View enable等于True 并且 onTouch方法返回true该事件将被消费
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            
            result = true;
        }
		
        //我们没有设置如上的onTouchLister所有将执行onTouchEvent方法
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

...
    return result;
    }
```

下面来看看onTouchEvent方法

```java
/**
     * Implement this method to handle touch screen motion events.
     * <p>
     * If this method is used to detect click actions, it is recommended that
     * the actions be performed by implementing and calling
     * {@link #performClick()}. This will ensure consistent system behavior,
     * including:
     * <ul>
     * <li>obeying click sound preferences
     * <li>dispatching OnClickListener calls
     * <li>handling {@link AccessibilityNodeInfo#ACTION_CLICK ACTION_CLICK} when
     * accessibility features are enabled
     * </ul>
     *
     * @param event The motion event.
     * @return True if the event was handled, false otherwise.
     */
    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

       ...

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        ....
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        ...
```

进入该方法返回true即代表事件被消费

下面看一下performClick方法

```java
    /**
     * Call this view's OnClickListener, if it is defined.  Performs all normal
     * actions associated with clicking: reporting accessibility event, playing
     * a sound, etc.
     *
     * @return True there was an assigned OnClickListener that was called, false
     *         otherwise is returned.
     */
    public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }
```

如上代码处理了如果view被注册了onClickListener监听，该onClick方法将被执行。

## 流程图

### Activity事件分发流程

![Activity](D:\zyblog\ZYBlog\source\_posts\Event\Activity.png)

<font color="red">该图来这互联网，如果侵权请告之删除</font>

### ViewGroup事件分发

![viewgroup](D:\zyblog\ZYBlog\source\_posts\Event\viewgroup.png)

<font color="red">该图来这互联网，如果侵权请告之删除</font>

### View事件分发

![View](D:\zyblog\ZYBlog\source\_posts\Event\View.png)

<font color="red">该图来这互联网，如果侵权请告之删除</font>

### 事件分发总流程

![20190823194222623](D:\zyblog\ZYBlog\source\_posts\Event\20190823194222623.png)

<font color="red">该图来这互联网，如果侵权请告之删除</font>