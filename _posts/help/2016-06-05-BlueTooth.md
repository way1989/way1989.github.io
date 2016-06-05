---
category: books-2016
published: true
layout: post
title: 蓝牙传输文件代码
description: 蓝牙传输文件简短小代码
---

``` 
java
  /**
     * 通过蓝牙发送文件
     */
    private void sendFile(Activity activity) {
        PackageManager localPackageManager = activity.getPackageManager();
        Intent localIntent = null;
        HashMap<String, ActivityInfo> localHashMap = null;
        try {
            localIntent = new Intent();
            localIntent.setAction(Intent.ACTION_SEND);
            File file = new File(TAExternalOverFroyoUtils.getDiskCacheDir(this,
                    Constants.DOWNLOAD_DIR).getAbsolutePath(),
                    TextUtils.genApkName(worm.getWormid()));
            localIntent.putExtra(Intent.EXTRA_STREAM, Uri.fromFile(file));
            // localIntent.putExtra(Intent.EXTRA_STREAM,
            // Uri.fromFile(new File(localApplicationInfo.sourceDir)));
            localIntent.setType("*/*");
            List<ResolveInfo> localList = localPackageManager.queryIntentActivities(
                    localIntent, 0);
            localHashMap = new HashMap<String, ActivityInfo>();
            Iterator<ResolveInfo> localIterator1 = localList.iterator();
            while (localIterator1.hasNext()) {
                ResolveInfo resolveInfo = (ResolveInfo) localIterator1.next();
                ActivityInfo localActivityInfo2 = resolveInfo.activityInfo;
                String str = localActivityInfo2.applicationInfo.processName;
                if (str.contains("bluetooth"))
                    localHashMap.put(str, localActivityInfo2);
            }
        } catch (Exception localException) {
            ToastHelper.showBlueToothSupportErr(activity);
        }
        if (localHashMap.size() == 0)
            ToastHelper.showBlueToothSupportErr(activity);
        ActivityInfo localActivityInfo1 = (ActivityInfo) localHashMap
                .get("com.android.bluetooth");
        if (localActivityInfo1 == null) {
            localActivityInfo1 = (ActivityInfo) localHashMap
                    .get("com.mediatek.bluetooth");
        }
        if (localActivityInfo1 == null) {
            Iterator<ActivityInfo> localIterator2 = localHashMap.values().iterator();
            if (localIterator2.hasNext())
                localActivityInfo1 = (ActivityInfo) localIterator2.next();
        }
        if (localActivityInfo1 != null) {
            localIntent.setComponent(new ComponentName(
                    localActivityInfo1.packageName, localActivityInfo1.name));
            activity.startActivityForResult(localIntent, 4098);
            return;
        }
        ToastHelper.showBlueToothSupportErr(activity);
    }
```