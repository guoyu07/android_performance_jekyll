---
layout: post
title:  "StrictMode 详解"
date:   2014-04-24 23:44:45
categories: android
---

StrictMode类是Android 2.3 （API 9）引入的一个工具类，可以用来帮助开发者发现代码中的一些不规范的问题。比如，如果你在UI线程中进行了网络或者磁盘操作，StrictMode就会通过Log（logcat ）或者对话框的方式把信息提示给你，因为让你的UI线程处理这里操作会被认为是不规范的做法，可能会让你的应用变得比较卡顿。

官网文档：
[http://developer.android.com/reference/android/os/StrictMode.html](http://developer.android.com/reference/android/os/StrictMode.html)

<!--more-->

#如何启用 StrictMode

我们通常在 Activity 或者自定义的Application类中启动 StrictMode，代码如下：

{% highlight java  %}

 public void onCreate() {
     if (DEVELOPER_MODE) {
         StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                 .detectDiskReads()
                 .detectDiskWrites()
                 .detectNetwork()   // or .detectAll() for all detectable problems
                 .penaltyLog()
                 .build());
         StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                 .detectLeakedSqlLiteObjects()
                 .detectLeakedClosableObjects()
                 .penaltyLog()
                 .penaltyDeath()
                 .build());
     }
     super.onCreate();
 }

{% endhighlight %}

**注意：**我们只需要在app的开发版本下使用 StrictMode，线上版本避免使用 StrictMode，随意需要通过 诸如 DEVELOPER_MODE 这样的配置变量来进行控制。

下面我们举几个例子来说明 StrictMode 是如何发挥作用的。

代码1：
{% highlight java  %}
public class ActivitySimple extends Activity {

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		StrictMode.setThreadPolicy(new ThreadPolicy.Builder()
				.detectAll()
				.penaltyDialog() //弹出违规提示对话框
				.penaltyLog()	//在Logcat 中打印违规异常信息
				.build());
		
		this.testNetwork();
	}

	
	private void testNetwork() {
		try {
			URL url = new URL("http://www.baidu.com");
			HttpURLConnection conn = (HttpURLConnection) url.openConnection();
			conn.connect();
			BufferedReader reader = new BufferedReader(new InputStreamReader(
					conn.getInputStream()));
			String lines = null;
			StringBuffer sb = new StringBuffer();
			while ((lines = reader.readLine()) != null) {
				sb.append(lines);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
{% endhighlight %}


在这里例子中，我们在主线程（UI线程）中执行了网络请求，ThreadPolicy 策略中的 `detectAll()`方法 包含而来对这类违规操作的检查，同时我们通过 `penaltyDialog()` 和 `penaltyLog()` 两个方法将违规信息提示给开发者。

在运行这段代码是，我们会看到下图中的对话框提示：

<img src="/images/android-strict-mode/dialog.png" />

在LogCat 中我们会看到这样的日志信息：
<pre>
... D/StrictMode(26365): **StrictMode policy violation; ~duration=58 ms: android.os.StrictMode$StrictModeNetworkViolation: policy=63 violation=4**
... D/StrictMode(26365): 	at android.os.StrictMode$AndroidBlockGuardPolicy.onNetwork(StrictMode.java:1134)
... D/StrictMode(26365): 	at libcore.io.BlockGuardOs.recvfrom(BlockGuardOs.java:163)
... D/StrictMode(26365): 	at libcore.io.IoBridge.recvfrom(IoBridge.java:557)
... D/StrictMode(26365): 	at java.net.PlainSocketImpl.read(PlainSocketImpl.java:490)
... D/StrictMode(26365): 	at java.net.PlainSocketImpl.access$000(PlainSocketImpl.java:46)
...
（后面的部分省略）
</pre>


#StrictMode 详解

StrictMode 通过策略方式来让你自定义需要检查哪方面的问题。
主要有两中策略，一个时线程方策略（[ThreadPolicy](http://developer.android.com/reference/android/os/StrictMode.ThreadPolicy.html)），一个是VM方面的策略（[VmPolicy](http://developer.android.com/reference/android/os/StrictMode.VmPolicy.html)）。

* ThreadPolicy 主要用于发现在UI线程中是否有读写磁盘的操作，是否有网络操作，以及检查UI线程中调用的自定义代码是否执行得比较慢。

* VmPolicy，主要用于发现内存问题，比如 Activity内存泄露， SQL 对象内存泄露， 资源未释放，能够限定某个类的最大对象数。

##ThreadPolicy 详解

[StrictMode.ThreadPolicy.Builder](http://developer.android.com/reference/android/os/StrictMode.ThreadPolicy.Builder.html) 主要方法如下：

* detectNetwork() 用于检查UI线程中是否有网络请求操作，上面的代码的就是网络请求违规的问题。

* detectDiskReads() 和 detectDiskReads() 是磁盘读写检查，触发时会打印出如下日志（以 detectDiskReads() 为例）：
<pre>
... D/StrictMode(27429): StrictMode policy violation; ~duration=33 ms: android.os.StrictMode$StrictModeDiskReadViolation: policy=31 violation=2
... D/StrictMode(27429): 	at android.os.StrictMode$AndroidBlockGuardPolicy.onReadFromDisk(StrictMode.java:1118)
... D/StrictMode(27429): 	at libcore.io.BlockGuardOs.open(BlockGuardOs.java:106)
... D/StrictMode(27429): 	at java.io.File.createNewFile(File.java:941)
... D/StrictMode(27429): 	at com.ap.teststrictmode.ActivityTestDisk.testWriteDisk(ActivityTestDisk.java:51)
... D/StrictMode(27429): 	at com.ap.teststrictmode.ActivityTestDisk.onCreate(ActivityTestDisk.java:40)
... D/StrictMode(27429): 	at android.app.Activity.performCreate(Activity.java:5122)
... D/StrictMode(27429): 	at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1081)
... D/StrictMode(27429): 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2270)
...
（后面的部分省略）
</pre>

* detectCustomSlowCalls() 主要用于帮助开发者发现UI线程调用的那些方法执行得比较慢，要和 `StrictMode.noteSlowCall` 配合使用，`StrictMode.noteSlowCall` 只有通过 `StrictMode.noteSlowCall`用来标记“可能会”执行比较慢的方法，只有标记过的方法才能被检测到，日志中会记录方法的执行时间（比如 ~duration=2019 ms）。看下面的例子：
<br/>代码2：{% highlight java  %}
public class ActivityTestDetectCustomSlowCalls extends Activity {
	
	private TextView textView = null;
	
	private static boolean isStrictMode = false;
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		this.textView = (TextView) findViewById(R.id.text_view);
		this.textView.setText("In ActivityTestDetectCustomSlowCalls");
		if(! isStrictMode){
			StrictMode.setThreadPolicy(new ThreadPolicy.Builder()
					.detectCustomSlowCalls()
					.penaltyLog()
					.build());
			isStrictMode = true;
		}
		this.slowCall_1();
		this.slowCall_2();
	}

	/**
	 * 没有标记的方法
	 */
	private void slowCall_1(){
		//用来标记潜在执行比较慢的方法
		SystemClock.sleep(1000 * 2);
	}
	
	/**
	 * 标记过的方法
	 */
	private void slowCall_2(){
		//用来标记潜在执行比较慢的方法
		StrictMode.noteSlowCall("slowCall 2");
		SystemClock.sleep(1000 * 2);
	}
}
{% endhighlight %}<br/>在logcat 中我们只能看到和方法 `slowCall_2()`（因为通过`StrictMode.noteSlowCall()`标记过）相关的日志：<br/><pre>
...: D/StrictMode(1349): StrictMode policy violation; **~duration=2019 ms**: android.os.StrictMode$StrictModeCustomViolation: policy=24 violation=8 msg=slowCall 2
...: D/StrictMode(1349): 	at android.os.StrictMode$AndroidBlockGuardPolicy.onCustomSlowCall(StrictMode.java:1105)
...: D/StrictMode(1349): 	at android.os.StrictMode.noteSlowCall(StrictMode.java:1903)
...: D/StrictMode(1349): 	at com.ap.teststrictmode.ActivityTestDetectCustomSlowCalls.slowCall_2(ActivityTestDetectCustomSlowCalls.java:52)
...: D/StrictMode(1349): 	at com.ap.teststrictmode.ActivityTestDetectCustomSlowCalls.onCreate(ActivityTestDetectCustomSlowCalls.java:35)
...: D/StrictMode(1349): 	at android.app.Activity.performCreate(Activity.java:5122)
...: D/StrictMode(1349): 	at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1081)
...: D/StrictMode(1349): 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2270)
...: D/StrictMode(1349): 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2358)
...: D/StrictMode(1349): 	at android.app.ActivityThread.access$600(ActivityThread.java:156)
...: D/StrictMode(1349): 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1340)
...: D/StrictMode(1349): 	at android.os.Handler.dispatchMessage(Handler.java:99)
...: D/StrictMode(1349): 	at android.os.Looper.loop(Looper.java:153)
...
（后面的部分省略）
</pre>
<br/>当然你也可以在其他线程中使用 detectCustomSlowCalls()，但是没有什么实际意义，也看不到方法执行时间，比如：

{% highlight java  %}
public class ActivityTestDetectCustomSlowCalls extends Activity {
	
	private TextView textView = null;
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		this.textView = (TextView) findViewById(R.id.text_view);
		
		this.textView.setText("In ActivityTestDetectCustomSlowCalls");
		
		new Thread(){
			public void run() {
				StrictMode.setThreadPolicy(new ThreadPolicy.Builder()
					.detectCustomSlowCalls()
					.penaltyLog()
					.build());
				
				this.slowCallInCustomThread();
			};
			
			private void slowCallInCustomThread(){
				//用来标记潜在执行比较慢的方法
				StrictMode.noteSlowCall("slowCallInCustomThread");
				SystemClock.sleep(1000 * 2);
			}			
		}.start();;
	}
}
{% endhighlight %}

日志输出如下：

<pre>
...: D/StrictMode(2418): StrictMode policy violation: android.os.StrictMode$StrictModeCustomViolation: policy=24 violation=8 msg=slowCallInCustomThread
...: D/StrictMode(2418): 	at android.os.StrictMode$AndroidBlockGuardPolicy.onCustomSlowCall(StrictMode.java:1105)
...: D/StrictMode(2418): 	at android.os.StrictMode.noteSlowCall(StrictMode.java:1903)
...: D/StrictMode(2418): 	at com.ap.teststrictmode.ActivityTestDetectCustomSlowCalls$1.slowCallInCustomThread(ActivityTestDetectCustomSlowCalls.java:35)
...: D/StrictMode(2418): 	at com.ap.teststrictmode.ActivityTestDetectCustomSlowCalls$1.run(ActivityTestDetectCustomSlowCalls.java:30)
...
（后面的部分省略）
</pre>


* penaltyDeath()，当触发违规条件时，直接Crash掉当前应用程序。

* penaltyDeathOnNetwork()，当触发网络违规时，Crash掉当前应用程序。

* penaltyDialog()，触发违规时，显示对违规信息对话框。

* penaltyFlashScreen()，会造成屏幕闪烁，不过一般的设备可能没有这个功能。

* penaltyDropBox()，将违规信息记录到 dropbox 系统日志目录中（/data/system/dropbox），你可以通过如下命令进行插件：
<pre>
*adb shell dumpsys dropbox data_app_strictmode  --print*
</pre>
会得到如下的信息：
<pre>
2014-05-04 14:56:32 data_app_strictmode (text, 2627 bytes)
Process: com.ap.teststrictmode
Flags: 0x40a8be46
Package: com.ap.teststrictmode v1 (1.0)
Build: Xiaomi/pisces/pisces:4.2.1/JOP40D/JXCCNBA13.0:user/release-keys
System-App: false
Uptime-Millis: 66679049
Loop-Violation-Number: 10
Duration-Millis: 24
android.os.StrictMode$StrictModeNetworkViolation: policy=191 violation=4
        at android.os.StrictMode$AndroidBlockGuardPolicy.onNetwork(StrictMode.java:1136)
        at libcore.io.BlockGuardOs.recvfrom(BlockGuardOs.java:163)
        at libcore.io.IoBridge.recvfrom(IoBridge.java:513)
        at java.net.PlainSocketImpl.read(PlainSocketImpl.java:488)
        at java.net.PlainSocketImpl.access$000(PlainSocketImpl.java:46)
        at java.net.PlainSocketImpl$PlainSocketInputStream.read(PlainSocketImpl.java:240)
        at java.io.InputStream.read(InputStream.java:163)
        at java.io.BufferedInputStream.fillbuf(BufferedInputStream.java:142)
        at java.io.BufferedInputStream.read(BufferedInputStream.java:227)
        at libcore.io.Streams.readAsciiLine(Streams.java:201)
        at libcore.net.http.ChunkedInputStream.readChunkSize(ChunkedInputStream.java:77)
        at libcore.net.http.ChunkedInputStream.read(ChunkedInputStream.java:68)
        at java.io.InputStream.read(InputStream.java:163)
        at java.util.zip.InflaterInputStream.fill(InflaterInputStream.java:200)
        at java.util.zip.InflaterInputStream.read(InflaterInputStream.java:154)
        at java.util.zip.GZIPInputStream.read(GZIPInputStream.java:167)
...
</pre>

* permitCustomSlowCalls()、permitDiskReads ()、permitDiskWrites()、permitNetwork： 如果你想关闭某一项检测，可以使用对应的permit*方法。


##VMPolicy 详解

* detectActivityLeaks() 用户检查 Activity 的内存泄露情况，比如下面的代码：
{% highlight java %}
public class ActivityTestActivityLeaks extends Activity {
	
	private static boolean isStrictMode = false;
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		if(! isStrictMode){
			
			StrictMode.setVmPolicy(new VmPolicy.Builder()
			.detectActivityLeaks()
			.penaltyLog()
			.build());
			isStrictMode = true;
		}
		
	    new Thread() {
		      @Override
		      public void run() {
		        while (true) {
		        	
		          SystemClock.sleep(1000);
		        }
		      }
		}.start();
	}
}
{% endhighlight %}

我们反复旋转屏幕就会输出如下信息（重点在 instances=4; limit=1 这一行）：
<pre>
...: E/StrictMode(4784): class com.ap.teststrictmode.ActivityTestActivityLeaks; instances=4; limit=1
...: E/StrictMode(4784): android.os.StrictMode$InstanceCountViolation: class com.ap.teststrictmode.ActivityTestActivityLeaks; instances=4; limit=1
...: E/StrictMode(4784): 	at android.os.StrictMode.setClassInstanceLimit(StrictMode.java:1)
</pre>

这时因为，我们在Activity中创建了一个Thread匿名内部类，而匿名内部类隐式持有外部类的引用。而每次旋转屏幕是，Android会新创建一个Activity，而原来的Activity实例又被我们启动的匿名内部类线程持有，所以不会释放，从日志上看，当先系统中该Activty有4个实例，而限制是只能创建1各实例。我们不断翻转屏幕，instances 的个数还会持续增加。

* detectLeakedClosableObjects() 和 detectLeakedSqlLiteObjects()，资源没有正确关闭时回触发，比如下面的代码：
{% highlight java %}
	
public class MainActivityTestDetectLeakedClosableObjects extends Activity {
	
	private static boolean isStrictMode = false;
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		
		if(! isStrictMode){
			StrictMode.setVmPolicy(new VmPolicy.Builder()
			.detectLeakedClosableObjects()
			.penaltyLog()
			.build());
			isStrictMode = true;
		}
		
		File newxmlfile = new File(Environment.getExternalStorageDirectory(), "aaa.txt");
		try {
			newxmlfile.createNewFile();
			FileWriter fw = new FileWriter(newxmlfile);
			fw.write("aaaaaaaaaaa");
			//fw.close(); 我们在这里故意没有关闭 fw
		} catch (IOException e) {
			e.printStackTrace();
		}

	}
}

{% endhighlight %}

会产生如下异常信息：

<pre>
... E/StrictMode(22056): A resource was acquired at attached stack trace but never released. See java.io.Closeable for information on avoiding resource leaks.
... E/StrictMode(22056): java.lang.Throwable: Explicit termination method 'close' not called
... E/StrictMode(22056): 	at dalvik.system.CloseGuard.open(CloseGuard.java:184)
... E/StrictMode(22056): 	at java.io.FileOutputStream.<init>(FileOutputStream.java:90)
... E/StrictMode(22056): 	at java.io.FileOutputStream.<init>(FileOutputStream.java:73)
... E/StrictMode(22056): 	at java.io.FileWriter.<init>(FileWriter.java:42)
... E/StrictMode(22056): 	at com.ap.teststrictmode.MainActivityTestDetectLeakedClosableObjects.onCreate(MainActivityTestDetectLeakedClosableObjects.java:44)
... E/StrictMode(22056): 	at android.app.Activity.performCreate(Activity.java:5122)
... E/StrictMode(22056): 	at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1081)
... E/StrictMode(22056): 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2270)
... E/StrictMode(22056): 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2358)
... E/StrictMode(22056): 	at android.app.ActivityThread.handleRelaunchActivity(ActivityThread.java:3865)
（后面的省略）
</pre>

detectLeakedSqlLiteObjects() 和 detectLeakedClosableObjects()的用法类似，只不过是用来检查  SQLiteCursor 或者 其他 SQLite 对象是否被正确关闭。

* detectLeakedRegistrationObjects() 用来检查 BroadcastReceiver 或者 ServiceConnection 注册类对象是否被正确释放，看下面的代码

{% highlight java %}
public class ActivityTestLeakedRegistrationObjects extends Activity {
	private TextView textView = null;
	private static boolean isStrictMode = false;
	private MyReceiver receiver = null;
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		this.textView = (TextView) findViewById(R.id.text_view);
		
		this.textView.setText("In ActivityTestLeakedRegistrationObjects");

		
		if(! isStrictMode){
			StrictMode.setVmPolicy(new VmPolicy.Builder()
			.detectLeakedRegistrationObjects()
			.penaltyLog()
			.build());
			isStrictMode = true;
		}
		
		this.receiver = new MyReceiver();  
		IntentFilter filter = new IntentFilter();  
		filter.addAction("android.intent.action.MY_BROADCAST"); 
		registerReceiver(this.receiver, filter); 
	}
	
	@Override
	protected void onDestroy() {
		super.onDestroy();
	}
}
{% endhighlight %}

输入信息如下：

<pre>
...: E/ActivityThread(24442): Activity com.ap.teststrictmode.ActivityTestLeakedRegistrationObjects has leaked IntentReceiver com.ap.teststrictmode.MyReceiver@41f1f128 that was originally registered here. Are you missing a call to unregisterReceiver()?
...: E/ActivityThread(24442): android.app.IntentReceiverLeaked: Activity com.ap.teststrictmode.ActivityTestLeakedRegistrationObjects has leaked IntentReceiver com.ap.teststrictmode.MyReceiver@41f1f128 that was originally registered here. Are you missing a call to unregisterReceiver()?
...: E/ActivityThread(24442): 	at android.app.LoadedApk$ReceiverDispatcher.<init>(LoadedApk.java:825)
...: E/ActivityThread(24442): 	at android.app.LoadedApk.getReceiverDispatcher(LoadedApk.java:596)
...: E/ActivityThread(24442): 	at android.app.ContextImpl.registerReceiverInternal(ContextImpl.java:1388)
...: E/ActivityThread(24442): 	at android.app.ContextImpl.registerReceiver(ContextImpl.java:1368)
...
</pre>

正确做法应该是在 onDestroy() 方法中将 receiver 释放掉：

{% highlight java %}

	@Override
	protected void onDestroy() {
		unregisterReceiver(this.receiver);
		super.onDestroy();
	}

{% endhighlight %}

* setClassInstanceLimit()，设置某个类的同时处于内存中的实例上限，可以协助检查内存泄露。比如下面的代码：

{% highlight java %}
public class ActivityTestObjectLimit extends Activity {
	private static class MyClass{}
	
	private static boolean isStrictMode = false;
	
	private static List<MyClass> classList = new ArrayList<ActivityTestObjectLimit.MyClass>();
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		if(! isStrictMode){
			StrictMode.setVmPolicy(new VmPolicy.Builder()
			.setClassInstanceLimit(MyClass.class, 2)
			.penaltyLog()
			.build());
			isStrictMode = true;
		}
		
		classList.add(new MyClass());
		classList.add(new MyClass());
		classList.add(new MyClass());
		classList.add(new MyClass());
		classList.add(new MyClass());
		classList.add(new MyClass());
		classList.add(new MyClass());
		classList.add(new MyClass());
	}
}
{% endhighlight %}

日志信息如下：
<pre>
...: E/StrictMode(27681): class com.ap.teststrictmode.ActivityTestObjectLimit$MyClass; instances=8; limit=2
...: E/StrictMode(27681): android.os.StrictMode$InstanceCountViolation: class com.ap.teststrictmode.ActivityTestObjectLimit$MyClass; instances=8; limit=2
...: E/StrictMode(27681): 	at android.os.StrictMode.setClassInstanceLimit(StrictMode.java:1)
</pre>

注意：上面的异常一般都在GC之后抛出，如果测试的时候没有现象，可以多翻转几次屏幕，或者通过DDMS工具手动触发一下。
