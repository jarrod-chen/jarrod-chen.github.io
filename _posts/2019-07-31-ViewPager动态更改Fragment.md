---
layout:     post
title:       从源码解析ViewPager动态更改Fragment的实现
subtitle:   
date:       2019-07-31
author:     Jarrod
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - Android 
    - ViewPager
---



# 从源码解析ViewPager动态更改Fragment的实现

### 需求背景（What）

项目中有个需求的实现，详情页中有两个Tab（概览、数据详情），概览页根据业务类型不同，显示不同的UI。并且详情可修该业务类型，并且动态更换掉概览页面。抽象出来就是ViewPager中包含AFragment、BFragment，当业务类型在ViewPager显示时被更改，需要把AFragment替换成CFragment。

### 问题(Why)

原本可供ViewPager使用的Adapter有FragmentPagerAdapter与FragmentStatePagerAdapter，前者和后者的区别是前者类内的每一个生成的 Fragment 都将保存在内存之中，后者只保留当前页面，当页面离开视线后，就会被消除，释放其资源。而我们系统在销毁前，会把Fragment的Bundle在我们的onSaveInstanceState(Bundle)保存下来；而在页面需要显示时，Fragment就会根据我们的savedInstanceState生成新的页面。

所以跟据以上分析，大部分场景FragmentPagerAdapter已经可以满足我们的需求了，满心欢喜的使用该Adapter，在需要动态改变的地方调用notificationDataChange()方法，期待结果就像ListView、RecyclerView一样，刷新出来新的UI。

但是......依然显示AFragment，纹丝不动。

### 解决方法（how）

没办法找资料，看FragmentPagerAdapter源码

```java
@NonNull
public Object instantiateItem(@NonNull ViewGroup container, int position) {
    if (this.mCurTransaction == null) {
        this.mCurTransaction = this.mFragmentManager.beginTransaction();
    }
	//获取itemId 默认是fragment的position
    long itemId = this.getItemId(position);
    String name = makeFragmentName(container.getId(), itemId);
    //通过tag获取Fragment
    Fragment fragment = this.mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
        this.mCurTransaction.attach(fragment);
    } else {
        fragment = this.getItem(position);
        //添加Fragment，并标记tag = "android:switcher:" + viewId + ":" + id
        this.mCurTransaction.add(container.getId(), fragment, 
                                 makeFragmentName(container.getId(), itemId));
    }

    if (fragment != this.mCurrentPrimaryItem) {
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
    }

    return fragment;
}
//省略部分代码......
public long getItemId(int position) {
    return (long)position;
}

private static String makeFragmentName(int viewId, long id) {
    return "android:switcher:" + viewId + ":" + id;
}
```

通过阅读这部分源码可以知道，FragmentPagerAdapter默认提供的getItemId是position，因此就算调用notificationDataChange()，第一个Fragment改成CFragment但itemId依旧是0，然后通过tag获取到的Fragment依旧还是AFragment，所以我们在自己实现FragmentPagerAdapter时需要实现getItemId方法，保证每个Fragment的itemid不重复，我用的是:

```
@override
public long getItemId(int position) {
    return mFragmentList.get(position).hashCode();
}
```

改变此处应该可以了吧，跑了一遍，依然纹丝不动......慌了，这怎么回事

定定神，接着看源码。instantiateItem是什么时候调用的呢？它是抽象类PagerAdapter 中定义的，PagerAdapter文件头注视中有段话：

```
PagerAdapter supports data set changes. Data set changes must occur on the
main thread and must end with a call to {@link #notifyDataSetChanged()} similar
to AdapterView adapters derived from {@link android.widget.BaseAdapter}. A data
set change may involve pages being added, removed, or changing position. The
ViewPager will keep the current page active provided the adapter implements the method {@link #getItemPosition(Object)}.
```

再看看getItemPosition方法

```java
   /**
    * Called when the host view is attempting to determine if an item's position
    * has changed. Returns {@link #POSITION_UNCHANGED} if the position of the given
    * item has not changed or {@link #POSITION_NONE} if the item is no longer present
    * in the adapter.
    *
    * <p>The default implementation assumes that items will never
    * change position and always returns {@link #POSITION_UNCHANGED}.
    *
    * @param object Object representing an item, previously returned by a call to
    *               {@link #instantiateItem(android.view.View, int)}.
    * @return object's new position index from [0, {@link #getCount()}),
    *         {@link #POSITION_UNCHANGED} if the object's position has not changed,
    *         or {@link #POSITION_NONE} if the item is no longer present.
    */
    public int getItemPosition(Object object) {
        return POSITION_UNCHANGED;
    }
```

从注释看，原来系统默认使用POSITION_UNCHANGED表示item的位置不变化；返回object新的index值用来更新位置，这个可以用来动态增减item用；返回POSITION_NONE则表示这个item不再出现，所以对于我们目前固定两个Fragment，只替换第一个位置的场景需要使用POSITION_NONE。如此信心大增，只要在自己实现FragmentPagerAdapter时实现getItemPosition方法。

最终我的Adapter实现是：

```java
mViewPager.setAdapter(new FragmentPagerAdapter(getSupportFragmentManager()) {
    @Override
    public Fragment getItem(int position) {
        return mFragments.get(position);
    }

    @Override
    public int getCount() {
        return mFragments.size();
    }

    @Override
    public int getItemPosition(@NonNull Object object) {
        return POSITION_NONE;
    }

    @Override
    public long getItemId(int position) {
        return mFragments.get(position).hashCode();
    }
});
```

如此终于，需求实现了。但是，依然没有说明上面提到instantiateItem是什么时候调用的。

PagerAdapter类中，第一行代码就是

```java
 private DataSetObservable mObservable = new DataSetObservable();
```

一看就知道使用了观察者模式，这是被观察对象，也就是Adapter对应的数据集，下面找到了观察者的注册方法

```java
 //省略其他代码......
 /**
  * This method should be called by the application if the data backing this adapter has changed and associated views should update.
  */
   public void notifyDataSetChanged() {
   	   mObservable.notifyChanged();
   }

 /**
   * Register an observer to receive callbacks related to the adapter's data      changing.
   *
   * @param observer The {@link android.database.DataSetObserver} which will receive callbacks.
   */
   public void registerDataSetObserver(DataSetObserver observer) {
       mObservable.registerObserver(observer);
   }

   /**
     * Unregister an observer from callbacks related to the adapter's data changing.
     *
     * @param observer The {@link android.database.DataSetObserver} which will be unregistered.
     */
    public void unregisterDataSetObserver(DataSetObserver observer) {
        mObservable.unregisterObserver(observer);
    }
//省略其他代码......
```

PagerAdapter的实现类是供ViewPager调用的，所以再看ViewPager代码，果然在其中是有PagerAdapter，以下是ViewPager部分代码，觉得太过冗长可以跳过看结论。

```java
//省略其他代码......
private PagerAdapter mAdapter;

//省略其他代码......
/**
 * Set a PagerAdapter that will supply views for this pager as needed.
 *
 * @param adapter Adapter to use
 */
public void setAdapter(PagerAdapter adapter) {
    if (mAdapter != null) {
        //解除之前的监听
        mAdapter.unregisterDataSetObserver(mObserver);
        mAdapter.startUpdate(this);
        for (int i = 0; i < mItems.size(); i++) {
            final ItemInfo ii = mItems.get(i);
            mAdapter.destroyItem(this, ii.position, ii.object);
        }
        mAdapter.finishUpdate(this);
        mItems.clear();
        removeNonDecorViews();
        mCurItem = 0;
        scrollTo(0, 0);
    }

    final PagerAdapter oldAdapter = mAdapter;
    mAdapter = adapter;
    mExpectedAdapterCount = 0;

    if (mAdapter != null) {
        if (mObserver == null) {
            mObserver = new PagerObserver();
        }
        //重新注册监听
        mAdapter.registerDataSetObserver(mObserver);
        mPopulatePending = false;
        final boolean wasFirstLayout = mFirstLayout;
        mFirstLayout = true;
        mExpectedAdapterCount = mAdapter.getCount();
        if (mRestoredCurItem >= 0) {
            mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
            setCurrentItemInternal(mRestoredCurItem, false, true);
            mRestoredCurItem = -1;
            mRestoredAdapterState = null;
            mRestoredClassLoader = null;
        } else if (!wasFirstLayout) {
            populate();
        } else {
            requestLayout();
        }
    }

    if (mAdapterChangeListener != null && oldAdapter != adapter) {
        mAdapterChangeListener.onAdapterChanged(oldAdapter, adapter);
    }
}
//省略其他代码......
//内部类
private class PagerObserver extends DataSetObserver {
    @Override
    public void onChanged() {
        dataSetChanged();
    }
    @Override
    public void onInvalidated() {
        dataSetChanged();
    }
}
//省略其他代码......
void dataSetChanged() {
    // This method only gets called if our observer is attached, so mAdapter is non-null.
    final int adapterCount = mAdapter.getCount();
    mExpectedAdapterCount = adapterCount;
    boolean needPopulate = mItems.size() < mOffscreenPageLimit * 2 + 1 &&
            mItems.size() < adapterCount;
    int newCurrItem = mCurItem;
    boolean isUpdating = false;
    for (int i = 0; i < mItems.size(); i++) {
        final ItemInfo ii = mItems.get(i);
        //获取当前新数据item的position
        final int newPos = mAdapter.getItemPosition(ii.object);
        //这是关键！！！
        if (newPos == PagerAdapter.POSITION_UNCHANGED) {
            continue;
        }
         //这是关键！！！
        if (newPos == PagerAdapter.POSITION_NONE) {
            mItems.remove(i);
            i--;
            if (!isUpdating) {
                mAdapter.startUpdate(this);
                isUpdating = true;
            }
            mAdapter.destroyItem(this, ii.position, ii.object);
            needPopulate = true;
            if (mCurItem == ii.position) {
                // Keep the current item in the valid range
                newCurrItem = Math.max(0, Math.min(mCurItem, adapterCount - 1));
                needPopulate = true;
            }
            continue;
        }
        if (ii.position != newPos) {
            if (ii.position == mCurItem) {
                // Our current item changed position. Follow it.
                newCurrItem = newPos;
            }
            ii.position = newPos;
            needPopulate = true;
        }
    }
   //省略其他代码......
｝
 
```

综上，当我们调用FragmentPagerAdapter的notifyDataSetChanged方法时，PagerAdapter的notifyDataSetChanged也会执行，由于观察者模式，它也会通知观察者ViewPager中的PagerObserver，继续调用ViewPager的dataSetChanged方法。这个方法中，当mAdapter.getItemPosition(object)拿到的position是POSITION_UNCHANGED时，什么都没变化；是POSITION_NONE才会移除当前位置旧的item对象，换上新的；其它不等于position的（增加、删除item）也会处理。

### 总结

我们通过源码FragmentPagerAdapter、PagerAdapter、ViewPager终于弄清楚了更新ViewPager的方法，以及为什么之前我们不能成功更新页面的原因。当我们需要实现这样ViewPager中Fragment数量不变，改变实体对象的时候，我们需要自己重新覆盖FragmentPagerAdapter的getItemPosition和getItemId两个方法。