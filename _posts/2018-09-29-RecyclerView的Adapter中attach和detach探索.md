# RecyclerView的Adapter中attach和detach探索

## 问题描述
今天App的日志捕获中收到了一条这样的crash日志：
![异常捕获图](http://oq54hiwcu.bkt.clouddn.com/2018-09-29-Screen%20Shot%202018-09-29%20at%2022.15.06.png)

刚看到这个日志的时候，分析了一下，复现的场景应该是这样的：RecyclerView的Item中一个按钮，点击了之后会发起一个异步请求，开始前会弹出一个ProgressDialog等待，如果这个时候按home键回到了后台，此时不巧被Activity被系统回收的话，就会出现这个问题。debug模式下，开启不保留活动发现能够稳定复现。原因是：异步操作回来的时候，在执行ProgressDialog的dismisss方法的时候，由于Activity已经被回收之后，就相当于这个ProgressDialog（它持有了Activity的Context）所依附的window已经被销毁了，所以会出现这个问题。

## 代码场景
具体到项目的场景中，我们的项目中一个RecyclerView中对应了很多种type类型，所以引用了MultiType的库来简洁的注册多种类型，这并没什么问题。

问题是：当初为了简单，按钮的点击效果是直接放到了ViewHolder中来处理了。参考代码如下：

```java

public class SongBinder extends ItemViewBinder<SongInfo, SongBinder.ViewHolder> {
    @NonNull @Override protected ViewHolder onCreateViewHolder(@NonNull LayoutInflater inflater,
        @NonNull ViewGroup parent) {
        ...
    }

    @Override protected void onBindViewHolder(@NonNull ViewHolder holder, @NonNull SongInfo item) {
        holder.bind(item);
        ...
    }

    static class ViewHolder extends RecyclerView.ViewHolder {
        private View mView;
        private ProgressDialog mProgressDialog;
        private CompositeDisposable mDisposables = new CompositeDisposable();
        ...

        public ViewHolder(View itemView) {
            super(itemView);
            mView = itemView;
            ...
        }

        public void bind(SongInfo info) {
            mView.setOnClickListener(view -> {
                ...
                // Step1 显示弹窗
                showProgress();
                mDisposables.add(PlayUtils.playSingleSong().subscribe(status -> {
                	// Step2 关闭弹窗
                    dismissProgress();
                    ...
                }, throwable -> {
                	// Step2 关闭弹窗
                    dismissProgress();
                    ...
                }));
            });
        }

        public void showProgress() {
            if (mProgressDialog == null) {
                mProgressDialog = new LoadingDialog(getContext());
            }
            mProgressDialog.show();
        }

        public void dismissProgress() {
            if (mProgressDialog != null && mProgressDialog.isShowing()) {
                mProgressDialog.dismiss();
            }
        }

        public Context getContext() {
            return mView.getContext();
        }
    }
}
```

如上面的代码所示，当初为了简便，点击事件的处理放到了ViewHolder中，所以出现了开头所述的问题。因为RecyclerView的Apdater有attach和detach的方法，所以看到这个问题，第一反应是增加这两个方法，然后在detach方法中执行异步任务取消的操作，代码如下：

```java

public class SongBinder extends ItemViewBinder<SongInfo, SongBinder.ViewHolder> {
	private CompositeDisposable mDisposables = new CompositeDisposable();
	...
	
    @Override protected void onViewAttachedToWindow(@NonNull ViewHolder holder) {
        super.onViewAttachedToWindow(holder);
    }

    @Override protected void onViewDetachedFromWindow(@NonNull ViewHolder holder) {
        super.onViewDetachedFromWindow(holder);
        // 取消异步操作
        mDisposables.clear();
    }

   class ViewHolder extends RecyclerView.ViewHolder {
        private View mView;
        private ProgressDialog mProgressDialog;
        
        ...
    }
}
```

如代码注释，设置取消的异步任务的操作。本以为这样设置应该可以解决这个问题，但是测试发现，还是会出现上述问题。调试发现，attach方法会执行，但是detach方法并没有被执行到。

后来在[这篇文章](https://eneim.github.io/2017/06/03/Why-you-should-call-setAdapter-null/)中找到了一些说明，这里选取部分要点如下：
> - `onAttachedToRecyclerView` is called when the Adapter is set to RecyclerView, after a call to RecyclerView#setAdapter(Adapter) or RecyclerView#swapAdapter(Adapter, boolean). This is quite obvious.
- `onDetachedFromRecyclerView`, on the other hand, is called when current Adapter if going to be replaced by another Adapter (this another ‘Adapter’ can be Null). What is the point here: if you don’t replace the Adapter, this method will never be called. And what happens if an Adapter is never be “detached” from a RecyclerView? Let’s see after I explain about the other couples.
- `onViewAttachedToWindow` is called once RecyclerView or its LayoutManager add a View into RecyclerView (hint: go to RecyclerView source code and search for the following keywords: dispatchChildAttached).
- `onViewDetachedFromWindow`, on opposite, is called when RecyclerView or its LayoutManager detach a View from current Window.

大致意思是说：`onViewDetachedFromWindow`只有当它的布局管理把一个子的Item View从当前Window中分离的时候才会调用。总结来说，在以下两种情况下会被调用：

- 显式的调用Adapter的remove方法；
- 重新设置RecyclerView的Adapter；

## 解决方案
知道了上述原因，想到两种解决方案：

1. 比较简单的改法：在Actiivty的onDestroy方法中调用RecyclerView的setAdapter方法，把原来的Adapter设置为null；这样可以保证onDetach的调用，实际测试发现可以解决问题；
2. 把所有的点击事件移到Activity中去处理，不要再ViewHolder中处理点击事件。弹对话框的操作也放到Activity中去处理。

项目中为了简单，采用了第一种改法：

```java
@Override protected void onViewDetachedFromWindow(@NonNull ViewHolder holder) {
    super.onViewDetachedFromWindow(holder);
    // 避免窗体泄漏
    holder.dismissProgress();
    mDisposables.clear();
}

public class SearchFragment extends Fragment{
	...
    @Override public void onDestroyView() {
        mResultListBinding.recyclerView.setAdapter(null);
        super.onDestroyView();
    }
}
```
