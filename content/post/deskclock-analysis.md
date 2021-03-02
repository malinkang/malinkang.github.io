---
title: 系统闹钟源码分析
date: 2021-01-19 09:21:04
tags: ["源码分析"]
draft: true
---



## 核心类

* UI部分

  * DeskClock：主界面，由ViewPager实现。
  * DeskClockFragment：首页四个Fragment的基类。
  * AlarmClockFragment：闹钟Fragment
  * ClockFragment：时钟Fragment
  * StopwatchFragment：秒表Fragment
  * TimerFragment：定时器Fragment

* 数据

  * ClockDatabaseHelper：数据库Helper，闹钟应用创建了两表，表`alarm_templates`存储所有的闹钟，`alarm_instances`存储了开启的闹钟。
  * ClockProvider：ContentProvider
  * Alarm：闹钟实体Bean，与`alarm_templates`对应，定义了对`alarm_templates`表增删改查的静态方法
  * AlarmInstance：闹钟实体Bean，与`alarm_instances`对应，定义了对`alarm_instances`表增删改查的静态方法。

* AlarmStateManager：闹钟状态管理类

* AlarmService：负责计时完成显示通知和铃声、以及关闭通知和铃声。

  

## 闹钟的工作流程

![image-20210120194327601](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/image-20210120194327601.png)



## 数据库设计

### ClockDatabaseHelper 

`alarm_templates`表结构

![image-20210119173719416](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/image-20210119173719416.png)

`alarm_instances`表结构

![image-20210119173836511](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/image-20210119173836511.png)

## 查询

### Alarm#getAlarmsCursorLoader()

```kotlin
//Alarm
fun getAlarmsCursorLoader(context: Context): CursorLoader {
    //从ContentProvider中获取
    return object : CursorLoader(context, AlarmsColumns.ALARMS_WITH_INSTANCES_URI,//uri
            QUERY_ALARMS_WITH_INSTANCES_COLUMNS, null, null, DEFAULT_SORT_ORDER) {
        override fun onContentChanged() {
            // There is a bug in Loader which can result in stale data if a loader is stopped
            // immediately after a call to onContentChanged. As a workaround we stop the
            // loader before delivering onContentChanged to ensure mContentChanged is set to
            // true before forceLoad is called.
            if (isStarted && !isAbandoned) {
                stopLoading()
                super.onContentChanged()
                startLoading()
            } else {
                super.onContentChanged()
            }
        }
        override fun loadInBackground(): Cursor? {
            // Prime the ringtone title cache for later access. Most alarms will refer to
            // system ringtones.
            DataModel.dataModel.loadRingtoneTitles()
            return super.loadInBackground()
        }
    }
}
```

### ClockProvider#query()

```kotlin
override fun query(
    uri: Uri,
    projectionIn: Array<String?>?,
    selection: String?,
    selectionArgs: Array<String?>?,
    sort: String?
): Cursor? {
    val qb = SQLiteQueryBuilder()
    val db: SQLiteDatabase = mOpenHelper.readableDatabase
    // Generate the body of the query
    when (sURIMatcher.match(uri)) {
        ALARMS -> qb.tables = ALARMS_TABLE_NAME
        ALARMS_ID -> {
            qb.tables = ALARMS_TABLE_NAME
            qb.appendWhere(BaseColumns._ID + "=")
            qb.appendWhere(uri.lastPathSegment!!)
        }
        INSTANCES -> qb.tables = INSTANCES_TABLE_NAME
        INSTANCES_ID -> {
            qb.tables = INSTANCES_TABLE_NAME
            qb.appendWhere(BaseColumns._ID + "=")
            qb.appendWhere(uri.lastPathSegment!!)
        }
        ALARMS_WITH_INSTANCES -> { //匹配到这里
            qb.tables = ALARM_JOIN_INSTANCE_TABLE_STATEMENT //表
            qb.appendWhere(ALARM_JOIN_INSTANCE_WHERE_STATEMENT)
            qb.projectionMap = sAlarmsWithInstancesProjection
        }
        else -> throw IllegalArgumentException("Unknown URI $uri")
    }
    val ret: Cursor? = qb.query(db, projectionIn, selection, selectionArgs, null, null, sort)
    if (ret == null) {
        LogUtils.e("Alarms.query: failed")
    } else {
        ret.setNotificationUri(context!!.contentResolver, uri)
    }
    return ret
}
```

```kotlin
private const val ALARM_JOIN_INSTANCE_TABLE_STATEMENT =
        ALARMS_TABLE_NAME + " LEFT JOIN " +
                INSTANCES_TABLE_NAME + " ON (" +
                ALARMS_TABLE_NAME + "." +
                BaseColumns._ID + " = " + InstancesColumns.ALARM_ID + ")"
//等价 select * from alarm_templates left join alarm_instances on( alarm_templates._id = alarm_instances.alarm_id)
```

![image-20210119191935953](https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/image-20210119191935953.png)

### onLoadFinished()

```java
override fun onLoadFinished(cursorLoader: Loader<Cursor>, data: Cursor) {
    val itemHolders: MutableList<AlarmItemHolder> = ArrayList(data.count)
    data.moveToFirst()
    while (!data.isAfterLast) {
        val alarm = Alarm(data)//Cursor转换为Alarm对象
        val alarmInstance = if (alarm.canPreemptivelyDismiss()) {
            AlarmInstance(data, joinedTable = true)//创建AlarmInstance
        } else {
            null
        }
        val itemHolder = AlarmItemHolder(alarm, alarmInstance, mAlarmTimeClickHandler)
        itemHolders.add(itemHolder)
        data.moveToNext()
    }
    setAdapterItems(itemHolders, SystemClock.elapsedRealtime()) //设置adapter
}
```

## 插入

```java
override fun onTimeSet(fragment: TimePickerDialogFragment?, hourOfDay: Int, minute: Int) {
    mAlarmTimeClickHandler.onTimeSet(hourOfDay, minute) //设置时间
}
```

```java
fun onTimeSet(hourOfDay: Int, minute: Int) {
    if (mSelectedAlarm == null) { //如果闹钟对象为空，说明是新建不是修改
        // If mSelectedAlarm is null then we're creating a new alarm.
        val a = Alarm()
        a.hour = hourOfDay
        a.minutes = minute
        a.enabled = true //默认闹钟打开
        mAlarmUpdateHandler.asyncAddAlarm(a)//异步添加闹钟
    } else {
        mSelectedAlarm!!.hour = hourOfDay
        mSelectedAlarm!!.minutes = minute
        mSelectedAlarm!!.enabled = true
        mScrollHandler.setSmoothScrollStableId(mSelectedAlarm!!.id)
        mAlarmUpdateHandler
                .asyncUpdateAlarm(mSelectedAlarm!!, popToast = true, minorUpdate = false)
        mSelectedAlarm = null
    }
}
```

```java
fun asyncAddAlarm(alarm: Alarm?) {
    val updateTask: AsyncTask<Void, Void, AlarmInstance> =
            object : AsyncTask<Void, Void, AlarmInstance>() {
        override fun doInBackground(vararg parameters: Void): AlarmInstance? {
            if (alarm != null) {
                Events.sendAlarmEvent(R.string.action_create, R.string.label_deskclock)
                val cr: ContentResolver = mAppContext.contentResolver
                // Add alarm to db
                val newAlarm = Alarm.addAlarm(cr, alarm) //添加闹钟
                // Be ready to scroll to this alarm on UI later.
                mScrollHandler?.setSmoothScrollStableId(newAlarm.id)
                // Create and add instance to db
                if (newAlarm.enabled) { //如果enable = true 设置闹钟实例
                    return setupAlarmInstance(newAlarm)
                }
            }
            return null
        }
        override fun onPostExecute(instance: AlarmInstance?) {
            if (instance != null) {
                AlarmUtils.popAlarmSetSnackbar(mSnackbarAnchor!!,
                        instance.alarmTime.timeInMillis)
            }
        }
    }
    updateTask.execute() //执行任务
}
```

```java
private fun setupAlarmInstance(alarm: Alarm): AlarmInstance {
    val cr: ContentResolver = mAppContext.contentResolver
    var newInstance = alarm.createInstanceAfter(Calendar.getInstance()) //创建AlarmInstance
    newInstance = AlarmInstance.addInstance(cr, newInstance) //添加实例
    // Register instance to state manager
    AlarmStateManager.registerInstance(mAppContext, newInstance, true)
    return newInstance
}
```

## AlarmStateManager

### registerInstance()

```kotlin
 @JvmStatic
 fun registerInstance(
     context: Context,
     instance: AlarmInstance,
     updateNextAlarm: Boolean
 ) {
     LogUtils.i("Registering instance: " + instance.mId)
     val cr: ContentResolver = context.contentResolver
     val alarm = Alarm.getAlarm(cr, instance.mAlarmId!!)
     val currentTime = currentTime
     val alarmTime: Calendar = instance.alarmTime
     val timeoutTime: Calendar? = instance.timeout
     val lowNotificationTime: Calendar = instance.lowNotificationTime
     val highNotificationTime: Calendar = instance.highNotificationTime
     val missedTTL: Calendar = instance.missedTimeToLive
     // Handle special use cases here
     if (instance.mAlarmState == InstancesColumns.DISMISSED_STATE) {
         // This should never happen, but add a quick check here
         LogUtils.e("Alarm Instance is dismissed, but never deleted")
         deleteInstanceAndUpdateParent(context, instance)
         return
     } else if (instance.mAlarmState == InstancesColumns.FIRED_STATE) {
         // Keep alarm firing, unless it should be timed out
         val hasTimeout = timeoutTime != null && currentTime.after(timeoutTime)
         if (!hasTimeout) {
             setFiredState(context, instance)
             return
         }
     } else if (instance.mAlarmState == InstancesColumns.MISSED_STATE) {
         if (currentTime.before(alarmTime)) {
             if (instance.mAlarmId == null) {
                 LogUtils.i("Cannot restore missed instance for one-time alarm")
                 // This instance parent got deleted (ie. deleteAfterUse), so
                 // we should not re-activate it.-
                 deleteInstanceAndUpdateParent(context, instance)
                 return
             }
             // TODO: This will re-activate missed snoozed alarms, but will
             // use our normal notifications. This is not ideal, but very rare use-case.
             // We should look into fixing this in the future.
             // Make sure we re-enable the parent alarm of the instance
             // because it will get activated by by the below code
             alarm!!.enabled = true
             Alarm.updateAlarm(cr, alarm)
         }
     } else if (instance.mAlarmState == InstancesColumns.PREDISMISSED_STATE) {
         if (currentTime.before(alarmTime)) {
             setPreDismissState(context, instance)
         } else {
             deleteInstanceAndUpdateParent(context, instance)
         }
         return
     }
     // Fix states that are time sensitive
     if (currentTime.after(missedTTL)) {
         // Alarm is so old, just dismiss it
         deleteInstanceAndUpdateParent(context, instance)
     } else if (currentTime.after(alarmTime)) {
         // There is a chance that the TIME_SET occurred right when the alarm should go off,
         // so we need to add a check to see if we should fire the alarm instead of marking
         // it missed.
         val alarmBuffer = Calendar.getInstance()
         alarmBuffer.time = alarmTime.time
         alarmBuffer.add(Calendar.SECOND, ALARM_FIRE_BUFFER)
         if (currentTime.before(alarmBuffer)) {
             setFiredState(context, instance)
         } else {
             setMissedState(context, instance)
         }
     } else if (instance.mAlarmState == InstancesColumns.SNOOZE_STATE) {
         // We only want to display snooze notification and not update the time,
         // so handle showing the notification directly
         AlarmNotifications.showSnoozeNotification(context, instance)
         scheduleInstanceStateChange(context, instance.alarmTime,
                 instance, InstancesColumns.FIRED_STATE)
     } else if (currentTime.after(highNotificationTime)) {
         setHighNotificationState(context, instance)
     } else if (currentTime.after(lowNotificationTime)) {
         // Only show low notification if it wasn't hidden in the past
         if (instance.mAlarmState == InstancesColumns.HIDE_NOTIFICATION_STATE) {
             setHideNotificationState(context, instance)
         } else {
             setLowNotificationState(context, instance)
         }
     } else {
         // Alarm is still active, so initialize as a silent alarm
         setSilentState(context, instance)
     }
     // The caller prefers to handle updateNextAlarm for optimization
     if (updateNextAlarm) {
         updateNextAlarm(context)
     }
 }
```

```kotlin
private fun setSilentState(context: Context, instance: AlarmInstance) {
    LogUtils.i("Setting silent state to instance " + instance.mId)
    // Update alarm in db
    val contentResolver: ContentResolver = context.contentResolver
    instance.mAlarmState = InstancesColumns.SILENT_STATE
    AlarmInstance.updateInstance(contentResolver, instance)
    // Setup instance notification and scheduling timers
    AlarmNotifications.clearNotification(context, instance) //清除通知
    //执行实例状态改变
    scheduleInstanceStateChange(context, instance.lowNotificationTime,
            instance, InstancesColumns.LOW_NOTIFICATION_STATE)
}
```

```java
private var sStateChangeScheduler: StateChangeScheduler = AlarmManagerStateChangeScheduler()
private fun scheduleInstanceStateChange(
    ctx: Context,
    time: Calendar,
    instance: AlarmInstance,
    newState: Int
) {
    sStateChangeScheduler.scheduleInstanceStateChange(ctx, time, instance, newState)
}
```

```kotlin
private class AlarmManagerStateChangeScheduler : StateChangeScheduler {
        override fun scheduleInstanceStateChange(
            context: Context,
            time: Calendar,
            instance: AlarmInstance,
            newState: Int
        ) {
            val timeInMillis = time.timeInMillis
            LogUtils.i("Scheduling state change %d to instance %d at %s (%d)", newState,
                    instance.mId, AlarmUtils.getFormattedTime(context, time), timeInMillis)
            val stateChangeIntent: Intent =
                    createStateChangeIntent(context, ALARM_MANAGER_TAG, instance, newState)
            // Treat alarm state change as high priority, use foreground broadcasts
            stateChangeIntent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND)
            //创建pendingIntent
            val pendingIntent: PendingIntent =
                    PendingIntent.getService(context, instance.hashCode(),
                    stateChangeIntent, PendingIntent.FLAG_UPDATE_CURRENT)
						//创建AlarmManager
            val am: AlarmManager = context.getSystemService(ALARM_SERVICE) as AlarmManager
            if (Utils.isMOrLater) { //大于等于6.0
                // Ensure the alarm fires even if the device is dozing.
                am.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, timeInMillis, pendingIntent)
            } else {
                am.setExact(AlarmManager.RTC_WAKEUP, timeInMillis, pendingIntent)
            }
        }

        override fun cancelScheduledInstanceStateChange(context: Context, instance: AlarmInstance) {
            LogUtils.v("Canceling instance " + instance.mId + " timers")

            // Create a PendingIntent that will match any one set for this instance
            val pendingIntent: PendingIntent? =
                    PendingIntent.getService(context, instance.hashCode(),
                    createStateChangeIntent(context, ALARM_MANAGER_TAG, instance, null),
                    PendingIntent.FLAG_NO_CREATE)

            pendingIntent?.let {
                val am: AlarmManager = context.getSystemService(ALARM_SERVICE) as AlarmManager
                am.cancel(it)
                it.cancel()
            }
        }
    }
```



```java
fun createStateChangeIntent(
    context: Context?,
    tag: String?,
    instance: AlarmInstance,
    state: Int?
): Intent {
    // This intent is directed to AlarmService, though the actual handling of it occurs here
    // in AlarmStateManager. The reason is that evidence exists showing the jump between the
    // broadcast receiver (AlarmStateManager) and service (AlarmService) can be thwarted by
    // the Out Of Memory killer. If clock is killed during that jump, firing an alarm can
    // fail to occur. To be safer, the call begins in AlarmService, which has the power to
    // display the firing alarm if needed, so no jump is needed.
    //闹钟完成，启动Service
    val intent: Intent =
            AlarmInstance.createIntent(context, AlarmService::class.java, instance.mId)
    intent.action = CHANGE_STATE_ACTION
    intent.addCategory(tag)
    intent.putExtra(ALARM_GLOBAL_ID_EXTRA, DataModel.dataModel.globalIntentId)
    if (state != null) {
        intent.putExtra(ALARM_STATE_EXTRA, state.toInt())
    }
    return intent
}
```



### unregisterInstance()

```java
fun unregisterInstance(context: Context, instance: AlarmInstance) {
    LogUtils.i("Unregistering instance " + instance.mId)
    // Stop alarm if this instance is firing it
    AlarmService.stopAlarm(context, instance)
    AlarmNotifications.clearNotification(context, instance)
    cancelScheduledInstanceStateChange(context, instance)
    setDismissState(context, instance)
}
```

## AlarmService

### onStartCommand()

```java
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        LogUtils.v("AlarmService.onStartCommand() with %s", intent)
        if (intent == null) {
            return Service.START_NOT_STICKY
        }

        val instanceId = AlarmInstance.getId(intent.data!!)
        when (intent.action) {
            AlarmStateManager.CHANGE_STATE_ACTION -> { //状态改变
                AlarmStateManager.handleIntent(this, intent)

                // If state is changed to firing, actually fire the alarm!
                val alarmState: Int = intent.getIntExtra(AlarmStateManager.ALARM_STATE_EXTRA, -1)
                if (alarmState == InstancesColumns.FIRED_STATE) {//如果当前状态是FIRED_STATE
                    val cr: ContentResolver = this.contentResolver
                    val instance: AlarmInstance? = AlarmInstance.getInstance(cr, instanceId)
                    if (instance == null) {
                        LogUtils.e("No instance found to start alarm: %d", instanceId)
                        if (mCurrentAlarm != null) {
                            // Only release lock if we are not firing alarm
                            AlarmAlertWakeLock.releaseCpuLock()
                        }
                    } else if (mCurrentAlarm != null && mCurrentAlarm!!.mId == instanceId) {
                        LogUtils.e("Alarm already started for instance: %d", instanceId)
                    } else {
                        startAlarm(instance) //开始闹钟
                    }
                }
            }
            STOP_ALARM_ACTION -> {//停止闹钟
                if (mCurrentAlarm != null && mCurrentAlarm!!.mId != instanceId) {
                    LogUtils.e("Can't stop alarm for instance: %d because current alarm is: %d",
                            instanceId, mCurrentAlarm!!.mId)
                } else {
                    stopCurrentAlarm()
                    stopSelf()
                }
            }
        }

        return Service.START_NOT_STICKY
    }
```

### startAlarm()

```kotlin
private fun startAlarm(instance: AlarmInstance) {
    LogUtils.v("AlarmService.start with instance: " + instance.mId)
    if (mCurrentAlarm != null) {
        AlarmStateManager.setMissedState(this, mCurrentAlarm!!)
        stopCurrentAlarm()
    }
    AlarmAlertWakeLock.acquireCpuWakeLock(this)
    mCurrentAlarm = instance
      //显示Notification
      AlarmNotifications.showAlarmNotification(this, mCurrentAlarm!!)
    mTelephonyManager.listen(mPhoneStateListener.init(), PhoneStateListener.LISTEN_CALL_STATE)
    AlarmKlaxon.start(this, mCurrentAlarm!!)
    sendBroadcast(Intent(ALARM_ALERT_ACTION))
}
```

## 参考

* [设置重复闹铃时间](https://developer.android.com/training/scheduling/alarms#kotlin)

















