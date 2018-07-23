## Fragment之间切换





那么我们现在开始分析：

1. `getSupportFragmentManager().beginTransaction()`

   ```java
   在Activity中调用getSupportFragmentManager
   public FragmentManager getSupportFragmentManager() {
           return mFragments.getSupportFragmentManager();
       }
   ```

   我们在**Fragment的生命周期**一文了解到，Activity中的mFragment就是FragmentController,用来控制Fragment的生命周期的；在FragmentController中通过mHost获取了FragmentManagerImpl

   ```java
   	/**
        * Returns a {@link FragmentManager} for this controller.
        */
       public FragmentManager getSupportFragmentManager() {
           return mHost.getFragmentManagerImpl();
       }
   ```

   那么我们有疑问了，这个mHost又是一个什么东西：

   ```java
   /**
    * Integration points with the Fragment host.
    * <p>
    * Fragments may be hosted by any object; such as an {@link Activity}. In order to
    * host fragments, implement {@link FragmentHostCallback}, overriding the methods
    * applicable to the host.
    */
   public abstract class FragmentHostCallback<E> extends FragmentContainer {
       private final Activity mActivity;
       final Context mContext;
       private final Handler mHandler;
       final int mWindowAnimations;
       final FragmentManagerImpl mFragmentManager = new FragmentManagerImpl();
       ...
   }
   ```

   通过类的声明文件我们知道，因为Fragment是可以依附于任意的对象的，那么具体依附的对象要实现管理Fragment的功能，就必须实现这个抽象类；我们在FragmentAcitivity中找到了实现类；

   ```java
   public class FragmentActivity extends BaseFragmentActivityApi16 implements
           ViewModelStoreOwner, 
           ActivityCompat.OnRequestPermissionsResultCallback,
           ActivityCompat.RequestPermissionsRequestCodeValidator {
           ...
           class HostCallbacks extends FragmentHostCallback<FragmentActivity> {
           ...
           }
           ...
   }
   //并且在创建FragmentContriller同时创建FragmentHostCallback
   FragmentController mFragments = FragmentController.createController(new HostCallbacks());
   ```

写到这里我们思考一个问题：
**FragmentContrller是管理整个Fragment的生命周期的，而FragmentHostCallback是存储Fragment宿主信息的，FragmentController为什么要持有FragmenHostCallback的引用，他要干什么？**

在FragementController中我们看到，当Activity调用FragmentContriller中的生命周期的时候，其实是执行FragmentManager对应的方法，而FragmentManager具体实现是通过FragmentManagerImpl，而创建对应的实例是在FragmentHostCallback中创建的。那么

```java
public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);
}
```

通过在FragmentManagerImpl中beginTransaction,实际就是创建了一个BackstackRecord

```java
/**
 * Entry of an operation on the fragment back stack.
 */
final class BackStackRecord extends FragmentTransaction implements
        FragmentManager.BackStackEntry, FragmentManagerImpl.OpGenerator {
    static final String TAG = FragmentManagerImpl.TAG;

    final FragmentManagerImpl mManager;

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
    ...
}
```

对BackStackRecord的描述很简单，想表达的意思就是，BackStackRecord就是一次事物的载体，包含了这次事物的所有操作，这也符合Java面向对象的思想；

```java
final class BackStackRecord extends FragmentTransaction implements
        FragmentManager.BackStackEntry, FragmentManagerImpl.OpGenerator {
    ...
	@Override
    public FragmentTransaction remove(Fragment fragment) {
        addOp(new Op(OP_REMOVE, fragment));
        return this;
    }
    @Override
    public FragmentTransaction hide(Fragment fragment) {
        addOp(new Op(OP_HIDE, fragment));
        return this;
    }
    @Override
    public FragmentTransaction show(Fragment fragment) {
        addOp(new Op(OP_SHOW, fragment));
        return this;
    }
	...
}

```

任何的操作最后都会创建一个Op对象并且调用addOp()这个方法；

```java
static final class Op {
        int cmd;
        Fragment fragment;
        int enterAnim;
        int exitAnim;
        int popEnterAnim;
        int popExitAnim;

        Op() {
        }

        Op(int cmd, Fragment fragment) {
            this.cmd = cmd;
            this.fragment = fragment;
        }
    }

ArrayList<Op> mOps = new ArrayList<>();

void addOp(Op op) {
        mOps.add(op);
        op.enterAnim = mEnterAnim;
        op.exitAnim = mExitAnim;
        op.popEnterAnim = mPopEnterAnim;
        op.popExitAnim = mPopExitAnim;
}
```

很明显，我们在事物中的每一次操作，包括replace、add、hide、remove都会创建一个Op的实例，并且将创建的所有Op实例添加到一ArrayList集合中.讲到这里我们在猜想一下**将一次事物中涉及的多有操作放在一个List中，应该提交之后，来一个for循环，将这个事物的所有操作一一执行，是这样吗？**

```java
@Override
    public FragmentTransaction addToBackStack(String name) {
        if (!mAllowAddToBackStack) {
            throw new IllegalStateException(
                    "This FragmentTransaction is not allowed to be added to the back stack.");
        }
        mAddToBackStack = true;
        mName = name;
        return this;
    }
```

加入返回栈，就是将当前的事物的一个Flag置为true.

```java
    @Override
    public int commit() {
        return commitInternal(false);
    }

    @Override
    public int commitAllowingStateLoss() {
        return commitInternal(true);
    }
```

创建完成之后，就要提交了，我们一般常用的就是这两种方式。这两者的差别就是参数不一样；

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

重复提交会报错，如果这个Fragment添加到返回栈中，那么会申请一个索引返回，最后调用FragmentManagerImpl中的enqueueActon。

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

`mPendingAction`是保存每一次操作（包括BackStackRecord和PopBackStackState），如果allowStateLoss为false,则调用checkStateLoss；

```java
private void checkStateLoss() {
        if (isStateSaved()) {
            throw new IllegalStateException(
                    "Can not perform this action after onSaveInstanceState");
        }
        if (mNoTransactionsBecause != null) {
            throw new IllegalStateException(
                    "Can not perform this action inside of " + mNoTransactionsBecause);
        }
    }
```

在`checkStateLoss`中我们可以看到，顾名思义，会检查是否允许状态丢失之后提交事务，我们之后，在Fragment崩溃（crash）的时候，会走`onSaveInstanceState()`这个生命周期，来保存页面数据，如果alllowStateLoss为true，是不关系是否调用了`onSaveInstanceState()`的，允许新提交一个事务；

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

在上面的代码中我们可以看到，执行`Runnable`的前提是，有延时执行的事务或者，`mPendingActions`的大小只能为1，当满足上述两种情况中的一种，则执行这个`Runnable`，这里我看源码的时候有一个疑问，既然mPengdingActions是在`enqueueActions()`方式中，添加元素的，那么肯定不可能只有add操作没有remove操作，带着疑问我们继续往下看。

```java
Runnable mExecCommit = new Runnable() {
        @Override
        public void run() {
            execPendingActions();
        }
    };

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

这个时候我就想到了`RxJava`，上面这中代码风格，一个方法里有5个方法，你是不知道这个方法的下一步是要干啥，如果用RxJava的话，会清晰很多。我们先沿着主干继续往下看，主要的方法就是`removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop)`

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
//优化操作，移除重复的操作，并执行操作
private void removeRedundantOperationsAndExecute(ArrayList<BackStackRecord> records,
                                                 ArrayList<Boolean> isRecordPop) {
    if (records == null || records.isEmpty()) {
        return;
    }

    if (isRecordPop == null || records.size() != isRecordPop.size()) {
        //因为isRecordPop和records是一一对应的，操作records也会操作isRecordPop，用来标记，这个Record有没有出栈，我那会也有考虑过，难道不能使用个Bean类,将这两个List合并一个
        throw new IllegalStateException("Internal error with the back stack records");
    }

    // Force start of any postponed transactions that interact with scheduled transactions:
    executePostponedTransaction(records, isRecordPop);

    final int numRecords = records.size();
    int startIndex = 0;
    for (int recordNum = 0; recordNum < numRecords; recordNum++) 
        //mReorderingAllowed:是否允许重新排序，默认是false
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

这里提到了一个特别重要的方法`executeOpsTogether(records, isRecordPop, recordNum, reorderingEnd);`

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

```java
/**
     * Expands all meta-ops into their more primitive equivalents. This must be called prior to
     * {@link #executeOps()} or any other call that operations on mOps for forward navigation.
     * It should not be called for pop/reverse navigation operations.
     *
     * <p>Removes all OP_REPLACE ops and replaces them with the proper add and remove
     * operations that are equivalent to the replace.</p>
     *
     * <p>Adds OP_UNSET_PRIMARY_NAV ops to match OP_SET_PRIMARY_NAV, OP_REMOVE and OP_DETACH
     * ops so that we can restore the old primary nav fragment later. Since callers call this
     * method in a loop before running ops from several transactions at once, the caller should
     * pass the return value from this method as the oldPrimaryNav parameter for the next call.
     * The first call in such a loop should pass the value of
     * {@link FragmentManager#getPrimaryNavigationFragment()}.</p>
     *
     * @param added Initialized to the fragments that are in the mManager.mAdded, this
     *              will be modified to contain the fragments that will be in mAdded
     *              after the execution ({@link #executeOps()}.
     * @param oldPrimaryNav The tracked primary navigation fragment as of the beginning of
     *                      this set of ops
     * @return the new oldPrimaryNav fragment after this record's ops would be run
     */
    @SuppressWarnings("ReferenceEquality")
    Fragment expandOps(ArrayList<Fragment> added, Fragment oldPrimaryNav) {
        for (int opNum = 0; opNum < mOps.size(); opNum++) {
            final Op op = mOps.get(opNum);
            switch (op.cmd) {
                case OP_ADD:
                case OP_ATTACH:
                    added.add(op.fragment);
                    break;
                case OP_REMOVE:
                case OP_DETACH: {
                    added.remove(op.fragment);
                    if (op.fragment == oldPrimaryNav) {
                        mOps.add(opNum, new Op(OP_UNSET_PRIMARY_NAV, op.fragment));
                        opNum++;
                        oldPrimaryNav = null;
                    }
                }
                break;
                case OP_REPLACE: {
                    final Fragment f = op.fragment;
                    final int containerId = f.mContainerId;
                    boolean alreadyAdded = false;
                    for (int i = added.size() - 1; i >= 0; i--) {
                        final Fragment old = added.get(i);
                        if (old.mContainerId == containerId) {
                            if (old == f) {
                                alreadyAdded = true;
                            } else {
                                // This is duplicated from above since we only make
                                // a single pass for expanding ops. Unset any outgoing primary nav.
                                if (old == oldPrimaryNav) {
                                    mOps.add(opNum, new Op(OP_UNSET_PRIMARY_NAV, old));
                                    opNum++;
                                    oldPrimaryNav = null;
                                }
                                final Op removeOp = new Op(OP_REMOVE, old);
                                removeOp.enterAnim = op.enterAnim;
                                removeOp.popEnterAnim = op.popEnterAnim;
                                removeOp.exitAnim = op.exitAnim;
                                removeOp.popExitAnim = op.popExitAnim;
                                mOps.add(opNum, removeOp);
                                added.remove(old);
                                opNum++;
                            }
                        }
                    }
                    if (alreadyAdded) {
                        mOps.remove(opNum);
                        opNum--;
                    } else {
                        op.cmd = OP_ADD;
                        added.add(f);
                    }
                }
                break;
                case OP_SET_PRIMARY_NAV: {
                    // It's ok if this is null, that means we will restore to no active
                    // primary navigation fragment on a pop.
                    mOps.add(opNum, new Op(OP_UNSET_PRIMARY_NAV, oldPrimaryNav));
                    opNum++;
                    // Will be set by the OP_SET_PRIMARY_NAV we inserted before when run
                    oldPrimaryNav = op.fragment;
                }
                break;
            }
        }
        return oldPrimaryNav;
    }
```



这个`oldPrimaryNav`困扰了好久

其中最核心的方法是

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

这个方法针对已经出栈和进栈的Fragment都做里处理。这里我们主要先分析`record.executeOps();`

```java
/**
 * Executes the operations contained within this transaction. The Fragment states will only
 * be modified if optimizations are not allowed.
 */
void executeOps() {
    final int numOps = mOps.size();
    for (int opNum = 0; opNum < numOps; opNum++) {
        final Op op = mOps.get(opNum);
        final Fragment f = op.fragment;
        if (f != null) {
            f.setNextTransition(mTransition, mTransitionStyle);
        }
        switch (op.cmd) {
            case OP_ADD:
                f.setNextAnim(op.enterAnim);
                mManager.addFragment(f, false);
                break;
            case OP_REMOVE:
                f.setNextAnim(op.exitAnim);
                mManager.removeFragment(f);
                break;
            case OP_HIDE:
                f.setNextAnim(op.exitAnim);
                mManager.hideFragment(f);
                break;
            case OP_SHOW:
                f.setNextAnim(op.enterAnim);
                mManager.showFragment(f);
                break;
            case OP_DETACH:
                f.setNextAnim(op.exitAnim);
                mManager.detachFragment(f);
                break;
            case OP_ATTACH:
                f.setNextAnim(op.enterAnim);
                mManager.attachFragment(f);
                break;
            case OP_SET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(f);
                break;
            case OP_UNSET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(null);
                break;
            default:
                throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
        }
        if (!mReorderingAllowed && op.cmd != OP_ADD && f != null) {
            mManager.moveFragmentToExpectedState(f);
        }
    }
    if (!mReorderingAllowed) {
        // Added fragments are added at the end to comply with prior behavior.
        mManager.moveToState(mManager.mCurState, true);
    }
}
```

接下来我们看`moveFragmentToExpectedState()`

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

我们继续看`moveToState()`

```java
void moveToState(Fragment f, int newState, int transit, int transitionStyle,
            boolean keepActive) {
        // Fragments that are not currently added will sit in the onCreate() state.
        if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
            newState = Fragment.CREATED;
        }
        if (f.mRemoving && newState > f.mState) {
            if (f.mState == Fragment.INITIALIZING && f.isInBackStack()) {
                // Allow the fragment to be created so that it can be saved later.
                newState = Fragment.CREATED;
            } else {
                // While removing a fragment, we can't change it to a higher state.
                newState = f.mState;
            }
        }
        // Defer start if requested; don't allow it to move to STARTED or higher
        // if it's not already started.
        if (f.mDeferStart && f.mState < Fragment.STARTED && newState > Fragment.STOPPED) {
            newState = Fragment.STOPPED;
        }
        if (f.mState <= newState) {
            // For fragments that are created from a layout, when restoring from
            // state we don't want to allow them to be created until they are
            // being reloaded from the layout.
            if (f.mFromLayout && !f.mInLayout) {
                return;
            }
            if (f.getAnimatingAway() != null || f.getAnimator() != null) {
                // The fragment is currently being animated...  but!  Now we
                // want to move our state back up.  Give up on waiting for the
                // animation, move to whatever the final state should be once
                // the animation is done, and then we can proceed from there.
                f.setAnimatingAway(null);
                f.setAnimator(null);
                moveToState(f, f.getStateAfterAnimating(), 0, 0, true);
            }
            switch (f.mState) {
                case Fragment.INITIALIZING:
                    if (newState > Fragment.INITIALIZING) {
                        if (DEBUG) Log.v(TAG, "moveto CREATED: " + f);
                        if (f.mSavedFragmentState != null) {
                            f.mSavedFragmentState.setClassLoader(mHost.getContext()
                                    .getClassLoader());
                            f.mSavedViewState = f.mSavedFragmentState.getSparseParcelableArray(
                                    FragmentManagerImpl.VIEW_STATE_TAG);
                            f.mTarget = getFragment(f.mSavedFragmentState,
                                    FragmentManagerImpl.TARGET_STATE_TAG);
                            if (f.mTarget != null) {
                                f.mTargetRequestCode = f.mSavedFragmentState.getInt(
                                        FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG, 0);
                            }
                            if (f.mSavedUserVisibleHint != null) {
                                f.mUserVisibleHint = f.mSavedUserVisibleHint;
                                f.mSavedUserVisibleHint = null;
                            } else {
                                f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
                                        FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
                            }
                            if (!f.mUserVisibleHint) {
                                f.mDeferStart = true;
                                if (newState > Fragment.STOPPED) {
                                    newState = Fragment.STOPPED;
                                }
                            }
                        }

                        f.mHost = mHost;
                        f.mParentFragment = mParent;
                        f.mFragmentManager = mParent != null
                                ? mParent.mChildFragmentManager : mHost.getFragmentManagerImpl();

                        // If we have a target fragment, push it along to at least CREATED
                        // so that this one can rely on it as an initialized dependency.
                        if (f.mTarget != null) {
                            if (mActive.get(f.mTarget.mIndex) != f.mTarget) {
                                throw new IllegalStateException("Fragment " + f
                                        + " declared target fragment " + f.mTarget
                                        + " that does not belong to this FragmentManager!");
                            }
                            if (f.mTarget.mState < Fragment.CREATED) {
                                moveToState(f.mTarget, Fragment.CREATED, 0, 0, true);
                            }
                        }

                        dispatchOnFragmentPreAttached(f, mHost.getContext(), false);
                        f.mCalled = false;
                        f.onAttach(mHost.getContext());
                        if (!f.mCalled) {
                            throw new SuperNotCalledException("Fragment " + f
                                    + " did not call through to super.onAttach()");
                        }
                        if (f.mParentFragment == null) {
                            mHost.onAttachFragment(f);
                        } else {
                            f.mParentFragment.onAttachFragment(f);
                        }
                        dispatchOnFragmentAttached(f, mHost.getContext(), false);

                        if (!f.mIsCreated) {
                            dispatchOnFragmentPreCreated(f, f.mSavedFragmentState, false);
                            f.performCreate(f.mSavedFragmentState);
                            dispatchOnFragmentCreated(f, f.mSavedFragmentState, false);
                        } else {
                            f.restoreChildFragmentState(f.mSavedFragmentState);
                            f.mState = Fragment.CREATED;
                        }
                        f.mRetaining = false;
                    }
                    // fall through
                case Fragment.CREATED:
                    // This is outside the if statement below on purpose; we want this to run
                    // even if we do a moveToState from CREATED => *, CREATED => CREATED, and
                    // * => CREATED as part of the case fallthrough above.
                    ensureInflatedFragmentView(f);

                    if (newState > Fragment.CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
                        if (!f.mFromLayout) {
                            ViewGroup container = null;
                            if (f.mContainerId != 0) {
                                if (f.mContainerId == View.NO_ID) {
                                    throwException(new IllegalArgumentException(
                                            "Cannot create fragment "
                                                    + f
                                                    + " for a container view with no id"));
                                }
                                container = (ViewGroup) mContainer.onFindViewById(f.mContainerId);
                                if (container == null && !f.mRestored) {
                                    String resName;
                                    try {
                                        resName = f.getResources().getResourceName(f.mContainerId);
                                    } catch (NotFoundException e) {
                                        resName = "unknown";
                                    }
                                    throwException(new IllegalArgumentException(
                                            "No view found for id 0x"
                                            + Integer.toHexString(f.mContainerId) + " ("
                                            + resName
                                            + ") for fragment " + f));
                                }
                            }
                            f.mContainer = container;
                            f.mView = f.performCreateView(f.performGetLayoutInflater(
                                    f.mSavedFragmentState), container, f.mSavedFragmentState);
                            if (f.mView != null) {
                                f.mInnerView = f.mView;
                                f.mView.setSaveFromParentEnabled(false);
                                if (container != null) {
                                    container.addView(f.mView);
                                }
                                if (f.mHidden) {
                                    f.mView.setVisibility(View.GONE);
                                }
                                f.onViewCreated(f.mView, f.mSavedFragmentState);
                                dispatchOnFragmentViewCreated(f, f.mView, f.mSavedFragmentState,
                                        false);
                                // Only animate the view if it is visible. This is done after
                                // dispatchOnFragmentViewCreated in case visibility is changed
                                f.mIsNewlyAdded = (f.mView.getVisibility() == View.VISIBLE)
                                        && f.mContainer != null;
                            } else {
                                f.mInnerView = null;
                            }
                        }

                        f.performActivityCreated(f.mSavedFragmentState);
                        dispatchOnFragmentActivityCreated(f, f.mSavedFragmentState, false);
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);
                        }
                        f.mSavedFragmentState = null;
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState > Fragment.ACTIVITY_CREATED) {
                        f.mState = Fragment.STOPPED;
                    }
                    // fall through
                case Fragment.STOPPED:
                    if (newState > Fragment.STOPPED) {
                        if (DEBUG) Log.v(TAG, "moveto STARTED: " + f);
                        f.performStart();
                        dispatchOnFragmentStarted(f, false);
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState > Fragment.STARTED) {
                        if (DEBUG) Log.v(TAG, "moveto RESUMED: " + f);
                        f.performResume();
                        dispatchOnFragmentResumed(f, false);
                        f.mSavedFragmentState = null;
                        f.mSavedViewState = null;
                    }
            }
        } else if (f.mState > newState) {
            switch (f.mState) {
                case Fragment.RESUMED:
                    if (newState < Fragment.RESUMED) {
                        if (DEBUG) Log.v(TAG, "movefrom RESUMED: " + f);
                        f.performPause();
                        dispatchOnFragmentPaused(f, false);
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState < Fragment.STARTED) {
                        if (DEBUG) Log.v(TAG, "movefrom STARTED: " + f);
                        f.performStop();
                        dispatchOnFragmentStopped(f, false);
                    }
                    // fall through
                case Fragment.STOPPED:
                    if (newState < Fragment.STOPPED) {
                        if (DEBUG) Log.v(TAG, "movefrom STOPPED: " + f);
                        f.performReallyStop();
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState < Fragment.ACTIVITY_CREATED) {
                        if (DEBUG) Log.v(TAG, "movefrom ACTIVITY_CREATED: " + f);
                        if (f.mView != null) {
                            // Need to save the current view state if not
                            // done already.
                            if (mHost.onShouldSaveFragmentState(f) && f.mSavedViewState == null) {
                                saveFragmentViewState(f);
                            }
                        }
                        f.performDestroyView();
                        dispatchOnFragmentViewDestroyed(f, false);
                        if (f.mView != null && f.mContainer != null) {
                            // Stop any current animations:
                            f.mContainer.endViewTransition(f.mView);
                            f.mView.clearAnimation();
                            AnimationOrAnimator anim = null;
                            if (mCurState > Fragment.INITIALIZING && !mDestroyed
                                    && f.mView.getVisibility() == View.VISIBLE
                                    && f.mPostponedAlpha >= 0) {
                                anim = loadAnimation(f, transit, false,
                                        transitionStyle);
                            }
                            f.mPostponedAlpha = 0;
                            if (anim != null) {
                                animateRemoveFragment(f, anim, newState);
                            }
                            f.mContainer.removeView(f.mView);
                        }
                        f.mContainer = null;
                        f.mView = null;
                        f.mInnerView = null;
                        f.mInLayout = false;
                    }
                    // fall through
                case Fragment.CREATED:
                    if (newState < Fragment.CREATED) {
                        if (mDestroyed) {
                            // The fragment's containing activity is
                            // being destroyed, but this fragment is
                            // currently animating away.  Stop the
                            // animation right now -- it is not needed,
                            // and we can't wait any more on destroying
                            // the fragment.
                            if (f.getAnimatingAway() != null) {
                                View v = f.getAnimatingAway();
                                f.setAnimatingAway(null);
                                v.clearAnimation();
                            } else if (f.getAnimator() != null) {
                                Animator animator = f.getAnimator();
                                f.setAnimator(null);
                                animator.cancel();
                            }
                        }
                        if (f.getAnimatingAway() != null || f.getAnimator() != null) {
                            // We are waiting for the fragment's view to finish
                            // animating away.  Just make a note of the state
                            // the fragment now should move to once the animation
                            // is done.
                            f.setStateAfterAnimating(newState);
                            newState = Fragment.CREATED;
                        } else {
                            if (DEBUG) Log.v(TAG, "movefrom CREATED: " + f);
                            if (!f.mRetaining) {
                                f.performDestroy();
                                dispatchOnFragmentDestroyed(f, false);
                            } else {
                                f.mState = Fragment.INITIALIZING;
                            }

                            f.performDetach();
                            dispatchOnFragmentDetached(f, false);
                            if (!keepActive) {
                                if (!f.mRetaining) {
                                    makeInactive(f);
                                } else {
                                    f.mHost = null;
                                    f.mParentFragment = null;
                                    f.mFragmentManager = null;
                                }
                            }
                        }
                    }
            }
        }

        if (f.mState != newState) {
            Log.w(TAG, "moveToState: Fragment state for " + f + " not updated inline; "
                    + "expected state " + newState + " found " + f.mState);
            f.mState = newState;
        }
    }
```











































































