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
            List<ResolveInfo> resolveInfos = localPackageManager.queryIntentActivities(
                    intent, 0);
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