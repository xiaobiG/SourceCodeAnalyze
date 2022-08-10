# APK 的安装方式

安装 APK 主要分为以下三种场景

- 安装系统应用：系统启动后调用 PackageManagerService.main() 初始化注册解析安装工作

```java
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    // Self-check for initial settings.
    PackageManagerServiceCompilerMapping.checkProperties();

    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);
    m.enableSystemUserPackages();
    ServiceManager.addService("package", m);
    final PackageManagerNative pmn = m.new PackageManagerNative();
    ServiceManager.addService("package_native", pmn);
    return m;
}
```

- 通过 adb 安装：通过 pm 参数，调用 PM 的 runInstall 方法，进入 PackageManagerService 安装安装工作
- 通过系统安装器 PackageInstaller 进行安装：先调用 InstallStart 进行权限检查之后启动 PackageInstallActivity，调用 PackageInstallActivity 的 startInstall 方法，点击 OK 按钮后进入 PackageManagerService 完成拷贝解析安装工作



## 普通应用调起安装APP界面

```java
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);

/*
* 自Android N开始，是通过FileProvider共享相关文件，但是Android Q对公
* 有目录 File API进行了限制，只能通过Uri来操作
*/
if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q){
    // filePath是通过ContentResolver得到的
    intent.setDataAndType(Uri.parse(filePath) ,"application/vnd.android.package-archive");
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
}else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    Uri contentUri = FileProvider.getUriForFile(mContext, "com.dhl.file.fileProvider", file);
    intent.setDataAndType(contentUri, "application/vnd.android.package-archive");
} else {
    intent.setDataAndType(Uri.fromFile(file), "application/vnd.android.package-archive");
}
startActivity(intent);

// 需要在AndroidManifest添加权限
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" /> 

```

