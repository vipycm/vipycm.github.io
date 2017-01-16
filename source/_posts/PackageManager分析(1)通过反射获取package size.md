layout: "post"
title: "PackageManager分析(1)通过反射获取package size"
date: "2017-01-16 21:01"
categories:
- Android
tags:
- PackageManager
---
## 通过反射获取package size
[源码地址](https://github.com/vipycm/mao-android/blob/master/app/src/main/java/com/vipycm/mao/ui/PmFragment.java)

### 添加权限
```
android.permission.GET_PACKAGE_SIZE
```
<!--more-->
### 核心代码
```
private void getPackageInfo(Context context, String pkg) {
    PackageManager pm = context.getPackageManager();
    try {
        Method method_getPackageSizeInfo = pm.getClass().getMethod("getPackageSizeInfo", String.class, IPackageStatsObserver.class);
        method_getPackageSizeInfo.invoke(pm, pkg, new IPackageStatsObserver.Stub() {

  [f2cd7e9b]: aa "title"

            @Override
            public IBinder asBinder() {
                log.d("asBinder");
                return super.asBinder();
            }

            @Override
            public void onGetStatsCompleted(PackageStats packageStats, boolean b) throws RemoteException {
                final StringBuilder sb = new StringBuilder("onGetStatsCompleted\n");
                sb.append("packageName:").append(packageStats.packageName).append("\n");
                sb.append("cacheSize:").append(packageStats.cacheSize).append("\n");
                sb.append("dataSize:").append(packageStats.dataSize).append("\n");
                sb.append("externalDataSize:").append(packageStats.externalDataSize).append("\n");
                log.i(sb.toString());
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        txt_content.setText(sb);
                    }
                });

            }
        });
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
