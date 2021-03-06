[TOC]

# HandlerThread和IntentService

## 简述

* HandlerThread 是一个内部已经创建好 Handler 底层的 Looper 和 MessageQueue 的一种 Thread，它是一种可以直接使用 Handler 的 Thread ；
* IntentService 是一种可以用来执行后台耗时任务的Service，任务结束时它会自动停止；里面封装了 HandlerThread 所以可以用来做耗时任务，又因为它是 Service 所以优先级比单纯的 Thread 高很多，所以**用处是比较适合执行一些高优先级的后台任务**；

## 源码分析

### HandlerThread

* HandlerThread实现很简单，继承了 Thread ，就是一个已经创建好消息队列的线程；

  ~~~java
  @Override
      public void run() {
          mTid = Process.myTid();
          Looper.prepare();
          synchronized (this) {
              mLooper = Looper.myLooper();
              notifyAll();
          }
          Process.setThreadPriority(mPriority);
          onLooperPrepared();
          Looper.loop();
          mTid = -1;
      }
  ~~~

* 调用 start 后就可以使用 这个线程的handler了；

### IntentService

* IntentService 就是一个 Service ，只不过内置了线程可以执行耗时任务，所以它是一个可以异步执行耗时任务的 Service，在完成了使命之后会自动停止；

* 使用示例

  ~~~java
  public class MyIntentService extends IntentService {

      /**
       * 是否正在运行
       */
      private boolean isRunning;

      /**
       *进度
       */
      private int count;

      /**
       * 广播
       */
      private LocalBroadcastManager mLocalBroadcastManager;

      public MyIntentService() {
          super("MyIntentService");
          Logout.e("MyIntentService");
      }

      @Override
      public void onCreate() {
          super.onCreate();
          Logout.e("onCreate");
          mLocalBroadcastManager = LocalBroadcastManager.getInstance(this);
      }
  //就是要覆写这个方法，可以根据 intent 所携带的判定码来判定需要 handler 执行哪个任务；任务可以有多个，执行一次 startService 就会执行一次任务，但是任务是线性执行的，因为是 handler 机制来执行
      @Override
      protected void onHandleIntent(Intent intent) {
          Logout.e("onHandleIntent");
          try {
              Thread.sleep(1000);
              isRunning = true;
              count = 0;
              while (isRunning) {
                  count++;
                  if (count >= 100) {
                      isRunning = false;
                  }
                  Thread.sleep(50);
                  sendThreadStatus("线程运行中...", count);
              }

          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }

      /**
       * 发送进度消息
       */
      private void sendThreadStatus(String status, int progress) {
          Intent intent = new Intent(IntentServiceActivity.ACTION_TYPE_THREAD);//发广播
          intent.putExtra("status", status);
          intent.putExtra("progress", progress);
          mLocalBroadcastManager.sendBroadcast(intent);
      }

      @Override
      public void onDestroy() {
          super.onDestroy();
          Logout.e("线程结束运行..." + count);
      }
  }
  ~~~

* 源码

* 根据 Service 的执行顺序可以先看 onCreate

  ~~~java
  @Override
      public void onCreate() {
          // TODO: It would be nice to have an option to hold a partial wakelock
          // during processing, and to have a static startService(Context, Intent)
          // method that would launch the service & hand off a wakelock.

          super.onCreate();
        //这里可以看到 IntentService 内置了 HandlerThread 线程
          HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        //启动它，此时这个 HandlerThread 线程内部已经创建好了handler所需要的环境
          thread.start();
        //接下来就可以拿到我们执行耗时任务的线程的loop er
          mServiceLooper = thread.getLooper();
        //有了这个looper就可以创建属于执行耗时任务的线程的 handler 了；
          mServiceHandler = new ServiceHandler(mServiceLooper);
      }
  ~~~

* 接下来可以去 onStartCommand 方法看一下

  ~~~java
  @Override
      public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
          onStart(intent, startId);
          return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
      }

  @Override
      public void onStart(@Nullable Intent intent, int startId) {
        //将外界传进来的intent放到消息中，然后就handler就发送消息去执行任务的线程了
          Message msg = mServiceHandler.obtainMessage();
          msg.arg1 = startId;
          msg.obj = intent;
          mServiceHandler.sendMessage(msg);
      }
  ~~~

* 这个时候应该去 handler 中看看handler 是如何处理的；

  ~~~java
  private final class ServiceHandler extends Handler {
          public ServiceHandler(Looper looper) {
              super(looper);
          }

          @Override
          public void handleMessage(Message msg) {
            //逻辑很清楚，调用开放给用户编写的onHandleIntent方法；
              onHandleIntent((Intent)msg.obj);
            //然后就是很重要的这里了，看名字，应该是停止服务，好奇怪，为什么这里不是使用我们熟悉的 stopSelf(),而是调用stopSelf(int startId)这个方法，这是因为线程中可能还有任务待执行，要等其他任务执行结束之后再关闭服务；至于关闭的原理，应该是这样子，这里这个参数是会去和启动服务的次数判断，相同了才关闭，因为相同了就意味着这是最后一个任务了，我再猜一下，每启动一次服务，AMS是会记录启动次数并赋给Service，也就是这个次数会到onStartCommand的startId中，所以每次启动的服务的startId都是不同的（累加的），此时每个任务的 startId 都是不同的，然后Service只是都将任务丢给 HandlerThread 执行，looper是一个一个执行任务的，执行完上面的onHandleIntent((Intent)msg.obj);就要执行下面的stopSelf(msg.arg1);此时就是判断是否是最后一个任务，如果是才停止服务，如果不是就不停止
              stopSelf(msg.arg1);
          }·
  }
  ~~~

  ​

  ​

