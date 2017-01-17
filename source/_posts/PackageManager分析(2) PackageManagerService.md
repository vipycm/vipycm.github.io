layout: "post"
title: "PackageManager分析(2) PackageManagerService"
date: "2017-01-17 14:41"
---
在第1节中,通过反射调用PackageManager的getPackageInfo方法实现了package size的获取,那么这个系统接口做了哪些事情呢,接下来就通过函数的调用堆栈来分析一下PackageManagerService这个服务.
<!--more-->
### PackageManager
首先看看我们调用的PackageManager
```
// android.content.pm.PackageManager

public void getPackageSizeInfo(String packageName, IPackageStatsObserver observer) {
    getPackageSizeInfoAsUser(packageName, UserHandle.myUserId(), observer);
}

public abstract void getPackageSizeInfoAsUser(String packageName, @UserIdInt int userId,
          IPackageStatsObserver observer);
```

### ApplicationPackageManager
getPackageSizeInfoAsUser是一个抽象方法,通过Context.getPackageManager()得到的时ApplicationPackageManager的实例,继续跟进ApplicationPackageManager
```
//android.app.ApplicationPackageManager

private final IPackageManager mPM;

public void getPackageSizeInfoAsUser(String packageName, int userHandle, IPackageStatsObserver observer) {
    try {
        this.mPM.getPackageSizeInfo(packageName, userHandle, observer);
    } catch (RemoteException var5) {
        throw var5.rethrowFromSystemServer();
    }
}
```

### PackageManagerService
ApplicationPackageManager 其实只是对IPackageManager这个aidl接口的封装,该接口的实现在PackageManagerService中.属于framework层的代码,代码位于
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
```
// com.android.server.pm.PackageManagerService

public void getPackageSizeInfo(final String packageName, int userHandle,
        final IPackageStatsObserver observer) {
    mContext.enforceCallingOrSelfPermission(
            android.Manifest.permission.GET_PACKAGE_SIZE, null);
    if (packageName == null) {
        throw new IllegalArgumentException("Attempt to get size of null packageName");
    }

    PackageStats stats = new PackageStats(packageName, userHandle);

    /*
     * Queue up an async operation since the package measurement may take a
     * little while.
     */
    Message msg = mHandler.obtainMessage(INIT_COPY);
    msg.obj = new MeasureParams(stats, observer);
    mHandler.sendMessage(msg);
}

```

getPackageSizeInfo里面把这个请求封装成MeasureParams之后通过mHandler send出去之后这个方法就返回了,顺着mHandler继续寻找,会调用到MeasureParams的handleStartCopy来处理请求
```
// com.android.server.pm.PackageManagerService.MeasureParams

class MeasureParams extends HandlerParams {
    private final PackageStats mStats;
    private boolean mSuccess;

    private final IPackageStatsObserver mObserver;

    public MeasureParams(PackageStats stats, IPackageStatsObserver observer) {
        super(new UserHandle(stats.userHandle));
        mObserver = observer;
        mStats = stats;
    }

    @Override
    public String toString() {
        return "MeasureParams{"
            + Integer.toHexString(System.identityHashCode(this))
            + " " + mStats.packageName + "}";
    }

    @Override
    void handleStartCopy() throws RemoteException {
        synchronized (mInstallLock) {
            mSuccess = getPackageSizeInfoLI(mStats.packageName, mStats.userHandle, mStats);
        }

        if (mSuccess) {
            final boolean mounted;
            if (Environment.isExternalStorageEmulated()) {
                mounted = true;
            } else {
                final String status = Environment.getExternalStorageState();
                mounted = (Environment.MEDIA_MOUNTED.equals(status)
                        || Environment.MEDIA_MOUNTED_READ_ONLY.equals(status));
            }

            if (mounted) {
                final UserEnvironment userEnv = new UserEnvironment(mStats.userHandle);

                mStats.externalCacheSize = calculateDirectorySize(mContainerService,
                        userEnv.buildExternalStorageAppCacheDirs(mStats.packageName));

                mStats.externalDataSize = calculateDirectorySize(mContainerService,
                        userEnv.buildExternalStorageAppDataDirs(mStats.packageName));

                // Always subtract cache size, since it's a subdirectory
                mStats.externalDataSize -= mStats.externalCacheSize;

                mStats.externalMediaSize = calculateDirectorySize(mContainerService,
                        userEnv.buildExternalStorageAppMediaDirs(mStats.packageName));

                mStats.externalObbSize = calculateDirectorySize(mContainerService,
                        userEnv.buildExternalStorageAppObbDirs(mStats.packageName));
            }
        }
    }

    @Override
    void handleReturnCode() {
        if (mObserver != null) {
            try {
                mObserver.onGetStatsCompleted(mStats, mSuccess);
            } catch (RemoteException e) {
                Slog.i(TAG, "Observer no longer exists.");
            }
        }
    }

    @Override
    void handleServiceError() {
        Slog.e(TAG, "Could not measure application " + mStats.packageName
                        + " external storage");
    }
}
```

从handleStartCopy方法可以看到,在调用getPackageSizeInfoLI方法(计算/data/data/{pkg}/xxx下的size)成功之后,再计算与扩展卡相关的各种size,接下来看看getPackageSizeInfoLI的实现

```
final Installer mInstaller;

private boolean getPackageSizeInfoLI(String packageName, int userHandle,
        PackageStats pStats) {
    if (packageName == null) {
        Slog.w(TAG, "Attempt to get size of null packageName.");
        return false;
    }
    PackageParser.Package p;
    boolean dataOnly = false;
    String libDirRoot = null;
    String asecPath = null;
    PackageSetting ps = null;
    synchronized (mPackages) {
        p = mPackages.get(packageName);
        ps = mSettings.mPackages.get(packageName);
        if(p == null) {
            dataOnly = true;
            if((ps == null) || (ps.pkg == null)) {
                Slog.w(TAG, "Package named '" + packageName +"' doesn't exist.");
                return false;
            }
            p = ps.pkg;
        }
        if (ps != null) {
            libDirRoot = ps.legacyNativeLibraryPathString;
        }
        if (p != null && (p.isForwardLocked() || p.applicationInfo.isExternalAsec())) {
            final long token = Binder.clearCallingIdentity();
            try {
                String secureContainerId = cidFromCodePath(p.applicationInfo.getBaseCodePath());
                if (secureContainerId != null) {
                    asecPath = PackageHelper.getSdFilesystem(secureContainerId);
                }
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
    }
    String publicSrcDir = null;
    if(!dataOnly) {
        final ApplicationInfo applicationInfo = p.applicationInfo;
        if (applicationInfo == null) {
            Slog.w(TAG, "Package " + packageName + " has no applicationInfo.");
            return false;
        }
        if (p.isForwardLocked()) {
            publicSrcDir = applicationInfo.getBaseResourcePath();
        }
    }
    // TODO: extend to measure size of split APKs
    // TODO(multiArch): Extend getSizeInfo to look at the full subdirectory tree,
    // not just the first level.
    // TODO(multiArch): Extend getSizeInfo to look at *all* instruction sets, not
    // just the primary.
    String[] dexCodeInstructionSets = getDexCodeInstructionSets(getAppDexInstructionSets(ps));

    String apkPath;
    File packageDir = new File(p.codePath);

    if (packageDir.isDirectory() && p.canHaveOatDir()) {
        apkPath = packageDir.getAbsolutePath();
        // If libDirRoot is inside a package dir, set it to null to avoid it being counted twice
        if (libDirRoot != null && libDirRoot.startsWith(apkPath)) {
            libDirRoot = null;
        }
    } else {
        apkPath = p.baseCodePath;
    }

    int res = mInstaller.getSizeInfo(p.volumeUuid, packageName, userHandle, apkPath,
            libDirRoot, publicSrcDir, asecPath, dexCodeInstructionSets, pStats);
    if (res < 0) {
        return false;
    }

    // Fix-up for forward-locked applications in ASEC containers.
    if (!isExternal(p)) {
        pStats.codeSize += pStats.externalCodeSize;
        pStats.externalCodeSize = 0L;
    }

    return true;
}
```
其中最重要的一句是调用mInstaller.getSizeInfo(...)来计算size.

### Installer
接着来看看Installer的getSizeInfo方法,代码位于:
frameworks/base/services/core/java/com/android/server/pm/Installer.java
```
private final InstallerConnection mInstaller;

public int getSizeInfo(String uuid, String pkgName, int persona, String apkPath,
        String libDirPath, String fwdLockApkPath, String asecPath, String[] instructionSets,
        PackageStats pStats) {
    for (String instructionSet : instructionSets) {
        if (!isValidInstructionSet(instructionSet)) {
            Slog.e(TAG, "Invalid instruction set: " + instructionSet);
            return -1;
        }
    }

    StringBuilder builder = new StringBuilder("getsize");
    builder.append(' ');
    builder.append(escapeNull(uuid));
    builder.append(' ');
    builder.append(pkgName);
    builder.append(' ');
    builder.append(persona);
    builder.append(' ');
    builder.append(apkPath);
    builder.append(' ');
    // TODO: Extend getSizeInfo to look at the full subdirectory tree,
    // not just the first level.
    builder.append(libDirPath != null ? libDirPath : "!");
    builder.append(' ');
    builder.append(fwdLockApkPath != null ? fwdLockApkPath : "!");
    builder.append(' ');
    builder.append(asecPath != null ? asecPath : "!");
    builder.append(' ');
    // TODO: Extend getSizeInfo to look at *all* instrution sets, not
    // just the primary.
    builder.append(instructionSets[0]);

    String s = mInstaller.transact(builder.toString());
    String res[] = s.split(" ");

    if ((res == null) || (res.length != 5)) {
        return -1;
    }
    try {
        pStats.codeSize = Long.parseLong(res[1]);
        pStats.dataSize = Long.parseLong(res[2]);
        pStats.cacheSize = Long.parseLong(res[3]);
        pStats.externalCodeSize = Long.parseLong(res[4]);
        return Integer.parseInt(res[0]);
    } catch (NumberFormatException e) {
        return -1;
    }
}
```
其中最重要的一句时调用mInstaller.transact(...)发送了一个getsize的命令来计算size.

### InstallerConnection
继续跟进InstallerConnection,代码位于:
frameworks/base/core/java/com/android/internal/os/InstallerConnection.java
```
public synchronized String transact(String cmd) {
    if (!connect()) {
        Slog.e(TAG, "connection failed");
        return "-1";
    }

    if (!writeCommand(cmd)) {
        /*
         * If installd died and restarted in the background (unlikely but
         * possible) we'll fail on the next write (this one). Try to
         * reconnect and write the command one more time before giving up.
         */
        Slog.e(TAG, "write command failed? reconnect!");
        if (!connect() || !writeCommand(cmd)) {
            return "-1";
        }
    }
    if (LOCAL_DEBUG) {
        Slog.i(TAG, "send: '" + cmd + "'");
    }

    final int replyLength = readReply();
    if (replyLength > 0) {
        String s = new String(buf, 0, replyLength);
        if (LOCAL_DEBUG) {
            Slog.i(TAG, "recv: '" + s + "'");
        }
        return s;
    } else {
        if (LOCAL_DEBUG) {
            Slog.i(TAG, "fail");
        }
        return "-1";
    }
}

private boolean connect() {
    if (mSocket != null) {
        return true;
    }
    Slog.i(TAG, "connecting...");
    try {
        mSocket = new LocalSocket();

        LocalSocketAddress address = new LocalSocketAddress("installd",
                LocalSocketAddress.Namespace.RESERVED);

        mSocket.connect(address);

        mIn = mSocket.getInputStream();
        mOut = mSocket.getOutputStream();
    } catch (IOException ex) {
        disconnect();
        return false;
    }
    return true;
}

private boolean writeCommand(String cmdString) {
    final byte[] cmd = cmdString.getBytes();
    final int len = cmd.length;
    if ((len < 1) || (len > buf.length)) {
        return false;
    }

    buf[0] = (byte) (len & 0xff);
    buf[1] = (byte) ((len >> 8) & 0xff);
    try {
        mOut.write(buf, 0, 2);
        mOut.write(cmd, 0, len);
    } catch (IOException ex) {
        Slog.e(TAG, "write error");
        disconnect();
        return false;
    }
    return true;
}

```
这里面的逻辑比较简单,就是通过LocalSocketAddress连接到一个服务器,然后写入命令.至于这个服务器在哪里,里面的逻辑时怎么样的,我们下一节再进一步分析.
