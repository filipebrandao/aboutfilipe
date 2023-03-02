---
title: "Get the RecyclerView scroll position - the reliable way."
date: 2023-03-01T23:00:00+00:00
draft: false
---

In case you haven't moved to Jetpack Compose yet and still rely on the `RecyclerView` to builds your list, you might face the problem of not having a reliable way of getting the scroll position of the `RecyclerView` at a given moment. **Specially** if the views feeding the `RecyclerView` have **dynamic sizes**, such as dynamic heights in a vertical list, there doesn't really seem to be a reliable mechanism in the SDK to get that scroll offset.


But I'm here to help :) 

You can workaround this problem by implementing your own `LayoutManager`. Here's an example for a `RecyclerView` which displays a vertical list of items with dynamic height:

 ```
 /**
 * Mimics LinearLayoutManager but adds reliable scroll Y offset calculation for RecyclerView list items with dynamic height.
 * The base implementation relies on the average list item height, which is unreliable, while this one stores the actual height of each item.
 */
class ReliableScrollOffsetLinearLayoutManager(context: Context?) : LinearLayoutManager(context) {

    private val childSizes = mutableMapOf<Int, Int>()

    /**
     * Useful when the RecyclerView has been sorted or new items have been added before the current scroll position
     */
    fun recalculateChildHeights() {
        childSizes.clear()
        for (i in 0 until childCount) {
            val child = getChildAt(i)!!
            childSizes[getPosition(child)] = child.height
        }
    }

    override fun onLayoutCompleted(state: RecyclerView.State?) {
        super.onLayoutCompleted(state)
        recalculateChildHeights()
    }

    override fun computeVerticalScrollOffset(state: RecyclerView.State): Int {
        if (childCount == 0) {
            return 0
        }
        val firstChildPosition = findFirstVisibleItemPosition()
        val firstChild = findViewByPosition(0) ?: return super.computeVerticalScrollOffset(state)

        var scrollOffsetY: Int = -getDecoratedTop(firstChild) // factors in the offset applied with decorations
        for (i in 0 until firstChildPosition) {
            scrollOffsetY += childSizes[i] ?: 0
        }
        return max(scrollOffsetY + paddingTop, 0) // factors in the top padding of the recycler view
    }

}
```

This solution has been battle tested by me and considers paddings addded to the RecyclerView or through `ItemDecoration`. Just make sure assign this `LayoutManager` to your RecyclerView:
```
    recyclerView.layoutManager = ReliableScrollOffsetLinearLayoutManager(context)
```
And call the `recalculateChildHeights()` method after you sort or update the list.
```
        val layoutManager = binding.resultsRv.layoutManager as ReliableScrollOffsetLinearLayoutManager
        layoutManager.recalculateChildHeights()
```
To make sure you don't run into issues where you update the data backing up the `Adapter`, if needed, I recommend setting a callback that is triggered once the first `Adapter.onBindViewHolder()` is called. Here's the skeleton code for this idea:

```

class MyListAdapter : ListAdapter<ListItemUIModel, RecyclerView.ViewHolder>(ListItemDiffCallback()) {

    var onUiUpdateCallback: (() -> Unit)? = null

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        onUiUpdateCallback?.let {
            // The only reliable way I found to notify of the UI being updated.
            // All of the more recommended approaches will trigger the callback too early due to the AsyncListDiffer in this Adapter
            it.invoke()
            onUiUpdateCallback = null // cleanup
        }
        // your ViewHolder binding logic follows:
        holder.bind(getItem(position)) 
    }

    
    // The usual Adapter implementation boilerplate code will follow:
    // (...) 

}
 ```
And of course wire that `onUiUpdateCallback` to the `recalculateChildHeights()` method from our custom `LayoutManager` right before you update the data in the `Adapter` backing up the RecyclerView:

```
        myListAdapter.onUiUpdateCallback = {
                // must recalculate scroll offset after sorting and after adding new items to the list before the current scroll position
                (binding.resultsRv.layoutManager as ReliableScrollOffsetLinearLayoutManager).recalculateChildHeights() 
       }

        myListAdapter.submitList(listItems) 
```



## In summary

1. Create a `LinearLayoutManager` like the `ReliableScrollOffsetLinearLayoutManager` I posted above.
2. Assign that layoutManager to your `RecyclerView`.
3. Make sure to call `recalculateChildHeights()` when you update the list data.
4. If needed for extra reliability: Set a callback wired to the `Adapter.onBindViewHolder()` to call the `recalculateChildHeights()` right before you update the `Adapter` data. 


Too much work for such a simple task, right? I hope this helps you not wasting so much time as I did :) 

