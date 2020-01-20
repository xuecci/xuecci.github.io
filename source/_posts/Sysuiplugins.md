## 一、SystemUI plugins机制介绍
插件是包含可以动态加载到SystemUI中的代码和资源的APK，插件提供了一种快速创建SystemUI功能的简便方法，通过创建插件来实现SysUI中已经定义的基本接口集，从而在运行时更改SystemUI的行为，由接口控制的代码可以更快的迭代。

## 二、基本架构
下图显示了插件编译/加载流程的工作原理
![Sysui plugins 解析(以SampleOverlayPlugin为例)](uploads/affc4d5bcc13947ed5d94d85c43b2c66/sysui-plugins.png)

## 三、代码结构
vendor\tran_os\packages\apps\SystemUI\plugin\ExamplePlugin\src\com\android\systemui\plugin\testoverlayplugin\ 插件目录，生成插件apk，sysui接口实现

vendor\tran_os\packages\apps\SystemUI\plugin\src\com\android\systemui\plugins\  功能接口定义

vendor\tran_os\packages\apps\SystemUI\plugin_core\  基础接口定义

vendor\tran_os\packages\apps\SystemUI\shared\src\com\android\systemui\shared\plugins\   plugin运行模块，主要负责插件加载，管理等

## 四、SampleOverlayPlugin加载流程解析
### 1、触发加载插件 (vendor\tran_os\packages\apps\SystemUI\src\com\android\systemui\SystemUIApplication.java)
```
Dependency.get(PluginManager.class).addPluginListener(
                new PluginListener<OverlayPlugin>() {
                    private ArraySet<OverlayPlugin> mOverlays = new ArraySet<>();

                    @Override
                    public void onPluginConnected(OverlayPlugin plugin, Context pluginContext) {
                        mainHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                StatusBar statusBar = getComponent(StatusBar.class);
                                if (statusBar != null) {
                                    plugin.setup(statusBar.getStatusBarWindow(),
                                            statusBar.getNavigationBarView(), new Callback(plugin));
                                }
                            }
                        });
                    }

                    @Override
                    public void onPluginDisconnected(OverlayPlugin plugin) {
                        mainHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                mOverlays.remove(plugin);
                                Dependency.get(StatusBarWindowController.class).setForcePluginOpen(
                                        mOverlays.size() != 0);
                            }
                        });
                    }

                    class Callback implements OverlayPlugin.Callback {
                        private final OverlayPlugin mPlugin;

                        Callback(OverlayPlugin plugin) {
                            mPlugin = plugin;
                        }

                        @Override
                        public void onHoldStatusBarOpenChange() {
                            if (mPlugin.holdStatusBarOpen()) {
                                mOverlays.add(mPlugin);
                            } else {
                                mOverlays.remove(mPlugin);
                            }
                            mainHandler.post(new Runnable() {
                                @Override
                                public void run() {
                                    Dependency.get(StatusBarWindowController.class)
                                            .setStateListener(b -> mOverlays.forEach(
                                                    o -> o.setCollapseDesired(b)));
                                    Dependency.get(StatusBarWindowController.class)
                                            .setForcePluginOpen(mOverlays.size() != 0);
                                }
                            });
                        }
                    }
                }, OverlayPlugin.class, true /* Allow multiple plugins */);
```
Dependency.get(PluginManager.class)得到的是PluginManagerImpl对象，调用了void addPluginListener(PluginListener<T> listener, Class<?> cls,
boolean allowMultiple)方法，实际调用到addPluginListener(String action, PluginListener<T> listener,
Class cls, boolean allowMultiple)方法

通过PluginManagerImpl将寻找OverlayPlugin的实现，绑定后同过OverlayPlugin定义好的接口去调用SampleOverlayPlugin的实现功能，下面我们一步步看如何寻找到接口OverlayPlugin的实现

### 2. addPluginListener

vendor\tran_os\packages\apps\SystemUI\shared\src\com\android\systemui\shared\plugins\PluginManagerImpl.java
```
    public <T extends Plugin> void addPluginListener(String action, PluginListener<T> listener,
            Class cls, boolean allowMultiple) {
        mPluginPrefs.addAction(action);
        PluginInstanceManager p = mFactory.createPluginInstanceManager(mContext, action, listener,
                allowMultiple, mLooper, cls, this);
        p.loadAll();
        mPluginMap.put(listener, p);
        startListening();
    }
```
其中action是通过PluginManager.Helper.getAction(cls)获取的
```
        public static <P> String getAction(Class<P> cls) {
            ProvidesInterface info = cls.getDeclaredAnnotation(ProvidesInterface.class);
            if (info == null) {
                throw new RuntimeException(cls + " doesn't provide an interface");
            }
            if (TextUtils.isEmpty(info.action())) {
                throw new RuntimeException(cls + " doesn't provide an action");
            }
            return info.action();
        }
```
getDeclaredAnnotation通过注解获取OverlayPlugin的ACTION
```
@ProvidesInterface(action = OverlayPlugin.ACTION, version = OverlayPlugin.VERSION)
public interface OverlayPlugin extends Plugin {
```

主要看一下loadAll方法
```
    public void loadAll() {
        if (DEBUG) Log.d(TAG, "startListening");
        mPluginHandler.sendEmptyMessage(PluginHandler.QUERY_ALL);
    }
    
    public void handleMessage(Message msg) {
            switch (msg.what) {
                case QUERY_ALL:
                    if (DEBUG) Log.d(TAG, "queryAll " + mAction);
                    for (int i = mPlugins.size() - 1; i >= 0; i--) {
                        PluginInfo<T> plugin = mPlugins.get(i);
                        mListener.onPluginDisconnected(plugin.mPlugin);
                        if (!(plugin.mPlugin instanceof PluginFragment)) {
                            // Only call onDestroy for plugins that aren't fragments, as fragments
                            // will get the onDestroy as part of the fragment lifecycle.
                            plugin.mPlugin.onDestroy();
                        }
                    }
                    mPlugins.clear();
                    handleQueryPlugins(null);
                    break;

    private void handleQueryPlugins(String pkgName) {
            // This isn't actually a service and shouldn't ever be started, but is
            // a convenient PM based way to manage our plugins.
            Intent intent = new Intent(mAction);
            if (pkgName != null) {
                intent.setPackage(pkgName);
            }
            // plugin在AndroidManifest中必须声明为<service>，packageManager读取到所有的service信息
            List<ResolveInfo> result = mPm.queryIntentServices(intent, 0);
            if (DEBUG) Log.d(TAG, "Found " + result.size() + " plugins");
            if (result.size() > 1 && !mAllowMultiple) {
                // TODO: Show warning.
                Log.w(TAG, "Multiple plugins found for " + mAction);
                if (DEBUG) {
                    for (ResolveInfo info : result) {
                        ComponentName name = new ComponentName(info.serviceInfo.packageName,
                                info.serviceInfo.name);
                        Log.w(TAG, "  " + name);
                    }
                }
                return;
            }
            for (ResolveInfo info : result) {
                ComponentName name = new ComponentName(info.serviceInfo.packageName,
                        info.serviceInfo.name);
                PluginInfo<T> t = handleLoadPlugin(name);
                if (t == null) continue;

                // add plugin before sending PLUGIN_CONNECTED message
                mPlugins.add(t);
                mMainHandler.obtainMessage(mMainHandler.PLUGIN_CONNECTED, t).sendToTarget();
            }
        }

        protected PluginInfo<T> handleLoadPlugin(ComponentName component) {
            // This was already checked, but do it again here to make extra extra sure, we don't
            // use these on production builds.
            if (!isDebuggable && !isPluginWhitelisted(component)) {
                // Never ever ever allow these on production builds, they are only for prototyping.
                Log.w(TAG, "Plugin cannot be loaded on production build: " + component);
                return null;
            }
            if (!mManager.getPluginEnabler().isEnabled(component)) {
                if (DEBUG) Log.d(TAG, "Plugin is not enabled, aborting load: " + component);
                return null;
            }
            String pkg = component.getPackageName();
            String cls = component.getClassName();
            try {
                ApplicationInfo info = mPm.getApplicationInfo(pkg, 0);
                // TODO: This probably isn't needed given that we don't have IGNORE_SECURITY on
                // 此处检查设置的权限
                if (mPm.checkPermission(PLUGIN_PERMISSION, pkg)
                        != PackageManager.PERMISSION_GRANTED) {
                    Log.d(TAG, "Plugin doesn't have permission: " + pkg);
                    return null;
                }
                // Create our own ClassLoader so we can use our own code as the parent.
                ClassLoader classLoader = mManager.getClassLoader(info);
                Context pluginContext = new PluginContextWrapper(
                        mContext.createApplicationContext(info, 0), classLoader);
                Class<?> pluginClass = Class.forName(cls, true, classLoader);
                // TODO: Only create the plugin before version check if we need it for
                // legacy version check.
                T plugin = (T) pluginClass.newInstance();
                try {
                    VersionInfo version = checkVersion(pluginClass, plugin, mVersion);
                    if (DEBUG) Log.d(TAG, "createPlugin");
                    return new PluginInfo(pkg, cls, plugin, pluginContext, version);
                } catch (InvalidVersionException e) {
                    final int icon = mContext.getResources().getIdentifier("tuner", "drawable",
                            mContext.getPackageName());
                    final int color = Resources.getSystem().getIdentifier(
                            "system_notification_accent_color", "color", "android");
                    final Notification.Builder nb = new Notification.Builder(mContext,
                            PluginManager.NOTIFICATION_CHANNEL_ID)
                                    .setStyle(new Notification.BigTextStyle())
                                    .setSmallIcon(icon)
                                    .setWhen(0)
                                    .setShowWhen(false)
                                    .setVisibility(Notification.VISIBILITY_PUBLIC)
                                    .setColor(mContext.getColor(color));
                    String label = cls;
                    try {
                        label = mPm.getServiceInfo(component, 0).loadLabel(mPm).toString();
                    } catch (NameNotFoundException e2) {
                    }
                    if (!e.isTooNew()) {
                        // Localization not required as this will never ever appear in a user build.
                        nb.setContentTitle("Plugin \"" + label + "\" is too old")
                                .setContentText("Contact plugin developer to get an updated"
                                        + " version.\n" + e.getMessage());
                    } else {
                        // Localization not required as this will never ever appear in a user build.
                        nb.setContentTitle("Plugin \"" + label + "\" is too new")
                                .setContentText("Check to see if an OTA is available.\n"
                                        + e.getMessage());
                    }
                    Intent i = new Intent(PluginManagerImpl.DISABLE_PLUGIN).setData(
                            Uri.parse("package://" + component.flattenToString()));
                    PendingIntent pi = PendingIntent.getBroadcast(mContext, 0, i, 0);
                    nb.addAction(new Action.Builder(null, "Disable plugin", pi).build());
                    mContext.getSystemService(NotificationManager.class)
                            .notifyAsUser(cls, SystemMessage.NOTE_PLUGIN, nb.build(),
                                    UserHandle.ALL);
                    // TODO: Warn user.
                    Log.w(TAG, "Plugin has invalid interface version " + plugin.getVersion()
                            + ", expected " + mVersion);
                    return null;
                }
            } catch (Throwable e) {
                Log.w(TAG, "Couldn't load plugin: " + pkg, e);
                return null;
            }
        }
        
       case PLUGIN_CONNECTED:
          if (DEBUG) Log.d(TAG, "onPluginConnected");
          PluginPrefs.setHasPlugins(mContext);
          PluginInfo<T> info = (PluginInfo<T>) msg.obj;
          mManager.handleWtfs();
          if (!(msg.obj instanceof PluginFragment)) {
           // Only call onDestroy for plugins that aren't fragments, as fragments
           // will get the onCreate as part of the fragment lifecycle.
           info.mPlugin.onCreate(mContext, info.mPluginContext);
         }
         mListener.onPluginConnected(info.mPlugin, info.mPluginContext);

```
查看plugin的service定义，定义的action要与接口OverlayPlugin中定义的一致，PackageManager通过此action查询到plugin
```
   vendor\tran_os\packages\apps\SystemUI\plugin\ExamplePlugin\AndroidManifest.xml
   <service android:name=".SampleOverlayPlugin"
            android:label="@string/plugin_label">
         <intent-filter>
                <action android:name="com.android.systemui.action.PLUGIN_OVERLAY" />
         </intent-filter>
   </service>
```
之后通过PathClassLoader加载类，PathClassLoader只能用于加载已安装过的apk的dex
```
    ClassLoader getParentClassLoader() {
        if (mParentClassLoader == null) {
            // Lazily load this so it doesn't have any effect on devices without plugins.
            mParentClassLoader = new ClassLoaderFilter(getClass().getClassLoader(),
                    "com.android.systemui.plugin");
        }
        return mParentClassLoader;
    }

    /** Returns class loader specific for the given plugin. */
    public ClassLoader getClassLoader(ApplicationInfo appInfo) {
        if (!isDebuggable && !isPluginPackageWhitelisted(appInfo.packageName)) {
            Log.w(TAG, "Cannot get class loader for non-whitelisted plugin. Src:"
                    + appInfo.sourceDir + ", pkg: " + appInfo.packageName);
            return null;
        }
        if (mClassLoaders.containsKey(appInfo.packageName)) {
            return mClassLoaders.get(appInfo.packageName);
        }

        List<String> zipPaths = new ArrayList<>();
        List<String> libPaths = new ArrayList<>();
        LoadedApk.makePaths(null, true, appInfo, zipPaths, libPaths);
        ClassLoader classLoader = new PathClassLoader(
                TextUtils.join(File.pathSeparator, zipPaths),
                TextUtils.join(File.pathSeparator, libPaths),
                getParentClassLoader());
        mClassLoaders.put(appInfo.packageName, classLoader);
        return classLoader;
    }
```
至此通过classLoader加载plugin类赋值到PluginInfo,然后发送PLUGIN_CONNECTED消息，调用到mListener.onPluginConnected，回到第一步addPluginListener-》PluginListener，调用plugin.setup，此plugin已经是SampleOverlayPlugin apk中的实现，从而完成布局加载

## 五、类图结构
![Sysui plugins 解析(以SampleOverlayPlugin为例)](uploads/affc4d5bcc13947ed5d94d85c43b2c66/class-structure.png)
