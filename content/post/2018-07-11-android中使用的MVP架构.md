---
title: android中使用的MVP架构
categories:
  - Android
date: 2018-07-11 21:38:22
updated: 2018-07-11 21:38:22
tags: 
  - Android
  - MVP
---
从未写过代码的人，现在转行做安卓，很多问题其实都不知道。就比如吧，刚上班一周就开始马上贡献代码给公司了，但其实做得非常的差。刚开始的时候是做一些，拉取数据，然后在RecyclerView中进行展示，或者在一LinearLayout内展示内容。但一般遇到业务逻辑比较复杂，而且后面老是变更各种需求的时候，这个问题就比较麻烦了。

# 典型问题

比如业务上，我们把不同的数据，如：人员信息，职务信息，基本情况分别放在三个表内，现在接口那边，为减轻服务器压力，并不会进行联合查询，而是分别返回，每个接口返回对应的数据。那么，当我们需要基于三个返回的数据进行业务判断的时候，那么我们就需要等待三个数据都返回后再进行操作。

那么，我们要么是通过同过串联式调用，第一个调用成功后，在其网络获取请求的回调函数中调用第二个接口。这是非常低的。

或者并发调用，使用一个handler，等待三个接口返回的数据都到达后进行处理。

最疯狂的时候，需要调用10个接口的数据来进行处理，这可能是顶层设计的问题，或者是后端优化不行的问题，但是已经不大可能改了。只能是这样将就着去适配。

偶然看到了 google 的  例子内看到了他们对于架构的描述。

# MVC

平常用得最多的可能就是 MVC，**model, view, controler**，view负责显示逻辑，model负责获取数据，controler负责业务逻辑。但事实上在安卓内，我们是把一大坨代码，都放在activity或者 fragment中的，哈哈，what's a fuck。等到业务需求变更的时候，一种想死的心油然而生。

就比如我们干的事情，在view中，写获取数据，写业务逻辑，简直就是坨屎在那里。

# MVP

这可能是现在用得最多的架构模式了。model属于数据获取模型，presenter用于处理业务逻辑，view负责显示就OK了。

但事实上这个架构不好理解，特别是连续不断的回调实在让人蛋疼。而对于是以接口的方式进行相互连接，而不是对象持有的方式也让人有点感觉茫然。好在，谷歌提供了一些比较实际的例子。

整个架构的关键就在于 Presenter，它对 Model，View都进行了引用，所以能在两者间进行通信和调度。最终看了代码的感觉是，其专心的只是逻辑，至于数据从哪里来，怎么显示，他不是很关心，这个交由 Model, View自己去实现。

## Contract
代码中使用了一个 接口类来定义每个 View, Presenter要实现的接口。

每个activity/fragment都会定义附加逻辑及model。

```java
public interface TasksContract {

    interface View extends BaseView<Presenter> {

        void setLoadingIndicator(boolean active);

        void showTasks(List<Task> tasks);

        void showAddTask();

        void showTaskDetailsUi(String taskId);

        void showTaskMarkedComplete();

        void showTaskMarkedActive();

        void showCompletedTasksCleared();

        void showLoadingTasksError();

        void showNoTasks();

        void showActiveFilterLabel();

        void showCompletedFilterLabel();

        void showAllFilterLabel();

        void showNoActiveTasks();

        void showNoCompletedTasks();

        void showSuccessfullySavedMessage();

        boolean isActive();

        void showFilteringPopUpMenu();
    }

    interface Presenter extends BasePresenter {

        void result(int requestCode, int resultCode);

        void loadTasks(boolean forceUpdate);

        void addNewTask();

        void openTaskDetails(@NonNull Task requestedTask);

        void completeTask(@NonNull Task completedTask);

        void activateTask(@NonNull Task activeTask);

        void clearCompletedTasks();

        void setFiltering(TasksFilterType requestType);

        TasksFilterType getFiltering();
    }
}
```

定义了V/P需要实现的接口。

## 初始化

通过在 View 中来初始化 Presenter：，在 `onCreate()`方法中：

```java
    private TasksPresenter mTasksPresenter;

        mTasksPresenter = new TasksPresenter(
                Injection.provideTasksRepository(getApplicationContext()), tasksFragment);
```

这样就获得了对 Presenter 的引用。第一个参数是传递给构造器的 model实现，这个有点特别，因为他实现了缓存等操作比较复杂，我们先不关注这个。


在 Presenter的构造器中：

```java
    public TasksPresenter(@NonNull TasksRepository tasksRepository, @NonNull TasksContract.View tasksView) {
        mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null");
        mTasksView = checkNotNull(tasksView, "tasksView cannot be null!");

        mTasksView.setPresenter(this);
    }
```

其持有了 model/view，所以其就能进行两者的调用操作了。


### Presenter 与 Model交互

看一个简单的方法，当需要获取所有的任务列表的时候，在 View中会看到如此的调用：

```java
                mPresenter.loadTasks(false);

```

但我们观察一下这个方法，会发现：

```java
        mTasksRepository.getTasks(new TasksDataSource.LoadTasksCallback() {
            @Override
            public void onTasksLoaded(List<Task> tasks) {
                List<Task> tasksToShow = new ArrayList<Task>();

                // This callback may be called twice, once for the cache and once for loading
                // the data from the server API, so we check before decrementing, otherwise
                // it throws "Counter has been corrupted!" exception.
                if (!EspressoIdlingResource.getIdlingResource().isIdleNow()) {
                    EspressoIdlingResource.decrement(); // Set app as idle.
                }

                // We filter the tasks based on the requestType
                for (Task task : tasks) {
                    switch (mCurrentFiltering) {
                        case ALL_TASKS:
                            tasksToShow.add(task);
                            break;
                        case ACTIVE_TASKS:
                            if (task.isActive()) {
                                tasksToShow.add(task);
                            }
                            break;
                        case COMPLETED_TASKS:
                            if (task.isCompleted()) {
                                tasksToShow.add(task);
                            }
                            break;
                        default:
                            tasksToShow.add(task);
                            break;
                    }
                }
                // The view may not be able to handle UI updates anymore
                if (!mTasksView.isActive()) {
                    return;
                }
                if (showLoadingUI) {
                    mTasksView.setLoadingIndicator(false);
                }

                processTasks(tasksToShow);
            }

            @Override
            public void onDataNotAvailable() {
                // The view may not be able to handle UI updates anymore
                if (!mTasksView.isActive()) {
                    return;
                }
                mTasksView.showLoadingTasksError();
            }
        });
```

其实是调用了 model去获取任务，调用方法的时候，同时传递给 model一个回调，其在获取数据成功后就会进行回调：

```java
        mTasksRepository.getTasks(new TasksDataSource.LoadTasksCallback() {
... ...

processTasks(tasks);

    private void processTasks(List<Task> tasks) {
        if (tasks.isEmpty()) {
            // Show a message indicating there are no tasks for that filter type.
            processEmptyTasks();
        } else {
            // Show the list of tasks
            mTasksView.showTasks(tasks);
            // Set the filter label's text.
            showFilterLabel();
        }
    }
}
```

回调过后，其就调用 View 的方法来更新界面显示了。

瞧，多么简单。
