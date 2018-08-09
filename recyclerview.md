# RecyclerView

```java
void initAdapterManager() {
        mAdapterHelper = new AdapterHelper(new AdapterHelper.Callback() {
            @Override
            public ViewHolder findViewHolder(int position) {
                final ViewHolder vh = findViewHolderForPosition(position, true);
                if (vh == null) {
                    return null;
                }
                // ensure it is not hidden because for adapter helper, the only thing matter is that
                // LM thinks view is a child.
                if (mChildHelper.isHidden(vh.itemView)) {
                    if (DEBUG) {
                        Log.d(TAG, "assuming view holder cannot be find because it is hidden");
                    }
                    return null;
                }
                return vh;
            }

            @Override
            public void offsetPositionsForRemovingInvisible(int start, int count) {
                offsetPositionRecordsForRemove(start, count, true);
                mItemsAddedOrRemoved = true;
                mState.mDeletedInvisibleItemCountSincePreviousLayout += count;
            }

            @Override
            public void offsetPositionsForRemovingLaidOutOrNewView(
                    int positionStart, int itemCount) {
                offsetPositionRecordsForRemove(positionStart, itemCount, false);
                mItemsAddedOrRemoved = true;
            }


            @Override
            public void markViewHoldersUpdated(int positionStart, int itemCount, Object payload) {
                viewRangeUpdate(positionStart, itemCount, payload);
                mItemsChanged = true;
            }

            @Override
            public void onDispatchFirstPass(AdapterHelper.UpdateOp op) {
                dispatchUpdate(op);
            }

            void dispatchUpdate(AdapterHelper.UpdateOp op) {
                switch (op.cmd) {
                    case AdapterHelper.UpdateOp.ADD:
                        mLayout.onItemsAdded(RecyclerView.this, op.positionStart, op.itemCount);
                        break;
                    case AdapterHelper.UpdateOp.REMOVE:
                        mLayout.onItemsRemoved(RecyclerView.this, op.positionStart, op.itemCount);
                        break;
                    case AdapterHelper.UpdateOp.UPDATE:
                        mLayout.onItemsUpdated(RecyclerView.this, op.positionStart, op.itemCount,
                                op.payload);
                        break;
                    case AdapterHelper.UpdateOp.MOVE:
                        mLayout.onItemsMoved(RecyclerView.this, op.positionStart, op.itemCount, 1);
                        break;
                }
            }

            @Override
            public void onDispatchSecondPass(AdapterHelper.UpdateOp op) {
                dispatchUpdate(op);
            }

            @Override
            public void offsetPositionsForAdd(int positionStart, int itemCount) {
                offsetPositionRecordsForInsert(positionStart, itemCount);
                mItemsAddedOrRemoved = true;
            }

            @Override
            public void offsetPositionsForMove(int from, int to) {
                offsetPositionRecordsForMove(from, to);
                // should we create mItemsMoved ?
                mItemsAddedOrRemoved = true;
            }
        });
    }

    mAdapterHelper


    private void initChildrenHelper() {
        mChildHelper = new ChildHelper(new ChildHelper.Callback() {
            @Override
            public int getChildCount() {
                return RecyclerView.this.getChildCount();
            }

            @Override
            public void addView(View child, int index) {
                if (VERBOSE_TRACING) {
                    TraceCompat.beginSection("RV addView");
                }
                RecyclerView.this.addView(child, index);
                if (VERBOSE_TRACING) {
                    TraceCompat.endSection();
                }
                dispatchChildAttached(child);
            }

            @Override
            public int indexOfChild(View view) {
                return RecyclerView.this.indexOfChild(view);
            }

            @Override
            public void removeViewAt(int index) {
                final View child = RecyclerView.this.getChildAt(index);
                if (child != null) {
                    dispatchChildDetached(child);

                    // Clear any android.view.animation.Animation that may prevent the item from
                    // detaching when being removed. If a child is re-added before the
                    // lazy detach occurs, it will receive invalid attach/detach sequencing.
                    child.clearAnimation();
                }
                if (VERBOSE_TRACING) {
                    TraceCompat.beginSection("RV removeViewAt");
                }
                RecyclerView.this.removeViewAt(index);
                if (VERBOSE_TRACING) {
                    TraceCompat.endSection();
                }
            }

            @Override
            public View getChildAt(int offset) {
                return RecyclerView.this.getChildAt(offset);
            }

            @Override
            public void removeAllViews() {
                final int count = getChildCount();
                for (int i = 0; i < count; i++) {
                    View child = getChildAt(i);
                    dispatchChildDetached(child);

                    // Clear any android.view.animation.Animation that may prevent the item from
                    // detaching when being removed. If a child is re-added before the
                    // lazy detach occurs, it will receive invalid attach/detach sequencing.
                    child.clearAnimation();
                }
                RecyclerView.this.removeAllViews();
            }

            @Override
            public ViewHolder getChildViewHolder(View view) {
                return getChildViewHolderInt(view);
            }

            @Override
            public void attachViewToParent(View child, int index,
                    ViewGroup.LayoutParams layoutParams) {
                final ViewHolder vh = getChildViewHolderInt(child);
                if (vh != null) {
                    if (!vh.isTmpDetached() && !vh.shouldIgnore()) {
                        throw new IllegalArgumentException("Called attach on a child which is not"
                                + " detached: " + vh + exceptionLabel());
                    }
                    if (DEBUG) {
                        Log.d(TAG, "reAttach " + vh);
                    }
                    vh.clearTmpDetachFlag();
                }
                RecyclerView.this.attachViewToParent(child, index, layoutParams);
            }

            @Override
            public void detachViewFromParent(int offset) {
                final View view = getChildAt(offset);
                if (view != null) {
                    final ViewHolder vh = getChildViewHolderInt(view);
                    if (vh != null) {
                        if (vh.isTmpDetached() && !vh.shouldIgnore()) {
                            throw new IllegalArgumentException("called detach on an already"
                                    + " detached child " + vh + exceptionLabel());
                        }
                        if (DEBUG) {
                            Log.d(TAG, "tmpDetach " + vh);
                        }
                        vh.addFlags(ViewHolder.FLAG_TMP_DETACHED);
                    }
                }
                RecyclerView.this.detachViewFromParent(offset);
            }

            @Override
            public void onEnteredHiddenState(View child) {
                final ViewHolder vh = getChildViewHolderInt(child);
                if (vh != null) {
                    vh.onEnteredHiddenState(RecyclerView.this);
                }
            }

            @Override
            public void onLeftHiddenState(View child) {
                final ViewHolder vh = getChildViewHolderInt(child);
                if (vh != null) {
                    vh.onLeftHiddenState(RecyclerView.this);
                }
            }
        });
    }

    mChildHelper


    if (orientation != mOrientation || mOrientationHelper == null) {
        mOrientationHelper =
                OrientationHelper.createOrientationHelper(this, orientation);//1
        mAnchorInfo.mOrientationHelper = mOrientationHelper;
        mOrientation = orientation;
        requestLayout();
    }

    //mOrientationHelper
    public static OrientationHelper createVerticalHelper(RecyclerView.LayoutManager layoutManager) {
        return new OrientationHelper(layoutManager) {
            @Override
            public int getEndAfterPadding() {
                return mLayoutManager.getHeight() - mLayoutManager.getPaddingBottom();
            }

            @Override
            public int getEnd() {
                return mLayoutManager.getHeight();
            }

            @Override
            public void offsetChildren(int amount) {
                mLayoutManager.offsetChildrenVertical(amount);
            }

            @Override
            public int getStartAfterPadding() {
                return mLayoutManager.getPaddingTop();
            }

            @Override
            public int getDecoratedMeasurement(View view) {
                final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams)
                        view.getLayoutParams();
                return mLayoutManager.getDecoratedMeasuredHeight(view) + params.topMargin
                        + params.bottomMargin;
            }

            @Override
            public int getDecoratedMeasurementInOther(View view) {
                final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams)
                        view.getLayoutParams();
                return mLayoutManager.getDecoratedMeasuredWidth(view) + params.leftMargin
                        + params.rightMargin;
            }

            @Override
            public int getDecoratedEnd(View view) {
                final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams)
                        view.getLayoutParams();
                return mLayoutManager.getDecoratedBottom(view) + params.bottomMargin;
            }

            @Override
            public int getDecoratedStart(View view) {
                final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams)
                        view.getLayoutParams();
                return mLayoutManager.getDecoratedTop(view) - params.topMargin;
            }

            @Override
            public int getTransformedEndWithDecoration(View view) {
                mLayoutManager.getTransformedBoundingBox(view, true, mTmpRect);
                return mTmpRect.bottom;
            }

            @Override
            public int getTransformedStartWithDecoration(View view) {
                mLayoutManager.getTransformedBoundingBox(view, true, mTmpRect);
                return mTmpRect.top;
            }

            @Override
            public int getTotalSpace() {
                return mLayoutManager.getHeight() - mLayoutManager.getPaddingTop()
                        - mLayoutManager.getPaddingBottom();
            }

            @Override
            public void offsetChild(View view, int offset) {
                view.offsetTopAndBottom(offset);
            }

            @Override
            public int getEndPadding() {
                return mLayoutManager.getPaddingBottom();
            }

            @Override
            public int getMode() {
                return mLayoutManager.getHeightMode();
            }

            @Override
            public int getModeInOther() {
                return mLayoutManager.getWidthMode();
            }
        };
    }

    //dispatchLayoutStep2()
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;
    mState.mInPreLayout = false;

    //LinearLayoutManager.onlayoutchildren(Recycler,State)
    mLayoutState =new LayoutState();
    mLayoutState.mRecycle = false;
    mShouldReverseLayout = mReverseLayout;//false
    //mAnchorInfo.reset();
    mPosition = NO_POSITION;
    mCoordinate = INVALID_OFFSET;
    mLayoutFromEnd = false;
    mValid = false;

    mAnchorInfo.mLayoutFromEnd=false;
    //anchorInfo.assignCoordinateFromPadding();
    mAnchorInfo.mCoordinate= 0//mLayoutManager.getPaddingTop()
    mAnchorInfo.mPosition=0;
    mAnchorInfo.mValid = true;

    final int extra=0;
    extraForEnd = extra;//0
    extraForStart = 0;

    int startOffset;
    int endOffset;
    final int firstLayoutDirection;//LayoutState.ITEM_DIRECTION_TAIL ==1

    mLayoutState.mInfinite = resolveIsInfinite();//false
    mLayoutState.mIsPreLayout = state.isPreLayout();//false
    // fill towards end
    //updateLayoutStateToFillEnd
    mLayoutState.mAvailable = mOrientationHelper.getEndAfterPadding() - offset;//mLayoutManager.getHeight() - mLayoutManager.getPaddingBottom() -offset==height
    mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD ://LayoutState.ITEM_DIRECTION_TAIL ==1
        LayoutState.ITEM_DIRECTION_TAIL;
    mLayoutState.mCurrentPosition = itemPosition;//0
    mLayoutState.mLayoutDirection = LayoutState.LAYOUT_END;//1
    mLayoutState.mOffset = offset;//0
    mLayoutState.mScrollingOffset = LayoutState.SCROLLING_OFFSET_NaN;
    mLayoutState.mExtra = extraForEnd;//0

    //fill(recycler, mLayoutState, state, false)
       final int start = layoutState.mAvailable;//height
       int remainingSpace = layoutState.mAvailable + layoutState.mExtra;//
       //layoutState.hasMore(state)
       return mCurrentPosition >= 0 && mCurrentPosition < state.getItemCount();//true if there are more items in the data adapter
       //layoutChunkResult.resetInternal();
            mConsumed = 0;
            mFinished = false;
            mIgnoreConsumed = false;
            mFocusable = false;
        //layoutChunk(recycler, state, layoutState, layoutChunkResult);重点是 layout
            //View view = layoutState.next(recycler);
            view = recycler.getViewForPosition(mCurrentPosition);
        layoutChunkResult.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);//消费了多少高度height
        int left, top, right, bottom;
        left = getPaddingLeft();
        right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);//width 宽度
        top = layoutState.mOffset;
        bottom = layoutState.mOffset + result.mConsumed;
        layoutDecoratedWithMargins(view, left, top, right, bottom);//child.layout()
        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
        layoutState.mAvailable -= layoutChunkResult.mConsumed;
        remainingSpace -= layoutChunkResult.mConsumed;
    endOffset = mLayoutState.mOffset;
    final int lastElement = mLayoutState.mCurrentPosition;

     // fill towards start 为什么
    //updateLayoutStateToFillStart
        mLayoutState.mAvailable = offset - mOrientationHelper.getStartAfterPadding();//0?
        mLayoutState.mCurrentPosition = itemPosition;//0
        mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL ://ITEM_DIRECTION_HEAD ==-1
                LayoutState.ITEM_DIRECTION_HEAD;
        mLayoutState.mLayoutDirection = LayoutState.LAYOUT_START;//-1
        mLayoutState.mOffset = offset;//0
        mLayoutState.mScrollingOffset = LayoutState.SCROLLING_OFFSET_NaN;
```