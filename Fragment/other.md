```java
/**
 * Set a currently active fragment in this FragmentManager as the primary navigation fragment.
 *
 * <p>The primary navigation fragment's
 * {@link Fragment#getChildFragmentManager() child FragmentManager} will be called first
 * to process delegated navigation actions such as {@link FragmentManager#popBackStack()}
 * if no ID or transaction name is provided to pop to. Navigation operations outside of the
 * fragment system may choose to delegate those actions to the primary navigation fragment
 * as returned by {@link FragmentManager#getPrimaryNavigationFragment()}.</p>
 *
 * <p>The fragment provided must currently be added to the FragmentManager to be set as
 * a primary navigation fragment, or previously added as part of this transaction.</p>
 *
 * @param fragment the fragment to set as the primary navigation fragment
 * @return the same FragmentTransaction instance
 */
    public abstract FragmentTransaction setPrimaryNavigationFragment(Fragment fragment);
```

1. 在当前这个FragmentManager中设置一个运行Fragment作为主要路由Fragment,
2. 如果在调用`popBackStack()`方法（没有指定ID和transactionName），这个Fragment的childFragmentManager将会代理执行这个返回操作
3. 所以当我们在在使用

在FragmentManager中有一个内部类来专门负责PopStackBack

> 27.1.1的support

```java
private class PopBackStackState implements OpGenerator {
        final String mName;
        final int mId;
        final int mFlags;

        PopBackStackState(String name, int id, int flags) {
            mName = name;
            mId = id;
            mFlags = flags;
        }

        @Override
        public boolean generateOps(ArrayList<BackStackRecord> records,
                ArrayList<Boolean> isRecordPop) {
            if (mPrimaryNav != null // We have a primary nav fragment
                    && mId < 0 // No valid id (since they're local)
                    && mName == null) { // no name to pop to (since they're local)
                final FragmentManager childManager = mPrimaryNav.peekChildFragmentManager();
                if (childManager != null && childManager.popBackStackImmediate()) {
                    // We didn't add any operations for this FragmentManager even though
                    // a child did do work.
                    return false;
                }
            }
            return popBackStackState(records, isRecordPop, mName, mId, mFlags);
        }
    }
```

> 25.4.0的support

```java
private class PopBackStackState implements OpGenerator {
    final String mName;
    final int mId;
    final int mFlags;

    PopBackStackState(String name, int id, int flags) {
        mName = name;
        mId = id;
        mFlags = flags;
    }

    @Override
    public boolean generateOps(ArrayList<BackStackRecord> records,
            ArrayList<Boolean> isRecordPop) {
        return popBackStackState(records, isRecordPop, mName, mId, mFlags);
    }
}
```

可以写Demo来分析 设置setPrimartNavigation和不设置的区别