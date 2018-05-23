### BackStackRecord是FragmentTransaction的实现类

> BackStackRecord：Entry of an operation on the fragment back stack.

```
static final int OP_NULL = 0;
static final int OP_ADD = 1;
static final int OP_REPLACE = 2;
static final int OP_REMOVE = 3;
static final int OP_HIDE = 4;
static final int OP_SHOW = 5;
static final int OP_DETACH = 6;
static final int OP_ATTACH = 7;
static final int OP_SET_PRIMARY_NAV = 8;
static final int OP_UNSET_PRIMARY_NAV = 9;
```

针对Fragment的不同操作，对于BackStackRecord来说，都是不同标识位的不同操作；她们最终都会执行BackStackRecord的doAddOp()方法

```java
    private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
        final Class fragmentClass = fragment.getClass();
        final int modifiers = fragmentClass.getModifiers();
        if (fragmentClass.isAnonymousClass() || !Modifier.isPublic(modifiers)
                || (fragmentClass.isMemberClass() && !Modifier.isStatic(modifiers))) {
            throw new IllegalStateException("Fragment " + fragmentClass.getCanonicalName()
                    + " must be a public static class to be  properly recreated from"
                    + " instance state.");
        }

        fragment.mFragmentManager = mManager;

        if (tag != null) {
            if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
                throw new IllegalStateException("Can't change tag of fragment "
                        + fragment + ": was " + fragment.mTag
                        + " now " + tag);
            }
            fragment.mTag = tag;
        }

        if (containerViewId != 0) {
            if (containerViewId == View.NO_ID) {
                throw new IllegalArgumentException("Can't add fragment "
                        + fragment + " with tag " + tag + " to container view with no id");
            }
            if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
                throw new IllegalStateException("Can't change container ID of fragment "
                        + fragment + ": was " + fragment.mFragmentId
                        + " now " + containerViewId);
            }
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }

        addOp(new Op(opcmd, fragment));
    }
```

调用commit/commitAllowingStateLoss之后都会走下面这个方法；

```java
    int commitInternal(boolean allowStateLoss) {
        if (mCommitted) throw new IllegalStateException("commit already called");
        if (FragmentManagerImpl.DEBUG) {
            Log.v(TAG, "Commit: " + this);
            LogWriter logw = new LogWriter(TAG);
            PrintWriter pw = new PrintWriter(logw);
            dump("  ", null, pw, null);
            pw.close();
        }
        mCommitted = true;
        if (mAddToBackStack) {
            mIndex = mManager.allocBackStackIndex(this);
        } else {
            mIndex = -1;
        }
        mManager.enqueueAction(this, allowStateLoss);
        return mIndex;
    }
```

而`commitInternal`最终都会调用FragmentManagerImpl中的enqueueAction方法；

```java
    /**
     * Adds an action to the queue of pending actions.
     *
     * @param action the action to add
     * @param allowStateLoss whether to allow loss of state information
     * @throws IllegalStateException if the activity has been destroyed
     */
    public void enqueueAction(OpGenerator action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            checkStateLoss();
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                if (allowStateLoss) {
                    // This FragmentManager isn't attached, so drop the entire transaction.
                    return;
                }
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<>();
            }
            mPendingActions.add(action);
            scheduleCommit();
        }
    }
```



```java

    /**
     * Schedules the execution when one hasn't been scheduled already. This should happen
     * the first time {@link #enqueueAction(OpGenerator, boolean)} is called or when
     * a postponed transaction has been started with
     * {@link Fragment#startPostponedEnterTransition()}
     */
    private void scheduleCommit() {
        synchronized (this) {
            boolean postponeReady =
                    mPostponedTransactions != null && !mPostponedTransactions.isEmpty();
            boolean pendingReady = mPendingActions != null && mPendingActions.size() == 1;
            if (postponeReady || pendingReady) {
                mHost.getHandler().removeCallbacks(mExecCommit);
                mHost.getHandler().post(mExecCommit);
            }
        }
    }
```



```java
Runnable mExecCommit = new Runnable() {
        @Override
        public void run() {
            execPendingActions();
        }
    };
```



```java
    /**
     * Only call from main thread!
     */
    public boolean execPendingActions() {
        ensureExecReady(true);

        boolean didSomething = false;
        while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
            mExecutingActions = true;
            try {
                removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
            } finally {
                cleanupExec();
            }
            didSomething = true;
        }

        doPendingDeferredStart();
        burpActive();

        return didSomething;
    }
```



```java
    /**
     * An add or pop transaction to be scheduled for the UI thread.
     */
    interface OpGenerator {
        /**
         * Generate transactions to add to {@code records} and whether or not the transaction is
         * an add or pop to {@code isRecordPop}.
         *
         * records and isRecordPop must be added equally so that each transaction in records
         * matches the boolean for whether or not it is a pop in isRecordPop.
         *
         * @param records A list to add transactions to.
         * @param isRecordPop A list to add whether or not the transactions added to records is
         *                    a pop transaction.
         * @return true if something was added or false otherwise.
         */
        boolean generateOps(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop);
    }
```



```java
    /**
     * Adds all records in the pending actions to records and whether they are add or pop
     * operations to isPop. After executing, the pending actions will be empty.
     *
     * @param records All pending actions will generate BackStackRecords added to this.
     *                This contains the transactions, in order, to execute.
     * @param isPop All pending actions will generate booleans to add to this. This contains
     *              an entry for each entry in records to indicate whether or not it is a
     *              pop action.
     */
    private boolean generateOpsForPendingActions(ArrayList<BackStackRecord> records,
            ArrayList<Boolean> isPop) {
        int numActions;
        synchronized (this) {
            if (mPendingActions == null || mPendingActions.size() == 0) {
                return false;
            }

            numActions = mPendingActions.size();
            for (int i = 0; i < numActions; i++) {
                mPendingActions.get(i).generateOps(records, isPop);
            }
            mPendingActions.clear();
            mHost.getHandler().removeCallbacks(mExecCommit);
        }
        return numActions > 0;
    }
```



```java
/**
     * Remove redundant BackStackRecord operations and executes them. This method merges operations
     * of proximate records that allow reordering. See
     * {@link FragmentTransaction#setReorderingAllowed(boolean)}.
     * <p>
     * For example, a transaction that adds to the back stack and then another that pops that
     * back stack record will be optimized to remove the unnecessary operation.
     * <p>
     * Likewise, two transactions committed that are executed at the same time will be optimized
     * to remove the redundant operations as well as two pop operations executed together.
     *
     * @param records The records pending execution
     * @param isRecordPop The direction that these records are being run.
     */
    private void removeRedundantOperationsAndExecute(ArrayList<BackStackRecord> records,
            ArrayList<Boolean> isRecordPop) {
        if (records == null || records.isEmpty()) {
            return;
        }

        if (isRecordPop == null || records.size() != isRecordPop.size()) {
            throw new IllegalStateException("Internal error with the back stack records");
        }

        // Force start of any postponed transactions that interact with scheduled transactions:
        executePostponedTransaction(records, isRecordPop);

        final int numRecords = records.size();
        int startIndex = 0;
        for (int recordNum = 0; recordNum < numRecords; recordNum++) {
            final boolean canReorder = records.get(recordNum).mReorderingAllowed;
            if (!canReorder) {
                // execute all previous transactions
                if (startIndex != recordNum) {
                    executeOpsTogether(records, isRecordPop, startIndex, recordNum);
                }
                // execute all pop operations that don't allow reordering together or
                // one add operation
                int reorderingEnd = recordNum + 1;
                if (isRecordPop.get(recordNum)) {
                    while (reorderingEnd < numRecords
                            && isRecordPop.get(reorderingEnd)
                            && !records.get(reorderingEnd).mReorderingAllowed) {
                        reorderingEnd++;
                    }
                }
                executeOpsTogether(records, isRecordPop, recordNum, reorderingEnd);
                startIndex = reorderingEnd;
                recordNum = reorderingEnd - 1;
            }
        }
        if (startIndex != numRecords) {
            executeOpsTogether(records, isRecordPop, startIndex, numRecords);
        }
    }
```


```java
/**
     * Executes a subset of a list of BackStackRecords, all of which either allow reordering or
     * do not allow ordering.
     * @param records A list of BackStackRecords that are to be executed
     * @param isRecordPop The direction that these records are being run.
     * @param startIndex The index of the first record in <code>records</code> to be executed
     * @param endIndex One more than the final record index in <code>records</code> to executed.
     */
    private void executeOpsTogether(ArrayList<BackStackRecord> records,
            ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
        final boolean allowReordering = records.get(startIndex).mReorderingAllowed;
        boolean addToBackStack = false;
        if (mTmpAddedFragments == null) {
            mTmpAddedFragments = new ArrayList<>();
        } else {
            mTmpAddedFragments.clear();
        }
        mTmpAddedFragments.addAll(mAdded);
        Fragment oldPrimaryNav = getPrimaryNavigationFragment();
        for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
            final BackStackRecord record = records.get(recordNum);
            final boolean isPop = isRecordPop.get(recordNum);
            if (!isPop) {
                oldPrimaryNav = record.expandOps(mTmpAddedFragments, oldPrimaryNav);
            } else {
                oldPrimaryNav = record.trackAddedFragmentsInPop(mTmpAddedFragments, oldPrimaryNav);
            }
            addToBackStack = addToBackStack || record.mAddToBackStack;
        }
        mTmpAddedFragments.clear();

        if (!allowReordering) {
            FragmentTransition.startTransitions(this, records, isRecordPop, startIndex, endIndex,
                    false);
        }
        executeOps(records, isRecordPop, startIndex, endIndex);

        int postponeIndex = endIndex;
        if (allowReordering) {
            ArraySet<Fragment> addedFragments = new ArraySet<>();
            addAddedFragments(addedFragments);
            postponeIndex = postponePostponableTransactions(records, isRecordPop,
                    startIndex, endIndex, addedFragments);
            makeRemovedFragmentsInvisible(addedFragments);
        }

        if (postponeIndex != startIndex && allowReordering) {
            // need to run something now
            FragmentTransition.startTransitions(this, records, isRecordPop, startIndex,
                    postponeIndex, true);
            moveToState(mCurState, true);
        }

        for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
            final BackStackRecord record = records.get(recordNum);
            final boolean isPop = isRecordPop.get(recordNum);
            if (isPop && record.mIndex >= 0) {
                freeBackStackIndex(record.mIndex);
                record.mIndex = -1;
            }
            record.runOnCommitRunnables();
        }
        if (addToBackStack) {
            reportBackStackChanged();
        }
    }
```



```

```

```java
/**
 * Run the operations in the BackStackRecords, either to push or pop.
 *
 * @param records The list of records whose operations should be run.
 * @param isRecordPop The direction that these records are being run.
 * @param startIndex The index of the first entry in records to run.
 * @param endIndex One past the index of the final entry in records to run.
 */
private static void executeOps(ArrayList<BackStackRecord> records,
        ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
    for (int i = startIndex; i < endIndex; i++) {
        final BackStackRecord record = records.get(i);
        final boolean isPop = isRecordPop.get(i);
        if (isPop) {
            record.bumpBackStackNesting(-1);
            // Only execute the add operations at the end of
            // all transactions.
            boolean moveToState = i == (endIndex - 1);
            record.executePopOps(moveToState);
        } else {
            record.bumpBackStackNesting(1);
            record.executeOps();
        }
    }
}
```



```java
/**
     * Reverses the execution of the operations within this transaction. The Fragment states will
     * only be modified if reordering is not allowed.
     *
     * @param moveToState {@code true} if added fragments should be moved to their final state
     *                    in ordered transactions
     */
    void executePopOps(boolean moveToState) {
        for (int opNum = mOps.size() - 1; opNum >= 0; opNum--) {
            final Op op = mOps.get(opNum);
            Fragment f = op.fragment;
            if (f != null) {
                f.setNextTransition(FragmentManagerImpl.reverseTransit(mTransition),
                        mTransitionStyle);
            }
            switch (op.cmd) {
                case OP_ADD:
                    f.setNextAnim(op.popExitAnim);
                    mManager.removeFragment(f);
                    break;
                case OP_REMOVE:
                    f.setNextAnim(op.popEnterAnim);
                    mManager.addFragment(f, false);
                    break;
                case OP_HIDE:
                    f.setNextAnim(op.popEnterAnim);
                    mManager.showFragment(f);
                    break;
                case OP_SHOW:
                    f.setNextAnim(op.popExitAnim);
                    mManager.hideFragment(f);
                    break;
                case OP_DETACH:
                    f.setNextAnim(op.popEnterAnim);
                    mManager.attachFragment(f);
                    break;
                case OP_ATTACH:
                    f.setNextAnim(op.popExitAnim);
                    mManager.detachFragment(f);
                    break;
                case OP_SET_PRIMARY_NAV:
                    mManager.setPrimaryNavigationFragment(null);
                    break;
                case OP_UNSET_PRIMARY_NAV:
                    mManager.setPrimaryNavigationFragment(f);
                    break;
                default:
                    throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
            }
            if (!mReorderingAllowed && op.cmd != OP_REMOVE && f != null) {
                mManager.moveFragmentToExpectedState(f);
            }
        }
        if (!mReorderingAllowed && moveToState) {
            mManager.moveToState(mManager.mCurState, true);
        }
    }
```
当Fragment添加到返回栈的时候，会在FragmentManager中维护的一个List中add一条记录，当弹出是remove掉这一条记录，并且将remove的这一条记录添加到record中，但是将isPopRecord中的值改为true.

```java
 /**
     * Changes the state of the fragment manager to {@code newState}. If the fragment manager
     * changes state or {@code always} is {@code true}, any fragments within it have their
     * states updated as well.
     *
     * @param newState The new state for the fragment manager
     * @param always If {@code true}, all fragments update their state, even
     *               if {@code newState} matches the current fragment manager's state.
     */ 
void moveToState(int newState, boolean always) {
        if (mHost == null && newState != Fragment.INITIALIZING) {
            throw new IllegalStateException("No activity");
        }

        if (!always && newState == mCurState) {
            return;
        }

        mCurState = newState;
    	//每次调用addFragement的时候，都会将对应的Fragment添加到mActive和mAdded中
        if (mActive != null) {
            boolean loadersRunning = false;

            // Must add them in the proper order. mActive fragments may be out of order
            final int numAdded = mAdded.size();
            for (int i = 0; i < numAdded; i++) {
                Fragment f = mAdded.get(i);
                moveFragmentToExpectedState(f);
                if (f.mLoaderManager != null) {
                    loadersRunning |= f.mLoaderManager.hasRunningLoaders();
                }
            }

            // Now iterate through all active fragments. These will include those that are removed
            // and detached.
            final int numActive = mActive.size();
            for (int i = 0; i < numActive; i++) {
                Fragment f = mActive.valueAt(i);
                if (f != null && (f.mRemoving || f.mDetached) && !f.mIsNewlyAdded) {
                    moveFragmentToExpectedState(f);
                    if (f.mLoaderManager != null) {
                        loadersRunning |= f.mLoaderManager.hasRunningLoaders();
                    }
                }
            }

            if (!loadersRunning) {
                startPendingDeferredFragments();
            }

            if (mNeedMenuInvalidate && mHost != null && mCurState == Fragment.RESUMED) {
                mHost.onSupportInvalidateOptionsMenu();
                mNeedMenuInvalidate = false;
            }
        }
    }
```



```java
/**
     * Moves a fragment to its expected final state or the fragment manager's state, depending
     * on whether the fragment manager's state is raised properly.
     *
     * @param f The fragment to change.
     */
    void moveFragmentToExpectedState(Fragment f) {
        if (f == null) {
            return;
        }
        int nextState = mCurState;
        if (f.mRemoving) {
            if (f.isInBackStack()) {
                nextState = Math.min(nextState, Fragment.CREATED);
            } else {
                nextState = Math.min(nextState, Fragment.INITIALIZING);
            }
        }
        moveToState(f, nextState, f.getNextTransition(), f.getNextTransitionStyle(), false);

        if (f.mView != null) {
            // Move the view if it is out of order
            Fragment underFragment = findFragmentUnder(f);
            if (underFragment != null) {
                final View underView = underFragment.mView;
                // make sure this fragment is in the right order.
                final ViewGroup container = f.mContainer;
                int underIndex = container.indexOfChild(underView);
                int viewIndex = container.indexOfChild(f.mView);
                if (viewIndex < underIndex) {
                    container.removeViewAt(viewIndex);
                    container.addView(f.mView, underIndex);
                }
            }
            if (f.mIsNewlyAdded && f.mContainer != null) {
                // Make it visible and run the animations
                if (f.mPostponedAlpha > 0f) {
                    f.mView.setAlpha(f.mPostponedAlpha);
                }
                f.mPostponedAlpha = 0f;
                f.mIsNewlyAdded = false;
                // run animations:
                AnimationOrAnimator anim = loadAnimation(f, f.getNextTransition(), true,
                        f.getNextTransitionStyle());
                if (anim != null) {
                    setHWLayerAnimListenerIfAlpha(f.mView, anim);
                    if (anim.animation != null) {
                        f.mView.startAnimation(anim.animation);
                    } else {
                        anim.animator.setTarget(f.mView);
                        anim.animator.start();
                    }
                }
            }
        }
        if (f.mHiddenChanged) {
            completeShowHideFragment(f);
        }
    }
```

moveToState系列方法有两种，

* 第一种`moveToState(int newState, boolean always)`
* 第二种`moveToState(Fragment f, int newState, int transit, int transitionStyle,boolean keepActive)`

在第一种对FramgmentManagerImpl中的`mCurState`进行了改变，同时**TODO**

对于FragmentManagerImpl中：

```java
final ArrayList<Fragment> mAdded = new ArrayList<>();
SparseArray<Fragment> mActive;
```

每次`addFragment`都会调用这个方法：

```java
void makeActive(Fragment f) {
        if (f.mIndex >= 0) {
            return;
        }

        f.setIndex(mNextFragmentIndex++, mParent);
        if (mActive == null) {
            mActive = new SparseArray<>();
        }
        mActive.put(f.mIndex, f);
        if (DEBUG) Log.v(TAG, "Allocated fragment index " + f);
    }
```









```java
/**
     * Moves a fragment to its expected final state or the fragment manager's state, depending
     * on whether the fragment manager's state is raised properly.
     *
     * @param f The fragment to change.
     */
    void moveFragmentToExpectedState(Fragment f) {
        if (f == null) {
            return;
        }
        int nextState = mCurState;
        if (f.mRemoving) {
            if (f.isInBackStack()) {
                nextState = Math.min(nextState, Fragment.CREATED);
            } else {
                nextState = Math.min(nextState, Fragment.INITIALIZING);
            }
        }
        moveToState(f, nextState, f.getNextTransition(), f.getNextTransitionStyle(), false);

        if (f.mView != null) {
            // Move the view if it is out of order
            Fragment underFragment = findFragmentUnder(f);
            if (underFragment != null) {
                final View underView = underFragment.mView;
                // make sure this fragment is in the right order.
                final ViewGroup container = f.mContainer;
                int underIndex = container.indexOfChild(underView);
                int viewIndex = container.indexOfChild(f.mView);
                if (viewIndex < underIndex) {
                    container.removeViewAt(viewIndex);
                    container.addView(f.mView, underIndex);
                }
            }
            if (f.mIsNewlyAdded && f.mContainer != null) {
                // Make it visible and run the animations
                if (f.mPostponedAlpha > 0f) {
                    f.mView.setAlpha(f.mPostponedAlpha);
                }
                f.mPostponedAlpha = 0f;
                f.mIsNewlyAdded = false;
                // run animations:
                AnimationOrAnimator anim = loadAnimation(f, f.getNextTransition(), true,
                        f.getNextTransitionStyle());
                if (anim != null) {
                    setHWLayerAnimListenerIfAlpha(f.mView, anim);
                    if (anim.animation != null) {
                        f.mView.startAnimation(anim.animation);
                    } else {
                        anim.animator.setTarget(f.mView);
                        anim.animator.start();
                    }
                }
            }
        }
        if (f.mHiddenChanged) {
            completeShowHideFragment(f);
        }
    }
```