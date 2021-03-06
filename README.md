# Android-Job

An utility library for Android to run jobs delayed in the background. Depending on the Android version either the `JobScheduler`, `GcmNetworkManager` or `AlarmManager` is getting used. You can find out in [this blog post](https://blog.evernote.com/tech/2015/10/26/unified-job-library-android/) or in [these slides](https://speakerdeck.com/vrallev/scheduling-background-job-on-android-at-the-right-time-1) why you should prefer this library than each separate API. All features from Android Nougat are backward compatible.

## Download

Download [the latest version](http://search.maven.org/#search|gav|1|g:"com.evernote"%20AND%20a:"android-job") or grab via Gradle:

```groovy
dependencies {
    compile 'com.evernote:android-job:1.1.9'
}
```

If you didn't turn off the manifest merger from the Gradle build tools, then no further step is required to setup the library. Otherwise you manually need to add the permissions and services like in this [AndroidManifest](library/src/main/AndroidManifest.xml).

You can read the [JavaDoc here](https://evernote.github.io/android-job/javadoc/).

## Usage

The class `JobManager` serves as entry point. Your jobs need to extend the class `Job`. Create a `JobRequest` with the corresponding builder class and schedule this request with the `JobManager`.

Before you can use the `JobManager` you must initialize the singleton. You need to provide a `Context` and add a `JobCreator` implementation after that. The `JobCreator` maps a job tag to a specific job class. It's recommend to initialize the `JobManager` in the `onCreate()` method of your `Application` object, but there is [an alternative](FAQ.md#i-cannot-override-the-application-class-how-can-i-add-my-jobcreator), if you don't have access to the `Application` class.

```java
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        JobManager.create(this).addJobCreator(new DemoJobCreator());
    }
}
```

```java
public class DemoJobCreator implements JobCreator {

    @Override
    public Job create(String tag) {
        switch (tag) {
            case DemoSyncJob.TAG:
                return new DemoSyncJob();
            default:
                return null;
        }
    }
}
```

After that you can start scheduling jobs.

```java
public class DemoSyncJob extends Job {

    public static final String TAG = "job_demo_tag";

    @Override
    @NonNull
    protected Result onRunJob(Params params) {
        // run your job here
        return Result.SUCCESS;
    }

    public static void scheduleJob() {
        new JobRequest.Builder(DemoSyncJob.TAG)
                .setExecutionWindow(30_000L, 40_000L)
                .build()
                .schedule();
    }
}
```

## Advanced

The `JobRequest.Builder` class has many extra options, e.g. you can specify a required network connection, make the job periodic, pass some extras with a bundle, restore the job after a reboot or run the job at an exact time.

Each job has a unique ID. This ID helps to identify the job later to update requirements or to cancel the job.

```java
private void scheduleAdvancedJob() {
    PersistableBundleCompat extras = new PersistableBundleCompat();
    extras.putString("key", "Hello world");

    int jobId = new JobRequest.Builder(DemoSyncJob.TAG)
            .setExecutionWindow(30_000L, 40_000L)
            .setBackoffCriteria(5_000L, JobRequest.BackoffPolicy.EXPONENTIAL)
            .setRequiresCharging(true)
            .setRequiresDeviceIdle(false)
            .setRequiredNetworkType(JobRequest.NetworkType.CONNECTED)
            .setExtras(extras)
            .setRequirementsEnforced(true)
            .setPersisted(true)
            .setUpdateCurrent(true)
            .build()
            .schedule();
}

private void schedulePeriodicJob() {
    int jobId = new JobRequest.Builder(DemoSyncJob.TAG)
            .setPeriodic(TimeUnit.MINUTES.toMillis(15), TimeUnit.MINUTES.toMillis(5))
            .setPersisted(true)
            .build()
            .schedule();
}

private void scheduleExactJob() {
    int jobId = new JobRequest.Builder(DemoSyncJob.class)
            .setExact(20_000L)
            .setPersisted(true)
            .build()
            .schedule();
}

private void cancelJob(int jobId) {
    JobManager.instance().cancel(jobId);
}
```

If a non periodic `Job` fails, then you can reschedule it with the defined back-off criteria.

```java
public class RescheduleDemoJob extends Job {

    @Override
    @NonNull
    protected Result onRunJob(Params params) {
        // something strange happened, try again later
        return Result.RESCHEDULE;
    }

    @Override
    protected void onReschedule(int newJobId) {
        // the rescheduled job has a new ID
    }
}
```

**Warning:** With Android Marshmallow Google introduced the auto backup feature. All job information are stored in a shared preference file called `evernote_jobs.xml` and in a database called `evernote_jobs.db`. You should exclude these files so that they aren't backed up.

You can do this by defining a resource XML file (i.e., `res/xml/backup_config.xml`) with content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<full-backup-content>
    <exclude domain="sharedpref" path="evernote_jobs.xml" />
    <exclude domain="database" path="evernote_jobs.db" />
</full-backup-content>
``` 

And then referring to it in your application tag in `AndroidManifest.xml`:

```xml
<application ...  android:fullBackupContent="@xml/backup_config">
```

#### Proguard

The library doesn't use reflection, but it relies on three `Service`s and two `BroadcastReceiver`s. In order to avoid any issues, you shouldn't obfuscate those four classes. The library bundles its own Proguard config and you don't need to do anything, but just in case you can add [these rules](library/proguard.txt) in your configuration.

## FAQ

See [here](FAQ.md).

## Google Play Services

This library does **not** automatically bundle the Google Play Services, because the dependency is really heavy and not all apps want to include them. That's why you need to add the dependency manually, if you want that the library uses the `GcmNetworkManager` on Android 4.
```groovy
dependencies {
    compile "com.google.android.gms:play-services-gcm:latest_version"
}
```
Crashes after removing the GCM dependency is a known limitation of the Google Play Services. Please take a look at [this workaround](FAQ.md#how-can-i-remove-the-gcm-dependency-from-my-app) to avoid those crashes.

## License
```
Copyright (c) 2007-2017 by Evernote Corporation, All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```