Qt and App Nap
==============

App Nap is a macOS power-saving feature which reduces CPU time given to background
applications. One effect of this is that timers are are no longer timely.

As a demonstration, here's logging output from a foreground Qt application that has
two timers, with intervals of 100 and 200 ms. runLoopTimerCallback is the native
timer callback, the timestamps are seconds-since-epoc:

    524429311.389913 runLoopTimerCallback
    524429311.389979 200ms fire
    524429311.389997 100ms fire
    524429311.490056 runLoopTimerCallback
    524429311.490130 100ms fire
    524429311.589752 runLoopTimerCallback
    524429311.589929 200ms fire
    524429311.589976 100ms fire

Both 100ms and 200ms timers fire as expected. Next lets look at the same setup
with App Nap active:

    524429428.690887 runLoopTimerCallback
    524429428.691002 100ms fire
    524429428.691143 200ms fire
    524429433.692103 runLoopTimerCallback
    524429433.692205 100ms fire
    524429433.692331 200ms fire

Now there is a 5-second delay between native timer firerings. Further, the 100ms
timer fires at the same rate as the 200ms timer. Qt does not make up for missed
timers by firering multiple times.

Feature or Problem?
-------------------

Isolated, reducing battery usage is a win. Also, timer fire delay is allowed by
the Qt documentation. However some applications may rely on timers continuing to
fire at the set rate, even in the background. To remedy this macOS provides the
[NSProcess beginActivityWithOptions] API.

We have suspected that App Nap was causing instability in the Qt CI system and
have now patched QTestLib to use this API to disable App Napp during test runs:

    https://codereview.qt-project.org/#/c/202515/
    
You may want to do the same thing in your applications, here is how:

beginActivityWithOptions and Objective-C
----------------------------------------

First (myapp.pro):

    OBJECTIVE_SOURCES += appnap.mm

Then (appnap.mm):

    // Begin NSActivityBackground to prevent App Napping
    id m_activity = [[NSProcessInfo processInfo]
                      beginActivityWithOptions:NSActivityBackground
                                        reason:@"MyReason"];
    [m_activity retain];

    // ... (use timers here)

    // End NSActivityBackground when done
    [[NSProcessInfo processInfo] endActivity:m_activity];
    [m_activity release];


Minimal Objective-C <-> C++ Rosetta Stone:

    [NSProcessInfo processInfo]                                              NSProcessInfo::processInfo()
    [foo beginActivityWithOptions:NSActivityBackground reason:@"MyReason"]   foo->beginActivity(NSActivityBackground, "MyReason")

If your reason is in a QString instead of a string literal then you may use
string.toNSString() to get a native string.

There are other NSActivityOptions as well, for example for preventing idle system
or display sleep. Refer to the Apple documentation for the complete list.
