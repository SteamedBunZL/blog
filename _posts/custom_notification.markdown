---
layout:     post
title:      "自定义通知总结"
subtitle:   " \"Hello World, Hello Blog\""
date:       2015-01-29 12:00:00
author:     "Steve"
catalog: true
---



#自定义通知总结


最近公司在做自定义通知栏的相关需求，查询了相关的资料，把用到的东西和踩的一些坑做一下总结。自定义通知栏，也就是有别于系统的默认的样式，我们在看比如360、墨迹天气等等，都采用了自己特有的通知，那么这些通知是怎么做的呢？

先不说自定义通知栏，我们先看一个能发送普通通知的小例子


```java

   /**
     * 测试最简单的通知
     * @param context
     */
    public void testNotification(Context context){
        android.app.NotificationManager manager = (android.app.NotificationManager) context.getSystemService(NOTIFICATION_SERVICE);
        NotificationCompat.Builder builder = new NotificationCompat.Builder(context);
        Notification notification = builder
                .setContentTitle("这是通知标题")
                .setContentText("这是通知内容")
                .setWhen(System.currentTimeMillis())
                .setSmallIcon(R.mipmap.ic_launcher)  //5.0  如果不设置小图标 会崩溃????
                //.setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))  //但是可以不设置大图标,哦哦,原来如此
                .build();
        manager.notify(1, notification);
    }
```

![](file:///Users/zhanglong/Desktop/自定义通知总结/notifcaiton1.png)

上面是最最简单的通知了，相信大部人都知道如何去发一个通知

先前我们弹出系统样式的通知，所以不需要去做适配，但是现在我们如果要做一个自定义通知的话，产品这时候就会去要求你"我们的通知，是自定义通知，但是要做到跟系统的差别尽量小，或者看上去只有各别地方不同，大部分要做到一致",这个问题实际上涉及了自定义通知栏的适配的问题，在网上也找了好多资料，发现最靠谱的要属hackware 在简书中的文章做的适配最好，即去获取系统通知栏的的字体颜色，然后进行自定义通知的实现。

文章的地址：http://www.jianshu.com/p/426d85f34561

我们直接拿来使用

```java

public class NotificationColorEngine {

    private static final String DUMMY_TITLE = "DUMMY_TITLE";
    private static int titleColor;
    private static final double COLOR_THRESHOLD = 180.0;

    public static int getNotificationColor(Context context){
        if (context instanceof AppCompatActivity){
            return getNotificationColorCompat(context);
        }else{
            return getNotificationColorInternal(context);
        }
    }


    private static int getNotificationColorInternal(Context context){
        NotificationCompat.Builder builder = new NotificationCompat.Builder(context);
        builder.setContentTitle(DUMMY_TITLE);
        Notification notification = builder.build();
        ViewGroup notificationRoot = (ViewGroup) notification.contentView.apply(context,new FrameLayout(context));
        TextView title = (TextView) notificationRoot.findViewById(android.R.id.title);
        if (title == null){
            iteratorView(notificationRoot, new Filter() {
                @Override
                public void filter(View view) {
                    if (view instanceof TextView){
                        TextView textView = (TextView) view;
                        if (DUMMY_TITLE.equals(textView.getText().toString())){
                            titleColor = textView.getCurrentTextColor();
                        }
                    }
                }
            });
            return titleColor;
        }else{
            return title.getCurrentTextColor();
        }
    }

    private static int getNotificationColorCompat(Context context){
        NotificationCompat.Builder builder = new NotificationCompat.Builder(context);
        Notification notification = builder.build();
        int layoutId = notification.contentView.getLayoutId();
        ViewGroup notificationRoot = (ViewGroup) LayoutInflater.from(context).inflate(layoutId,null);
        TextView title = (TextView) notificationRoot.findViewById(android.R.id.title);
        if (title == null){
            final List<TextView> textViews = new ArrayList<>();
            iteratorView(notificationRoot, new Filter() {
                @Override
                public void filter(View view) {
                    if (view instanceof TextView){
                        textViews.add((TextView) view);
                    }
                }
            });

            float minTextSize = Integer.MIN_VALUE;
            int index = 0;
            float currentSize = 0;
            for(int i = 0,j = textViews.size();i<j;i++){
                currentSize = textViews.get(i).getTextSize();
                if (currentSize>minTextSize){
                    minTextSize = currentSize;
                    index = i;
                }
            }
            return textViews.get(index).getCurrentTextColor();

        }else {
            return title.getCurrentTextColor();
        }
    }

    private static void iteratorView(View view, Filter filter){
        if (view == null || filter == null)
            return;

        filter.filter(view);

        if (view instanceof ViewGroup){
            ViewGroup container = (ViewGroup) view;
            for(int i = 0,j = container.getChildCount();i<j;i++){
                iteratorView(container.getChildAt(i),filter);
            }
        }
    }

    interface Filter{
        void filter(View view);
    }

    public static boolean isDarkNotificationBar(Context context){
        return Build.VERSION.SDK_INT>=Build.VERSION_CODES.N?false:!isColorSimilar(Color.BLACK,getNotificationColor(context));
    }

    public static boolean isColorSimilar(int baseColor,int color){
        int simpleBaseColor = baseColor | 0xff000000;
        int simpleColor = color | 0xff000000;
        int baseRed = Color.red(simpleBaseColor) - Color.red(simpleColor);
        int baseBlue = Color.blue(simpleBaseColor) - Color.blue(simpleColor);
        int baseGreen = Color.green(simpleBaseColor) - Color.green(simpleColor);
        double value = Math.sqrt(baseRed * baseRed + baseGreen * baseGreen + baseBlue * baseBlue);
        if (value < COLOR_THRESHOLD)
            return true;
        return false;
    }
}


```

基本思想就是去拿系统通知的view 或者反射，或者什么的反正我拿到了，然后使用isColorSimilar方法去判断这个颜色更加贴近黑或白(毕竟多数的系统通知也无非黑或者白么，至于黑到什么程度白到什么程度，这里就不关心啦)，这里使用了颜色相比的算法，是算三原色的平方和的根，这个值越大，表示两个颜色相差的越多，越小表示颜色越贴近，原文中说的非常详细了，这里就不再赘述了。

拿到了系统的颜色，好开心啊，这回产品无论要求怎么适配，我们都游刃有余了，慢着，只是单单显示出来好像并没有什么难度啊，那么问题来了，到底我们应该使用什么API去让自定义通知显示出来呢？

先看一下最终的效果

![](file:///Users/zhanglong/Desktop/notification2.png)

既然说到了自定义通知栏，那么我们就离不开remoteviews的使用，因为离开它，你也实现不了自定义通知栏，感觉说了一大堆废话。。。。

不多说，上代码

```java
RemoteViews remoteViews = new RemoteViews(mContext.getPackageName(),R.layout.notification_two_line);
```
我们可以使用自定义的布局去生成一个remoteviews 然后再把remoteviews设置给notification的contentview属性,如果这么做的话,通过notify弹出的通知就是你自定义的布局

remoteviews去设置文字和图片有区别于我们一般使用的view,一般的view我们有findviewbyid去找到，但是remoteviews的更新，使用的是另外的方式



这里实际上系统的底色是白色的，但是我们应用里可以去显示黑色的主题，也就是强行把通知栏的底色设置为黑色，刚开始在这里还纠结了一下下，因为我们