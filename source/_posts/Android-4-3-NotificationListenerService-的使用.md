title: Android 4.3+ NotificationListenerService 的使用
date: 2015-12-11 18:35:15
categories: 技术
tags: [Android,Notification] 
---
> A service that receives calls from the system when new notifications are posted or removed, or their ranking changed.

官方文档有这么一句话，告诉我们**NotificationListenerService**可以用来监听到通知的发送以及移除和排名位置变化。那么如果我们注册了这个服务，当系统任何一条通知到来或者被移除掉，我们都能通过这个service来监听到，甚至可以做一些管理工作。

<!-- more -->

### 如何使用？

既然是系统提供的服务，那么申请权限以及在配置文件中声明是基本的套路。在**AndroidManifest**中需要加入以下代码：
``` xml
 <service android:name=".NotificationListener"
          android:label="@string/service_name"
          android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
     <intent-filter>
         <action android:name="android.service.notification.NotificationListenerService" />
     </intent-filter>
 </service>
```
这样就声明了我可以拥有处理notification的能力了。但是系统怎么知道你能处理呢？显然，你得向系统申请权限，系统会记录下一些信息，当通知到来或有被移除的时候，会通知所有有权限的应用这条新来的通知对象。
那么当你申请权限的时候，系统到底做了啥事儿呢？

### 原理分析

当你首次使用的时候，需要去设置页面开启一个权限，这个页面叫做Notification access，会列出所有申请了这个权限的应用供你选择，勾选了这个应用后，那么对应的上面注册的**NotificationListener**就会被启动起来。

当我们使用NotificationListenerService的时候，我们只需要继承自它，暴露给我们两个抽象方法：

- **onNotificationPosted(StatusBarNotification sbn)**

- **onNotificationRemoved(StatusBarNotification sbn)**

当系统收到一条通知的时候，onNotificationPosted会回调，当从通知栏移除一条通知时，onNotificationPosted会回调。
>注意到**StatusBarNotification**这个对象，是系统对notification对象的重新包装，携带了一些基本信息，我们可以拿着这个对象干很多有意思的事情，稍后介绍。

从实现上来看，NotificationListenerService继承自系统的Service，那么这个服务是何时，何地，被谁启动的呢？
这里我们不妨猜想一下：
- 1.如果我们已经在设置中勾选了应用，必然会触发系统会更新一些设置，如果你有一些android开发经验，可能知道这种更新往往是通过contenprovider更新了一张系统数据表，告诉系统我有这个权限了；
- 2.如果已经勾选过，重新开机系统在准备的时候，应该会知道有谁已经有了这个权限，到时候要为这个应用服务；
- 3.如果卸载了有权限的应用，应该会告诉系统从权限列表移除数据，显然这里肯定有谁在监听一个广播；

那么，到底是谁在做这件事儿呢？我们先来分析一下**NotificationListenerService**的实现。
查看SDK源码，当绑定服务后，返回的IBinder对象如下：

``` java
     @Override
    public IBinder onBind(Intent intent) {
        if (mWrapper == null) {
            mWrapper = new INotificationListenerWrapper();
        }
        return mWrapper;
    }

    private class INotificationListenerWrapper extends INotificationListener.Stub {
        @Override
        public void onNotificationPosted(StatusBarNotification sbn) {
            try {
                NotificationListenerService.this.onNotificationPosted(sbn);
            } catch (Throwable t) {
                Log.w(TAG, "Error running onNotificationPosted", t);
            }
        }
        @Override
        public void onNotificationRemoved(StatusBarNotification sbn) {
            try {
                NotificationListenerService.this.onNotificationRemoved(sbn);
            } catch (Throwable t) {
                Log.w(TAG, "Error running onNotificationRemoved", t);
            }
        }
    }
```

注意到返回的这个binder对象其实是**INotificationListener.Stub**，可以看到onNotificationPosted方法是会被INotificationListener对象调用的，一定有一个客户端得到这个INotificationListener对象，当通知来的时候回调到**NotificationListenerService.this.onNotificationPosted(sbn)**方法，就是我们继承**NotificationListenerService**后需要实现的方法。注意到源码中还有一个方法：
``` java
    private final INotificationManager getNotificationInterface() {
        if (mNoMan == null) {
            mNoMan = INotificationManager.Stub.asInterface(
                    ServiceManager.getService(Context.NOTIFICATION_SERVICE));
        }
        return mNoMan;
    }
```
这个mNoman是一个**INotificationManager**对象，并且是通过ServiceManager注册的，有经验的同学应该知道了，类似这种都能在系统中找到对应的服务，所有**NotificationManagerService**就呼之欲出了，我们来分析看看它的源码。

### NotificationmanagerService中的相关源码分析

前面我们猜想了可能启动NotificationListenerService的时机，事实上在NotificationmanagerService源码中我们都能找到具体实现。
- 开机启动
在系统开机的时候，NotificationmanagerService在初始化后，会调用到**systemReady()**方法，如下所示：
``` java
    void systemReady() {
        mAudioService = IAudioService.Stub.asInterface(
                ServiceManager.getService(Context.AUDIO_SERVICE));
        // no beeping until we're basically done booting
        mSystemReady = true;
        // make sure our listener services are properly bound
        rebindListenerServices();
    }
```
首先会拿到audioservice对象，因为有些通知可以携带声音信息，然后调用**rebindListenerServices**这个关键方法。

``` java
void rebindListenerServices() {
        final int currentUser = ActivityManager.getCurrentUser();
        //通过secure表查出那些注册了ENABLED_NOTIFICATION_LISTENERS
        String flat = Settings.Secure.getStringForUser(
                mContext.getContentResolver(),
                Settings.Secure.ENABLED_NOTIFICATION_LISTENERS,
                currentUser);
        NotificationListenerInfo[] toRemove = new NotificationListenerInfo[mListeners.size()];
        final ArrayList<ComponentName> toAdd;
        synchronized (mNotificationList) {
            // unbind and remove all existing listeners
            toRemove = mListeners.toArray(toRemove);
            toAdd = new ArrayList<ComponentName>();
            final HashSet<ComponentName> newEnabled = new HashSet<ComponentName>();
            final HashSet<String> newPackages = new HashSet<String>();
            // decode the list of components
            if (flat != null) {
                String[] components = flat.split(ENABLED_NOTIFICATION_LISTENERS_SEPARATOR);
                for (int i=0; i<components.length; i++) {
                    final ComponentName component
                            = ComponentName.unflattenFromString(components[i]);
                    if (component != null) {
                        newEnabled.add(component);
                        toAdd.add(component);
                        newPackages.add(component.getPackageName());
                    }
                }
                mEnabledListenersForCurrentUser = newEnabled;
                mEnabledListenerPackageNames = newPackages;
            }
        }
        for (NotificationListenerInfo info : toRemove) {
            final ComponentName component = info.component;
            final int oldUser = info.userid;
            Slog.v(TAG, "disabling notification listener for user " + oldUser + ": " + component);
            unregisterListenerService(component, info.userid);
        }
        final int N = toAdd.size();
        for (int i=0; i<N; i++) {
            final ComponentName component = toAdd.get(i);
            Slog.v(TAG, "enabling notification listener for user " + currentUser + ": "
                    + component);
            registerListenerService(component, currentUser);
        }
    }
```
可以看到，首先从secure表中查处哪些应用注册了ENABLED_NOTIFICATION_LISTENERS服务，添加到toAdd列表中，然后把当前已经注册的mListeners全部移除掉，调用** unregisterListenerService(component, info.userid)**移除之后，重新注册toAdd列表中的服务，调用**registerListenerService(component, currentUser)**。可以猜到这两个方法就是去bind或者unbind我们自己实现的**NotificationListenerService**了。

> 通过secure表查询到的字段，我们可以通过RE浏览器看到里面的数据，在com.android.provider.setting中找到secure表。

接下来在**registerListenerservice**中有如下实现：
``` java
    ....
    try {
     if (DBG) Slog.v(TAG, "binding: " + intent);
      if (!mContext.bindServiceAsUser(intent,
              new ServiceConnection() {
                  INotificationListener mListener;
                  @Override
                  public void onServiceConnected(ComponentName name, IBinder service) {
                      synchronized (mNotificationList) {
                          mServicesBinding.remove(servicesBindingTag);
                          try {
                              mListener = INotificationListener.Stub.asInterface(service);
                              NotificationListenerInfo info = new NotificationListenerInfo(
                                      mListener, name, userid, this);
                              service.linkToDeath(info, 0);
                              mListeners.add(info);
                          } catch (RemoteException e) {
                              // already dead
                          }
                      }
                  }
                  @Override
                  public void onServiceDisconnected(ComponentName name) {
                      Slog.v(TAG, "notification listener connection lost: " + name);
                  }
              },
              Context.BIND_AUTO_CREATE,
              new UserHandle(userid)))
                        
                        ...
```

这部分代码大家应该比较熟悉了，跟平时用AIDL服务一样，我们注册的**NotificationListenerService**服务在这里通过
`INotificationListener.Stub.asInterface(service);`转换成需要的INotificationListener对象，将它加入到mListeners集合中，然后就可以通过这个对象回调到我们自己注册的服务代码中了。

- 用户在设置中开启权限的时候
这种情况下，由于**NotificationManagerService**中使用`ContentObserver`监听了`SettingProvider`的变化，当secure数据表发生变化，也会调用**rebindListenerServices**重新绑定服务。如下：
``` java
class SettingsObserver extends ContentObserver {
        private final Uri NOTIFICATION_LIGHT_PULSE_URI
                = Settings.System.getUriFor(Settings.System.NOTIFICATION_LIGHT_PULSE);
        private final Uri ENABLED_NOTIFICATION_LISTENERS_URI
                = Settings.Secure.getUriFor(Settings.Secure.ENABLED_NOTIFICATION_LISTENERS);
        SettingsObserver(Handler handler) {
            super(handler);
        }
        void observe() {
            ContentResolver resolver = mContext.getContentResolver();
            resolver.registerContentObserver(NOTIFICATION_LIGHT_PULSE_URI,
                    false, this, UserHandle.USER_ALL);
            resolver.registerContentObserver(ENABLED_NOTIFICATION_LISTENERS_URI,
                    false, this, UserHandle.USER_ALL);
            update(null);
        }
        @Override public void onChange(boolean selfChange, Uri uri) {
            update(uri);
        }
        public void update(Uri uri) {
            ContentResolver resolver = mContext.getContentResolver();
            if (uri == null || NOTIFICATION_LIGHT_PULSE_URI.equals(uri)) {
                boolean pulseEnabled = Settings.System.getInt(resolver,
                            Settings.System.NOTIFICATION_LIGHT_PULSE, 0) != 0;
                if (mNotificationPulseEnabled != pulseEnabled) {
                    mNotificationPulseEnabled = pulseEnabled;
                    updateNotificationPulse();
                }
            }
            if (uri == null || ENABLED_NOTIFICATION_LISTENERS_URI.equals(uri)) {
                rebindListenerServices();
            }
        }
    }
```

- 通过广播监听
当监听到注册了**NotificationListenerService**的应用被卸载时，同样会调用**rebindListenerServices**重新绑定服务，之后的流程跟前面分析一样。
> 这部分代码较长，由于篇幅原因就不贴出来了，大家可以自行查看源码。

上面就是**NotificationListenerService**的启动情况。既然服务已经启动了，那么当有通知到来或者删除的时候，怎么告诉注册的这些应用呢？前面分析到，注册的服务都会被添加到一个**mListeners**的集合中，我们猜想当通知到来的时候，NotificationListenerService肯定会遍历整个集合，然后调用listener的**onNotificationPosted(sbnf)**方法。平时我们在发送通知的时候，是通过NotificationManager对象调用notify发送，其实最终都是通过**NotificationManagerService**来完成的，我们看源码，在**NotificationManager**中：
``` java
 public void notify(String tag, int id, Notification notification)
    {
       ...
        try {
            service.enqueueNotificationWithTag(pkg, mContext.getOpPackageName(), tag, id,
                    notification, idOut, UserHandle.myUserId());
            ...
        } catch (RemoteException e) {
        }
    }
```
这里的service对象显然是**NotificationManagerService**，调用其enqueueNotificationWithTag发送一条通知，接下来辗转调用到**notifyPostedLocked()**，这里就符合了我们上面的猜想：
```java
 private void notifyPostedLocked(NotificationRecord n) {
        // make a copy in case changes are made to the underlying Notification object
        final StatusBarNotification sbn = n.sbn.clone();
        for (final NotificationListenerInfo info : mListeners) {
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    info.notifyPostedIfUserMatch(sbn);
                }});
        }
    }
```
可以看到这里遍历mListeners集合，调用notifyPostedIfUserMatch方法，到这里我们可以知道离我们想要看到的方法已经不远了。
```java
   public void notifyPostedIfUserMatch(StatusBarNotification sbn) {
            if (!enabledAndUserMatches(sbn)) {
                return;
            }
            try {
                listener.onNotificationPosted(sbn);
            } catch (RemoteException ex) {
                Log.e(TAG, "unable to notify listener (posted): " + listener, ex);
            }
        }
```

终于，我们熟悉的方法看到了，这不就是我们自己注册服务中的方法吗？这样整个调用链就清晰了。

> 这里我们可以发现一个有意思的地方，如果我们注册了这个服务，当系统有通知到来的时候，会回调这个服务。如果有些应用有常驻进程，当由于某些原因常驻进程挂掉了，这是不是有一定机会系统帮我们唤醒了进程呢？

> 另外的删除通知类似，这里就不再分析了。

现在总结一下整个流程：
从整体上来看，**NotificationListenerService**相当于一个本地代理，真正起到控制作用的是**NotificationManagerService**。通过三种方式注册到**NotificationManagerService**中的listeners会保存在一个集合中，当通知到来的时候，会遍历整个集合回调listener中的方法。

### 取消通知
**NotificationListenerService**还提供了一个方法，可以取消到收到的通知，如果用户使用了这个服务，可以对收到的通知取消，那么在通知栏上我们就看不到这条通知了。这对于有洁癖的用户来说，这也许是一个好方法。

    - cancelNotification(String pkg, String tag, int id)
    - cancelAllNotifications() 
    - cancelNotification(String key) 5.0

看名字知道这两个方法的作用。5.0以后，第一个方法过期了，所以这个方法取消不了5.0版本以后发送的通知，第三个方法是5.0提供的方法。





