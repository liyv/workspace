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

    public void setVisibility(@Visibility int visibility) {
        setFlags(visibility, VISIBILITY_MASK);
    }
void setFlags(int flags, int mask) {
        final boolean accessibilityEnabled =
                AccessibilityManager.getInstance(mContext).isEnabled();
        final boolean oldIncludeForAccessibility = accessibilityEnabled && includeForAccessibility();

        int old = mViewFlags;
        mViewFlags = (mViewFlags & ~mask) | (flags & mask);

        int changed = mViewFlags ^ old;
        if (changed == 0) {
            return;
        }
        int privateFlags = mPrivateFlags;

        // If focusable is auto, update the FOCUSABLE bit.
        int focusableChangedByAuto = 0;
        if (((mViewFlags & FOCUSABLE_AUTO) != 0)
                && (changed & (FOCUSABLE_MASK | CLICKABLE)) != 0) {
            // Heuristic only takes into account whether view is clickable.
            final int newFocus;
            if ((mViewFlags & CLICKABLE) != 0) {
                newFocus = FOCUSABLE;
            } else {
                newFocus = NOT_FOCUSABLE;
            }
            mViewFlags = (mViewFlags & ~FOCUSABLE) | newFocus;
            focusableChangedByAuto = (old & FOCUSABLE) ^ (newFocus & FOCUSABLE);
            changed = (changed & ~FOCUSABLE) | focusableChangedByAuto;
        }

        /* Check if the FOCUSABLE bit has changed */
        if (((changed & FOCUSABLE) != 0) && ((privateFlags & PFLAG_HAS_BOUNDS) != 0)) {
            if (((old & FOCUSABLE) == FOCUSABLE)
                    && ((privateFlags & PFLAG_FOCUSED) != 0)) {
                /* Give up focus if we are no longer focusable */
                clearFocus();
                if (mParent instanceof ViewGroup) {
                    ((ViewGroup) mParent).clearFocusedInCluster();
                }
            } else if (((old & FOCUSABLE) == NOT_FOCUSABLE)
                    && ((privateFlags & PFLAG_FOCUSED) == 0)) {
                /*
                 * Tell the view system that we are now available to take focus
                 * if no one else already has it.
                 */
                if (mParent != null) {
                    ViewRootImpl viewRootImpl = getViewRootImpl();
                    if (!sAutoFocusableOffUIThreadWontNotifyParents
                            || focusableChangedByAuto == 0
                            || viewRootImpl == null
                            || viewRootImpl.mThread == Thread.currentThread()) {
                        mParent.focusableViewAvailable(this);
                    }
                }
            }
        }

        final int newVisibility = flags & VISIBILITY_MASK;
        if (newVisibility == VISIBLE) {
            if ((changed & VISIBILITY_MASK) != 0) {
                /*
                 * If this view is becoming visible, invalidate it in case it changed while
                 * it was not visible. Marking it drawn ensures that the invalidation will
                 * go through.
                 */
                mPrivateFlags |= PFLAG_DRAWN;
                invalidate(true);

                needGlobalAttributesUpdate(true);

                // a view becoming visible is worth notifying the parent
                // about in case nothing has focus.  even if this specific view
                // isn't focusable, it may contain something that is, so let
                // the root view try to give this focus if nothing else does.
                if ((mParent != null) && (mBottom > mTop) && (mRight > mLeft)) {
                    mParent.focusableViewAvailable(this);
                }
            }
        }

        /* Check if the GONE bit has changed */
        if ((changed & GONE) != 0) {
            needGlobalAttributesUpdate(false);
            requestLayout();

            if (((mViewFlags & VISIBILITY_MASK) == GONE)) {
                if (hasFocus()) {
                    clearFocus();
                    if (mParent instanceof ViewGroup) {
                        ((ViewGroup) mParent).clearFocusedInCluster();
                    }
                }
                clearAccessibilityFocus();
                destroyDrawingCache();
                if (mParent instanceof View) {
                    // GONE views noop invalidation, so invalidate the parent
                    ((View) mParent).invalidate(true);
                }
                // Mark the view drawn to ensure that it gets invalidated properly the next
                // time it is visible and gets invalidated
                mPrivateFlags |= PFLAG_DRAWN;
            }
            if (mAttachInfo != null) {
                mAttachInfo.mViewVisibilityChanged = true;
            }
        }

        /* Check if the VISIBLE bit has changed */
        if ((changed & INVISIBLE) != 0) {
            needGlobalAttributesUpdate(false);
            /*
             * If this view is becoming invisible, set the DRAWN flag so that
             * the next invalidate() will not be skipped.
             */
            mPrivateFlags |= PFLAG_DRAWN;

            if (((mViewFlags & VISIBILITY_MASK) == INVISIBLE)) {
                // root view becoming invisible shouldn't clear focus and accessibility focus
                if (getRootView() != this) {
                    if (hasFocus()) {
                        clearFocus();
                        if (mParent instanceof ViewGroup) {
                            ((ViewGroup) mParent).clearFocusedInCluster();
                        }
                    }
                    clearAccessibilityFocus();
                }
            }
            if (mAttachInfo != null) {
                mAttachInfo.mViewVisibilityChanged = true;
            }
        }

        if ((changed & VISIBILITY_MASK) != 0) {
            // If the view is invisible, cleanup its display list to free up resources
            if (newVisibility != VISIBLE && mAttachInfo != null) {
                cleanupDraw();
            }

            if (mParent instanceof ViewGroup) {
                ((ViewGroup) mParent).onChildVisibilityChanged(this,
                        (changed & VISIBILITY_MASK), newVisibility);
                ((View) mParent).invalidate(true);
            } else if (mParent != null) {
                mParent.invalidateChild(this, null);
            }

            if (mAttachInfo != null) {
                dispatchVisibilityChanged(this, newVisibility);

                // Aggregated visibility changes are dispatched to attached views
                // in visible windows where the parent is currently shown/drawn
                // or the parent is not a ViewGroup (and therefore assumed to be a ViewRoot),
                // discounting clipping or overlapping. This makes it a good place
                // to change animation states.
                if (mParent != null && getWindowVisibility() == VISIBLE &&
                        ((!(mParent instanceof ViewGroup)) || ((ViewGroup) mParent).isShown())) {
                    dispatchVisibilityAggregated(newVisibility == VISIBLE);
                }
                notifySubtreeAccessibilityStateChangedIfNeeded();
            }
        }

        if ((changed & WILL_NOT_CACHE_DRAWING) != 0) {
            destroyDrawingCache();
        }

        if ((changed & DRAWING_CACHE_ENABLED) != 0) {
            destroyDrawingCache();
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            invalidateParentCaches();
        }

        if ((changed & DRAWING_CACHE_QUALITY_MASK) != 0) {
            destroyDrawingCache();
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }

        if ((changed & DRAW_MASK) != 0) {
            if ((mViewFlags & WILL_NOT_DRAW) != 0) {
                if (mBackground != null
                        || mDefaultFocusHighlight != null
                        || (mForegroundInfo != null && mForegroundInfo.mDrawable != null)) {
                    mPrivateFlags &= ~PFLAG_SKIP_DRAW;
                } else {
                    mPrivateFlags |= PFLAG_SKIP_DRAW;
                }
            } else {
                mPrivateFlags &= ~PFLAG_SKIP_DRAW;
            }
            requestLayout();
            invalidate(true);
        }

        if ((changed & KEEP_SCREEN_ON) != 0) {
            if (mParent != null && mAttachInfo != null && !mAttachInfo.mRecomputeGlobalAttributes) {
                mParent.recomputeViewAttributes(this);
            }
        }

        if (accessibilityEnabled) {
            if ((changed & FOCUSABLE) != 0 || (changed & VISIBILITY_MASK) != 0
                    || (changed & CLICKABLE) != 0 || (changed & LONG_CLICKABLE) != 0
                    || (changed & CONTEXT_CLICKABLE) != 0) {
                if (oldIncludeForAccessibility != includeForAccessibility()) {
                    notifySubtreeAccessibilityStateChangedIfNeeded();
                } else {
                    notifyViewAccessibilityStateChangedIfNeeded(
                            AccessibilityEvent.CONTENT_CHANGE_TYPE_UNDEFINED);
                }
            } else if ((changed & ENABLED_MASK) != 0) {
                notifyViewAccessibilityStateChangedIfNeeded(
                        AccessibilityEvent.CONTENT_CHANGE_TYPE_UNDEFINED);
            }
        }
    }
public void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }

    void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        if (mGhostView != null) {
            mGhostView.invalidate(true);
            return;
        }

        if (skipInvalidate()) {
            return;
        }

        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            if (fullInvalidate) {
                mLastIsOpaque = isOpaque();
                mPrivateFlags &= ~PFLAG_DRAWN;
            }

            mPrivateFlags |= PFLAG_DIRTY;

            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            // Propagate the damage rectangle to the parent view.
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }

            // Damage the entire projection receiver, if necessary.
            if (mBackground != null && mBackground.isProjected()) {
                final View receiver = getProjectionReceiver();
                if (receiver != null) {
                    receiver.damageInParent();
                }
            }
        }
    }
    public final void invalidateChild(View child, final Rect dirty) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null && attachInfo.mHardwareAccelerated) {
            // HW accelerated fast path
            onDescendantInvalidated(child, child);
            return;
        }

        ViewParent parent = this;
        if (attachInfo != null) {
            // If the child is drawing an animation, we want to copy this flag onto
            // ourselves and the parent to make sure the invalidate request goes
            // through
            final boolean drawAnimation = (child.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0;

            // Check whether the child that requests the invalidate is fully opaque
            // Views being animated or transformed are not considered opaque because we may
            // be invalidating their old position and need the parent to paint behind them.
            Matrix childMatrix = child.getMatrix();
            final boolean isOpaque = child.isOpaque() && !drawAnimation &&
                    child.getAnimation() == null && childMatrix.isIdentity();
            // Mark the child as dirty, using the appropriate flag
            // Make sure we do not set both flags at the same time
            int opaqueFlag = isOpaque ? PFLAG_DIRTY_OPAQUE : PFLAG_DIRTY;

            if (child.mLayerType != LAYER_TYPE_NONE) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            final int[] location = attachInfo.mInvalidateChildLocation;
            location[CHILD_LEFT_INDEX] = child.mLeft;
            location[CHILD_TOP_INDEX] = child.mTop;
            if (!childMatrix.isIdentity() ||
                    (mGroupFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
                RectF boundingRect = attachInfo.mTmpTransformRect;
                boundingRect.set(dirty);
                Matrix transformMatrix;
                if ((mGroupFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
                    Transformation t = attachInfo.mTmpTransformation;
                    boolean transformed = getChildStaticTransformation(child, t);
                    if (transformed) {
                        transformMatrix = attachInfo.mTmpMatrix;
                        transformMatrix.set(t.getMatrix());
                        if (!childMatrix.isIdentity()) {
                            transformMatrix.preConcat(childMatrix);
                        }
                    } else {
                        transformMatrix = childMatrix;
                    }
                } else {
                    transformMatrix = childMatrix;
                }
                transformMatrix.mapRect(boundingRect);
                dirty.set((int) Math.floor(boundingRect.left),
                        (int) Math.floor(boundingRect.top),
                        (int) Math.ceil(boundingRect.right),
                        (int) Math.ceil(boundingRect.bottom));
            }

            do {
                View view = null;
                if (parent instanceof View) {
                    view = (View) parent;
                }

                if (drawAnimation) {
                    if (view != null) {
                        view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                    } else if (parent instanceof ViewRootImpl) {
                        ((ViewRootImpl) parent).mIsAnimating = true;
                    }
                }

                // If the parent is dirty opaque or not dirty, mark it dirty with the opaque
                // flag coming from the child that initiated the invalidate
                if (view != null) {
                    if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
                        opaqueFlag = PFLAG_DIRTY;
                    }
                    if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                        view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                    }
                }

                parent = parent.invalidateChildInParent(location, dirty);
                if (view != null) {
                    // Account for transform on current parent
                    Matrix m = view.getMatrix();
                    //是否单位矩阵
                    if (!m.isIdentity()) {
                        RectF boundingRect = attachInfo.mTmpTransformRect;
                        boundingRect.set(dirty);
                        m.mapRect(boundingRect);
                        dirty.set((int) Math.floor(boundingRect.left),
                                (int) Math.floor(boundingRect.top),
                                (int) Math.ceil(boundingRect.right),
                                (int) Math.ceil(boundingRect.bottom));
                    }
                }
            } while (parent != null);
        }
    }
```

## 公共方法

```java
//viewrootimpl
 @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            //遍历
            scheduleTraversals();
        }
}

```

## 问题

- requestlayout() 和 invalidate() 是什么关系?什么时候,什么情况下应该用哪个方法?什么时候用组合方法,有哪些组合的先后顺序?

```java
//requestlayout() 在前
@Override
    public void bringChildToFront(View child) {
        final int index = indexOfChild(child);
        if (index >= 0) {
            removeFromArray(index);
            addInArray(child, mChildrenCount);
            child.mParent = this;
            //
            requestLayout();
            invalidate();
        }
}
// viewgroup 也是 requestLayout() 在前
public void addView(View child, int index, LayoutParams params) {
        if (DBG) {
            System.out.println(this + " addView");
        }

        if (child == null) {
            throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
        }

        // addViewInner() will call child.requestLayout() when setting the new LayoutParams
        // therefore, we call requestLayout() on ourselves before, so that the child's request
        // will be blocked at our level
        //上面这话是什么意思
        requestLayout();
        invalidate(true);
        addViewInner(child, index, params, false);
    }
```