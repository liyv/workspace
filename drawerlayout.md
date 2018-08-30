# DrawerLayout

1. a

## 问题

- Drawerlayout 和 Viewpager 的实现方式有何不同?
- scrollTo 方式和 invalide() 方式有何却别

## 代码

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
        final int action = ev.getActionMasked();

        // "|" used deliberately here; both methods should be invoked.
        final boolean interceptForDrag = mLeftDragger.shouldInterceptTouchEvent(ev)
                | mRightDragger.shouldInterceptTouchEvent(ev);
        //....
        return interceptForDrag || interceptForTap || hasPeekingDrawer() || mChildrenCanceledTouch;
    }

    //shouldInterceptTouchEvent down事件
    case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                final int pointerId = ev.getPointerId(0);
                saveInitialMotion(x, y, pointerId);

                final View toCapture = findTopChildUnder((int) x, (int) y);

                // Catch a settling view if possible.
                if (toCapture == mCapturedView && mDragState == STATE_SETTLING) {
                    tryCaptureViewForDrag(toCapture, pointerId);
                }

                final int edgesTouched = mInitialEdgesTouched[pointerId];
                if ((edgesTouched & mTrackingEdges) != 0) {
                    mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                }
                break;
            }
    // mCallback.onEdgeTouched
    public void onEdgeTouched(int edgeFlags, int pointerId) {
            postDelayed(mPeekRunnable, PEEK_DELAY);
        }
    //mPeekRunnable
    void peekDrawer() {
            final View toCapture;
            final int childLeft;
            final int peekDistance = mDragger.getEdgeSize();//20dp
            final boolean leftEdge = mAbsGravity == Gravity.LEFT;//true
            if (leftEdge) {
                toCapture = findDrawerWithGravity(Gravity.LEFT);//drawer
                childLeft = (toCapture != null ? -toCapture.getWidth() : 0) + peekDistance;//- drawer width + 20dp
            } else {
                toCapture = findDrawerWithGravity(Gravity.RIGHT);
                childLeft = getWidth() - peekDistance;
            }
            // Only peek if it would mean making the drawer more visible and the drawer isn't locked
            if (toCapture != null && ((leftEdge && toCapture.getLeft() < childLeft)//true
                    || (!leftEdge && toCapture.getLeft() > childLeft))
                    && getDrawerLockMode(toCapture) == LOCK_MODE_UNLOCKED) {
                final LayoutParams lp = (LayoutParams) toCapture.getLayoutParams();
                mDragger.smoothSlideViewTo(toCapture, childLeft, toCapture.getTop());
                lp.isPeeking = true;
                invalidate();//invalidate的作用是什么 重点!!!

                closeOtherDrawer();

                cancelChildViewTouch();
            }
        }
        // mDragger.smoothSlideViewTo(toCapture, childLeft, toCapture.getTop());
        public boolean smoothSlideViewTo(@NonNull View child, int finalLeft, int finalTop) {//finalLeft= -drawerwidth+edgesize(==20dp)
        mCapturedView = child;
        mActivePointerId = INVALID_POINTER;

        boolean continueSliding = forceSettleCapturedViewAt(finalLeft, finalTop, 0, 0);
        if (!continueSliding && mDragState == STATE_IDLE && mCapturedView != null) {
            // If we're in an IDLE state to begin with and aren't moving anywhere, we
            // end up having a non-null capturedView with an IDLE dragState
            mCapturedView = null;
        }

        return continueSliding;
    }
```