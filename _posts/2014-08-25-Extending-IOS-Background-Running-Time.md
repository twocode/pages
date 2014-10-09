---
title: Utilizing BackgroundTask list to extend Application Background Running Time for Scheduled Tasks
date: 2014-08-25 16:00
layout: post
category: projects
comments: true
tags: ios app backgroundtask nstimer
description: Despite strict limitations on IOS background tasks by IOS kernel (3 minutes allowed for extra running, if no music, location services running), there is a way to extend its running time with help of BackgroundTask lists.

---


IOS has improved multiprocesses ever since iOS4.*. However, background tasks are still strictly restricted by the kernel for battery saving purposes. Unless the app requires music playing, location tracking, or background fetching, iOS does not allow the app running in the background after the 3 minutes it offers and kills it.

In my senario, I would want the app to run in the background with a timer-triggered location tracking, in contrast to the default always-running location tracking, to save some battery, while using algorithms to rebuild the location histories based on the sampled GPS values.

As an introdution 101, iOS provides `CLLocationManager` class for location tracking services and `CLLocationManagerDelegate` protocal for data-retrieving callbacks. Namely, implementing `-locationManager:didUpdateLocations:` protocal function will enable the delegate class to get the location values. This function is called everytime location gets updated, so it is very sensitive and gets called frequently. 

Another feature iOS provides for background related feasibilities is `UIBackgroundTaskIdentifier`. It is like a pid in linux representing processes, functioning as the descriptor to a background task. Actually, the identifier is defined as:

    typedef unsigned int NSUInteger;
    typedef NSUInteger UIBackgroundTaskIdentifier;

Normally, a background task is added in this way:

    UIBackgroundTaskIdentifier bgTask = [[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler: ^ {
                    [application endBackgroundTask: bgTask];
                    bgTask = UIBackgroundTaskInvalid;
                }];

After one background task has begun, the app will be entitled extra several (3) minutes to run in the background. It should be noted that **1. If the app occupys the resources and battery for background tasks which runs for nothing, the app will not be permitted to be distributed on the App Store; 2. If `bgTask` is never called by `endBackgroundTask`, the app will be terminated by the kernel.** 

Last and most, we need `NSTimer` to bridge the features above to enable the timer-triggered location tracking function in the background.

The procedures of the whole model can be described as follows.

1. Application entering background, hook it, add one background task and appends it to the background list. While adding the task, other existing background task identifiers should be ended to ensure that there always exists only one background task activated in the list. This rule needs to be maintained whenever a new background task is added to the list.

2. When `didUpdateLocations` callback gets called, call `CLLocationManger`'s `stopUpdatingLocation` to stop GPS tracking and activate the sampling timer to set up the next time to restart location tracking with `startUpdatingLocation`. Then the app will enter a state where location tracking is disabled and timer set. Then timer up and it will re-enable location tracking and get into this function again.

3. Other extensions can be achieved with other timers. For example, add another timer to set the running time for each location tracker's running time makes it possible to get a more previous GPS value; add another timer to process the location data, say, according to which calculate them with algorthms and such.

The reason why it is relatively easy to combine location tracking and background tasks is because`didUpdateLocation` gets called automatically with `CLLocationManagerDelegate`. If such method is intended to perform other background tasks other than a location tracking scenario, a timer-triggered function should be implemented where the same logic in `didUpdateLocation` is ported.

<br />

