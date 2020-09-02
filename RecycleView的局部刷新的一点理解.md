### RecyclerView的局部刷新的一点理解
- 两种方式
	- 局部整体刷新：notifyItemChanged(int position)
	- 局部单项刷新：notifyItemChanged(int position, @Nullable Object payload)

- 为什么“单项刷新”的效率更高？一句话说明
	- detachView的效率更高于removeView

- 几个关键流程
	- 真正布局：dispatchLayoutStep2 (RecyclerView)
	- 布局实现：onLayoutChildren (LinearLayoutManager)
		- 寻找锚点
		- 将所有View detach并回收：detachAndScrapAttachedViews
		- 重新attach view，并铺满屏幕：fill->layoutChunk->next
	- 决定view解除关系，是detach，还是remove：detachAndScrapAttachedViews
	- 上述决定的关键：viewHolder.isInvalid()
		- 局部刷新：isInvalid == false
		- 整体刷新：sInvalid == true
	- 找到isInvalid置位的地方：
		- RecyclerViewDataObserver中只有“onChanged” 中的processDataSetCompletelyChanged、markKnownViewsInvalid会置位valid
		- 其他“onItemRangeXXXX”均无该操作

- 为什么“detachView的效率更高于removeView”？
	-  detach只是简单将view从数组中移除，见"removeFromArray"
	-  removeView中会调用requestLayout()、invalidate进行重绘
