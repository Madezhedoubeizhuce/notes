# 事件分发

## Activity组成

Android的事件分发是将touch事件从Activity往具体的view进行分发，所以在介绍具体的事件分发机制之前，需要先了解Activity的组成。

先来看一张图片：

![activity](.\imgs\activity.png)

从上图可以看出，Activity顶层是PhoneWindow，Window由DecorView组成，DecorView中一般是包含TitleView和ContentView，其实这里的ContentView就是我们Activity对应的布局文件。下面我们根据源码来分析一下Activity的构成。

在覆盖Activity的onCreate方法时都会调用Activity的setContentView()方法，所以先看看setContentView的实现：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

这里调用了getWindow()来获取Window对象，看下这个方法的实现：

```java
public Window getWindow() {
    return mWindow;
}
```

再查看那里创建mWindow对象，可以发现mWindow是一个PhoneWindow对象：

```java
 mWindow = new PhoneWindow(this);
```

所以，Activity的顶层是一个PhoneWindow。

在setContentView()方法中，又调用了PhoneWindow的setContentView()，接下来看一下这个方法：

```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }
	
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                                                       getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
}
```

其中，mContentParent为PhoneWindow的属性，它是Window中内容的根View，这个后面会说到：

```java
// This is the view in which the window contents are placed. It is either
// mDecor itself, or a child of mDecor where the contents go.
private ViewGroup mContentParent;
```

即查看installDecor()的实现：

```java
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor();
        ...
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
		...
    }
}
```

可以看出，当mDecor为空时，通过generateDecor()方法生成Decor，mDecor就是PhoneWindow中的DecorView：

```java
// This is the top-level view of the window, containing the window decor.
private DecorView mDecor;
```

接下去看一下DevorView是如何创建的：

```java
protected DecorView generateDecor() {
    return new DecorView(getContext(), -1);
}
```

其实就是直接new一个DecorView的实例，这里也就不再深入了。接下去看看后面的if语句，当mContentParent为空时，通过generateLayout创建布局：

```java
protected ViewGroup generateLayout(DecorView decor) {
	...
    // Inflate the window decor.

    int layoutResource;
    int features = getLocalFeatures();
    // System.out.println("Features: 0x" + Integer.toHexString(features));
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
    } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                R.attr.dialogTitleIconsDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title_icons;
        }
        // XXX Remove this once action bar supports these features.
        removeFeature(FEATURE_ACTION_BAR);
        // System.out.println("Title Icons!");
    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
               && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
        // Special case for a window with only a progress bar (and title).
        // XXX Need to have a no-title version of embedded windows.
        layoutResource = R.layout.screen_progress;
        // System.out.println("Progress!");
    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
        // Special case for a window with a custom title.
        // If the window is floating, we need a dialog layout
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                R.attr.dialogCustomTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_custom_title;
        }
        // XXX Remove this once action bar supports these features.
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        // If no other features and not embedded, only need a title.
        // If the window is floating, we need a dialog layout
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                R.styleable.Window_windowActionBarFullscreenDecorLayout,
                R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title;
        }
        // System.out.println("Title!");
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        layoutResource = R.layout.screen_simple_overlay_action_mode;
    } else {
        // Embedded, so no decoration is needed.
        layoutResource = R.layout.screen_simple;
        // System.out.println("Simple!");
    }

    mDecor.startChanging();

    View in = mLayoutInflater.inflate(layoutResource, null);
    decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    mContentRoot = (ViewGroup) in;

    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }
	...
    mDecor.finishChanging();

    return contentParent;
}
```

这个方法很长，这里把只截取了在DecorView中添加view的部分。可以看出，上面有一堆if-else，用来根据当前Activity的特性来创建不同的布局，如果去查看每个布局文件，会发现每个布局文件中都有个R.id.content的FrameLayout，也就是ID_ANDROID_CONTENT变量的值：

```java
/**
  * The ID that the main layout in the XML layout file should have.
  */
public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
```

可以看出，generateDecor最终把id为R.id.content的FrameLayout作为返回值，也就是说mContentParent就是这个ViewGroup。

最后，再回到PhoneWindow的setContentView方法中，注意这个方法的参数*layoutResID*，即Activity对应的布局文件ID，可以看到这样一行代码：

```java
 mLayoutInflater.inflate(layoutResID, mContentParent);
```

也就是说，在这里把Activity的布局文件创建并且添加到mContentParent中，因此，Activity的布局文件最终都是添加到DecorView的R.id.content这个FrameLayout中了。

至此，Activity的组成也就分析完成了，接下去就要开始正式分析touch事件分发机制了。

## 事件分发

从上一节我们了解到Activity中包含一个DecorView，DecorView中又包含Activity的布局文件，一般Activity的布局文件都是由一系列的ViewGroup和View组成的，Android的触摸事件分发机制也就是在这些ViewGroup和View中传递，寻找合适的View或ViewGroup处理该事件的过程。

首先，我们来看一下下面的这张图：

![event_dispatch](.\imgs\event_dispatch.png)

首先，当触摸事件发生时，它被传递到Activity的dispatchTouchEvent方法中进行处理，在Activity中，将事件传递给child进行处理，也就是传递给ViewGroup的dispatchTouchEvent处理，在ViewGroup中，一般来说都会调用onInterceptTouchEvent来判断是否要把当前事件拦截，如果要拦截的话，就把当前事件传递给当前ViewGroup的onTouchEvent处理，否则，将事件传递给child处理，也即是View的dispatchTouchEvent方法进行处理，在View中，会将该事件传递到onTouchEvent中进行处理。当事件到达onTouchEvent时，如果事件被处理了，则事件传递结束，否则，将事件往上层传递，即交给ViewGroup的onTouchEvent处理，如果ViewGroup也未处理此事件，则继续往上传递，直到Activity为止。

关于事件分发主体的流程基本就是这样，接下来从下面三个方面来详细分析事件分发的实现：

1. 事件如何从Activity中分发到ViewGroup中
2. ViewGroup如何将事件分发到View中
3. View中如何处理事件

### Activity中的事件分发

首先，看一下Activity的dispatchTouchEvent方法：

```java
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

可以看出，这里将触摸事件通过getWindow().superDispatchTouchEvent(ev)转发到了Window中处理，从第一节中可以知道，这个Window是PhoneWindow的实例，故直接查看PhoneWindow的实现：

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

仍旧根据第一节的分析可知，这里又将事件转发到DecorView中进行处理，故查看DecorView对应的方法：

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

可以看到DecorView的superDispatchTouchEvent方法实现直接调用了super.dispatchTouchEvent(event)来处理事件，所以我们来看看DecorView的类声明：

```java
private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
	...
}
```

可以看出，DecorView继承自FrameLayout，而FrameLayout又继承自ViewGroup：

```java
public class FrameLayout extends ViewGroup {
    ...
}
```

因为之前分析过Activity的组成，可以知道DecorView其实是Activity的根View，所以这里可以知道，Activity通过Window将触摸事件转发到DecorView中，在DecorView中就开始进行触摸事件的处理。

### ViewGroup对于触摸事件的处理

从前一小节分析可知，触摸事件经过Activity之后，首先被分发到根View中，并且根View是一个ViewGroup，所里这里ViewGroup的dispatchTouchEvent方法，因为这个方法比较长，所以分几部分进行分析：

```java
final int action = ev.getAction();
final int actionMasked = action & MotionEvent.ACTION_MASK;

// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The framework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}
```

首先，获取当前触摸事件的类型，这里先说一下，一次触摸事件由一次ACTION_DOWN、一系列的ACTION_MOVE以及一次ACTION_UP组成。

因为触摸事件系列都是以一个ACTION_DOWN事件开始的，所以在接收到新的ACTION_DOWN事件时，需要将当前的触摸状态清空，主要就是取消之前未处理的触摸事件，并把mFirstTouchTarget这个链表清空。

关于mFirstTouchTarget，这里需要简单介绍一下，这是一个TouchTarget类型的变量，当ViewGroup找到了子View处理ACTION_DOWN事件时，会将处理该子View添加到TarchTarget对象中，并把它赋值给mFirstTouchTarget 。

接下去就是调用onInterceptTouchEvent来判断是否需要拦截了：

```java
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
    || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
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
```

可以看到开头判断当前事件是否是ACTION_DOWN事件，表明如果是新的触摸事件则进入该if语句。

这里要注意，还有个条件mFirstTouchTarget != null，这个条件有什么作用呢？前面说过，当ViewGroup找到子View处理当前ACTION_DOWN事件时，会给mFirstTouchTarget 赋值并指向对应的子View，因此后续的ACTION_MOVE或ACTION_UP事件到来时， mFirstTouchTarget 就不为空，此时就会进入if语句内将拦截标记为置为false，以便将事件分发给对应的子View进行处理。那么如果ViewGroup没有找到子View处理ACTION_DOWN事件或者决定自身拦截该触摸事件时，会发生什么呢？这种情况下，mFirstTouchTarget不会被赋值，从而导致直接进入else分支，也就是将拦截标记为置为true，表明当前ViewGroup将事件拦截，不进行后续的事件分发动作了。

在if分支内，有一个disallowIntercept标记，这个标记的作用是用来取消拦截除了ACTION_DOWN以外的其他点击事件，一般在子View中使用。

一般情况下，disallowIntercept都为false，故这里会调用ViewGroup的onInterceptTouchEvent方法来判断ViewGroup是否需要拦截当前事件。

```java
boolean alreadyDispatchedToNewTouchTarget = false;
if (!canceled && !intercepted) {
    ...

    if (actionMasked == MotionEvent.ACTION_DOWN
        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
        ...

        final int childrenCount = mChildrenCount;
        if (newTouchTarget == null && childrenCount != 0) {
            final float x = ev.getX(actionIndex);
            final float y = ev.getY(actionIndex);
            // Find a child that can receive the event.
            // Scan children from front to back.
            final ArrayList<View> preorderedList = buildOrderedChildList();
            final boolean customOrder = preorderedList == null
                && isChildrenDrawingOrderEnabled();
            final View[] children = mChildren;
            for (int i = childrenCount - 1; i >= 0; i--) {
                final int childIndex = customOrder
                    ? getChildDrawingOrder(childrenCount, i) : i;
                final View child = (preorderedList == null)
                    ? children[childIndex] : preorderedList.get(childIndex);

                ...

                if (!canViewReceivePointerEvents(child)
                    || !isTransformedTouchPointInView(x, y, child, null)) {
                    ev.setTargetAccessibilityFocus(false);
                    continue;
                }

                ...

                resetCancelNextUpFlag(child);
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

                // The accessibility focus didn't handle the event, so clear
                // the flag and do a normal dispatch to all children.
                ev.setTargetAccessibilityFocus(false);
            }
            if (preorderedList != null) preorderedList.clear();
        }

       	...
    }
}
```

上面的代码起始就是ViewGroup分发事件的核心逻辑，其主要就是当收到ACTION_DOWN事件时，遍历ViewGroup的所有子View，然后判断子View是否能够接收触摸响应，和当前触点的位置是否在子View之内，如果两个条件都满足，则调用dispatchTransformedTouchEvent进行事件分发。如果事件分发成功了，则更新相应的状态，其中addTouchTarget中更新了mFirstTouchTarget的值：

```java
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

这里需要看一下dispatchTransformedTouchEvent的实现：

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
                                              View child, int desiredPointerIdBits) {
    final boolean handled;

   	...

    // Perform any necessary transformations and dispatch.
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }

        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
```

这里剔除了其他的逻辑，剩余主要实现。当传入的child为空时，则调用View的dispatchTouchEvent（因为ViewGroup继承View），否则，进行一次坐标转换，并且调用子View的dispatchTouchEvent进行事件分发。

上面分析的都是ViewGroup不拦截事件时的逻辑，当ViewGroup拦截事件时，intercepted为true，也就不会进入对View分发的逻辑，我们可以在后面看到这样一段代码：

```java
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                            TouchTarget.ALL_POINTER_IDS);
}
```

因为没有进入对子View分发的逻辑，所以走到这里的时候mFirstTouchTarget仍旧未空，这里会调用dispatchTransformedTouchEvent分发事件，注意，这里的child参数传入为null，根据dispatchTransformedTouchEvent的实现，可以知道这时候会调用super.dispatchTouchEvent()，也就是调用ViewGroup对应View的dispatchTouchEvent方法分发事件。

目前已经清楚了在ACTION_DOWN事件来临时事件分发的机制， 那么当ACTION_MOVE和ACTION_UP来时，事件时如何分发的呢？看下面的代码：

```java
if (mFirstTouchTarget == null) {
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
        } else { // ***1
            final boolean cancelChild = resetCancelNextUpFlag(target.child)
                || intercepted;
            if (dispatchTransformedTouchEvent(ev, cancelChild,
                                              target.child, target.pointerIdBits)) {
                handled = true;
            }
            ...
        }
        predecessor = target;
        target = next;
    }
}
```

可以看到，在if (mFirstTouchTarget == null) 之后还有一个else分支，前面说过，当处理ACTION_DOWN事件之后会给mFirstTouchTarget 赋值，并且在新的ACTION_DOWN事件到来之前该值一直存在，所以当 ACTION_MOVE或ACTION_UP事件到来时，会进入该分支。注意这里有一个alreadyDispatchedToNewTouchTarget标记，他表示是否分发了当前事件，根据前面处理ACTION_DOWN事件的代码可知，ACTION_MOVE或ACTION_UP时间来临时，该标记为一定为false，所以这里会进入注释为"***1"处的else分支。这里有一个cancelChild变量，在ACTION_MOVE或ACTION_UP事件到来时，resetCancelNextUpFlag方法返回值一般都是false，并且intercepted也为false，所以cancelChild也是false，接下去仍旧是调用dispatchTransformedTouchEvent方法分发事件，就不再多说了。

### View处理触摸事件

到这里为止，我们已经了解到ViewGroup是如何将事件分发到View的，接下来就要看下View中如何处理触摸事件。

```java
public boolean dispatchTouchEvent(MotionEvent event) {
	...

    boolean result = false;

    ...

    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
            && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    ...

    return result;
}
```

如果OnTouchListener不是空的，则直接调用OnTouchListener的onTouch方法处理此事件，当OnTouchListener的onTouch方法返回true之后，便不再调用onTouchEvent处理该事件了。

如果OnTouchListener为空或者其的onTouch方法返回false，则调用onTouchEvent方法处理当前事件：

```java
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
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    ...

                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
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
                }
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    setPressed(true, x, y);
                    checkForLongClick(0);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                setPressed(false);
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_MOVE:
                drawableHotspotChanged(x, y);

                // Be lenient about moving outside of buttons
                if (!pointInView(x, y, mTouchSlop)) {
                    // Outside button
                    removeTapCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        // Remove any future long press/tap checks
                        removeLongPressCallback();

                        setPressed(false);
                    }
                }
                break;
        }

        return true;
    }

    return false;
}
```

这里是对ACTION_UP、ACTION_DOWN、ACTION_CANCEL和ACTION_MOVE事件的处理，主要就是在ACTION_UP事件时调用performClick触发点击事件，在ACTION_DOWN事件时调用checkForLongClick方法根据具体方法触发长按事件，这里就不详细介绍了。

对事件分发的知识到这里就介绍完了，这里只是主要分析了事件分发相关的代码，如果想要了解更多，可以自己去看源码。