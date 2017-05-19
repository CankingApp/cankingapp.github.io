---
title: 知识总结之 JobQueue 源码解析
date: 2017-05-02 11:33:14
categories: android 
tags: JobQueue
---

本文主要分析及调研开源项目android-priority-jobqueue的基本原理及知识点，目的为更加深入的了解安卓平台后台
任务处理，及多线程调度的理解。

![Just do it](http://wx4.sinaimg.cn/mw690/70e618a5ly1fdzhtte5alj20zk0qoqa6.jpg)

<!--more-->

# Android Priority Job Queue

## 一、JobQueue是什么？

> Priority Job Queue is an implementation of a Job Queue specifically written for Android
 to easily schedule jobs (tasks) that run in the background, improving UX and application stability.

> 出自github：[android-priority-jobqueue](https://github.com/path/android-priority-jobqueue)

可看出JobQueue是一个处理后台任务的控件，比AsyncTask的多任务调度能力要强，默认的缓存队列由数据库存储，
具有一定稳定性。

*JobQueue = 线程池+线程调度优化+定时任务*

## 二、基本使用方法

考虑到资源共享，JobQueue推荐以单例形式存在，在起第一次调用或者程序入口添加JobQueue的初始化。

1, 代码地址

```
Gradle: compile 'com.path:android-priority-jobqueue:1.1.2'

Maven:

<dependency>
    <groupId>com.path</groupId>
    <artifactId>android-priority-jobqueue</artifactId>
    <version>1.1.2</version>
</dependency>

```

2，初始化代码

```
  Configuration.Builder builder = new Configuration.Builder(this)
                .customLogger(new CustomLogger() {
                    private static final String TAG = "zhangphil job";

                    @Override
                    public boolean isDebugEnabled() {
                        return true;
                    }

                    @Override
                    public void d(String text, Object... args) {
                        Log.d(TAG, String.format(text, args));
                    }

                    @Override
                    public void e(Throwable t, String text, Object... args) {
                        Log.e(TAG, String.format(text, args), t);
                    }

                    @Override
                    public void e(String text, Object... args) {
                        Log.e(TAG, String.format(text, args));
                    }

                    @Override
                    public void v(String text, Object... args) {

                    }
                })
                .minConsumerCount(1)//always keep at least one consumer alive
                .maxConsumerCount(3)//up to 3 consumers at a time
                .loadFactor(3)//3 jobs per consumer
                .consumerKeepAlive(120);//wait 2 minute

        jobManager = new JobManager(builder.build());
```

3，定义一个需要执行的任务

```
// A job to send a tweet
public class PostTweetJob extends Job {
    public static final int PRIORITY = 1;
    private String text;
    public PostTweetJob(String text) {
        // This job requires network connectivity,
        // and should be persisted in case the application exits before job is completed.
        super(new Params(PRIORITY).requireNetwork().persist());
    }
    @Override
    public void onAdded() {
        // Job has been saved to disk.
        // This is a good place to dispatch a UI event to indicate the job will eventually run.
        // In this example, it would be good to update the UI with the newly posted tweet.
    }
    @Override
    public void onRun() throws Throwable {
        // Job logic goes here. In this example, the network call to post to Twitter is done here.
        webservice.postTweet(text);
    }
    @Override
    protected boolean shouldReRunOnThrowable(Throwable throwable) {
        // An error occurred in onRun.
        // Return value determines whether this job should retry running (true) or abort (false).
    }
    @Override
    protected void onCancel() {
        // Job has exceeded retry attempts or shouldReRunOnThrowable() has returned false.
    }
}
```

4，添加到执行器，合适的时机，onRun方法会在后台调用。

```
	jobManager.addJobInBackground(new PostTweetJob(status));
```

## 三、主要类结构

![UML](jobuml.png)





