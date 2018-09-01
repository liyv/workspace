# View

```java
//
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        mAttachInfo = info;
        //ViewOverlay 代表了什么
        if (mOverlay != null) {
            mOverlay.getOverlayView().dispatchAttachedToWindow(info, visibility);
        }
        mWindowAttachCount++;//Count of how many windows this view has been attached to.
        // We will need to evaluate the drawable state at least once.
        mPrivateFlags |= PFLAG_DRAWABLE_STATE_DIRTY;
        //ViewTreeObserver: Special tree observer used when mAttachInfo is null.
        if (mFloatingTreeObserver != null) {
            info.mTreeObserver.merge(mFloatingTreeObserver);
            mFloatingTreeObserver = null;
        }

        registerPendingFrameMetricsObservers();

        if ((mPrivateFlags&PFLAG_SCROLL_CONTAINER) != 0) {
            mAttachInfo.mScrollContainers.add(this);
            mPrivateFlags |= PFLAG_SCROLL_CONTAINER_ADDED;
        }
        // Transfer all pending runnables.
        if (mRunQueue != null) {
            mRunQueue.executeActions(info.mHandler);
            mRunQueue = null;
        }
        //和 systemui 有关?
        performCollectViewAttributes(mAttachInfo, visibility);
        onAttachedToWindow();

        ListenerInfo li = mListenerInfo;
        final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners =
                li != null ? li.mOnAttachStateChangeListeners : null;
        if (listeners != null && listeners.size() > 0) {
            // NOTE: because of the use of CopyOnWriteArrayList, we *must* use an iterator to
            // perform the dispatching. The iterator is a safe guard against listeners that
            // could mutate the list by calling the various add/remove methods. This prevents
            // the array from being modified while we iterate it.
            for (OnAttachStateChangeListener listener : listeners) {
                listener.onViewAttachedToWindow(this);
            }
        }

        int vis = info.mWindowVisibility;
        if (vis != GONE) {
            onWindowVisibilityChanged(vis);
            if (isShown()) {
                // Calling onVisibilityAggregated directly here since the subtree will also
                // receive dispatchAttachedToWindow and this same call
                onVisibilityAggregated(vis == VISIBLE);
            }
        }

        // Send onVisibilityChanged directly instead of dispatchVisibilityChanged.
        // As all views in the subtree will already receive dispatchAttachedToWindow
        // traversing the subtree again here is not desired.
        onVisibilityChanged(this, visibility);

        if ((mPrivateFlags&PFLAG_DRAWABLE_STATE_DIRTY) != 0) {
            // If nobody has evaluated the drawable state yet, then do it now.
            refreshDrawableState();
        }
        needGlobalAttributesUpdate(false);

        notifyEnterOrExitForAutoFillIfNeeded(true);
    }

    //This is called when the view is attached to a window.
    protected void onAttachedToWindow() {
        if ((mPrivateFlags & PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            mParent.requestTransparentRegion(this);
        }

        mPrivateFlags3 &= ~PFLAG3_IS_LAID_OUT;

        jumpDrawablesToCurrentState();

        resetSubtreeAccessibilityStateChanged();

        // rebuild, since Outline not maintained while View is detached
        //真有 outline 这种东西吗
        rebuildOutline();

        if (isFocused()) {
            //这是和输入有关吗?
            InputMethodManager imm = InputMethodManager.peekInstance();
            if (imm != null) {
                imm.focusIn(this);
            }
        }
    }
```

## 公共方法

```java

```