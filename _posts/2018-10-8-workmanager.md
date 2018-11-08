---
layout: post
title: 使用workManager实现每日定时推送的尝试
tags: [Android]
---

使用workManager实现每日定时推送通知

> 写这篇博客的缘由来自一个需求：
>
> 每日定时推送通知，提醒用户完成签到。如果后台被清理，则在打开App后立即推送。

众所周知，如今的Android国产ROM想要实现定时操作需要做极强的保活。

##### 然而一旦做了保活，那么程序就可能会消耗没有必要的资源，变得很流氓，这并不是我们想要看到的。

于是~~**jobSchedule**~~应运而生，这个库应该是最完美的解决方案，但有**api**限制，并不能满足我的需要。

后来尝试了**~~AlarmManager~~**，这个工具也能较精确地定时工作，但是一旦程序被杀后台，就再也不起作用。而且由于采用继承Receiver的方式，**在8点打开app无法收到本应是7点的通知，因此也无法满足需要。**

于是选择了**[workManager](https://github.com/googlecodelabs/android-workmanager)**，这个比较新鲜的官方框架。

| 各种情况是否定时推送 | 后台未被清理 | 后台被清理        |
| :------------------- | ------------ | ----------------- |
| 原生ROM              | 推送         | 推送              |
| 国产ROM              | 推送         | 打开App后继续推送 |

在程序没有被杀的情况下，能够完成定时工作。

在程序被杀后，若是原生ROM，不会有影响；若是国产ROM，不会自动推送通知，但会在打开App的第一时间自动调用代码，完成推送。

###### 愚以为，微博，知乎App也是用类似这样的操作实现的推送，因为你杀掉后台后并不能接收到推送（一加5t 氢os 8.1），但是一旦重新打开App，会收到之前的提醒。

##### 当然，这样做的缺点就是无法做到精确定时，因为workManager的重复工作间隔必须大于15分钟。因此即使使用最短的时间间隔，也最多只能保证精度为15*2=30分钟。

做到定时的原理就是，每15分钟调用一次，查看时间是否是在指定的时间段。

如果在时间段内，且未推送，那么推送，并记录。如果不在时间段内，则重置记录为未推送。

有点绕，需要理一理=_=。如果说精度大于一小时，就不用判断了。

至于使用方法就很简单了，其他博客也有详细的讲解，这里主要示例定时代码，定时7点-7点30分推送任务。

1.添加依赖

```groovy
implementation "android.arch.work:work-runtime:1.0.0-alpha09"
```

2.实现Worker的子类

```kotlin
import android.app.PendingIntent
import android.content.Intent
import androidx.work.Worker
import com.mredrock.cyxbs.common.utils.extensions.defaultSharedPreferences
import com.mredrock.cyxbs.common.utils.extensions.editor
import com.mredrock.cyxbs.mine.util.NotificationUtil
import java.util.*

/**
 * Created by zia on 2018/10/8.
 * 每日签到提醒的work
 */
class SignWorker : Worker() {

    private val FLAG = "SIGNPUSH"

    override fun doWork(): Worker.Result {

        //读取是否通知过
        val isPush = applicationContext.defaultSharedPreferences.getBoolean(FLAG, false)

        if (compareCurrentHour(7)) {
            if (!isPush) {
                //如果在指定时间段，并且没有推送过通知
                applicationContext.defaultSharedPreferences.editor {
                    //写入已通知
                    putBoolean(FLAG, true)
                }
                //继续后面的推送通知代码
            } else {
                //在指定时间段，已推送过了，则不再推送
                return Result.RETRY
            }
        } else {
            //不在时间段，重置标志位false
            applicationContext.defaultSharedPreferences.editor {
                putBoolean(FLAG, false)
            }
            return Result.RETRY
        }

        val resultIntent = Intent(applicationContext, DailySignActivity::class.java)
        val intent = PendingIntent.getActivity(applicationContext, 0, resultIntent, PendingIntent.FLAG_UPDATE_CURRENT)

        //推送通知
        NotificationUtil.makeNotification(applicationContext, "赶快去签到领取积分哦~", CHANNEL_ID = "sign", pendingIntent = intent)

        return Worker.Result.SUCCESS
    }

    private fun compareCurrentHour(targetHour: Int): Boolean {
        val current = Calendar.getInstance().get(Calendar.HOUR_OF_DAY)
        return current == targetHour
    }
}

```

3.在其他地方调用

```kotlin
//获取一个builder
val request = PeriodicWorkRequest
                        .Builder(SignWorker::class.java, 15, TimeUnit.MINUTES)
                        .build()
//插入worker队列，并且使用enqueueUniquePeriodicWork方法，防止重复
WorkManager.getInstance().enqueueUniquePeriodicWork(workName,ExistingPeriodicWorkPolicy.KEEP, request)
```

4.用户取消推送

```kotlin
WorkManager.getInstance().cancelUniqueWork(workName)
```

