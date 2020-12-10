# ActivityLifecycleCallback检测APP是否处于前台



ActivityLifecycleCallback是Application中的一个接口，用于监听Activity的生命周期的事件，

此回调运行在主线程，不应该在其中运行耗时操作。可以使用这个回调检测APP是否处于前台：

## 实现ActivityLifecycleCallback接口

```java
import android.app.Activity;
import android.app.Application;
import android.os.Bundle;
import android.util.Log;

import com.reconova.visitor.util.DeviceUtil;
import com.reconova.visitor.util.SystemUtil;

public class ActivityListener implements Application.ActivityLifecycleCallbacks {
    private static final String TAG = "ActivityListener";
    private int count = 0;

    @Override
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {

    }

    @Override
    public void onActivityStarted(Activity activity) {
        // 应用进入前台
        if (count == 0) {
            Log.d(TAG, "onActivityStarted: app in foreground");
        }
        count++;
    }

    @Override
    public void onActivityResumed(Activity activity) {

    }

    @Override
    public void onActivityPaused(Activity activity) {

    }

    @Override
    public void onActivityStopped(Activity activity) {
        count--;
        // 应用进入后台
        if (count == 0) {
            Log.d(TAG, "onActivityStopped: app in background");
        }
    }

    @Override
    public void onActivitySaveInstanceState(Activity activity, Bundle outState) {

    }

    @Override
    public void onActivityDestroyed(Activity activity) {

    }
}
```

## 在Application中注册回调

```java
import android.app.Application;

public class MApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();

        registerActivityLifecycleCallbacks(new ActivityListener());
    }
}

```

## 原理

这里通过一个count来对当前处于前台的Activity进行计数，在onActivityStarted中先判断，如果count为0，则说明此时是从后台切换到前台，然后将count+1。在onActivityStopped中现将count递减，然后进行判断，如果count为0，说明此时应用从前台切换到后台。为什么在onActivityStarted和onActivityStopped中进行计数和判断，而不是其他方法，这里需要了解一下Activity在各种切换其生命周期的变化。

一般APP切到后台可以分为这几种情况，

1. 从当前页面跳转到其他APP，这是属于两个Activity之前的切换
2. 按下Home键，APP进入后台
3. 按下back键退出应用，这不算进入后台。

下面分别看下这几种情况下Activity的生命周期变化

## 两个Activity之间的切换

假设我们现在两个Activity：MainActivity和AnimatorActivity，现在从MainActivity跳转到AnimatorActivity，其生命周期的调用事件如下：

```
D/MainActivity: onPause: 
D/AnimatorActivity: onCreate:
D/AnimatorActivity: onStart: 
D/AnimatorActivity: onResume: 
D/MainActivity: onStop: 
```

从日志可以看出，首先调用MainActivity的onPause方法，然后开始创建AnimatorActivity，等AnimatorActivity创建完成并显示出来，即调用onResume后，才调用MainActivity的onStop方法。

这样看来，应该在onStop中将count递减，然后再onCreate、onStart和onResume中将count递增都可以，但是别忘了还有直接按HOME键返回的，下面来看下按HOME键将APP进入后台的情况吧。

## 按下HOME键的生命周期变化

当前正在MainActivity，当按下HOME键时，生命周期变化如下：

```
D/MainActivity: onPause: 
D/MainActivity: onStop: 
D/MainActivity: onStart: 
D/MainActivity: onResume: 
```

 可以看出，在按下HOME键切到后台再切回前台时，不一定会调用onCreate，所以一定不能再onCreate中对count递增，这样会导致计数错误。

所以最好还是在onStat和onStop中对前台页面计数会比较准确。