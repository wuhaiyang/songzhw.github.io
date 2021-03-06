## 1. Why Do You Need Another Library: ViewModel?

### 1.1 You need to handle the configuration change
The configuration change is totally out of your control. Last minute, your user is using your app. Next minute, he/she switch out and change a configuration(i.e. rotate the screen), your app may remove your current Activity and re-create it again. This is completely out of your control, but you have to deal with it. 

I know some developer would say, "It's okay for me. My project just make sure every Activity's orientation is portrait. So I have no such problems." But screen rotation is just one example of configuration change. You can make your Activity portrait, but you can forbit your user to change the language, or font size. Every time the user changed the language, like from English to Chinese, your activity will get notified this is a configuration change, and it may get removed and re-created. And you, as a developer, have to deal with it.

So, **The [ViewModel](https://developer.android.com/reference/android/arch/lifecycle/ViewModel.html) class allows data to survive configuration changes such as screen rotations**. 

### 1.2 Why onSaveInstanceState() is not enough for us?
A tranditional way to handle the configuration change is to save data in the onSaveInstanceState() and to restore the data in onCreate(). 

But there are two limits about onSaveInstanceState(). 
1. onSaveInstanceState() can not save a large amounts of data. I've seen some posts that some developer saved a lot of data in the onSaveInstanceState(), and they got a `TransactionTooLargeException`.
2. The data you want to save in the onSaveInstanceState() must be serializable. So you'd better to use Parceable (Serializable is not a good option for Android applications). This is not just creating a class and making it implement Parceable. Sometimes the data is from third-party liarbry, and you can not modify the data. So for some scenarios, it's hard to save data in onSaveInstanceState().

## 2. A Simplest Demo
**Step 01** - Create your ViewModel class
Just make it extend ViewModel. 
```java
public class ZeroViewModel extends ViewModel {
    public User user;
}
```

**Step 02** - use it in the Fragment/FragmentActivity
```java

public class ZeroDemo extends AppCompatActivity {
    private TextView tv;
    private ZeroViewModel vm;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tv_btn);
        tv = findViewById(R.id.tv_simple);
        
        vm = ViewModelProviders.of(this).get(ZeroViewModel.class);
        System.out.println("szw vm.user = " + vm.user);
    }
    
    // android:onClick="onClickSimpleButton"
    public void onClickSimpleButton(View v) {
        vm.user = new User(23, "jorden");
    }
}
```
Note that `ViewModelProvides.of()` needs a Fragment or a FragmentActivity as a parameter. 
By the way, AppCompatActivity is a subclass of Fragment, so you can pass an AppCompatActivity object here.

You can rotate screen now, and you will find out the value of `vm.user` is always there. That's what the ViewModel's for.


## 3. Different Instance of One Activity Class

### 3.1 Having two instance at the same time
Let's do an experiment. Here is the code.

```java
public class SameClass01 extends AppCompatActivity {
    private TextView tv;
    private ZeroViewModel vm;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tv_btn);
        tv = findViewById(R.id.tv_simple);

        vm = ViewModelProviders.of(this).get(ZeroViewModel.class);
        System.out.println("szw SameClass01 : " + vm.user);

    }
    // launch the second instance
    // android:onClick="onClickSimpleButton"
    public void onClickSimpleButton(View v) {
        vm.user = new User(100, "SuperMario");
        startActivity(new Intent(this, SameClass01.class));
    }
}
```

And when the second launch is created, the log is :
`I/System.out: szw SameClass01 : null`

So even you have two instance of one Activity, the ViewModels is not a mess. The ViewModel in the first instance holds the "Mario" user, and the ViewModel in the second instance holds the null user. 

### 3.2 Creating the second instance later
Now if I open SameClass02 (an Activity), save one user onto ViewModel, then exit SameClass02. 
Later, I open SameClass02 again, what value would I get from the ViewModel's user.

```java
public class SameClass02 extends AppCompatActivity {
    private TextView tv;
    private ZeroViewModel vm;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tv_btn);
        tv = findViewById(R.id.tv_simple);
        vm = ViewModelProviders.of(this).get(ZeroViewModel.class);
        System.out.println("szw SameClass02 onCreate() : " + vm.user);
    }
    // android:onClick="onClickSimpleButton"
    public void onClickSimpleButton(View v) {
        vm.user = new User(22, "test");
    }

    // android:onClick="onClickSimpleButton2"
    public void onClickSimpleButton2(View v) {
        System.out.println("szw SameClass02 : saved = "+vm.user);
    }
}

```

From the log, `System.out: szw SameClass02 onCreate() : null`, we are glad to see the ViewModel is not messed.

## 4. Static, An Alternative?

### 4.1 Experiment 01 : Rotate Screen
I was told ViewModel can survive through configuration change. And the previous code shows if your activity exit, the data on ViewModel would be erased.  So I was wondering, this seems a job of static value. 

Let's write some code to verify that. 

```java
public class SameVm {
    public static User user;
}
```

And then save the value in the Activity. 

```java
public class ZeroDemo extends AppCompatActivity {
    private TextView tv;
    private ZeroViewModel vm;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tv_btn);
        tv = findViewById(R.id.tv_simple);

        String value = savedInstanceState == null ? "emptyBundle" : savedInstanceState.getString("key");
        System.out.println("szw onCreate() " + value);

        vm = ViewModelProviders.of(this).get(ZeroViewModel.class);
        System.out.println("szw vm.user = " + vm.user);

        System.out.println("szw static = "+SameVm.user);
    }

    // android:onClick="onClickSimpleButton"
    public void onClickSimpleButton(View v) {
        vm.user = new User(23, "jorden");
        SameVm.user = new User(21, "king");
    }
}

```

After rotating the screen,  we save the value:
```
szw vm.user = User{id=23, name='jorden'}
szw static = User{id=21, name='king’}
```

So we still have the same user after we rotating the screen when we are storing the user object as a static value. 

### 4.2 Experiment 02 : Terminate Application
Same code, but this time we do something different. After executing onClickSimpleButton(), we put the app to the background. Then terminate the application. And then bring the app again.

Here is what happened. The log showed the ViewModel and static value can not both survive the application termination.
```java
szw vm.user = null
szw static = null
```

### 4.3 Then What's the Difference?
You now can see that static value and ViewModel seems no different. They can both survive the configuration change, and they neither can survive the application termination. 

But they do have some difference
1. [Design] ViewModel is designed as a decouple part. Just like the Preseneter in the MVP, ViewModel is the VM in the MVVM pattern.  So you can do asyncronous operations in the ViewModel to fetch data, you can change the data and let the View know the change (you may need the [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) to do that.)
2. [Saving Value] The static value can be modified by every class. But the value in the ViewModel is activity-local variable, which is like [ThreadLocal](https://docs.oracle.com/javase/7/docs/api/java/lang/ThreadLocal.html) class.
3. ViewModel can tell whether the destroy of one Activity is a normal finish or a configuration change. If this is a normal finish, ViewModel would erase the value that binds to this activity. And static value can not do that.

## 5. Caution
(1). A ViewModel must never reference a view, Lifecycle, or any class that may hold a reference to the activity context.
Just like what I said before, ViewModel is somehow like static value. If you hold an Activity context there, you will get a memory leak.

(2). If you do need a Context object to try to get the dimens, strings, or some kind of Manager (i.e. LocationManager, SensorManager, ...), you should use [AndroidViewModel](https://developer.android.com/reference/android/arch/lifecycle/AndroidViewModel.html). Then you can get an application object in the AndroidViewModel. 

Here is an example of how to use AndroidViewModel.
```java
public class SensorViewModel extends AndroidViewModel {
    private SensorManager sensorManager;

    public SensorViewModel(@NonNull Application application) {
        super(application);
        sensorManager = (SensorManager) application.getSystemService(Context.SENSOR_SERVICE);
    }
}
```

## 6. Trap
According to [this post](https://medium.com/google-developers/viewmodels-and-livedata-patterns-antipatterns-21efaef74a54), ViewModel does not fit perfectly with event. 

Consider a ViewModel with a MutableLiveData<String> field:

```java
public class DupliViewModel extends ViewModel {
    private MutableLiveData<String> message = new MutableLiveData<>();

    public void fetchMessage(){
        message.setValue("A New Value");
    }

    public LiveData<String> getMessage() {
        return message;
    }
}

```
And here is our Activity:

```java

public class DupliObserverDemo extends AppCompatActivity {
    private TextView tv;
    private DupliObserverDemo self;
    private DupliViewModel vm;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tv_btn);
        self = this;
        tv = findViewById(R.id.tv_simple);

        vm = ViewModelProviders.of(this).get(DupliViewModel.class);
        vm.getMessage().observe(this, new Observer<String>() {
            @Override
            public void onChanged(@Nullable String s) {
                System.out.println("szw updated ~");
                Toast.makeText(self, "updated "+s, Toast.LENGTH_SHORT).show();
            }
        });
    }
    
    // android:onClick="onClickSimpleButton"
    public void onClickSimpleButton(View v) {
        vm.fetchMessage();
    }

}
```


If the user rotate the screen, the new activity will register a observer again. When the LiveData observation starts, the activity will immediately receives the old value, which cause the toast to show again!

If you want to avoid this, you should use [SingleLiveEvent](https://github.com/googlesamples/android-architecture/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/SingleLiveEvent.java). Here is the source of SingleLiveEvent.

```java
/*
 *  Copyright 2017 Google Inc.
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 *  you may not use this file except in compliance with the License.
 *  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 */

/**
 * A lifecycle-aware observable that sends only new updates after subscription, used for events like
 * navigation and Snackbar messages.
 * <p>
 * This avoids a common problem with events: on configuration change (like rotation) an update
 * can be emitted if the observer is active. This LiveData only calls the observable if there's an
 * explicit call to setValue() or call().
 * <p>
 * Note that only one observer is going to be notified of changes.
 */
public class SingleLiveEvent<T> extends MutableLiveData<T> {

    private static final String TAG = "SingleLiveEvent";

    private final AtomicBoolean mPending = new AtomicBoolean(false);

    @MainThread
    public void observe(LifecycleOwner owner, final Observer<T> observer) {

        if (hasActiveObservers()) {
            Log.w(TAG, "Multiple observers registered but only one will be notified of changes.");
        }

        // Observe the internal MutableLiveData
        super.observe(owner, new Observer<T>() {
            @Override
            public void onChanged(@Nullable T t) {
                if (mPending.compareAndSet(true, false)) {
                    observer.onChanged(t);
                }
            }
        });
    }

    @MainThread
    public void setValue(@Nullable T t) {
        mPending.set(true);
        super.setValue(t);
    }

    /**
     * Used for cases where T is Void, to make calls cleaner.
     */
    @MainThread
    public void call() {
        setValue(null);
    }
}
```




