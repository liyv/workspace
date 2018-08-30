# Activity

```java
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
    public void startActivity(Intent intent, @Nullable Bundle options)//options==null
        startActivityForResult(intent, -1);
    startActivityForResult(intent, requestCode, null);

    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        //mParent 是什么,什么是嵌套的Activity
        if (mParent == null) {
            //1
            options = transferSpringboardActivityOptions(options);
            //2
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        }
    }
    Activity 的attach方法是什么时候调用的,方法内的一些变量赋值代表了什么意思,
    //startActivityForResult-1
    private Bundle transferSpringboardActivityOptions(Bundle options) {
        // mWindow 在这里的意思是什么,为什么可以 非null和 非 Active
        if (options == null && (mWindow != null && !mWindow.isActive())) {
            // ActivityOptions 的作用是什么
            final ActivityOptions activityOptions = getActivityOptions();
            if (activityOptions != null &&
                    activityOptions.getAnimationType() == ActivityOptions.ANIM_SCENE_TRANSITION) {
                return activityOptions.toBundle();
            }
        }
        return options;
    }
    //startActivityForResult-2
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        //onProvideReferrer==null
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        //monitor的作用是什么
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    //是true吗?
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```