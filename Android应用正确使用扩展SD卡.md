# Android应用正确使用扩展SD卡

------

## 先介绍一下Android的存储

在 2.x 版本中，Android设备都是单存储，第三方App写文件，必须申请 WRITE_EXTERNAL_STORAGE 权限；

在4.0之后，Android设备开始有了内置闪存，即 primary storage，并且可以外置SD卡，即 secondary external storage device；

WRITE_EXTERNAL_STORAGE 权限变成了仅仅控制 primary storage，同时引入了 WRITE_MEDIA_STORAGE 权限来控制secondary external storage device的操作。

到了Android 4.4 KitKat，WRITE_MEDIA_STORAGE 权限仅提供给系统应用，不再授予第三方App。

关于 secondary external storage device 的写操作也有了新规定。

Android的文档是这么写的：

Link： [http://source.android.com/devices/tech/storage/index.html:][1]

> The WRITE_EXTERNAL_STORAGE permission must only grant write access to
> the primary external storage on a device. Apps must not be allowed to
> write to secondary external storage devices, except in their
> package-specific directories as allowed by synthesized permissions.
> Restricting writes in this way ensures the system can clean up files
> when applications are uninstalled.

翻译：
WRITE_EXTERNAL_STORAGE 权限，仅仅用于授权用户写 primary external storage，除了与自己包名相关的文件夹之外，应用程序不允许写secondary external storage devices。

举例来说，如果应用的包名是com.example.foo，那么外部存储上的Android/data/com.example.foo/文件夹就可随意访问，其他任何地方都不允许写，并且，存储在自己包名相关的文件夹的文件，当该应用被卸载时候也会随之被清除。

分情况来说：

> * 只有外部存储的设备
这种设备一般是android4.0之前的，只有一个存储，不受这个规则限制，还是可以随便读写，但如果你刷了4.4系统，那么就只能写自己包名相关的文件夹了。

> * 只有内部存储的设备
比如Nexus系列，sony L系列，不受这个规则限制，但是建议在自己的包名相关的文件夹写数据。

> * 既有内部存储又有外部存储
需要遵守这个规定，不能在外部存储乱写了，需要在自己的包名相关的文件夹写数据。

Google做了这个限制后解决了这个问题：

随便一个App，都会在/sdcard、/sdcard1 上建一个目录，删了也会重新建，即使被卸载，也会留下一些垃圾文件。

但是，也产生了一个问题：

类似于视频、图像处理这种想在外部存储缓存大量音视频文件，并且App被卸载后还想保留的，就没办法了。
 
## 开发中应该怎么使用？

作为一个程序员，想必你也很讨厌App在SD卡根目录乱建目录吧，那就从我做起，来遵守Google的这一规定吧。

通过Context.getExternalFilesDir()方法可以获取到 SDCard/Android/data/{package_name}/files/ ，储存一些长时间保存的数据；

通过Context.getExternalCacheDir()方法可以获取到 SDCard/Android/data/{package_name}/cache/，储存临时缓存数据；

这两个目录分别对应 设置->应用->应用详情里面的”清除数据“与”清除缓存“选项。

一个获取外部存储Cache的例子：

>      /**
>       * 获取拓展存储Cache的绝对路径
>       *
>       * @param context
>       */
>      public static String getExternalCacheDir(Context context) {
> 
>           if (!isMounted())
>                return null;
>          
>           StringBuilder sb = new StringBuilder();
> 
>           File file = context.getExternalCacheDir();
> 
>           // In some case, even the sd card is mounted,
>           // getExternalCacheDir will return null
>           // may be it is nearly full.
>          
>           if (file != null) {
>                sb.append(file.getAbsolutePath()).append(File.separator);
>           } else {
>                sb.append(Environment.getExternalStorageDirectory().getPath()).append("/Android/data/").append(context.getPackageName())
>                          .append("/cache/").append(File.separator).toString();
>           }
>          
>           return sb.toString();
>      }


参考：<br/>
 [Google Plus](https://plus.google.com/+TodLiebeck/posts/gjnmuaDM8sn)<br/>
 [Android doc](http://source.android.com/devices/tech/storage/index.html)
