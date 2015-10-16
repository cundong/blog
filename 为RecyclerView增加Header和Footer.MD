# 为RecyclerView增加Header和Footer

最近在开发新产品，为了跟上Google的脚步，项目中使用到ListView的地方都换成RecyclerView了。

换完之后，遇到最紧急的一个问题就是：ListView是自带addHeaderView()、addFooterView()的，RecyclerView不知出于什么原因的考虑，没有自带这两个方法。

我们只能自己想办法去自己加上去。

## 把Header、Footer当做RecyclerView的一个ViewType

比较通用的做法是把Header、Footer当做RecyclerView的一个ViewType来处理，诸如：

```java
@Override
public int getItemViewType(int position) {
if (isHeader(position)) {
return TYPE_HEADER_VIEW;
}

if (isFooter(position)) {
return TYPE_FOOTER_VIEW;
}

int thePosition = position - (hasHeader() ? 1 : 0);
return getItemViewType2(thePosition) + TYPE_DATA_OFFSET;
}
```

原理就是，向原先的dataList中插入两个元素，一个头一个尾，然后用户去定义这个头尾的布局，这样做的坏处是会改变元素的真实位置。当你在dataList中插入了一个Header的时，相当于整个dataList中元素都后移了，因此我们想要获取dataList中的元素，必须要重新计算一下：

int thePosition = position - (hasHeader() ? 1 : 0);

为了做的通用一点，需要自己封装一个抽象类出来，继承 RecyclerView.Adapter，并且实现几个主要的方法：
onBindViewHolder()、onCreateViewHolder()、getItemCount()等。
```java
@Override
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

if (viewType == TYPE_HEADER_VIEW) {
return mHeaderViewHolder;
} else if (viewType == TYPE_FOOTER_VIEW) {
return mFooterViewHolder;
}

return onCreateViewHolder2(parent, viewType - TYPE_DATA_OFFSET);
}

@Override
public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {

int thePosition = position - (hasHeader() ? 1 : 0);

if (!isHeader(position) && !isFooter(position)) {

holder.itemView.setTag(mDataList.get(thePosition));

onBindViewHolder2((VH) holder, thePosition);
} else {
ViewGroup.LayoutParams layoutParams = holder.itemView.getLayoutParams();
if(layoutParams instanceof StaggeredGridLayoutManager.LayoutParams) {
((StaggeredGridLayoutManager.LayoutParams) layoutParams).setFullSpan(true);
}
}
}

/**
* 用于替代RecyclerView.Adapter.onCreateViewHolder()方法
*
* @param parent
* @param viewType
* @return
*/
public abstract VH onCreateViewHolder2(ViewGroup parent, int viewType);

/**
* 用于替代RecyclerView.Adapter.onBindViewHolder()方法
*
* @param holder
* @param position
*/
public abstract void onBindViewHolder2(VH holder, int position);
```
我们在自己的业务Adapter中，需要实现onCreateViewHolder2()这个抽象方法，并且用它来代替原生的onCreateViewHolder()方法。

## 自己写一个OuterAdapter，包裹真实的业务逻辑Adapter

另一种方法是自己写一个OuterAdapter，继承自RecyclerView.Adapter，把自己写的innerAdapter传进来，我们在OuterAdapter中插入头和尾，这样就不会影响用户自己写的innerAdapter中数据位置了。

并且，在OuterAdapter中重新为innerAdapter设置监听器，当数据变化时，计算插入Header、Footer之后的新位置，调用notifyXXXX系列方法来通知数据的改变。

```java
/**
* 设置adapter
* @param adapter
*/
public void setAdapter(RecyclerView.Adapter<RecyclerView.ViewHolder> adapter) {

if (adapter != null) {
if (!(adapter instanceof RecyclerView.Adapter))
throw new IllegalArgumentException("your adapter must be a RecyclerView.Adapter");
}

if (mInnerAdapter != null) {
notifyItemRangeRemoved(getHeaderViewsCount(), mInnerAdapter.getItemCount());
mInnerAdapter.unregisterAdapterDataObserver(mDataObserver);
}

this.mInnerAdapter = adapter;
mInnerAdapter.registerAdapterDataObserver(mDataObserver);
notifyItemRangeInserted(getHeaderViewsCount(), mInnerAdapter.getItemCount());
}

private RecyclerView.AdapterDataObserver mDataObserver = new RecyclerView.AdapterDataObserver() {

@Override
public void onChanged() {
super.onChanged();
notifyDataSetChanged();
}

@Override
public void onItemRangeChanged(int positionStart, int itemCount) {
super.onItemRangeChanged(positionStart, itemCount);
notifyItemRangeChanged(positionStart + getHeaderViewsCount(), itemCount);
}

@Override
public void onItemRangeInserted(int positionStart, int itemCount) {
super.onItemRangeInserted(positionStart, itemCount);
notifyItemRangeInserted(positionStart + getHeaderViewsCount(), itemCount);
}

@Override
public void onItemRangeRemoved(int positionStart, int itemCount) {
super.onItemRangeRemoved(positionStart, itemCount);
notifyItemRangeRemoved(positionStart + getHeaderViewsCount(), itemCount);
}

@Override
public void onItemRangeMoved(int fromPosition, int toPosition, int itemCount) {
super.onItemRangeMoved(fromPosition, toPosition, itemCount);
int headerViewsCountCount = getHeaderViewsCount();
notifyItemRangeChanged(fromPosition + headerViewsCountCount, toPosition + headerViewsCountCount + itemCount);
}
};
```

## 两层布局嵌套

还有一种比较巧妙的方式，如果你只需要为RecyclerView增加一个Header，不需要Footer，可以使用。

### 设置第一个元素的top padding
通过RecyclerView.ItemDecoration，将RecyclerView第一个item的top值设置的很大（Header多高就设置多大），然后在一个FrameLayout中放置RecyclerView，再把Header盖上去，这样给用户的视觉感受就是RecyclerView有了一个Header

### 滑动的时候联动
然后，让Header随着RecyclerView的滑动而滑动，并且在滑动Header的时候，带动RecyclerView的滑动

### 一些限制
如果你增加的Header中有能消费事件的控件，比如Button，EditText，则会中断事件的传递，导致滑动的时候Header和RecyclerView不联动，因此如果你想增加一个简单的HeaderView（比如只有简单的TextView展示基本信息之类），可以使用这种方式，否则，请使用以上两种方式。

我使用的是第2种方式，最简单。

##代码
代码整理完毕后会push到GitHub上。




