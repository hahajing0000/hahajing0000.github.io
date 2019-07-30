---
title: Android插件化——动态资源加载
date: 2019-07-24
tags: [Android,插件化]
categories: Android插件化
toc: true
---
之前我们聊了一下Android插件化中的热修复，参考：（[Android 热修复](http://www.zydeveloper.com/2019/07/15/hotupdate/)），现在我们聊聊资源的动态加载。

<!--more-->

顾名思义我们之前的热更新只是解决了代码的加载，对于外部资源我们还不能直接使用。我们工程的资源最终打包后都会放到R.java文件中，然后我们可以获取Resources然后通过类似resources.getDrawable(id);方式通过id来获取我们具体的资源。但对于外部资源也就是不在我们工程中没有被放到R.java文件中的资源我们如果使用呢？

```java
Drawable drawable = resources.getDrawable(resId);
```
上面这种方式是我们平时获取资源的使用方式。

我们使用Resources的getXXX 通过resId来获取资源。
那我们考虑考虑是否可以获取插件的Resources呢？如果可以我们是不是就可以实现资源的动态加载了呢？
答案是可以的！

我们先来看看Activity是如何获取Resources的：
我们直接定位到代码，ContextThemeWrapper
继承关系，如

ContextThemeWrapper > ContextWrapper > Context

```java
@Override
public Resources getResources() {
    return getResourcesInternal();
}

private Resources getResourcesInternal() {
    if (mResources == null) {
        if (mOverrideConfiguration == null) {
            mResources = super.getResources();
        } else {
            final Context resContext = createConfigurationContext(mOverrideConfiguration);
            mResources = resContext.getResources();
        }
    }
    return mResources;
}
```

我们发现获取Resources的方法，再来看看Resources.java的构造函数
```java
/**
* Create a new Resources object on top of an existing set of assets in an
* AssetManager.
*
* @deprecated Resources should not be constructed by apps.
* See {@link android.content.Context#createConfigurationContext(Configuration)}.
*
* @param assets Previously created AssetManager.
* @param metrics Current display metrics to consider when
*                selecting/computing resource values.
* @param config Desired device configuration to consider when
*               selecting/computing resource values (optional).
*/
@Deprecated
public Resources(AssetManager assets, DisplayMetrics metrics, Configuration config) {
this(null);
mResourcesImpl = new ResourcesImpl(assets, metrics, config, new DisplayAdjustments());
}
```
我们发现Resources是由AssetManager创建处理的。
再来分析一下AssetManger
```java
/**
 * Provides access to an application's raw asset files; see {@link Resources}
 * for the way most applications will want to retrieve their resource data.
 * This class presents a lower-level API that allows you to open and read raw
 * files that have been bundled with the application as a simple stream of
 * bytes.
 */
public final class AssetManager implements AutoCloseable {
    ...
    /**
    * @deprecated Use {@link #setApkAssets(ApkAssets[], boolean)}
    * @hide
    */
    @Deprecated
    public int addAssetPath(String path) {
        return addAssetPathInternal(path, false /*overlay*/, false /*appAsLib*/);
    }
    ...
}
```
我们发现一个addAssetPath的方法，实际这个方法可以使我们传入的apk path为我们构建出AssetManager 然后通过AssetManager创建出Resources。如上是我们实现动态加载资源的初步思路。
下面实践一下是否可行？
```java
/**
 * 获取对应插件的Resource对象
 * @param context 宿主apk的上下文
 * @param pluginPath 插件apk的路径，带apk名
 * @return
 */
public static Resources getPluginResources(Context context, String pluginPath) {
    try {
        AssetManager assetManager = AssetManager.class.newInstance();
        // 反射调用方法addAssetPath(String path)
        Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);
        // 将插件Apk文件添加进AssetManager
        addAssetPath.invoke(assetManager, pluginPath);
        // 获取宿主apk的Resources对象
        Resources superRes = context.getResources();
        // 获取插件apk的Resources对象
        Resources mResources = new Resources(assetManager, superRes.getDisplayMetrics(), superRes.getConfiguration());
        return mResources;
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}

pp
搜索
×
广告
Android插件化——资源加载
96  oceanLong 
2017.11.24 17:57 字数 731 阅读 1335评论 6喜欢 2
前言
资源，是APK包体积过大的病因之一。插件化技术将模块解耦，通过插件的形式加载。插件化技术中，每个插件都能够作为单独的APK独立运行。宿主启动插件的类，难免要涉及插件类中的资源问题。

那么，如何加载插件资源，就成为一个待解决的问题。

原理
参考APK打包流程：Android插件化基础-APK打包流程

Android工程在打包成apk时，会使用aapt将工程中的资源名与id在R.java中一一映射起来。

R.java

    public static final int ic_launcher=0x7f060054;
    public static final int ic_launcher_background=0x7f060055;
    public static final int ic_launcher_foreground=0x7f060056;
    public static final int notification_action_background=0x7f060057;
我们每次加载资源时，先要获取Resources。然后通过：

Drawable drawable = resources.getDrawable(resId);
获取对应的资源。

因此，我们的核心思路就是：获取插件的Resources和插件的resId。

实践
那么我们该如何获得插件的Resources呢？

ContextThemeWrapper.java

    @Override
    public Resources getResources() {
        return getResourcesInternal();
    }

    private Resources getResourcesInternal() {
        if (mResources == null) {
            if (mOverrideConfiguration == null) {
                mResources = super.getResources();
            } else {
                final Context resContext = createConfigurationContext(mOverrideConfiguration);
                mResources = resContext.getResources();
            }
        }
        return mResources;
    }
Resources.java是App资源的管理类。

    /**
     * Create a new Resources object on top of an existing set of assets in an
     * AssetManager.
     *
     * @param assets Previously created AssetManager.
     * @param metrics Current display metrics to consider when
     *                selecting/computing resource values.
     * @param config Desired device configuration to consider when
     *               selecting/computing resource values (optional).
     */
    public Resources(AssetManager assets, DisplayMetrics metrics, Configuration config) {
        this(null);
        mResourcesImpl = new ResourcesImpl(assets, metrics, config, new DisplayAdjustments());
    }
通过注释我们可以清晰的看到，真正让Resources与众不同的是AssetManager。

我们注意到这个构造方法在PackageParser#parseBaseApk方法中调用。在此我们可以想到，我们是不是可以仿照Apk的安装过程，为一个未安装的Apk创建一个Resources呢？

因此，我们继续看AssetManager.java：

/**
 * Provides access to an application's raw asset files; see {@link Resources}
 * for the way most applications will want to retrieve their resource data.
 * This class presents a lower-level API that allows you to open and read raw
 * files that have been bundled with the application as a simple stream of
 * bytes.
 */
public final class AssetManager implements AutoCloseable {
...
}
通过注释我们可以看到，这个类提供了我们访问资源文件的方式。读一下源码，可以找到一个方法：

AssetManager.java

    /**
     * Add an additional set of assets to the asset manager.  This can be
     * either a directory or ZIP file.  Not for use by applications.  Returns
     * the cookie of the added asset, or 0 on failure.
     * {@hide}
     */
    public final int addAssetPath(String path) {
        return  addAssetPathInternal(path, false);
    }
通过注释我们可以看到，这个方法提供了包装ZIP文件的方法。注释中说这不是用于Applications。但我们知道APK文件其实就是ZIP文件。

看到这里我们的思路就有了。通过这个方法，我们将插件APK的path传入，包装一个AssetManager。然后用AssetManager生成Resources，那么这个Resources就是插件的Resources。虽然插件APK并未安装，但我们仿照了安装的流程。

通过上面的分析，我们能够得到一个获取插件Resources的方法：

/**
 * 获取对应插件的Resource对象
 * @param context 宿主apk的上下文
 * @param pluginPath 插件apk的路径，带apk名
 * @return
 */
public static Resources getPluginResources(Context context, String pluginPath) {
    try {
        AssetManager assetManager = AssetManager.class.newInstance();
        // 反射调用方法addAssetPath(String path)
        Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);
        // 将插件Apk文件添加进AssetManager
        addAssetPath.invoke(assetManager, pluginPath);
        // 获取宿主apk的Resources对象
        Resources superRes = context.getResources();
        // 获取插件apk的Resources对象
        Resources mResources = new Resources(assetManager, superRes.getDisplayMetrics(), superRes.getConfiguration());
        return mResources;
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```

至此我们获得了插件的Resources，我们通过获取资源，如Drawable的方法是:

Drawable drawable = resources.getDrawable(resId);
因此，我们还缺一个resId，即插件资源在插件R.java中对应的id。

我们可以通过反射的方式，获取R.java中的id:
```java
    /**
     * 加载apk获得内部资源id
     *
     * @param context 宿主上下文
     * @param pluginPath apk路径
     */
    public static int getResId(Context context, String pluginPath, String apkPackageName, String resName) {
        try {
            //在应用安装目录下创建一个名为app_dex文件夹目录,如果已经存在则不创建
            File optimizedDirectoryFile = context.getDir("dex", Context.MODE_PRIVATE);
            // 构建插件的DexClassLoader类加载器，参数：
            // 1、包含dex的apk文件或jar文件的路径，
            // 2、apk、jar解压缩生成dex存储的目录，
            // 3、本地library库目录，一般为null，
            // 4、父ClassLoader
            DexClassLoader dexClassLoader = new DexClassLoader(pluginPath, optimizedDirectoryFile.getPath(), null, ClassLoader.getSystemClassLoader());
            //通过使用apk自己的类加载器，反射出R类中相应的内部类进而获取我们需要的资源id
            Class<?> clazz = dexClassLoader.loadClass(apkPackageName + ".R$drawable");
            Field field = clazz.getDeclaredField(resName);//得到名为resName的这张图片字段
            return field.getInt(R.id.class);//得到图片id
        } catch (Exception e) {
            e.printStackTrace();
        }
        return 0;
    }

    int resId = getResId(MainActivity.this.getApplication(), PATH, PLUGIN_PACKAGE_NAME, "ic_launcher");
    Resources resources = getPluginResources(MainActivity.this.getApplication(), PATH);
    Drawable drawable = resources.getDrawable(resId);
    mIvTest.setImageDrawable(drawable);
```
