---
date: 2016-06-05 11:16:24
title: 蓝牙传输文件代码
description: 蓝牙传输文件简短小代码
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 笔记
 - Bluetooth
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 笔记
---

## 主要代码：

``` java
private void sendFile(Activity activity) {
    PackageManager localPackageManager = activity.getPackageManager();
    Intent intent = new Intent();
    HashMap<String, ActivityInfo> shareItems = new HashMap<>();
    try {
        intent.setAction(Intent.ACTION_SEND);
        String filepath = App.getContext().getPackageManager()
              .getApplicationInfo(BuildConfig.APPLICATION_ID, 0).sourceDir;
        File file = new File(filepath);
        intent.putExtra(Intent.EXTRA_STREAM, Uri.fromFile(file));
        // intent.putExtra(Intent.EXTRA_STREAM,
        // Uri.fromFile(new File(localApplicationInfo.sourceDir)));
        intent.setType("*/*");
        List<ResolveInfo> resolveInfos = localPackageManager.queryIntentActivities(intent, 0);
        for(ResolveInfo resolveInfo : resolveInfos){
           ActivityInfo activityInfo = resolveInfo.activityInfo;
           String progressName = activityInfo.applicationInfo.processName;
           if(progressName.contains("bluetooth"))
                shareItems.put(progressName, activityInfo);
            }
      } catch (Exception e) {
          Log.e(e.getMessage(), e);
      }
      ActivityInfo activityInfo = shareItems.get("com.android.bluetooth");
      if (activityInfo == null)
          activityInfo = shareItems.get("com.mediatek.bluetooth");
      if (activityInfo == null) {
          Iterator<ActivityInfo> iterator = shareItems.values().iterator();
          if (iterator.hasNext())
                activityInfo =  iterator.next();
      }
      if (activityInfo != null) {
          intent.setComponent(new ComponentName(activityInfo.packageName, activityInfo.name));
          activity.startActivityForResult(intent, BLUETOOTH_SHARE_REQUEST_CODE);
      }
  }

```
