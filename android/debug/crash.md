# 应用崩溃及处理

在开发安卓应用程序时不可避免的会出现崩溃，在发生崩溃时要如何定位并解决呢？

首先要明确崩溃的类型，安卓的崩溃大致可以分为三类：

1. java层崩溃
2. ANR
3. native崩溃

在发生崩溃时首先要搞清楚是什么类型的崩溃，在明白是什么类型崩溃之后就可以开始分析这几类崩溃了：

## java层崩溃

java层崩溃一般都是由于发生运行时异常而导致崩溃，崩溃发生的堆栈基本上都打印在日志里面了，一般只要在发生崩溃之后查看logcat日志就可以很快的定位到异常原因。

可以监听全局未捕获异常，使用`Thread.setDefaultUncaughtExceptionHandler()`方法监听未捕获异常，在其中获取崩溃的堆栈信息，并将其保存起来或者上传到服务器收集，其基本实现如下：

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

在捕获到崩溃时，还可以保存更多进程相关的信息，比如各线程堆栈、内存使用情况等等，以便开发人员定位问题

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

### ANR定位

#### FileObserver

在安卓版本为5.0以下的设备上，可以通过`FileObserver`监听`/data/anr/`来获取anr事件

#### Looper

[BlockCanary](https://github.com/markzhai/AndroidPerformanceMonitor)就使用Looper监控主线程卡顿问题，在Looper中有一个Printer打印日志，他在每隔Message处理前后被调用，可以使用Printer来检测是否发生ANR，`Looper.loop()`方法如下：

```java
public static void loop() {
    // ....

    for (;;) {
        // ...

        // This must be in a local variable, in case a UI event sets the logger
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                            msg.callback + ": " + msg.what);
        }

        // ...
        msg.target.dispatchMessage(msg);
        // ...

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
       // ...
    }
}
```

设置代码如下：

```java
public class BlockDetectByPrinter {
    private static LogMonitor logMonitor;

    public static void start() {
        if (logMonitor == null) {
            logMonitor = new LogMonitor();
        }
        Looper.getMainLooper().setMessageLogging(new Printer() {
            //分发和处理消息开始前的log
            private static final String START = ">>>>> Dispatching";
            //分发和处理消息结束后的log
            private static final String END = "<<<<< Finished";

            @Override
            public void println(String x) {
                if (logMonitor == null) {
                    return;
                }

                if (x.startsWith(START)) {
                    //开始计时
                    logMonitor.startMonitor();
                }
                if (x.startsWith(END)) {
                    //结束计时，并计算出方法执行时间
                    logMonitor.removeMonitor();
                }
            }
        });
    }

    public static void stop() {
        Looper.getMainLooper().setMessageLogging(null);
        if (logMonitor != null) {
            logMonitor.destroy();
        }
        logMonitor = null;
    }
}

public class LogMonitor {
    private static final String TAG = "LogMonitor";
    private Handler mIoHandler;
    private HandlerThread mLogThread;
    //方法耗时的卡口,300毫秒
    private static final long TIME_BLOCK = 100L;

    public LogMonitor() {
        mLogThread = new HandlerThread("log");
        mLogThread.start();
        mIoHandler = new Handler(mLogThread.getLooper());
    }

    private static Runnable mLogRunnable = new Runnable() {
        @Override
        public void run() {
            //打印出执行的耗时方法的栈消息
            StringBuilder sb = new StringBuilder();
            StackTraceElement[] stackTrace = Looper.getMainLooper().getThread().getStackTrace();
            for (StackTraceElement s : stackTrace) {
                sb.append(s.toString());
                sb.append("\n");
            }
            MLog.i(TAG, sb.toString());
        }
    };


    /**
     * 开始计时
     */
    public void startMonitor() {
        mIoHandler.postDelayed(mLogRunnable, TIME_BLOCK);
    }

    /**
     * 停止计时
     */
    public void removeMonitor() {
        mIoHandler.removeCallbacks(mLogRunnable);
    }

    public void destroy() {
        mLogThread.quit();
    }
}
```



Printer收到`>>>>> Dispatching`时就是开始处理当前消息，此时启动一个延时任务，在收到`<<<<< Finished`时就将验视人物取消，如果发生了ANR，那么在超时的时候就会调用mLogRunnable，把相关的堆栈打印出来，同时可以收集更多的调试信息。

可以通过设置LogMonitor中的TIME_BLOCK来控制超时时间。

这种方法有以下缺点：

1. 在Printer输出之前，有一段代码queue.next()也会可能发生ANR，从而造成无法监控到ANR。
2.  无法监控CPU资源紧张造成系统卡顿无法响应的ANR。

#### ANR-WatchDog

ANR-WatchDog是参考Android WatchDog机制，起了个单独线程向主线程发送一个变量+1的操作，然后休眠一定时间阈值（阈值可自定义，例如5s），休眠过后再判断变量是否已经+1，如果未完成则ANR告警。其原理如下图所示：

![image](.\imgs\format,png)



缺点：

-  无法保证能捕获所有ANR，对阈值设置影响捕获概率。如时间过长，中间发生的ANR则可能被遗漏掉。

#### 捕获SIGQUIT信号

所有的Java进程都运行在Java虚拟机上，当应用发生ANR时，其最终的一个环节是向目标进程通过kill -3，发送信号SIGNAL_QUIT。Android进程收到SIGQUIT时，虚拟机会捕获这个信号，并输出相应的traces信息，保存到 `/data/anr/traces.txt` 中。

因此我们可以通过`int sigaction(int __signal, const struct sigaction* __new_action, struct sigaction* __old_action);`函数注册`SIGQUIT`信号的处理程序，在信号处理中保存相应的trace信息。关于linux信号处理将在native崩溃中介绍。

爱奇艺的xCrash库实现了捕获SIGQUIT来获取ANR信息功能，可以直接使用xCrash。

## native崩溃

一般在发生java层崩溃时，都比较好定位问题，很快就能找到解决方案，但是在发生native崩溃时就比较棘手了。

### tombstone

在有root权限的设备可以获取Android生成的tombstone文件分析，或者可以使用bugreport获取tombstone。那么下面来介绍一下tombstone分析native崩溃。

下面是tombstone文件的崩溃信息：

```
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'Android/rk3288_box/rk3288_box:5.1.1/LMY49F/ty05151933:userdebug/test-keys'
Revision: '0'
ABI: 'arm'
pid: 3459, tid: 3477, name: pool-1-thread-1  >>> com.reconova.visitor <<<
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0xdeadbaad
Abort message: 'invalid address or address of corrupt block 0x20 passed to dlfree'
    r0 00000000  r1 a25efef0  r2 deadbaad  r3 00000000
    r4 00000020  r5 b6df20e8  r6 9bfec000  r7 00000028
    r8 00000000  r9 a25f03b4  sl a25f03b0  fp b80ab000
    ip b6dec64c  sp a25f02e8  lr b6dc0f53  pc b6dc0f54  cpsr 60070030
    d0  0000000000000000  d1  0000000000000000
    d2  6f20737365726466  d3  707572726f632072
    d4  7374657373612f73  d5  65646f6d5f77722f
    d6  4d2f3332312f736c  d7  6d2f2f736c65646f
    d8  0000000000000000  d9  0000000000000000
    d10 0000000000000000  d11 0000000000000000
    d12 0000000000000000  d13 0000000000000000
    d14 0000000000000000  d15 0000000000000000
    d16 0000000000000000  d17 0000000000000fff
    d18 bb030f493ac97969  d19 3d3a30c6bac2b209
    d20 3a3e3cdd3b47a9c0  d21 3d3a30c63ab1edd7
    d22 3c75a986bdb7df6f  d23 3a0540b23b9a3464
    d24 be638e39be638e39  d25 be638e39be638e39
    d26 3cb60b613cb60b61  d27 3cb60b613cb60b61
    d28 3c360b613c360b61  d29 3c360b613c360b61
    d30 bcb60b61bcb60b61  d31 bcb60b61bcb60b61
    scr 20000011

backtrace:
    #00 pc 0002bf54  /system/lib/libc.so (dlfree+1239)
    #01 pc 00012257  /system/lib/libc.so (free+10)
    #02 pc 001f07d9  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #03 pc 001e7968  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #04 pc 0002433c  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #05 pc 0001d5bc  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #06 pc 0002a5e8  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #07 pc 00026690  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #08 pc 0002263c  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #09 pc 00021a7c  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #10 pc 0002ed18  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so (Java_com_reconova_java_RecoFaceEx_initModule+136)
    #11 pc 00412e45  /data/dalvik-cache/arm/data@app@com.reconova.visitor-1@base.apk@classes.dex

stack:
         a25f02a8  0000006b  
         a25f02ac  f542ff37  
         a25f02b0  00000000  
         a25f02b4  00000020  
         a25f02b8  b6df20e8  
         a25f02bc  9bfec000  [anon:libc_malloc]
         a25f02c0  00000028  
         a25f02c4  b6daa481  /system/lib/libc.so (__libc_fatal_no_abort+16)
         a25f02c8  b6de27de  /system/lib/libc.so
         a25f02cc  a25f02dc  [stack:3477]
         a25f02d0  b6de5ffe  /system/lib/libc.so
         a25f02d4  b6dc0f53  /system/lib/libc.so (dlfree+1238)
         a25f02d8  b6de27de  /system/lib/libc.so
         a25f02dc  00000020  
         a25f02e0  b6de5ffe  /system/lib/libc.so
......
```

`pid: 3459, tid: 3477, name: pool-1-thread-1  >>> com.reconova.visitor <<<`

在文件开头，记录了崩溃锦成相关的信息，包括进程号（pid），崩溃线程id（tid）、线程名以及进程名。

接下去记录了崩溃的一些信息：

`signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0xdeadbaad`

包括signal类型、以及崩溃的地址信息以及一些寄存器的信息。对于应用开发者来说，直接拿到这些信息可能不会有直接的帮助，因此我们需要继续往下分析。

```
backtrace:
    #00 pc 0002bf54  /system/lib/libc.so (dlfree+1239)
    #01 pc 00012257  /system/lib/libc.so (free+10)
    #02 pc 001f07d9  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #03 pc 001e7968  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #04 pc 0002433c  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #05 pc 0001d5bc  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #06 pc 0002a5e8  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #07 pc 00026690  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #08 pc 0002263c  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #09 pc 00021a7c  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so
    #10 pc 0002ed18  /data/app/com.reconova.visitor-1/lib/arm/libRWFaceSDK.so (Java_com_reconova_java_RecoFaceEx_initModule+136)
```

这里打印了崩溃的堆栈，很多情况下，再这里就能直接找到崩溃的so库以及堆栈信息了。其中下面的表示在栈顶，也就是说离我们调用的方法越近，因此我们需要从底部往上去寻找我们熟悉的so库以及函数信息。

在这里，我们的应用使用了libRWFaceSDK.so这个库，因此我们要找到这个库的堆栈信息，这里直接打印出来的方法是Java_com_reconova_java_RecoFaceEx_initModule方法，在这里还不能直接定位，需要继续往上找。但是上面的都没有函数信息，这样就需要从第三列中获取出问题的地址来分析了。这里我们去libRWFaceSDK.so最底层的地址来分析，也就是第四行的001f07d9。

拿到这个地址之后，我们需要借助工具来继续定位，有一下几种选择，

**1、addr2line**，将崩溃的地址作为参数输入，可以找到对应的源代码信息

```powershell
D:\DJ_Software\Android\Ndk_Download\android-ndk-r10e-windows-x86_64\android-ndk-r10e\toolchains\arm-linux-androideabi-4.8\prebuilt\windows-x86_64\bin>arm-linux-androideabi-addr2line -f -e D:\flash_anr_pc\libipcsdk.so 00023d6f

LoopBuffWrite
??:?
```

这里需要注意的是，这里使用的so库需要是没有strip过的，因为strip过的so库，是查不到代码信息的，的出来的只有一对？号

**2、objdump**，将so库进行反编译，可以根据偏移地址得到更详细的函数调用上下文

```powershell
D:\DJ_Software\Android\Ndk_Download\android-ndk-r10e-windows-x86_64\android-ndk-r10e\toolchains\arm-linux-androideabi-4.8\prebuilt\windows-x86_64\arm-linux-androideabi\bin>objdump -S -D D:\flash_anr_pc\libipcsdk.so > D:\flash_anr_pc\deassmble_libipc.log
```

反汇编后可以直接在反汇编的文件中寻找偏移地址，进行上下文的分析，不过这样的分析需要身后的功底才好进行分析，这里其实也没有具体这样分析过，也没办法说的详细了。

### linux signal

由于native崩溃发生在机器指令运行层面，比如app中的so库、系统so库、jvm本身等。如果这部分程序发生了错误（比如除数为零、空指针异常等），kernel就会向app中对应的线程发送相应的信号（signal），用户态进程也可以发送 signal 终止其他进程或自身。这些信号主要分为两类：

**kernel信号**

SIGFPE: 除数为零。

SIGILL: 无法识别的 CPU 指令。

SIGSYS: 无法识别的系统调用（system call）。

SIGSEGV: 错误的虚拟内存地址访问。

SIGBUS: 错误的物理设备地址访问。

**用户态进程信号**

SIGABRT: 调用 abort() / kill() / tkill() / tgkill() 自杀，或被其他进程通过 kill() / tkill() / tgkill() 他杀。

**信号处理**

linux进程发生错误时，其状态转换如下：

![img](.\imgs\exception_status.jpg)

因此我们可以在native中使用**sigaction**函数添加信号处理函数来记录发生异常时程序的各种信息，下面介绍一下**sigaction**函数：

**一、函数原型：sigaction函数的功能是检查或修改与指定信号相关联的处理动作（可同时两种操作）**

```c
int sigaction(int signum, const struct sigaction *act,
                     struct sigaction *oldact);
```

signum参数指出要捕获的信号类型，act参数指定新的信号处理方式，oldact参数输出先前信号的处理方式（如果不为NULL的话）。

**二、 struct sigaction结构体介绍**

```c
struct sigaction {
    void (*sa_handler)(int);
    void (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t sa_mask;
    int sa_flags;
    void (*sa_restorer)(void);
}
```

- sa_handler此参数和signal()的参数handler相同，代表新的信号处理函数
- sa_mask 用来设置在处理该信号时暂时将sa_mask 指定的信号集搁置
- sa_flags 用来设置信号处理的其他相关操作，下列的数值可用。 
- SA_RESETHAND：当调用信号处理函数时，将信号的处理函数重置为缺省值SIG_DFL
- SA_RESTART：如果信号中断了进程的某个系统调用，则系统自动启动该系统调用
- SA_NODEFER ：一般情况下， 当信号处理函数运行时，内核将阻塞该给定信号。但是如果设置了 SA_NODEFER标记， 那么在该信号处理函数运行时，内核将不会阻塞该信号

由于可能有各种崩溃原因，比如栈溢出、内存耗尽、FD耗尽、Flash空间耗尽等，故在sa_handler中有许多注意事项，具体可以参考爱奇艺的xCrash，这里不再深入介绍了。

## 日志保存

有时在崩溃发生或者应用发生错误时，我们需要需要获取相关的日志才能深入定位问题，但是logcat日志只能保存一段时间，故需要将应用日志保存到本地，以上传到服务器，方便开发人员分析定位问题。这里有两种方式保存：

#### android-log4j保存

在build.gradle中添加如下依赖：

```groovy
implementation files('libs/log4j-1.2.17.jar')
implementation files('libs/android-logging-log4j-1.0.3.jar')
```

使用MLog替代系统的Log

```java
public class MLog {
    private static Logger logger;
    private static final long MAX_SIZE = 1024 * 1024L;// 单个日志文件容量1M
    private static final int MSG_MAX_LEN = 2048;// 单条日志的最大长度
    private static final int MAX_LOG_FILE = 10;// 最多生成几个日志文件

    static {
        configLog(Environment.getExternalStorageDirectory() + "/RecoGateData/RecoLogger/Log/", "reco_log", true);
        logger = Logger.getLogger("GateFace");
    }

    public static void v(String tag, String msg) {
        Log.v(tag, msg);
    }

    public static void d(String tag, String msg) {
        logger.debug(tag + ": " + subMsgIfNeed(msg));
    }

    public static void i(String tag, String msg) {
        logger.info(tag + ": " + subMsgIfNeed(msg));
    }

    public static void w(String tag, String msg) {
        logger.warn(tag + ": " + subMsgIfNeed(msg));
    }

    public static void w(String tag, String msg, Throwable tr) {
        logger.warn(tag + ": " + subMsgIfNeed(msg), tr);
    }

    public static void w(String tag, Throwable tr) {
        logger.warn(tag, tr);
    }

    public static void e(String tag, String msg) {
        logger.error(tag + ": " + subMsgIfNeed(msg));
    }

    public static void e(String tag, String msg, Throwable tr) {
        logger.error(tag + ": " + subMsgIfNeed(msg), tr);
    }

    private static String subMsgIfNeed(String msg) {
        if (msg.length() > MSG_MAX_LEN) {
            return msg.substring(0, MSG_MAX_LEN);
        }
        return msg;
    }

    /**
     * 生成日志对象
     *
     * @param filePath 日志输出路径
     * @param fileName 日志文件名
     * @param flag     true:在已存在log文件后面追加 false:新log覆盖以前的log
     */
    public static void configLog(String filePath, String fileName, boolean flag) {
        LogConfigurator logConfigurator = new LogConfigurator();
        logConfigurator.setFileName(filePath + fileName + ".log");
        logConfigurator.setRootLevel(Level.DEBUG);
        logConfigurator.setUseFileAppender(flag);
        logConfigurator.setLevel("org.apache", Level.ERROR);
        logConfigurator.setFilePattern("%d{yyyy-MM-dd HH:mm:ss.SS} " + +Process.myPid() + "-%t/%p/%m%n");// 设置日志文件格式
        logConfigurator.setMaxFileSize(MAX_SIZE);// 日志文件大小
        logConfigurator.setMaxBackupSize(MAX_LOG_FILE);// 最多生成日志文件数量
        logConfigurator.setImmediateFlush(true);
        logConfigurator.configure();
    }
}
```

#### logcat日志保存

在应用中启动一条线程执行logcat命令，并将输出的结果保存到指定目录中，其实现如下：

```kotlin
object LogcatSaver {
    private const val TAG = "LogcatSaver"

    private var process: Process? = null
    private var stop = false
    private var thread: Thread? = null

    fun startSave(logPath: String) {
        stopSave()

        LogFileWriter.init(logPath)

        Log.d(TAG, "startSave: ")
        stop = false
        thread {
            val command = "logcat -v threadtime"
            val isRoot = checkRoot()

            while (!stop && !Thread.currentThread().isInterrupted) {
                process = exec(command, isRoot)
                process?.let { p ->
                    var bufReader: BufferedReader? = null
                    try {
                        bufReader = BufferedReader(InputStreamReader(p.inputStream))
                        var line: String?
                        do {
                            line = bufReader.readLine()
                            line?.let { s ->
                                LogFileWriter.writeLine(s)
                            }
                        } while (line != null)
                    } catch (e: Exception) {
                        Log.e(TAG, "startSave: ", e)
                        if (e is InterruptedException) {
                            Thread.currentThread().interrupt()
                        }
                    } finally {
                        bufReader?.close()
                    }
                }
            }
        }
    }

    private fun stopSave() {
        Log.d(TAG, "stopSave: ")
        stop = true
        thread?.interrupt()
        process?.destroy()
        process = null
    }

    fun saveCrashLog(crashPath: String) {
        Log.d(TAG, "saveCrashLog: ")
        val crashLog = File(crashPath, "crash.log")
        if (crashLog.parentFile?.exists() == false) {
            crashLog.parentFile?.mkdirs()
        }

        LogFileWriter.backupLog(crashLog)

        val command = "logcat -v threadtime -d -f ${crashLog.absolutePath}"
        val isRoot = checkRoot()

        exec(command, isRoot)
    }

    private const val COMMAND_SU = "su"
    private const val COMMAND_SH = "sh"
    private const val COMMAND_EXIT = "exit\n"
    private const val COMMAND_NEW_LINE = "\n"
    private fun exec(command: String, isRoot: Boolean = false): Process? {
        Log.d(TAG, "exec: $command")

        var process: Process? = null
        try {
            process = Runtime.getRuntime().exec(if (isRoot) COMMAND_SU else COMMAND_SH)
            val os = DataOutputStream(process?.outputStream)
            // donnot use os.writeBytes(commmand), avoid chinese charset error
            os.write(command.toByteArray())
            os.writeBytes(COMMAND_NEW_LINE)
            os.flush()

            os.writeBytes(COMMAND_EXIT)
            os.flush()
        } catch (e: Exception) {
            Log.e(TAG, "exec: ", e)
        }

        return process
    }

    private fun checkRoot(): Boolean {
        val checkProcess = exec("echo root", true)

        checkProcess?.let { process ->
            val result = process.waitFor()

            val errMsg = readInputStream(process.errorStream)
            val stdMsg = readInputStream(process.inputStream)

            Log.d(TAG, "checkRoot: stdMsg $stdMsg")
            Log.e(TAG, "checkRoot: errMsg $errMsg")

            return result == 0
        }
        return false
    }

    private fun readInputStream(inputStream: InputStream): String {
        var bufReader: BufferedReader? = null
        val result = StringBuilder()

        try {
            bufReader = BufferedReader(InputStreamReader(inputStream))
            var line: String?
            do {
                line = bufReader.readLine()
                if (line?.isNotEmpty() == true) {
                    result.append(line).append("\n");
                }
            } while (line != null)
        } catch (e: Exception) {
            Log.e(TAG, "startSave: ", e)
        } finally {
            bufReader?.close()
        }

        return result.toString()
    }
}

object LogFileWriter {
    private const val TAG = "LogFileWriter"

    private var logcatPath = Environment.getExternalStorageDirectory().absolutePath + "/test/log/"
    private const val LOG_FILE = "logcat.log"
    private const val MAX_FILE_NUM = 10 // 最多保存十个logcat日志
    private const val MAX_FILE_SIZE = 10 * 1024 * 1024 // 每个日志文件最多保存10M

    private var file: File? = null

    fun init(logPath: String) {
        logcatPath = logPath
    }

    fun writeLine(line: String) {
        if (file == null) {
            file = File(logcatPath + LOG_FILE)
            file?.let { f ->
                if (f.parentFile?.exists() == false) {
                    f.parentFile?.mkdirs()
                }
            }
        }

        file?.let { f ->
            if (f.length() >= MAX_FILE_SIZE) {
                backupLog(f)
            }

            var bufWriter: BufferedWriter? = null
            try {
                bufWriter = BufferedWriter(OutputStreamWriter(FileOutputStream(f, true)))

                bufWriter.write(line)
                bufWriter.newLine()
                bufWriter.flush()
            } catch (e: IOException) {
                Log.e(TAG, "writeLine: ", e)
            } finally {
                bufWriter?.close()
            }
        }
    }

     fun backupLog(logFile: File) {
        for (i in MAX_FILE_NUM - 1 downTo 1) {
            val bakLog = File(logFile.absolutePath + "." + i)
            if (bakLog.exists()) {
                val nextLog = File(logFile.absolutePath + "." + (i + 1))
                if (nextLog.exists()) {
                    nextLog.delete()
                }
                bakLog.renameTo(nextLog)
            }
        }
        val firstBak = File(logFile.absolutePath + ".1")
        logFile.renameTo(firstBak)
    }
}
```

将logcat日志全部保存还能再有root权限的设备中看到系统的日志，在碰上一些和系统相关的bug就可以看到系统的日志来协助定位问题了。

## 第三方工具库

当然了，在发生崩溃的时候往往是在用户手机或者客户现场的各种Android设备上发生的，上面说的都是开发测试过程中可以使用的手段，放到正式环境中就可能效率比较低了，因此我们可以集成一些第三方框架。

### Bugly

集成腾讯的Bugly框架可以自动将崩溃信息上传到bugly后台，以便开发人员分析定位问题

### Breakpad

Breakpad是Google提供的一个native奔溃分析工具，这里推荐一篇博客：

https://www.jianshu.com/p/295ebf42b05b

### xCrash

[xCrash](https://github.com/iqiyi/xCrash)是爱奇艺开源的进程崩溃处理工具，它能够捕获Java层和native层的崩溃时间，并在指定目录中生成一个tombstone文件。开发者可以将生成的tombstone文件上传到服务器。

