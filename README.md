## Activity的身份证号 -- mToken

我们知道在Android系统中，Activity的启动，销毁以及生命周期是由**ActivityManagerService（下文简称AMS）**来进行统一管理的，而AMS本身是运行在系统进程system_server中的，因此它与我们app之间所有的通信都是通过IPC跨进程通信（Android中使用了Binder机制）来完成的。具体到Activity来说，它的一系列生命周期的调用都是由AMS和app所在进程的ActivityThread通过IPC完成的。

一个事实是，AMS与ActivityThread之间对于Activity的生命周期的交互，并没有直接使用Activity对象进行交互，而是使用一个token来标识，这个token是binder对象，因此可以方便地跨进程传递。Activity里面有一个成员变量mToken代表的就是它，token可以唯一地标识一个Activity对象，这也就是我们今天的主角--Activity的身份证号。

先对于上面提出的事实举个🌰， 就拿我们最常用的Activity的finish()方法来看，当我们调用finish()后，Activity内部最终会调用到下面这个方法：

	private void finish(int finishTask) {
		//省略部分代码
		......
		
		//这里的mToken便是上面提到的ID，app进程会把它传递给AMS
		if (ActivityManagerNative.getDefault()
                        .finishActivity(mToken, resultCode, resultData, finishTask)) {
                    mFinished = true;
                }
		
		......
	}
	
ActivityManagerNative.getDefault()会返回AMS的远程Binder代理对象ActivityManagerProxy的一个实例给我们，然后便通过IPC把事情移交给了另一端的AMS来处理。AMS之后会进行一系列针对finishActivity这个命令的工作（具体代码流程这里不详述，大家感兴趣的话可以到ActivityManagerService.java中去追踪），最后到达**ApplicationThread**通过sendMessage到**ActivityThread**的**H**，然后通过**H**转发消息到ActivityThread的handleDestroyActivity，接着这个方法把任务交给performDestroyActivity完成。**注：**这一套流程看起来很绕，其实AMS和ActivityThread之间的通信基本都是这个套路，大家感兴趣的话可以阅读老罗的文章：

	Android应用程序的Activity启动过程简要介绍和学习计划
	http://blog.csdn.net/luoshengyang/article/details/6685853

这里我们关心的是performDestroyActivity中的实现，关键代码如下：

		final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();
		
		//这里的第一个参数token，便是我们之前发给AMS的那个mToken，转了一圈又回来了
	    private ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
            int configChanges, boolean getNonConfigInstance) 
        {
        	ActivityClientRecord r = mActivities.get(token);
        	
        	//省略部分代码
        	......
        	
        	r.activity.mCalled = false;
            mInstrumentation.callActivityOnDestroy(r.activity);
            
            ......
            
            mActivities.remove(token);
         }
我们看到回到app所在进程后，这里通过token从mActivities拿到了一个ActivityClientRecord，然后直接把这个record里面的Activity交给Instrument类完成了onDestroy的调用，最后从mActivities中移除该token。

从这个🌰可以看出，ActivityThread与AMS进行Activity相关的IPC通信时并没有传递具体的Activity对象实例，而是通过token这个唯一标识来指定待操作的Activity对象。

那么问题来了，这个身份证号是在**何时**被**谁**创建出来并赋给Activity对象的呢？下面我们分别来看这两个问题。

- **何时**

翻看Activity的源码，可以看到对于这个token的声明：

	//android.app.Activity.java
	
	private IBinder mToken;
	
可以看到这个mToken变量的类型是IBinder，具体类型为IApplicationToken.Stub的子类ActivityRecord.Token,这个后面会再讲到。继续翻看代码，发现mToken是在attach方法中被初始化的：

	//android.app.Activity.java
	
	final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) 
   	{
   		......
   		
   		mToken = token;
   		
   		......
   	}
   	
这是Activity初始化阶段的一个方法，调用者是AcivityThread的**performLaunchActivity**方法：

	//android.app.ActivityThread.java

	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent)
	{
		......
		
		activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);
                        
     	......
     	
    }
    
上面第四个参数r.token便是我们要找的对象，可以看到它是通过ActivityClientRecord实例r传递过来的。在继续分析函数调用链之前插一句，大家还记得上面我们提到的finish()过程中最后调用的performDestroyActivity方法吧，有木有发现这两个方法的名字很套路，其实就像上面说的，整个AMS和ActivityThread针对Activity的IPC通信过程就是一个套路，这里我们很容易就能猜到performLaunchActivity这个方法是startActivity()过程的一个步骤，事实也是如此，无非就是:

	Activity.startActivity() -> ActivityManagerNative -> ActivityManagerProxy -> ActivityManagerService -> ApplicationThreadProxy -> ApplicationThread -> ActivityThread.H -> ActivityThread.performLaunchActivity()

到这里，第一个问题就解决了，mToken是在我们调用startActivity启动某个Activity时在attach方法中被赋值的。

下面来看第二个问题：

- **被谁创造**

上面第一步中，我们已经知道了mToken是在launch Activity的过程中被初始化的，并且得知在ActivityThread.performLaunchActivity方法中，token是作为ActivityClientRecord中的成员变量被传入的。那么可想而知该token一定是在调用连中performLaunchActivity方法之前的某个方法中被创建的，因此我们可以顺着调用链反推。

**ActivityThread.H**

	public void handleMessage(Message msg)
	{
		......
		
		switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    
       ......
       
  	}
  	
可以看到这一步中，ActivityClientRecord对象是从接受到msg中获取的，那么继续往上一步。

**ApplicationThread**

	 @Override
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            
			  ......

            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
        
 首先明确一下，这个方法是调用链从AMS所在进程回到app所在进程之后的入口方法，该方法是定义在IApplicationThread中的，因此它的调用者应该是位于AMS进程的一端。另外可以看到ActivityClientRecord是在这里被new出来的，然而这个发型并没有什么卵用，还得继续往上一步看。。
 
 **ActivityManagerService**
 
 这里我们就回到了AMS进程中，这里先列出AMS这端内部的函数调用链：
 
 	IApplicationThread.scheduleLaunchActivity() <- ActivityStackSupervisor.realStartActivityLocked() <- ActivityStackSupervisor.startSpecificActivityLocked() <- ActivityStack.makeVisibleAndRestartIfNeeded() <- ActivityStack.ensureActivitiesVisibleLocked() <- ActivityStarter.startActivityUnchecked() <- ActivityStarter.startActivityLocked() <- ActivityStarter.startActivityMayWait() <- ActivityManagerService.startActivity()
 	
 讲真的，我在找这条调用链的时候内心是崩溃的，因为几乎每个方法的参数中都带有 'IBinder token' 这个参数。。。而调用的时候都是传入ActivityRecord.appToken，这个appToken变量是在ActivityRecord的构造函数中被初始化的，因此我们需要找到整个调用链中哪一步new了一个ActivityRecord对象。随着在我心中奔腾的草泥马的数量越来越多，突然。。突然一个神圣的方法出现了--**ActivityStarter.startActivityLocked()**：
 
 		//当我把这个长到令人发指的参数列表一个个看完的时候，我仿佛看见了前方的曙光，木有token！！
 	    final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
            TaskRecord inTask) 
            {
            	......
            	
            	//找到了！
            	ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, mSupervisor, container,
                options, sourceRecord);
                
              ......
            }
山路十八弯，总算是找到了创建ActivityRecord的地方，再进一步，点进去看看：

	ActivityRecord(ActivityManagerService _service, ProcessRecord _caller,
            int _launchedFromUid, String _launchedFromPackage, Intent _intent, String _resolvedType,
            ActivityInfo aInfo, Configuration _configuration,
            ActivityRecord _resultTo, String _resultWho, int _reqCode,
            boolean _componentSpecified, boolean _rootVoiceInteraction,
            ActivityStackSupervisor supervisor,
            ActivityContainer container, ActivityOptions options, ActivityRecord sourceRecord)
            {
            	service = _service;
        		appToken = new Token(this, service);
        		
        		......
        	
        	}
        	
        	static class Token extends IApplicationToken.Stub 
        	{
        		private final WeakReference<ActivityRecord> weakActivity;
        		private final ActivityManagerService mService;

        		Token(ActivityRecord activity, ActivityManagerService service) 
        		{
            		weakActivity = new WeakReference<>(activity);
            		mService = service;
        		}
        		
        		......
        	}
        	
 就是这里了，可以看到appToken的类型是ActivityRecord的静态内部类Token,这是一个IApplicationToken.Stub的子类，也即是一个Binder对象。它里面保存了AMS本身的实例以及所对应的ActivityRecord的弱引用，而ActivityRecord中则保存了所有该Activity的相关信息，因此我们完全可以通过这个token来获得目标activity。
 
 这个在AMS所在进程被new出来的ActivityRecord对象，记录了全部从app进程传来的目标Activity的相关信息，并且在随后的调用链中一路被传递下去，最后里面的信息又都被通过binder传回给了app进程的ActivityThread，这就回到了第一步中我们提到的ApplicationThread中new ActivityClientRecord的地方。
 
 到这里，第二个问题也解决了，这个mToken是在ActivityStarter类中的startActivityLocked方法中被创建的，该动作发生在AMS所在的进程中。
 
 写在最后：
 
 在我们搞清楚了这个Activity的身份证号是怎么来的之后，再顺带聊一聊它的用处。时下比较热的插件化框架技术，在处理动态加载插件中的Activity时其实就利用了Activity token的特性。我们知道在Android系统中，每一个被启动的Activity都必须在AndroidManifest中进行注册，不然的话会抛错。系统对于一个Activity有没有注册的检查是在AMS所在的system_server完成的，因此插件化框架无法更改这一块的逻辑，但是插件中的Activity又是不可能提前在主项目的AndroidManifest里进行注册的，这样一来就只能通过对于app进程端代码的Hook来完成动态加载的需求。大体做法是假装启动一个已经在AndroidManifest里面声明过的替身Activity，让这个Activity进入AMS进程接受检验；最后在调用链回到app进程的时候换成我们真正需要启动的Activity；这样就成功欺骗了AMS进程，瞒天过海。这个方法之所以能行的通，就是因为不论是app进程的ActivityThread还是AMS对于目标Activity都是只认ID（mToken）不认对象，所以我们才能在同一个ID的掩护下狸猫换太子，成功瞒过AMS的检查。
            	
         
