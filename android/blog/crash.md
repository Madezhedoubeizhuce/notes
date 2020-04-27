# 应用崩溃及处理

在开发安卓应用程序时不可避免的会出现崩溃，在发生崩溃时要如何定位并解决呢？

首先要明确崩溃的类型，安卓的崩溃大致可以分为三类：

1. java层崩溃
2. ANR
3. native崩溃

在发生崩溃时首先要搞清楚是什么类型的崩溃，在明白是什么类型崩溃之后就可以开始分析这几类崩溃了：

## java层崩溃

java层崩溃一般都是由于发生运行时异常而导致崩溃，崩溃发生的堆栈基本上都打印在日志里面了，一般只要在发生崩溃之后查看logcat日志就可以很快的定位到异常原因。

还可以监听全局未捕获异常，在发生运行时异常时，可以进行一些处理，比如将崩溃日志保存起来或者上传到服务器收集，其基本实现如下：

```java
package com.reconova.visitor.core;

import android.content.Context;
import android.os.Environment;
import android.util.Log;

import com.reconova.visitor.constants.Constants;
import com.reconova.visitor.util.ActivityContainer;
import com.reconova.visitor.util.SystemUtil;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.PrintWriter;
import java.io.StringWriter;
import java.io.Writer;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

/**
 * 记录程序崩溃的日志 UncaughtException处理类,当程序发生Uncaught异常的时候,由该类来接管程序,并记录发送错误报告.
 *
 * @author ljp
 */
public class CrashHandler implements Thread.UncaughtExceptionHandler {

    private static final String TAG = "CrashHandler";
    private Thread.UncaughtExceptionHandler mDefaultHandler;// 系统默认的UncaughtException处理类
    private volatile static CrashHandler INSTANCE;// CrashHandler实例
    private Context mContext;// 程序的Context对象
    private Map<String, String> info = new HashMap<String, String>();// 用来存储设备信息和异常信息
    private File file;
    private static final long FILESIZE = 1024 * 1024 * 30;// 文件的大小为30M；
    private static final String FILENAME = "CrashLog.txt";

    /**
     * 保证只有一个CrashHandler实例
     */
    private CrashHandler(Context context) {
        mContext = context.getApplicationContext();
        file = getSaveFile();
    }

    /**
     * 获取保存日志文件的路径
     *
     * @return
     */
    private File getSaveFile() {
        File file = null;
        if (Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED)) {//SD卡可用
            File path = new File(Constants.CRASH_LOG_PATH);

            if (!path.exists()) {
                path.mkdirs();
            }

            file = new File(path, FILENAME);
        } else {
            file = new File(mContext.getCacheDir(), FILENAME);
        }
        return file;
    }

    /**
     * 获取CrashHandler实例 ,单例模式
     */
    public static CrashHandler getInstance(Context context) {
        if (INSTANCE == null) {
            synchronized (CrashHandler.class) {
                if (INSTANCE == null) {
                    if (context == null) {
                        throw new NullPointerException("context cannet NULL");
                    }
                    INSTANCE = new CrashHandler(context);
                }
            }
        }
        return INSTANCE;
    }

    /**
     * 初始化
     */
    public void init() {
        mDefaultHandler = Thread.getDefaultUncaughtExceptionHandler();// 获取系统默认的UncaughtException处理器
        Thread.setDefaultUncaughtExceptionHandler(this);// 设置该CrashHandler为程序的默认处理器
    }

    /**
     * 当UncaughtException发生时会转入该重写的方法来处理
     */
    @Override
    public void uncaughtException(Thread thread, Throwable ex) {
        Log.e(TAG, "uncaughtException: ", ex);

        if (!handleException(ex) && mDefaultHandler != null) {
            mDefaultHandler.uncaughtException(thread, ex); // 如果自定义的没有处理则让系统默认的异常处理器来处理
        } else {
            DogFeeder.getInstance().stopFeedDog();
            // 如果自定义的处理过了则重启app
            SystemUtil.getInstance().restartApp(mContext, ActivityContainer.getInstance());
        }
    }

    /**
     * 自定义错误处理,收集错误信息 发送错误报告等操作均在此完成.
     *
     * @param ex 异常信息
     * @return true 如果处理了该异常信息;否则返回false.
     */
    public boolean handleException(Throwable ex) {
        if (ex == null) {
            return false;
        }

        if (file.exists() && file.isFile() && file.length() > FILESIZE) {
            file.delete();
        }

        saveCrashInfo2File(ex);// 保存日志文件
        return true;
    }

    private void saveCrashInfo2File(Throwable ex) {
        Log.d(TAG, "saveCrashInfo2File: ");

        StringBuffer sb = new StringBuffer();
        sb.append(new Date().toString());
        sb.append(": ");
        sb.append("发生崩溃的异常，设备的信息如下：******************************************************分割线***********************" + "\r\n");
        for (Map.Entry<String, String> entry : info.entrySet()) {
            String key = entry.getKey();
            String value = entry.getValue();
            sb.append(key + "\t=\t" + value + "\r\n");
        }
        Writer writer = new StringWriter();
        PrintWriter pw = new PrintWriter(writer);
        ex.printStackTrace(pw);
        Throwable cause = ex.getCause();
        // 循环着把所有的异常信息写入writer中
        while (cause != null) {
            cause.printStackTrace(pw);
            cause = cause.getCause();
        }
        pw.close();// 记得关闭
        String result = writer.toString();
        sb.append("发生崩溃的异常信息如下：" + "\r\n");
        sb.append(result);
        Log.e(TAG, result);
        // 保存文件
        try {
            //判断文件夹是否存在
            if (!file.getParentFile().exists()) {
                file.getParentFile().mkdir();
            }
            FileOutputStream fos = new FileOutputStream(file, true);
            fos.write(sb.toString().getBytes("UTF-8"));
            fos.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

这里实现了Thread.UncaughtExceptionHandler接口，在发生未补货的异常时，都会回调到uncaughtException(Thread thread, Throwable ex)方法中，其中ex表示发生的异常信息。在这里，我们选择了将异常信息保存到文件中，以便随时取出进行分析。

## ANR

ANR是**Application Not Responding**的缩写，表示应用程序无响应。

Android系统中，**ActivityManagerService(简称AMS)**和**WindowManagerService(简称WMS)**会检测App的响应时间，如果App在特定时间无法相应屏幕触摸或键盘输入时间，或者特定事件没有处理完毕，就会出现ANR。

以下四个条件都可以造成ANR发生：

- **InputDispatching Timeout**：5秒内无法响应屏幕触摸事件或键盘输入事件
- **BroadcastQueue Timeout** ：在执行前台广播（BroadcastReceiver）的`onReceive()`函数时10秒没有处理完成，后台为60秒。
- **Service Timeout** ：前台服务20秒内，后台服务在200秒内没有执行完毕。
- **ContentProvider Timeout** ：ContentProvider的publish在10s内没进行完。

因此，我们要尽量避免在主线程中做耗时操作，以此来避免发生ANR。

### ANR分析

一般来说，在系统发生ANR之后，都会生成/data/anr/traces.txt文件，同时也会在logcat中输出对应的日志，因此我们可以直接分析日志，或者分析/data/anr/traces.txt，这里我们来将一些/data/anr/traces.txt文件的分析。

首先来看一下此文件的格式：

```
----- pid 3298 at 2019-12-27 22:59:54 -----
Cmd line: com.reconova.visitor
ABI: arm
Build type: optimized
Zygote loaded classes=3405 post zygote classes=1574
Intern table: 38088 strong; 972 weak
JNI: CheckJNI is off; globals=353
Libraries: /data/app/com.reconova.visitor-1/lib/arm/libImageUtil.so /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so /data/app/com.reconova.visitor-1/lib/arm/libpl_droidsonroids_gif.so /data/app/com.reconova.visitor-1/lib/arm/libsdtapi.so /data/app/com.reconova.visitor-1/lib/arm/libzkusbapi.so /system/lib/libandroid.so /system/lib/libaudioeffect_jni.so /system/lib/libboot_optimization.so /system/lib/libcompiler_rt.so /system/lib/libjavacrypto.so /system/lib/libjnigraphics.so /system/lib/libmedia_jni.so /system/lib/librs_jni.so /system/lib/libsoundpool.so /system/lib/libwebviewchromium_loader.so libjavacore.so (16)
Heap: 10% free, 49MB/55MB; 470818 objects
Dumping cumulative Gc timings
Start Dumping histograms for 150 iterations for concurrent mark sweep
ProcessMarkStack:	Sum: 598.560ms 99% C.I. 0.001ms-9.500ms Avg: 1.330ms Max: 13.155ms
MarkConcurrentRoots:	Sum: 211.539ms 99% C.I. 1us-3900us Avg: 705.130us Max: 4370us
UpdateAndMarkImageModUnionTable:	Sum: 106.538ms 99% C.I. 46us-2287.500us Avg: 710.253us Max: 2820us
SweepMallocSpace:	Sum: 33.605ms 99% C.I. 1us-825us Avg: 112.016us Max: 1042us
MarkRootsCheckpoint:	Sum: 25.021ms 99% C.I. 17us-241.666us Avg: 83.403us Max: 924us
ImageModUnionClearCards:	Sum: 20.311ms 99% C.I. 44us-275us Avg: 67.703us Max: 1495us
SweepLargeObjects:	Sum: 17.822ms 99% C.I. 4.687us-725us Avg: 118.813us Max: 1814us
ScanGrayAllocSpaceObjects:	Sum: 10.457ms 99% C.I. 0.362us-197.499us Avg: 34.856us Max: 247us
AllocSpaceClearCards:	Sum: 8.861ms 99% C.I. 0.260us-95.652us Avg: 14.768us Max: 371us
ScanGrayImageSpaceObjects:	Sum: 8.757ms 99% C.I. 43us-162.500us Avg: 58.380us Max: 189us
(Paused)ScanGrayImageSpaceObjects:	Sum: 8.668ms 99% C.I. 43us-212.500us Avg: 57.786us Max: 229us
MarkNonThreadRoots:	Sum: 8.566ms 99% C.I. 8us-123us Avg: 28.553us Max: 123us
MarkAllocStackAsLive:	Sum: 7.689ms 99% C.I. 16us-300us Avg: 51.260us Max: 596us
SweepSystemWeaks:	Sum: 7.029ms 99% C.I. 7us-262.500us Avg: 46.860us Max: 284us
FinishPhase:	Sum: 6.923ms 99% C.I. 20us-112.500us Avg: 46.153us Max: 120us
ReMarkRoots:	Sum: 5.263ms 99% C.I. 13us-91us Avg: 35.086us Max: 91us
(Paused)ScanGrayAllocSpaceObjects:	Sum: 3.966ms 99% C.I. 0.260us-97.727us Avg: 13.220us Max: 184us
EnqueueFinalizerReferences:	Sum: 1.765ms 99% C.I. 1us-53us Avg: 11.766us Max: 53us
SwapBitmaps:	Sum: 1.482ms 99% C.I. 5us-29us Avg: 9.880us Max: 29us
ProcessCards:	Sum: 1.326ms 99% C.I. 2us-18us Avg: 4.420us Max: 18us
(Paused)PausePhase:	Sum: 1.297ms 99% C.I. 5us-31us Avg: 8.646us Max: 31us
MarkingPhase:	Sum: 1.204ms 99% C.I. 4us-41us Avg: 8.026us Max: 41us
PreCleanCards:	Sum: 1.157ms 99% C.I. 4us-28us Avg: 7.713us Max: 28us
ReclaimPhase:	Sum: 767us 99% C.I. 3us-15us Avg: 5.113us Max: 15us
RevokeAllThreadLocalAllocationStacks:	Sum: 710us 99% C.I. 2us-62.500us Avg: 4.733us Max: 65us
Sweep:	Sum: 605us 99% C.I. 2us-16us Avg: 4.033us Max: 16us
ZygoteModUnionClearCards:	Sum: 432us 99% C.I. 1us-8us Avg: 1.479us Max: 8us
MarkRoots:	Sum: 412us 99% C.I. 2us-14us Avg: 2.746us Max: 14us
SweepZygoteSpace:	Sum: 377us 99% C.I. 1us-34us Avg: 2.582us Max: 34us
ProcessReferences:	Sum: 336us 99% C.I. 1us-6us Avg: 2.240us Max: 6us
RecursiveMark:	Sum: 302us 99% C.I. 1us-5us Avg: 2.013us Max: 5us
InitializePhase:	Sum: 269us 99% C.I. 0.250us-17us Avg: 1.793us Max: 17us
BindBitmaps:	Sum: 194us 99% C.I. 0.250us-6us Avg: 1.293us Max: 6us
SwapStacks:	Sum: 141us 99% C.I. 250ns-3000ns Avg: 940ns Max: 3000ns
ScanGrayZygoteSpaceObjects:	Sum: 60us 99% C.I. 250ns-4000ns Avg: 410ns Max: 4000ns
(Paused)ScanGrayZygoteSpaceObjects:	Sum: 32us 99% C.I. 250ns-10000ns Avg: 219ns Max: 10000ns
FindDefaultSpaceBitmap:	Sum: 17us 99% C.I. 250ns-11000ns Avg: 113ns Max: 11000ns
UnBindBitmaps:	Sum: 16us 99% C.I. 250ns-2000ns Avg: 106ns Max: 2000ns
(Paused)ProcessMarkStack:	Sum: 3us 99% C.I. 250ns-1000ns Avg: 20ns Max: 1000ns
PreSweepingGcVerification:	Sum: 1us 99% C.I. 250ns-1000ns Avg: 6ns Max: 1000ns
Done Dumping histograms 
concurrent mark sweep paused:	Sum: 21.822ms 99% C.I. 90us-409us Avg: 145.480us Max: 409us
concurrent mark sweep total time: 1.102s mean time: 7.349ms
concurrent mark sweep freed: 90735 objects with total size 6MB
concurrent mark sweep throughput: 82336.7/s / 6MB/s
Start Dumping histograms for 2499 iterations for partial concurrent mark sweep
ProcessMarkStack:	Sum: 41.733s 99% C.I. 0.006ms-30.344ms Avg: 5.566ms Max: 42.907ms
SweepLargeObjects:	Sum: 11.516s 99% C.I. 1.274ms-12.650ms Avg: 4.608ms Max: 18.026ms
MarkRootsCheckpoint:	Sum: 9.407s 99% C.I. 0.203ms-5.900ms Avg: 1.882ms Max: 14.363ms
MarkConcurrentRoots:	Sum: 4.483s 99% C.I. 3us-3533.500us Avg: 897.020us Max: 6220us
UpdateAndMarkImageModUnionTable:	Sum: 2.560s 99% C.I. 0.627ms-3.750ms Avg: 1.024ms Max: 10.810ms
SweepMallocSpace:	Sum: 2.372s 99% C.I. 7us-5067.999us Avg: 474.702us Max: 38904us
ScanGrayAllocSpaceObjects:	Sum: 1.500s 99% C.I. 2us-1880.199us Avg: 300.209us Max: 5825us
(Paused)ScanGrayObjects:	Sum: 951.615ms 99% C.I. 144us-2601us Avg: 380.798us Max: 7430us
ReMarkRoots:	Sum: 813.652ms 99% C.I. 151.750us-1375.250us Avg: 325.591us Max: 4762us
AllocSpaceClearCards:	Sum: 536.128ms 99% C.I. 2us-362.583us Avg: 53.634us Max: 3362us
FinishPhase:	Sum: 503.310ms 99% C.I. 59us-2301us Avg: 201.404us Max: 8852us
MarkNonThreadRoots:	Sum: 488.670ms 99% C.I. 19us-1000.999us Avg: 97.773us Max: 6037us
ImageModUnionClearCards:	Sum: 446.906ms 99% C.I. 46us-883.499us Avg: 89.416us Max: 9189us
EnqueueFinalizerReferences:	Sum: 331.540ms 99% C.I. 28us-1350.500us Avg: 132.669us Max: 4814us
MarkAllocStackAsLive:	Sum: 222.864ms 99% C.I. 22us-1150.166us Avg: 89.181us Max: 4937us
SweepSystemWeaks:	Sum: 216.490ms 99% C.I. 50.034us-443.812us Avg: 86.630us Max: 3518us
ScanGrayImageSpaceObjects:	Sum: 192.995ms 99% C.I. 45us-487.541us Avg: 77.228us Max: 3329us
RevokeAllThreadLocalAllocationStacks:	Sum: 103.496ms 99% C.I. 9us-378.642us Avg: 41.414us Max: 7707us
UpdateAndMarkZygoteModUnionTable:	Sum: 68.193ms 99% C.I. 18us-94.802us Avg: 27.288us Max: 1099us
MarkingPhase:	Sum: 40.578ms 99% C.I. 9us-49.990us Avg: 16.237us Max: 941us
SwapBitmaps:	Sum: 38.098ms 99% C.I. 6us-90.403us Avg: 15.245us Max: 4437us
PreCleanCards:	Sum: 34.364ms 99% C.I. 6us-49.949us Avg: 13.751us Max: 3217us
ProcessReferences:	Sum: 31.920ms 99% C.I. 3us-86.557us Avg: 12.773us Max: 582us
ProcessCards:	Sum: 29.285ms 99% C.I. 2us-49.819us Avg: 5.859us Max: 404us
(Paused)PausePhase:	Sum: 26.943ms 99% C.I. 5us-49.929us Avg: 10.781us Max: 478us
Sweep:	Sum: 21.916ms 99% C.I. 3us-49.889us Avg: 8.769us Max: 2454us
ReclaimPhase:	Sum: 18.988ms 99% C.I. 4us-49.849us Avg: 7.598us Max: 307us
MarkRoots:	Sum: 13.916ms 99% C.I. 2us-49.929us Avg: 5.568us Max: 299us
ZygoteModUnionClearCards:	Sum: 10.433ms 99% C.I. 1us-49.769us Avg: 2.087us Max: 97us
RecursiveMark:	Sum: 5.931ms 99% C.I. 1us-49.789us Avg: 2.373us Max: 74us
BindBitmaps:	Sum: 5.627ms 99% C.I. 0.250us-38us Avg: 2.251us Max: 38us
InitializePhase:	Sum: 5.167ms 99% C.I. 0.250us-49.769us Avg: 2.067us Max: 69us
SwapStacks:	Sum: 3.936ms 99% C.I. 0.250us-25us Avg: 1.575us Max: 25us
UnBindBitmaps:	Sum: 3.560ms 99% C.I. 0.250us-32us Avg: 1.424us Max: 32us
ScanGrayZygoteSpaceObjects:	Sum: 2.685ms 99% C.I. 0.250us-49.769us Avg: 1.074us Max: 128us
(Paused)ProcessMarkStack:	Sum: 1.108ms 99% C.I. 250ns-25000ns Avg: 443ns Max: 25000ns
SweepZygoteSpace:	Sum: 902us 99% C.I. 250ns-49769ns Avg: 360ns Max: 212000ns
FindDefaultSpaceBitmap:	Sum: 191us 99% C.I. 250ns-25000ns Avg: 76ns Max: 25000ns
PreSweepingGcVerification:	Sum: 99us 99% C.I. 250ns-20000ns Avg: 39ns Max: 20000ns
Done Dumping histograms 
partial concurrent mark sweep paused:	Sum: 2.173s 99% C.I. 321us-5313.124us Avg: 869.781us Max: 8372us
partial concurrent mark sweep total time: 78.746s mean time: 31.511ms
partial concurrent mark sweep freed: 6094114 objects with total size 27GB
partial concurrent mark sweep throughput: 77389.5/s / 354MB/s
Start Dumping histograms for 2501 iterations for sticky concurrent mark sweep
SweepArray:	Sum: 12.287s 99% C.I. 1.800ms-12.866ms Avg: 4.913ms Max: 20.363ms
MarkRootsCheckpoint:	Sum: 7.432s 99% C.I. 0.202ms-4.919ms Avg: 1.485ms Max: 10.187ms
MarkConcurrentRoots:	Sum: 3.816s 99% C.I. 2us-3174.750us Avg: 762.901us Max: 6134us
ScanGrayAllocSpaceObjects:	Sum: 2.670s 99% C.I. 2.689us-2570.285us Avg: 266.977us Max: 25798us
FreeList:	Sum: 1.634s 99% C.I. 2us-1028.571us Avg: 151.349us Max: 10686us
(Paused)ScanGrayObjects:	Sum: 832.939ms 99% C.I. 145us-1475us Avg: 333.175us Max: 10565us
MarkingPhase:	Sum: 694.864ms 99% C.I. 165us-806.187us Avg: 277.834us Max: 2675us
ReMarkRoots:	Sum: 692.726ms 99% C.I. 70.918us-874.750us Avg: 276.979us Max: 2375us
AllocSpaceClearCards:	Sum: 542.476ms 99% C.I. 0.383us-387.449us Avg: 54.225us Max: 2088us
ScanGrayImageSpaceObjects:	Sum: 409.174ms 99% C.I. 43us-574.750us Avg: 81.802us Max: 2784us
ImageModUnionClearCards:	Sum: 398.393ms 99% C.I. 44us-599.750us Avg: 79.646us Max: 3243us
ProcessMarkStack:	Sum: 210.760ms 99% C.I. 0.506us-699.700us Avg: 21.069us Max: 7653us
SweepSystemWeaks:	Sum: 203.111ms 99% C.I. 50.011us-334.949us Avg: 81.211us Max: 1292us
MarkNonThreadRoots:	Sum: 190.691ms 99% C.I. 12us-139.980us Avg: 38.122us Max: 1627us
FinishPhase:	Sum: 113.743ms 99% C.I. 25us-231.187us Avg: 45.479us Max: 2172us
RevokeAllThreadLocalAllocationStacks:	Sum: 92.064ms 99% C.I. 3us-437.375us Avg: 36.810us Max: 2368us
ResetStack:	Sum: 78.887ms 99% C.I. 12us-329.900us Avg: 31.542us Max: 5436us
EnqueueFinalizerReferences:	Sum: 48.889ms 99% C.I. 2us-624.949us Avg: 19.547us Max: 2545us
ProcessCards:	Sum: 27.890ms 99% C.I. 2us-49.789us Avg: 5.575us Max: 354us
PreCleanCards:	Sum: 23.636ms 99% C.I. 4us-49.909us Avg: 9.450us Max: 333us
(Paused)PausePhase:	Sum: 23.590ms 99% C.I. 5us-49.849us Avg: 9.432us Max: 81us
ReclaimPhase:	Sum: 19.017ms 99% C.I. 3us-49.909us Avg: 7.603us Max: 1138us
SwapBitmaps:	Sum: 18.063ms 99% C.I. 3us-53.535us Avg: 7.222us Max: 331us
ProcessReferences:	Sum: 13.978ms 99% C.I. 2us-49.969us Avg: 5.588us Max: 795us
ScanGrayZygoteSpaceObjects:	Sum: 12.908ms 99% C.I. 0.250us-49.819us Avg: 2.580us Max: 356us
MarkRoots:	Sum: 12.404ms 99% C.I. 2us-49.829us Avg: 4.959us Max: 357us
ZygoteModUnionClearCards:	Sum: 8.276ms 99% C.I. 0.250us-49.759us Avg: 1.654us Max: 55us
RecordFree:	Sum: 5.764ms 99% C.I. 0.250us-49.789us Avg: 2.304us Max: 58us
BindBitmaps:	Sum: 5.400ms 99% C.I. 0.250us-24us Avg: 2.159us Max: 24us
InitializePhase:	Sum: 4.977ms 99% C.I. 0.250us-28us Avg: 1.990us Max: 28us
UnBindBitmaps:	Sum: 4.919ms 99% C.I. 1us-26us Avg: 1.966us Max: 26us
ForwardSoftReferences:	Sum: 3.932ms 99% C.I. 0.250us-49.809us Avg: 1.572us Max: 489us
SwapStacks:	Sum: 3.894ms 99% C.I. 0.250us-49.829us Avg: 1.556us Max: 79us
FindDefaultSpaceBitmap:	Sum: 3.443ms 99% C.I. 0.250us-49.789us Avg: 1.376us Max: 95us
(Paused)ProcessMarkStack:	Sum: 1.107ms 99% C.I. 250ns-49789ns Avg: 442ns Max: 154000ns
PreSweepingGcVerification:	Sum: 174us 99% C.I. 250ns-17000ns Avg: 69ns Max: 17000ns
(Paused)ScanGrayImageSpaceObjects:	Sum: 49us 99% C.I. 49us-49us Avg: 49us Max: 49us
(Paused)ScanGrayAllocSpaceObjects:	Sum: 25us 99% C.I. 0.250us-25us Avg: 12.500us Max: 25us
(Paused)ScanGrayZygoteSpaceObjects:	Sum: 0 99% C.I. 0ns-0ns Avg: 0ns Max: 0ns
Done Dumping histograms 
sticky concurrent mark sweep paused:	Sum: 1.714s 99% C.I. 127us-2561.875us Avg: 685.607us Max: 10953us
sticky concurrent mark sweep total time: 32.544s mean time: 13.012ms
sticky concurrent mark sweep freed: 9595329 objects with total size 31GB
sticky concurrent mark sweep throughput: 294842/s / 998MB/s
Total time spent in GC: 112.393s
Mean GC size throughput: 7MB/s
Mean GC object throughput: 139894 objects/s
Total number of allocations 16193973
Total bytes allocated 934MB
Free memory 5MB
Free memory until GC 5MB
Free memory until OOME 142MB
Total memory 55MB
Max memory 192MB
Total mutator paused time: 3.910s
Total time waiting for GC to complete: 138.681ms

DALVIK THREADS (45):
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 obj=0x74a26700 self=0xb7a67300
  | sysTid=3298 nice=0 cgrp=default sched=0/0 handle=0xb6fc4bec
  | state=D schedstat=( 189737241764 44454468940 671065 ) utm=14436 stm=4537 core=2 HZ=100
  | stack=0xbe038000-0xbe03a000 stackSize=8MB
  | held mutexes=
  native: #00 pc 0003d990  /system/lib/libc.so (fdatasync+12)
  native: #01 pc 0001aedb  /system/lib/libsqlite.so (???)
  native: #02 pc 00007517  /system/lib/libsqlite.so (???)
  native: #03 pc 00014835  /system/lib/libsqlite.so (???)
  native: #04 pc 0002852f  /system/lib/libsqlite.so (???)
  native: #05 pc 000286d9  /system/lib/libsqlite.so (???)
  native: #06 pc 0002a19d  /system/lib/libsqlite.so (???)
  native: #07 pc 00037a4b  /system/lib/libsqlite.so (???)
  native: #08 pc 0003ad53  /system/lib/libsqlite.so (sqlite3_step+426)
  native: #09 pc 00075009  /system/lib/libandroid_runtime.so (???)
  native: #10 pc 000b370b  /data/dalvik-cache/arm/system@framework@boot.oat (Java_android_database_sqlite_SQLiteConnection_nativeExecute__JJ+110)
  at android.database.sqlite.SQLiteConnection.nativeExecute(Native method)
  at android.database.sqlite.SQLiteConnection.execute(SQLiteConnection.java:555)
  at android.database.sqlite.SQLiteSession.endTransactionUnchecked(SQLiteSession.java:437)
  at android.database.sqlite.SQLiteSession.endTransaction(SQLiteSession.java:401)
  at android.database.sqlite.SQLiteDatabase.endTransaction(SQLiteDatabase.java:522)
  at org.greenrobot.greendao.database.StandardDatabase.endTransaction(:-1)
  at org.greenrobot.greendao.AbstractDao.deleteByKey(:-1)
  at org.greenrobot.greendao.AbstractDao.delete(:-1)
  at com.reconova.visitor.b.b.b.a.a(:-1)
  at com.reconova.visitor.b.b.b.d.a(:-1)
  at com.reconova.visitor.register.la.a(:-1)
  at com.reconova.visitor.register.la.onSuccess(:-1)
  at com.reconova.visitor.b.a.b.c.o.a(:-1)
  at com.reconova.visitor.b.a.b.c.o.onSuccess(:-1)
  at com.reconova.visitor.b.a.b.c.B.a(:-1)
  at com.reconova.visitor.b.a.b.c.B.a(:-1)
  at com.reconova.visitor.b.a.b.c.z.onSuccess(:-1)
  at io.reactivex.internal.operators.single.SingleObserveOn$ObserveOnSingleObserver.run(:-1)
  at io.reactivex.android.schedulers.HandlerScheduler$ScheduledRunnable.run(:-1)
  at android.os.Handler.handleCallback(Handler.java:739)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:135)
  at android.app.ActivityThread.main(ActivityThread.java:5280)
  at java.lang.reflect.Method.invoke!(Native method)
  at java.lang.reflect.Method.invoke(Method.java:372)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:963)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:758)

"Heap thread pool worker thread 2" prio=5 tid=2 Native (still starting up)
  | group="" sCount=1 dsCount=0 obj=0x0 self=0xb7a846a0
  | sysTid=3306 nice=0 cgrp=default sched=0/0 handle=0xb7a6d928
  | state=S schedstat=( 496102460 155821197 5637 ) utm=29 stm=20 core=3 HZ=100
  | stack=0xb4821000-0xb4823000 stackSize=1020KB
  | held mutexes=
  native: #00 pc 00012960  /system/lib/libc.so (syscall+28)
......
```

可以看出，首先显示的是应用的一些基本信息，这部分可以不用重点关注，需要重点关注的是接下去各线程在发生ANR时的线程堆栈信息。例子中的main线程在anr是运行的堆栈是org.greenrobot.greendao.AbstractDao.deleteByKey，这里看起来像是因为在删除数据库时发生了阻塞，这就需要继续到看堆栈的下层函数，可以发现是com.reconova.visitor.b.b.b.a.a(:-1)，这表明app进行过混淆，需要在build/outputs/mapping文件夹下找到mapping文件进行对应的分析了。可能在这里找到了对应的代码也不一定能够找出产生ANR的原因，那就需要在观察其他线程的堆栈以及进行代码分析来寻找anr原因。

当然了，简单的ANR是比较容易定位的，只要找到堆栈信息就可以直接定位到具体原因了，而复杂的anr问题就需要从多方面进行分析。 