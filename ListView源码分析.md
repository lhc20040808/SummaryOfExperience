# ListView源码分析

谈到ListView，大家最熟悉的套路就是要设置一个适配器，这也是安卓源码中经典的适配器模式。那我们从setAdapter方法中开始对ListView进行分析。

### 首先看一下ListView的类结构

![ListView类结构](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/源码分析/ListView类结构.png)

### 接着我们从setAdapter方法开始入手

```Java
    /**
     * Sets the data behind this ListView.
     *
     * The adapter passed to this method may be wrapped by a {@link WrapperListAdapter},
     * depending on the ListView features currently in use. For instance, adding
     * headers and/or footers will cause the adapter to be wrapped.
     *
     * @param adapter The ListAdapter which is responsible for maintaining the
     *        data backing this list and for producing a view to represent an
     *        item in that data set.
     *
     * @see #getAdapter() 
     */
    @Override
    public void setAdapter(ListAdapter adapter) {
        if (mAdapter != null && mDataSetObserver != null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver);
        }

        resetList();
        mRecycler.clear();

        if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
            mAdapter = wrapHeaderListAdapterInternal(mHeaderViewInfos, mFooterViewInfos, adapter);
        } else {
            mAdapter = adapter;
        }

        mOldSelectedPosition = INVALID_POSITION;
        mOldSelectedRowId = INVALID_ROW_ID;

        // AbsListView#setAdapter will update choice mode states.
        super.setAdapter(adapter);

        if (mAdapter != null) {
            mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();
            mOldItemCount = mItemCount;
            mItemCount = mAdapter.getCount();
            checkFocus();

            mDataSetObserver = new AdapterDataSetObserver();
            mAdapter.registerDataSetObserver(mDataSetObserver);

            mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

            int position;
            if (mStackFromBottom) {
                position = lookForSelectablePosition(mItemCount - 1, false);
            } else {
                position = lookForSelectablePosition(0, true);
            }
            setSelectedPositionInt(position);
            setNextSelectedPositionInt(position);

            if (mItemCount == 0) {
                // Nothing selected
                checkSelectionChanged();
            }
        } else {
            mAreAllItemsSelectable = true;
            checkFocus();
            // Nothing selected
            checkSelectionChanged();
        }

        requestLayout();
    }
```
由于设置的adapter会被观察者者监听，所以每次设置adapter时首先检查原先是否有adapter，如果有则解绑观察者。接着就是给mAdapter这是属性赋值，如果设置了header或footer，会调用`wrapHeaderListAdapterInternal`。这个方法会返回一个`HeaderViewListAdapter`，该类实现了`WrapperListAdapter`接口，而`WrapperListAdapter`接口继承自`ListAdapter`。这是一个典型的*装饰者模式*，有了adapter以后，进行一系列初始化工作，包括设置观察者，设置`mRecycler`需要的参数。`mStackFromBottom`这个布尔值的属性主要设置ListView的绘制方向，可以通过`setStackFromBottom`方法设置这个属性。

![适配器装饰者模式uml图](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/设计模式/decoration_mode_of_listview.png)



那么问题来了，这个`mRecycler`到底是什么。这是ListView种核心的部分之一，View的回收分发都是由其来操控，它的功能相当于一个对象池。`mRecycler`的实现是`RecycleBin`，它是`AbsListView`的一个*非静态内部类*，意味着它可以访问外部类的属性和方法。

```Java
  /**
     * The data set used to store unused views that should be reused during the next layout
     * to avoid creating new ones
     */
    final RecycleBin mRecycler = new RecycleBin();
```

```java
 /**
     * The RecycleBin facilitates reuse of views across layouts. The RecycleBin has two levels of
     * storage: ActiveViews and ScrapViews. ActiveViews are those views which were onscreen at the
     * start of a layout. By construction, they are displaying current information. At the end of
     * layout, all views in ActiveViews are demoted to ScrapViews. ScrapViews are old views that
     * could potentially be used by the adapter to avoid allocating views unnecessarily.
     *
     * @see android.widget.AbsListView#setRecyclerListener(android.widget.AbsListView.RecyclerListener)
     * @see android.widget.AbsListView.RecyclerListener
     */
    class RecycleBin {
       private RecyclerListener mRecyclerListener;

        /**
         * The position of the first view stored in mActiveViews.
         */
        private int mFirstActivePosition;

        /**
         * Views that were on screen at the start of layout. This array is populated at the start of
         * layout, and at the end of layout all view in mActiveViews are moved to mScrapViews.
         * Views in mActiveViews represent a contiguous range of Views, with position of the first
         * view store in mFirstActivePosition.
         */
        private View[] mActiveViews = new View[0];

        /**
         * Unsorted views that can be used by the adapter as a convert view.
         */
        private ArrayList<View>[] mScrapViews;

        private int mViewTypeCount;

        private ArrayList<View> mCurrentScrap;

        private ArrayList<View> mSkippedScrap;

        private SparseArray<View> mTransientStateViews;
        private LongSparseArray<View> mTransientStateViewsById;
      
      //...省略方法
    }
```

简单看一下`RecycleBin`的属性，`mScrapViews`中存在过期的view，为什么它是个存放列表的数组呢？回到`setAdapter`第44行的代码。大家可以看一下BaseAdapter源码，如果不覆盖这个方法，默认返回值为1。

```Java
 mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());
```

```Java
        public void setViewTypeCount(int viewTypeCount) {
            if (viewTypeCount < 1) {
                throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
            }
            //noinspection unchecked
            ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
            for (int i = 0; i < viewTypeCount; i++) {
                scrapViews[i] = new ArrayList<View>();
            }
            mViewTypeCount = viewTypeCount;
            mCurrentScrap = scrapViews[0];
            mScrapViews = scrapViews;
        }
```

它会根据adapter返回的View类型的数量来生成同等长度的数组来存储View。也就是说，将两种不一样样式（类型）的view存放在了不同数组中。



### 设置完adapter了，我们来看看ListView的绘制流程



#### `onMeasure`

#### `onLayout`

在ListView中是没有实现`onLayout`的，实现方法在父类`AbsListView`中。

```Java
    /**
     * Subclasses should NOT override this method but
     *  {@link #layoutChildren()} instead.
     */
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);

        mInLayout = true;

        final int childCount = getChildCount();
        if (changed) {
            for (int i = 0; i < childCount; i++) {
                getChildAt(i).forceLayout();
            }
            mRecycler.markChildrenDirty();
        }

        layoutChildren();
        mInLayout = false;

        mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;

        // TODO: Move somewhere sane. This doesn't belong in onLayout().
        if (mFastScroll != null) {
            mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
        }
    }
```

注释中也强调了，子类不应该覆盖这个方法而是实现`layoutChildren`。那么来看一下`ListView#layoutChildren`方法，这个方法非常的长。

着重看这一段，mLayoutMode默认情况下值为LAYOUT_NORMAL，`switch-case`最后进入default分支。

```java
 @Override
    protected void layoutChildren() {     

		final int childrenTop = mListPadding.top;
        final int childrenBottom = mBottom - mTop - mListPadding.bottom;
        final int childCount = getChildCount();
      
        switch (mLayoutMode) {
            //...省略部分代码
            default:
                // Remember the previously selected view
                index = mSelectedPosition - mFirstPosition;
                if (index >= 0 && index < childCount) {
                    oldSel = getChildAt(index);
                }

                // Remember the previous first child
                oldFirst = getChildAt(0);

                if (mNextSelectedPosition >= 0) {
                    delta = mNextSelectedPosition - mSelectedPosition;
                }

                // Caution: newSel might be null
                newSel = getChildAt(index + delta);
            }
      
		switch (mLayoutMode) {
                //...省略代码
            default:
                if (childCount == 0) {
                    if (!mStackFromBottom) {
                        final int position = lookForSelectablePosition(0, true);
                        setSelectedPositionInt(position);
                        sel = fillFromTop(childrenTop);
                    } else {
                        final int position = lookForSelectablePosition(mItemCount - 1, false);
                        setSelectedPositionInt(position);
                        sel = fillUp(mItemCount - 1, childrenBottom);
                    }
                } else {
                    if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
                        sel = fillSpecific(mSelectedPosition,
                                oldSel == null ? childrenTop : oldSel.getTop());
                    } else if (mFirstPosition < mItemCount) {
                        sel = fillSpecific(mFirstPosition,
                                oldFirst == null ? childrenTop : oldFirst.getTop());
                    } else {
                        sel = fillSpecific(0, childrenTop);
                    }
                }
                break;
            }
    }
```

第一次加载的时候ListView还没有子view，所以childCount为0。





推荐资料：[Android 优雅的为RecyclerView添加HeaderView和FooterView](http://blog.csdn.net/lmj623565791/article/details/51854533)

[Android ListView工作原理完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/44996879)