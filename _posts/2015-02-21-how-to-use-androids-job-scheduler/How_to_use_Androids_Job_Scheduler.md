---
layout: post
title: "How to use Android's Job Scheduler"
author: Chris Pierick
categories: [development]
tags: [job scheduler, support library]
---

One of the really cool things about Lollipop is the new [`JobScheduler`][1] API. Not only is this API exciting for developers but end users should also be excited.
 This surprisingly easy to use API lets your app schedule a job to take place according to a number of parameters.
 The cool thing about this is that you can easily set parameters that will save the end user lots of battery life, giving your users one less reason to uninstall your app.
 There are three main parts to this API: [`JobInfo`][3], [`JobService`][2] and [`JobScheduler`][1].
 <!--more-->

## JobInfo and Available Parameters

Parameters are defined by the [`JobInfo`][3] class and constructed using the [`JobInfo.Builder`][4].
 The following are the different parameters that you can set.

- Back-off Criteria is the policy that kicks in when a job is finished and a retry is requested. You can set the initial back-off time and if it is linear or exponential. The default is 30 sec and exponential.[`setBackoffCriteria(long initialBackoffMillis, int backoffPolicy)`][6]
- A Bundle of extras. This lets you send specific data to your job. [`setExtras(PersistableBundle extras)`][7]
- A minimum amount of time your job should be delayed. [`setMinimumLatency(long minLatencyMillis)`][8]
- A maximum amount of time to wait to execute your job. If you hit this time your job will be executed immediately regardless of your other parameters. [`setOverrideDeadline(long maxExecutionDelayMillis)`][9]
- If you want this job repeated, you can specify the interval between repeats. You are guaranteed to be executed within an interval but not at what point during that interval. This can sometimes lead to jobs being run closely together.  [`setPeriodic(long intervalMillis)`][10]
- You can persist the job across boot. This requires the [RECEIVE\_BOOT\_COMPLETED][5] permission added to your manifest. [`setPersisted(boolean isPersisted)`][11]
- The network type you want the device to have when your job is executed. You can choose between none, any and unmetered (Wifi). [`setRequiredNetworkType(int networkType)`][12]
- Whether or not the device should be charging. [`setRequiresCharging(boolean requiresCharging)`][13]
- If the device should be idle when running the job. This is a great time to do resource heavy jobs. [`setRequiresDeviceIdle(boolean requiresDeviceIdle)`][14]

If we were the PlayStore we would probably want to update apps when: 
- on wifi to save the user data. 
- the device is idle so that we don't slow down the user's device experience. No more updating when the screen turns on!
- the device is charging, allowing us to save the user battery life during the day.

```java
ComponentName serviceName = new ComponentName(context, MyJobService.class);
JobInfo jobInfo = new JobInfo.Builder(JOB_ID, serviceName)
        .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED)
        .setRequiresDeviceIdle(true)
        .setRequiresCharging(true)
        .build();
```

What's MyJobService.class? Don't freak out, we're getting to that now.

## JobService

The [`JobService`][2] is the actual service that is going to run your job.
 This service has different methods to implement than normal services.
 The first method is [`onStartJob(JobParameters params)`][15].
 This method is what gets called when the [`JobScheduler`][1] decides to run your job based on its parameters.
 You can get the jobId from the [`JobParameters`][18] and you will have to hold on to these parameters to finish the job later.
 The next method is [`onStopJob(JobParameters params)`][16]. This will get called when your parameters are no longer being met.
 In our previous example this would happen when the user switches off of wifi, unplugs or turns the screen on their device.
 This means that we want to stop our job as soon as we can; in this case we wouldn't update any more apps.

There are **three very important things** to know when using a [`JobService`][2].
- The [`JobService`][2] runs on the main thread. It is your responsibility to move your work off thread. If the user tries to open your app while a job is running on the main thread they might get an Android Not Responding error (ANR).
- You must finish your job when it is complete. The [`JobScheduler`][1] keeps a wake lock for your job. If you don't call [`jobFinished(JobParameters params, boolean needsReschedule)`][17] with the [`JobParameters`][18] from [`onStartJob(JobParameters params)`][15] the [`JobScheduler`][1] will keep a wake lock for your app and burn the devices battery. Even worse is that the battery history will blame your app. Here at Two Toasters we call that type of bug a `1 star, uninstall`.
- You have to register your job service in the AndroidManifest. If you do not, the system will not be able to find your service as a component and it will not start your jobs. You'll never even know as this does not produce an error.

```xml
<application
    .... stuff ....
    >

    <service
        android:name=".MyJobService"
        android:permission="android.permission.BIND_JOB_SERVICE"
        android:exported="true"/>
</application>
```

```java
public class MyJobService extends JobService {

    private UpdateAppsAsyncTask updateTask = new UpdateAppsAsyncTask();

    @Override
    public boolean onStartJob(JobParameters params) {
        // Note: this is preformed on the main thread.

        updateTask.execute(params);

        return true;
    }

    // Stopping jobs if our job requires change.

    @Override
    public boolean onStopJob(JobParameters params) {
        // Note: return true to reschedule this job.

        boolean shouldReschedule = updateTask.stopJob(params);

        return shouldReschedule;
    }

    private class UpdateAppsAsyncTask extends AsyncTask<JobParameters, Void, JobParameters[]> {


        @Override
        protected JobParameters[] doInBackground(JobParameters... params) {

          // Do updating and stopping logical here.
          return params;
        }

        @Override
        protected void onPostExecute(JobParameters[] result) {
            for (JobParameters params : result) {
                if (!hasJobBeenStopped(params)) {
                    jobFinished(params, false);
                }
            }
        }

        private boolean hasJobBeenStopped(JobParameters params) {
            // Logic for checking stop.
        }

        public boolean stopJob(JobParameters params) {
            // Logic for stopping a job. return true if job should be rescheduled.
        }

    }
}
```

Of course this is just stubbed out and you can handle your off thread logic however you so wish.
 Just remember that however you handle threading you have to call [`jobFinished(JobParameters params, boolean needsReschedule)`][17] when you're done.

## JobScheduler, wait... its how easy?

This part is really darn easy.
 We now have our [`JobInfo`][3] and our [`JobService`][2] so it is time to schedule our job!
 All we have to do is get the [`JobService`][2] the same way you would get any system service and hand it our [`JobInfo`][3] with the [`schedule(JobInfo job)`][19] method. *Noob tip: your activity is a context object.*

```java
JobScheduler scheduler = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);
int result = scheduler.schedule(jobInfo);
if (result == JobScheduler.RESULT_SUCCESS) Log.d(TAG, "Job scheduled successfully!");
```

## Conclusion

Not too bad eh?
 Way easier than setting up a SyncAdapter plus it's more powerful and flexible than the AlarmManager!
 If you want to check out the code in action, head to our [Github repo][20] and build it yourself!
 Thanks for the read and as always, keep on toasting.


[1]: https://developer.android.com/reference/android/app/job/JobScheduler.html
[2]: https://developer.android.com/reference/android/app/job/JobService.html
[3]: https://developer.android.com/reference/android/app/job/JobInfo.html
[4]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html
[5]: https://developer.android.com/reference/android/Manifest.permission.html#RECEIVE_BOOT_COMPLETED
[6]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setBackoffCriteria(long,%20int)
[7]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setExtras(android.os.PersistableBundle)
[8]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setMinimumLatency(long)
[9]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setOverrideDeadline(long)
[10]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setPeriodic(long)
[11]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setPersisted(boolean)
[12]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setRequiredNetworkType(int)
[13]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setRequiresCharging(boolean)
[14]: https://developer.android.com/reference/android/app/job/JobInfo.Builder.html#setRequiresDeviceIdle(boolean)
[15]: https://developer.android.com/reference/android/app/job/JobService.html#onStartJob(android.app.job.JobParameters)
[16]: https://developer.android.com/reference/android/app/job/JobService.html#onStopJob(android.app.job.JobParameters)
[17]: https://developer.android.com/reference/android/app/job/JobService.html#jobFinished(android.app.job.JobParameters,%20boolean)
[18]: https://developer.android.com/reference/android/app/job/JobParameters.html
[19]: https://developer.android.com/reference/android/app/job/JobScheduler.html#schedule(android.app.job.JobInfo)
[20]: https://github.com/twotoasters/toastdroid/tree/master/chris/how_to_use_jobscheduler
