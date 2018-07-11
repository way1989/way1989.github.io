---
date: 2016-07-02 11:16:24
title: 下载动画，图标飞入下载管理动画实现
description: 图标飞入动画
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 笔记
 - Animation
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 笔记
---

## 主要代码：

``` java
public static Button sActionBarDownload;  //title栏的下载  
    public static FrameLayout sParentView;  //获取父view  
    protected ImageView mImageSwitcher;  
    // 下载飞出动画的视图  
    public ImageView mAnimationImageView = null;  

    //获取屏幕的整个大view  
    sParentView = (FrameLayout) findViewById(R.id.parent_view);  
    mActionBarContainer = (ActionBarContainer) findViewById(R.id.actionbar);  
    sActionBarDownload = (Button) findViewById(R.id.btn_download_recommend);  

    private void iconFlyOutAnimation(final AppInfoDataBean app, final View v) {  
        int[] location = new int[2];  
        mImageSwitcher.getLocationInWindow(location);  //要下载应用图标的位置,相对于window的位置  
        int width = mImageSwitcher.getWidth();  
        int height = mImageSwitcher.getHeight();  


        if (MyActivity.sParentView != null)  //title下载的位置  
        {  
            //获取到目标view相对于window的位置  
            int[] locationRelative = new int[2];  
            MyActivity.sActionBarDownload.getLocationInWindow(locationRelative);  

            //将源view放到ImageView中  
            mAnimationImageView = new ImageView(getContext());  
            FrameLayout.LayoutParams params = new FrameLayout.LayoutParams(width, height);  
            params.leftMargin = locationRelative[0] - location[0];  
            params.topMargin = location[1] - locationRelative[1];  

            //获取到源图片的当前view，并设置到Drawable  
            ImageView iconImage = (ImageView) mImageSwitcher.getCurrentView();  
            mAnimationImageView.setImageDrawable(iconImage.getDrawable());  
            //将动画添加到控件内，源加到目标中  
            MyActivity.sParentView.addView(mAnimationImageView, params);  

            //获取目标的位置控件的长、宽  
            int[] locationDst = new int[2];  
            locationDst[0] = MyActivity.sActionBarDownload.getWidth();  
            locationDst[1] = MyActivity.sActionBarDownload.getHeight();  

            int destXp = locationDst[0] / 2;  
            int destYp = locationDst[1] / 2;  
            int destX = params.leftMargin - destXp;  
            int destY = params.topMargin - destYp + DrawUtils.dip2px(20);  
            //位移  
            Animation translateAnimation = new TranslateAnimation(-destX, DrawUtils.dip2px(12), 0, -destY);  
            translateAnimation.setDuration(600);  
            //缩放  
            Animation scaleAnimation = new ScaleAnimation(1, 0.1f, 1, 0.1f, Animation.RELATIVE_TO_SELF, 0.5f,  
                    Animation.RELATIVE_TO_SELF, 0.5f);  
            scaleAnimation.setDuration(600);  
            //旋转动画效果    
            Animation rotateAnimation = new RotateAnimation(0f, 720, Animation.RELATIVE_TO_SELF, 0.5f,  
                    Animation.RELATIVE_TO_SELF, 0.5f);  
            rotateAnimation.setDuration(600);  

            AnimationSet animationSet = new AnimationSet(true);  
            animationSet.setInterpolator(new DecelerateInterpolator(2.0f));  

            animationSet.addAnimation(rotateAnimation);  
            animationSet.addAnimation(scaleAnimation);  
            animationSet.addAnimation(translateAnimation);  
            //            BaseDataActivity.mActionBarDownload.InterceptTouchEvent(true); //设置不可点击  

            animationSet.setAnimationListener(new AnimationListener() {  
                @Override  
                public void onAnimationEnd(Animation animation) {  
                    if (mAnimationImageView != null && mAnimationImageView.getTag() == animation) {  
                        MyActivity.sParentView.removeView(mAnimationImageView);  

                        mAnimationImageView = null;  
                        // 运行完动画再下载  
                        downLoad(app);  
                    }  
                    //                    BaseDataActivity.mActionBarDownload.onInterceptTouchEvent(false);  
                }  

                @Override  
                public void onAnimationRepeat(Animation animation) {  

                }  

                @Override  
                public void onAnimationStart(Animation animation) {  

                }  
            });  

            mAnimationImageView.startAnimation(animationSet);  
            mAnimationImageView.setTag(animationSet);  
        }  
    }

```
