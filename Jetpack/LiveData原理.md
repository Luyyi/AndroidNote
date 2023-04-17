# LiveData原理

### 1. LiveData是什么
[官网链接](https://developer.android.com/topic/libraries/architecture/livedata?hl=zh-cn)

```
LiveData是一种可观察的数据存储器类。
遵循其他组件的生命周期，确保LiveData仅更新处于活跃状态的应用组件观察者。

宿主生命周期DESTROYED时会自动注销。
宿主生命周期从非活跃状态恢复活跃状态，会立即接收最新的数据。

粘性事件:
先发送一条数据，再注册观察者，观察者可以收到注册前发送的那条数据。
```

### 2. 原理

#### （1）LiveData的观察者注册流程

##### ① 代码中为LiveData注册观察者

```kotlin
viewModel.liveData.observe(this) { _ ->
    // onChanged，处理接收到数据变化后的页面逻辑
}
```

</br>
对应到的是LiveData#observe(): 
```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        // 如果生命周期状态是DESTROYED，就不注册了
        return;
    }
    // 将宿主owner和观察者observer整合包装起来
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    // 添加到保存的mObservers（是个SafeIterableMap）
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    // 返回值不为空表示这个observer之前就添加过
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    //包装后的wrapper添加到Lifecycle，开始监听生命周期
    owner.getLifecycle().addObserver(wrapper);
}
```

##### ② 注册流程总结

1. observe()调用开始注册。
2. 注册时先判断生命周期，如果处于DESTROYED，就不注册了。
3. 将宿主owner和观察者observer整合包装成wrapper。
4. 将wrapper保存到一个SafeIterableMap里面，其中key是observer，value是wrapper。
5. 保存时判断是否已经添加过。
6. 如果已经添加过，而且添加过的wrapper绑定的不是当前owner，会报错。
7. 未添加过的wrapper最后会添加到owner的Lifecycle，开始监听生命周期，注册完成。

这样完成了注册观察者的流程。而LiveData还存在`Destroy时自动注销` `活跃时才能收到数据` `黏性事件` 等特性，这些都封装在了LifecycleBoundObserver里面。

</br>

#### （2）事件回调
从上面的注册流程可以看到，回调的逻辑主要封装在LifecycleBoundObserver里面。

##### ① LifecycleBoundObserver

```java
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        // 保存owner
        mOwner = owner;
    }
    
    @Override
    boolean shouldBeActive() {
        // 判断owner的生命周期至少是STARTED，为活跃状态
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
        if (currentState == DESTROYED) {
            //生命周期为DESTROYED时，注销观察者
            removeObserver(mObserver);
            return;
        }
        
        // 循环进行状态对齐，至少会循环一次
        Lifecycle.State prevState = null;
        while (prevState != currentState) {
            prevState = currentState;
            // 这个方法会判断状态，判断是否执行事件分发
            activeStateChanged(shouldBeActive());
            currentState = mOwner.getLifecycle().getCurrentState();
        }
    }

    //...
}
```

1. 这个类的构造方法只保存了LifecycleOwner，用来获取宿主的生命周期状态。
2. 实现了LifecycleEventObserver接口，当宿主生命周期发生变化时，会回调onStateChanged()。
3. 监听到生命周期变为DESTROYED时，注销观察者。`Destroy时自动注销特性`
4. 其他状态则调用父类ObserverWrapper的activeStateChanged(shouldBeActive())来处理。
5. 覆写了父类ObserverWrapper的抽象方法shouldBeActive()，判断owner的生命周期至少是STARTED，为活跃状态

##### ② 父类ObserverWrapper

```java
private abstract class ObserverWrapper {
    // 注册的Observer
    final Observer<? super T> mObserver;
    // 宿主是否活跃状态
    boolean mActive;
    // 版本号，从-1开始，后续根据版本号判断是否执行事件分发
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<? super T> observer) {
        mObserver = observer;
    }

    //...

    // 这就是上面提到的子类LifecycleBoundObserver调用的方法
    // 判断状态，执行事件分发
    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            // 状态未改变
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        // 赋值新的状态
        mActive = newActive;
        // 变更活跃的观察者数量
        changeActiveCounter(mActive ? 1 : -1);
        // 判断状态，执行事件分发
        if (mActive) {
            dispatchingValue(this);
        }
    }
}
```

##### ③ 回调流程总结

1. 回调流程包装在LifecycleBoundObserver类中。
2. LifecycleBoundObserver实现了LifecycleEventObserver，监听宿主生命周期发生变化时，onStateChanged()会被回调。
3. 当监听到宿主生命周期变为DESTROYED，会调用removeObserver(mObserver)注销当前观察者。
4. 当生命周期不是DESTROYED，判断生命周期至少是STARTED为活跃状态。
5. 调用父类ObserverWrapper的activeStateChanged()，根据是否活跃状态决定是否执行事件分发。

这样完成了观察者事件回调流程，解答了LiveData`在DESTROYED时自动注销观察者` `活跃状态才会发送数据`的特性问题。

</br>

#### （3）事件分发

##### ① LiveData#dispatchingValue()

这是上面回调流程提到的事件分发方法。

```java
 void dispatchingValue(@Nullable ObserverWrapper initiator) {
    // 标记是否正在执行事件分发
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        // initiator是上面提到的ObserverWrapper
        if (initiator != null) {
            // 不为空，说明指定了观察者Observer
            // 通知指定的Observer更新数据
            considerNotify(initiator);
            initiator = null;
        } else {
            // 没有指定Observer，则遍历mObservers
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                //一个一个通知更新
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```

##### ② LiveData#considerNotify()

```java
private void considerNotify(ObserverWrapper observer) {
    // 当前观察者不在活跃状态，就不更新了
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    
    //shouldBeActive()在LifecycleBoundObserver中被重写
    // 当生命周期至少为STARTED时返回true
    if (!observer.shouldBeActive()) {
        // 宿主非活跃状态，不更新
        observer.activeStateChanged(false);
        return;
    }
    
    // 根据版本号判断
    // 记录的版本号>=当前版本号，不更新
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    // 记录新的版本号
    observer.mLastVersion = mVersion;
    // onChanged回调
    observer.mObserver.onChanged((T) mData);
}
```

##### ③ 事件主动更新：setValue()、postValue()

上面触发事件分发的时机是监听到生命周期发生变化且生命周期至少是STARTED时，更新最新的事件。

而在代码中，是通过LiveData的setValue()、postValue()来主动更新事件的。

###### LiveData#setValue()

```java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    // 版本号自增1
    mVersion++;
    // 设置数据
    mData = value;
    // 事件分发流程
    dispatchingValue(null);
}
```
1. setValue()是在主线程执行的
2. 版本号自增1，后续在事件分发流程中与旧的版本号作对比决定是否发送数据
3. dispatchingValue(null)，observer传入null，后续会通知mObservers中所有观察者更新数据

###### LiveData#postValue()

```java
protected void postValue(T value) {
    boolean postTask;
    // 加锁
    synchronized (mDataLock) {
        // NOT_SET是mPendingData的初始值
        postTask = mPendingData == NOT_SET;
        // 暂存value
        mPendingData = value;
    }
    // postTask为false，表示上一个postValue的runnable还没执行，直接返回
    // 注意，就算返回了，mPendingData也已经是新的value
    if (!postTask) {
        return;
    }
    // 切换到主线程更新
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    @SuppressWarnings("unchecked")
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            // mPendingData是最新的value
            newValue = mPendingData;
            // 重置mPendingData
            mPendingData = NOT_SET;
        }
        // 调用setValue()
        setValue((T) newValue);
    }
};
```

##### ④ 事件分发流程总结

1. 通过调用postValue()或setValue()主动更新事件
2. postValue()可以在子线程调用，自动切换到主线程调用setValue()
3. setValue()只能在主线程调用，每次setValue()，版本号自增1，接着调用dispatchingValue(null)
4. LiveData监听到宿主生命周期发生变化且不是`DESTROYED`时也会自动调用dispatchingValue(observer)
5. dispatchingValue()传入的observer为null，会遍历mObservers通知所有observer；不为空则通知指定observer
6. 通过调用considerNotify()通知observer
7. 判断当前observer不活跃，返回，不分发
8. 判断observer的宿主生命周期没到STARTED，不活跃，返回，不分发
9. 判断版本号，如果版本号已经是最新，返回，不分发
10. 满足了全部条件。更新版本号，调用observer.onChanged()，观察者观察到数据变化，到了View层我们自己的更新逻辑。
