## Fragment之间切换

这样的代码一定很熟悉；

```java
public static void  replaceSupportFragment(AppCompatActivity activity, int containerId, Fragment fragment, String tag, boolean addToBackStack, boolean haveAnim) {
        if (activity == null) {
            return;
        }

        FragmentTransaction transaction = activity.getSupportFragmentManager().beginTransaction();
        if (addToBackStack) {
            transaction.addToBackStack(null);
        }
        if (haveAnim) {
            transaction.setCustomAnimations(R.anim.backstack_push_enter, R.anim.backstack_push_exit, R.anim.backstack_pop_enter, R.anim.backstack_pop_exit);
        }
        transaction.replace(containerId, fragment);
        transaction.commitAllowingStateLoss();
    }

public static void addSupportFragment(AppCompatActivity activity, int container, Fragment fragment, String tag, boolean addToBackStack, boolean haveAnim) {
        if (activity == null) {
            return;
        }
        FragmentTransaction transaction = activity.getSupportFragmentManager().beginTransaction();
        if (addToBackStack) {
            transaction.addToBackStack(null);
        }
        if (haveAnim) {
            transaction.setCustomAnimations(R.anim.backstack_push_enter, R.anim.backstack_push_exit, R.anim.backstack_pop_enter, R.anim.backstack_pop_exit);
        }
        transaction.add(container, fragment, tag);
        transaction.commitAllowingStateLoss();
    }
```

我们在使用Fragment在Activity中切换的时候，一般会使用这样的方式`replace`或者（`add`+`remove`+`hide`+`show`)这两种方式；关于这两种方式的差异我将在最后的总结中论述；

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

很明显，我们在事物中的每一次操作，包括replace、add、hide、remove都会创建一个Op的是咧，并且将创建的所有Op实例添加到一ArrayList集合中.讲到这里我们在猜想一下**将一次事物中涉及的多有操作放在一个List中，应该提交之后，来一个for循环，将这个事物的所有操作一一执行，是这样吗？**

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

如果allowStateLoss为false,则调用checkStateLoss

```java
private void checkStateLoss() {
        if (mStateSaved) {
            throw new IllegalStateException(
                    "Can not perform this action after onSaveInstanceState");
        }
        if (mNoTransactionsBecause != null) {
            throw new IllegalStateException(
                    "Can not perform this action inside of " + mNoTransactionsBecause);
        }
    }
```



























































































