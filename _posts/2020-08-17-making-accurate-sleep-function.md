---
layout: post
title: "Making an accurate Sleep() function"
date: 2020-08-17 13:00:00 +0300
categories: timing
---

There are many cases in programming where I wish the system `Sleep()` function was more accurate. Real-time rendering immediately comes to mind, as do animated GUI's. In both of these cases you want some piece of code to run on a fixed timer, executing _precisely_ every 16.67 milliseconds for 60fps.

I haven't really found any usefull posts or tutorials about how to do precise timing anywhere else, so I decided to make a short post about my own precise timing solution.

**Note** this is only usefull if you can't use [vsync](https://en.wikipedia.org/wiki/Vsync_(computing)) for synchronization, or if you have no way to guarantee that vsync will be turned on on the user's machine.

**Also note** this is by no means a _perfect_ solution, even though it seems to be pretty accurate, it isn't very robust and might act differently on different systems.

## Sleep

Before we move on, it's important that you understand exactly what the problem with `Sleep()` is.

I should also clarify that by "sleep", I mean the [`Sleep()`](https://en.wikipedia.org/wiki/Sleep_(system_call)) system call. It's a function that makes the calling thread do nothing for a given period of time. In C++ we can use [`this_thread::sleep_for()`](https://en.cppreference.com/w/cpp/thread/sleep_for) from [`<thread>`](https://en.cppreference.com/w/cpp/header/thread) as a cross-platform way to sleep.

Let's pretend like we're making a simple game loop that needs to run _precisely_ 60 times per second, so approximately once every 16.67 milliseconds. Let's see what happens if we try to use `Sleep()` for our timing:

```cpp
#include <thread>
#include <chrono>
#include <stdio.h>
using namespace std;
using namespace std::chrono;

int main() {
    auto prevClock = high_resolution_clock::now();
    while (true) {
        auto nextClock = high_resolution_clock::now();
        double deltaTime = (nextClock - prevClock).count() / 1e9;
        printf(" frame time: %.2lf ms\n", deltaTime * 1e3);
        // updateGame();
        
        // make sure each frame takes *at least* 1/60th of a second
        auto frameClock = high_resolution_clock::now();
        double sleepSecs = 1.0 / 60 - (frameClock - nextClock).count() / 1e9;
        if (sleepSecs > 0)
            this_thread::sleep_for(nanoseconds((int64_t)(sleepSecs * 1e9)));

        prevClock = nextClock;
    }
    return 0;
}
```

So far so good. This seems like it will do exactly what we want. But look at what happens when we run it:

```
 ...
 frame time: 17.99 ms
 frame time: 17.57 ms
 frame time: 17.53 ms
 frame time: 18.04 ms
 frame time: 17.53 ms
 frame time: 17.27 ms
 frame time: 17.76 ms
 frame time: 18.30 ms
 frame time: 17.35 ms
 frame time: 17.08 ms
 ...
```

Yikes. That doesn't look precise at all. Remember that we were looking for steady 16.67ms, and not only are all frames taking longer than that, but the timings are completely inconsistent. 

So why is the system sleep so imprecise? Well the answer is complicated. When you call Sleep, your application releases it's assigned CPU core to the OS's scheduler, so that it's free to do other tasks while your app waits. Ideally, the schedule will reschedule your application exactly at the time you requested, but things obviously don't happen this way for 2 main reasons:

1. The scheduler wakes up approximately every 3ms. So if you tried to sleep for 1ms, you would instead sleep for _at least_ 3ms. The exact frequency is system dependant, but it's usually 1-10ms these days. 
2. when the time to reschedule comes, all threads might be in use by some higher-priority system services or apps, meaning you don't get your thread back until _they_ decide to sleep, or terminate.

So, calling sleep only guarantees that your thread will wait for _at least_ the given amount of time. Usually, this means 1-2ms more than you asked for. Also, this extra wait time is _random_ for all intents and purposes. It depends on your exact system, and what other programs are running.

One very good thing about sleep in general is that it conserves the CPU. While your program is sleeping, other programs can utilize "your" CPU time to do usefull work, and if nothing needs to run on the CPU, it can be put into low-power mode to save power and battery life.

## Spin-lock

If you want to make your program wait for a precise amount of time without risking your thread to the OS's scheduler, then spin-locking is pretty much the only thing you can do.

Spin-locking generally look like something like this: `while needToWaitMore(): doNothing()`. As you can see it's pretty much the most precise method of waiting, as the CPU does absolutely nothing until the exact moment that it needs to stop waiting. We can incorporate this into our pretend-game loop like so:

```cpp
while (true) {
    auto nextClock = high_resolution_clock::now();
    double deltaTime = (nextClock - prevClock).count() / 1e9;
    printf(" frame time: %.2lf ms\n", deltaTime * 1e3);
    // updateGame();
    
    // make sure each frame takes *at least* 1/60th of a second
    auto frameClock = high_resolution_clock::now();
    double sleepSecs = 1.0 / 60 - (frameClock - nextClock).count() / 1e9;
    auto spinStart = high_resolution_clock::now();
    while ((high_resolution_clock::now() - spinStart).count() / 1e9 < seconds);

    prevClock = nextClock;
}
```

As you can see, it's very easy to set up. And this is what the timing's look like when we run it:

```
 ...
 frame time: 16.67 ms
 frame time: 16.79 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.68 ms
 ...
```

Even this sadly isn't fool proof though. You can clearly see that while _most_ frames run for 16.67ms, one ran for 16.79ms and another ran for 16.68ms. I don't really know exactly why this happens, but my guess is that this is a result of the scheduler poking our thread at a bad time, right before the wait was supposed to end.

Still, this is _the most precision we can possibly get here_.

The problem with this is that spin-locking eats up CPU time and does absolutely nothing. This time could have been better used by other threads that actually have _usefull_ work to do. Or if no other threads are running, the CPU core could have been switched into low-power mode in order to save power and battery life on mobile systems.

## Combining sleep and spin-lock

So far we've tried two options, each with their own pros and cons:
1. Sleep: conserves the CPU but is innacurate.
2. Spin-lock: accurate but thrashes the CPU.

These are pretty much on oposite ends of the spectrum. If we could somehow get the best of both worlds, we would have a perfect sleep function.

Without further ado, here is the `preciseSleep()` function that I came up with that combines these two:

```cpp
#include <chrono>
#include <thread>
#include <math.h>

void preciseSleep(double seconds) {
    using namespace std;
    using namespace std::chrono;

    static double estimate = 5e-3;
    static double mean = 5e-3;
    static double m2 = 0;
    static int64_t count = 1;

    while (seconds > estimate) {
        auto start = high_resolution_clock::now();
        this_thread::sleep_for(milliseconds(1));
        auto end = high_resolution_clock::now();

        double observed = (end - start).count() / 1e9;
        seconds -= observed;

        ++count;
        double delta = observed - mean;
        mean += delta / count;
        m2   += delta * (observed - mean);
        double stddev = sqrt(m2 / (count - 1));
        estimate = mean + stddev;
    }

    // spin lock
    auto start = high_resolution_clock::now();
    while ((high_resolution_clock::now() - start).count() / 1e9 < seconds);
}
```

In short, `preciseSleep()` tries to sleep for 1ms only while it is sure (within reasonable doubt) that the `Sleep(1ms)` call won't take longer than the amount of seconds left to wait for. Then it spin-locks for any remaining time.

This is what we get if we update our pretend-game loop from before to use this function:

```
 ...
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.69 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.67 ms
 frame time: 16.98 ms
 frame time: 16.67 ms
 ...
```

That's looking very similar to the output for the spin-lock wait that we did before. Sometimes there are slight hiccups like that 16.98ms frame, but overall it's pretty much as perfect as we can hope to get. 

And the best part about it is that it uses way less CPU than a pure spin-lock. Mind you it still uses _a lot_ more CPU than a pure Sleep loop. On my laptop it measures somewhere around 5% core usage on average, but that's way better than 100% with a pure spin-lock. We've basically cut down power usage by **95%** without a noticeable accuracy penalty.

## How we got here

If you're curious how I derived the `preciseSleep()` function above, then I'll try to explain it now.

Let's walk through a simple example that shows how we can combine Sleep with spin-locking. Let's say that a perfect `preciseSleep()` function was called to sleep for exactly 3.4ms. Let's also _assume_ that our function somehow knows that calling `Sleep(1ms)` will sleep for _exactly_ 1.1ms. The function would then obviously do something like this pseudo code:

```ruby
def preciseSleep(3.4ms)
    Sleep(1ms)      # 1.1ms elapsed
    Sleep(1ms)      # 2.2ms elapsed
    Sleep(1ms)      # 3.3ms elapsed
    spinLock(0.1ms) # 3.4ms elapsed
end
```

This is perfect if we want to sleep for 3.4ms, but what about 2.4ms, or 10ms? Well we can deal with those in a loop like this:

```ruby
def preciseSleep(time)
    while time > 1.1ms
        Sleep(1ms)     # we still assumes this takes precisely 1.1ms
        time -= 1.1ms
    end
    spinLock(time)
end
```

Great, and now to deal with the elephant in the room.. We don't actually know how long `Sleep(1ms)` will take. But we can measure how long it takes to finish a bunch of times with our clock to get some estimate. And we can also keep updating our estimate while sleeping.

```ruby
estimate = ??

def preciseSleep(time)
    while time > estimate
        start = clock()
        Sleep(1ms)
        observed = clock() - start
        time -= observed
        estimate = updateEstimate(observed)
    end
    spinLock(time)
end
```

This is looking promising, but what's a good estimate here, and how do we find it? Well one thing we know for certain is that `Sleep(1ms)` will take _at least_ 1ms. It might take more, upwards of 10ms even, who knows, but we can safely set our initial estimate to something higher, like 5ms and then decrease it once we actually have data.

The key realization here is that when we call `Sleep(1ms)`, we actually end up sleeping for 1ms + some random amount of noise time:

```ruby
duration of Sleep(1ms) = 1ms + random()
```

So really, we can treat the whole `Sleep(1ms)` call as having a random duration with some mean duration and some random spread. We can calculate the mean and standard deviation from our observed `Sleep(1ms)` durations to make reasonable updates to estimates. We can calculate these on-the-fly as we get new observations using [Welford's algorithm](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Welford%27s_online_algorithm). So our `updateEstimate()` can look something like this:

```ruby
mean  = 5ms
m2    = 0
count = 1

def updateEstimate(observed)
    delta  = observed - mean
    count += 1
    mean  += delta / count
    m2    += delta * (observed - mean)
    stddev = sqrt(m2 / (count - 1))
    return estimate = mean + stddev
end
```

Notice how we set the estimate to be one standard deviation above the mean. This is us being a bit pessimistic about how long `Sleep(1)` will take, because it can randomly take a lot longer than expected. If you want a higher precision at the cost of more wasted CPU cycles, you can change this to:

```ruby
return estimate = mean + X * stddev
```

and then replace `X` with any constant you like. I have it set to 1 because it's simple and seems to work reasonably well.

## Timers on Windows

If you're dead set on writting portable standard C++ code, than the above `preciseSleep()` function is probably as good as you're gonna get. But if you're willing to step down into the pits of Windows hell, you can try using [waitable timers](https://docs.microsoft.com/en-us/windows/win32/sync/waitable-timer-objects) to drive your sleep function.

The upside of this is that it's very CPU friendly. I got 0% reported CPU usage when using this in the pretend-game loop. The accuracy is significantly worse than `preciseSleep()`, but if an error of ~0.5ms is good enough for your needs then here you go:

```cpp
#include <windows.h>
#include <chrono>
#include <math.h>

void timerSleep(double seconds) {
    using namespace std::chrono;

    static HANDLE timer = CreateWaitableTimer(NULL, FALSE, NULL);
    static double estimate = 5e-3;
    static double mean = 5e-3;
    static double m2 = 0;
    static int64_t count = 1;
    
    while (seconds - estimate > 1e-7) {
        double toWait = seconds - estimate;
        LARGE_INTEGER due;
        due.QuadPart = -int64_t(toWait * 1e7);
        auto start = high_resolution_clock::now();
        SetWaitableTimerEx(timer, &due, 0, NULL, NULL, NULL, 0);
        WaitForSingleObject(timer, INFINITE);
        auto end = high_resolution_clock::now();

        double observed = (end - start).count() / 1e9;
        seconds -= observed;

        ++count;
        double error = observed - toWait;
        double delta = error - mean;
        mean += delta / count;
        m2   += delta * (error - mean);
        double stddev = sqrt(m2 / (count - 1));
        estimate = mean + stddev;
    }

    // spin lock
    auto start = high_resolution_clock::now();
    while ((high_resolution_clock::now() - start).count() / 1e9 < seconds);
}
```

## Data: accuracy

How accurate is this precise sleeping function exactly? I'm not the sort of programmer that gets easily swayed without some numbers. So, let's get some.

First, I ran all 4 different sleep functions we've disscussed above for 1000 frames in our 60 FPS game loop test, and I measured the average, minimum, and maximum deviation from 16.67ms frame times when using each sleep function:

```cpp
for (int i = 0; i < 1000; ++i) {
    auto start = high_resolution_clock::now();
    sleep(1 / 60.0);
    auto end = high_resolution_clock::now();
    double duration = (end - start).count() / 1e9;
    printf("%.2lf\n", duration * 1e3);
    // data analysis was done later with a python script
}
```

Here's a graph comparing the errors for all 4 sleep functions:

![Figure 1](../assets/sleep/fig1.svg#center)

> **Figure 1** A graph comparing the absolute error of each sleep function in the 60 FPS game loop test. The bars represent the average error over 1000 runs, while the error lines represent the [minimum, maximum] error.

You can clearly see how closely `preciseSleep()` matches spin-locking with a negligable error. `timerSleep()` does a bit worse with ~40μs of error on average.

Note that in the 60 FPS game loop test we were basically always sleeping for 16.67ms, but these functions are also very precise at higher/lower sleep intervals.

To show this, I ran each function 100 times in a loop with sleep intervals in 1ns, 10ns, 100ns, 1μs, 10μs, ... 1 second.

```cpp
for (int64_t ns = 1; ns <= 1'000'000'000; ns *= 10) {
    // note that I collected other data but this isn't shown for conciseness
    double mean = 0;
    for (int i = 1; i <= 100; ++i) {
        auto start = high_resolution_clock::now();
        sleep(sec);
        auto end = high_resolution_clock::now();
        double elapsed = (end - start).count() / 1e9;
        mean += (elapsed - mean) / i;
    }
    printf("%.1lf\n", mean);
}
```

And here's a graph showing the results.

![Figure 2](../assets/sleep/fig2.svg#center)

> **Figure 2** a log-log plot of the requested sleep time vs the actual measured sleep time when calling the different functions for various time intervals.

Note how both `preciseSleep()` and `timerSleep()` match the requested sleeping time almost perfectly when it is higher than 10µs. Below 100ns not even spin locking can really match the requested time well, probably because calling `high_resolution_clock::now()` takes about 100ns on my system.

I have to admit it's a bit difficult to tell how the different sleep function compare at higher times here because the errors are so miniscule. I can tell you that there _is_ a small difference: spin-lock is obviously the most precise, followed by `preciseSleep()` and then followed by `timerSleep()`. If you want to see for yourself you can always look at the raw data.

## Data: CPU usage

So there's no way cross platform way to get CPU usage in C++ that I know of, so I resorted to using Windows' [`GetProcessTimes()`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getprocesstimes) function.

```cpp
double getProcessTime() {
    static DWORD pid = GetCurrentProcessId();
    static HANDLE process = OpenProcess(PROCESS_QUERY_INFORMATION, FALSE, pid);

    FILETIME creation, exit, kern, user;
    GetProcessTimes(process, &creation, &exit, &kern, &user);
    uint64_t t1 = (uint64_t(kern.dwHighDateTime) << 32) | kern.dwLowDateTime;
    uint64_t t2 = (uint64_t(user.dwHighDateTime) << 32) | user.dwLowDateTime;
    return (t1 + t2) / 1e7;
}
```

This returns the fractional number of seconds that our process was in use so far. If we divide this by the total number of seconds that elapsed since our program started, we can get a % CPU usage.

I then ran all of the different sleep functions for 1000 frames of the 60 FPS game loop and measured the total % CPU usage over the entire 1000 frames.

```cpp
double startProcess = getProcessTime();
auto start = high_resolution_clock::now();

for (int i = 0; i < 1000; ++i)
    sleep(1 / 60.0);

double endProcess = getProcessTime();
double duration = (high_resolution_clock::now() - start).count() / 1e9;
double cpuUsage = 100 * (getProcessTime() - startProcess) / duration;
printf("cpu usage: %.1lf\n", name, cpuUsage);
```

I didn't measure CPU usage for each individial sleep call as I'm not exactly sure how granular `GetProcessTimes()` is.

Anyway, here are the results:

|                   |**CPU usage**|
| ----------------- | ----------- |
| **system sleep**  | 0.0%        |
| **spin-lock**     | 99.8%       |
| **precise sleep** | 4.8%        |
| **timer sleep**   | 0.4%        |

So system sleep and spin-lock get 0 and ~100%, as expected. Precise sleep only uses 4.8%, that's significantly lower than what I expected. And timer sleep also basically uses 0% of the CPU, which is great.

Now, it's also interesting to look at how `preciseSleep()` and `timerSleep()` behave when they try to sleep for smaller time intervals, so I also wrote a little test for that. 

I checked to CPU usage of both of them when they were waiting for 1ms .. 25ms in a loop for 10 seconds using something like this:

```cpp
for (int64_t ms = 1; ms <= 10; ++ms) {
    double processStart = getProcessTime();
    auto start = high_resolution_clock::now();
    while ((high_resolution_clock::now() - start).count() / 1e9 < 5)
        sleep(ms / 1e3);
    auto end = high_resolution_clock::now();
    double cpuTime = getProcessTime() - processStart;
    double allTime = (end - start).count() / 1e9;
    printf("CPU usage: %.1lf\n", ms, 100 * cpuTime / allTime);
}
```

Here are the results:

![Figure 3](../assets/sleep/fig3.svg#center)

> **Figure 3** The CPU usage of `preciseSleep()` and `timerSleep()` at different sleep intervals.

So it seems like somewhere around 5ms-6ms is when both `preciseSleep()` and `timerSleep()` decide that actually sleeping instead of spin-locking is safe enough, on my system at least. `timerSleep()` then immediately falls down to 0% CPU usage at 6ms+, while the CPU usage of `preciseSleep()` slowly falls off from ~12% to ~3%. 

## Data: robustness

All of this data was obviously collected only from _my_ computer, but OS scheduler granularity can vary a lot between different systems, and I wanted to make sure that the `preciseSleep()` routine performs well in general.

In order to test the behaviour on different scheduling periods, I made a little "fake" sleep routine that looks like this:

```cpp
void fakeSystemSleep(double seconds) {
    // sleep for at least the requested time
    spinLock(seconds);

    // align with the fake scheduler period
    double untilNextSchedule = SCHEDULER_PERIOD * (rand() / (double)RAND_MAX);
    spinLock(untilNextSchedule);
    
    // emulate another process grabbing the CPU
    while ((rand() / (double)RAND_MAX) < 0.05)
        spinLock(SCHEDULER_PERIOD);
}
```

It emulates the bahaviour of the system sleep function, although it spin locks instead of actually sleeping - but we don't need to worry about that for this test.

We've already seen that the system sleep function behaves something like the above. But the nice part about `fakeSystemSleep()` is that we can control this `SCHEDULER_PERIOD` parameter which is supposed to emulate the period of an acual OS scheduler.

With this fake sleep function I measured the accuracy of `preciseSleep()` when `SCHEDULER_PERIOD` was set to 1ms .. 10ms. These are common periods for real OS schedulers so they should be representative. Either way, here are the results.

![Figure 4](../assets/sleep/fig4.svg#center)

> **Figure 4** A heatmap showing the relative error when sleeping with `preciseSleep()` for different time intervals and with different scheduler periods. The relative error is calculated as (t<sub>a</sub> - t<sub>r</sub> ) / t<sub>r</sub> where t<sub>r</sub> is the requested sleep time, and t<sub>a</sub> is the actual time spent sleeping.

Ok so from the heatmap you can see how the error gradually increases as the scheduler period increases. This shows that `preciseSleep()` isn't very robust against high scheduler periods when small intervals of sleep are requested. 

We can also see that the error doesn't increase as much with higher scheduler periods, so `preciseSleep()` is pretty robust against higher scheduler periods when higher sleep intervals are requested (10+ ms).

## Conclusion

Well, we've seen first-hand the problems with using the system's `Sleep()` function for precise timing, and we've come up with 2 alternative sleeping functions to fix this. `preciseSleep()` is a cross platform function with higher precision, while `timerSleep()` is a Windows-only solution with a lower precision but lower CPU usage. 

I think I've also demonstrated that both of these functions can be very precise with an average error less than 0.1ms. _But_ the caveat is that they behave unpredictably if ran on an operating system whose scheduler operates on a period > 5ms. 

Luckily, modern operating systems have scheduler periods in the low range of 1-5ms, at least [according to Microsoft](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-2000-server/cc938613(v=technet.10)?redirectedfrom=MSDN). On my machine it seems to be around 2ms based on the `Sleep()` timings I'm getting. `preciseSleep()` seems to be pretty stable in this range.

However, if you're depending on this working on most computers, you're probably not going to want to use `preciseSleep()` or anything based even remotely on the system `Sleep()` call. Use a spin-lock, it will eat up your CPU, but sometimes it's a necessary evil.

I really wish OS API's provided more accurate timing measures. I'm really not sure what the excuse is for how frankly pathetic the OS provided timing measures truly are. Even adding something like a `beginLowPowerCPUMode()` function would go a long way, because then we could at least conserve the CPU while we spin lock for more precise timings. But, who knows, maybe there are some security issues with there that I'm not considering.

Either way, see you next time.

## Source Code

You can download all of the source code used in this post, as well as the script I used to process the data by clicking [here](/assets/sleep/src.zip).

If you want to take a look at the raw data used for the figures and data analysis you can download it by clicking here [here](/assets/sleep/data.zip).