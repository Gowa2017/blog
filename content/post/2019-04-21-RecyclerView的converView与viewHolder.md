---
title: RecyclerView的converView与viewHolder
categories:
  - Android
date: 2019-04-21 12:54:17
updated: 2019-04-21 12:54:17
tags: 
  - Android
---

当前 *ListView* 很多时候都已经被 *RecyclerView* 替代了。相对而言，*RecyclerView* 更加的灵活和高级。

在 *RecyclerView* 中，几种不同的组件相互工作来显示数据。*RecyclerView* 通过我们提供的 *LayoutManager* 来获取 View 进行填充。 

<!--more-->

*RecyclerView* 中的 View 通过 *view holder* 来进行表示。*view holder* 对象是我们继承 了 *RecyclerView.ViewHolder* 的类的实例。每个 *View Holder* 负责用一个 View 来显示单个的数据项。

*RecyclerView* 只会创建能在屏幕上显示的那么多个 *view holder*。当用户滑动列表的时候，*RecyclerView* 会将离开屏幕的 view 绑定到新滑动到屏幕的数据项。

**view holder** 通过 *adapter* 进行管理，我们会通过继承 *RecyclerView.Adapter* 来实现。adapter 会根据需要来创建 *view holder*，然后将数据绑定到 *view holder*。其通过将 *view holder* 给到一个位置，然后调用 adapter 的 `onBindViewHolder()` 方法。这个方法会使用 *view holder* 的位置来决定其应该显示什么数据。


*RecyclerView* 做了很多优化，我们自己就不用干这些事情了：

- 当列表第一次展示，其会在列表的两边都绑定一些 *view holder*。例如，如果视图要显示 0-9 位置的数据，*RecyclerView* 会创建这些 *view holder*，但是他也可能会创建位置 10 的 *view holder*。这样，如果用户滑动列表的话，下一个元素就会立马显示出来。
- 当用户滑动列表的时候，*RecyclerView* 根据需要创建 *view holder*。其也会保存已经离开了屏幕的 *view holder*，以便复用。这样， *view holder* 不用重新创建及从布局扩张，而只需要更新 *view holder* 的内容就行了。
- 当显示的内容变了。可以耐用 `RecyclerView.Adapter.notify...()`方法。

# 使用

如果我们在 Activity 内添加一个 RecyclerView，我们一般会有如下的代码：

```java
public class MyActivity extends Activity {
    private RecyclerView recyclerView;
    private RecyclerView.Adapter mAdapter;
    private RecyclerView.LayoutManager layoutManager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.my_activity);
        recyclerView = (RecyclerView) findViewById(R.id.my_recycler_view);

        // use this setting to improve performance if you know that changes
        // in content do not change the layout size of the RecyclerView
        recyclerView.setHasFixedSize(true);

        // use a linear layout manager
        layoutManager = new LinearLayoutManager(this);
        recyclerView.setLayoutManager(layoutManager);

        // specify an adapter (see also next example)
        mAdapter = new MyAdapter(myDataset);
        recyclerView.setAdapter(mAdapter);
    }
    // ...
}
```

同时我们还需要一个 适配器：

```java
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.MyViewHolder> {
    private String[] mDataset;

    // Provide a reference to the views for each data item
    // Complex data items may need more than one view per item, and
    // you provide access to all the views for a data item in a view holder
    public static class MyViewHolder extends RecyclerView.ViewHolder {
        // each data item is just a string in this case
        public TextView textView;
        public MyViewHolder(TextView v) {
            super(v);
            textView = v;
        }
    }

    // Provide a suitable constructor (depends on the kind of dataset)
    public MyAdapter(String[] myDataset) {
        mDataset = myDataset;
    }

    // Create new views (invoked by the layout manager)
    @Override
    public MyAdapter.MyViewHolder onCreateViewHolder(ViewGroup parent,
                                                   int viewType) {
        // create a new view
        TextView v = (TextView) LayoutInflater.from(parent.getContext())
                .inflate(R.layout.my_text_view, parent, false);
        ...
        MyViewHolder vh = new MyViewHolder(v);
        return vh;
    }

    // Replace the contents of a view (invoked by the layout manager)
    @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        // - get element from your dataset at this position
        // - replace the contents of the view with that element
        holder.textView.setText(mDataset[position]);

    }

    // Return the size of your dataset (invoked by the layout manager)
    @Override
    public int getItemCount() {
        return mDataset.length;
    }
}
```

布局管理器会调用 适配器 的 `onCreateViewHolder()` 方法。这个方法需要构造一个 *RecyclerView.ViewHolder*，同时设置其用来显示内容的 View。**ViewHoler** 的类型必须与 Adapter 类签名声明的类型一致。通常，会通过扩张一个 xml 布局文件来获取视图。因为此时 VH 并没有被赋予实际的值，所以方法不会设置View 的内容。

接着布局管理器会将 VH 与数据相绑定。其通过调用 `onBindViewHolder()`方法来达成，同时会将 VH 在 RecyclerView 中的位置作为参数。


# 总结

*RecyclerView* 的内容通过 VH 进行表示，VH 会与一个 VIEW 绑定，这个VIEW 一般通过 xml 布局文件进行扩张。

*RecyclerView* 只会创建屏幕上能能够显示的那么多个 VH，滑出屏幕的会缓存也进行复用。

*RecyclerView* 中的 VH 保留了对 VIEW 中各个控件的引用，每次需要改变内容的时候不需要 `findViewById()`，直接对引用的控件进行操作即可。
