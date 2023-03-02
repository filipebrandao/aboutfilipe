---
title: "Android UI Testing: Flaky Espresso vs ViewPager"
date: 2021-05-01T22:00:00+01:00
draft: false
---


Testing is usually hard and expensive. UI Testing on Android it's not only that, it's also painful.   

One of the culprits is the Espresso framework. Android Developers need all kinds of hacks to get a deterministic behavior out of it.
I personally experienced being able to running my UI test suite successfully in a local emulator while the same test suite would fail when running on an emulator in a remote host: 
- Both emulators were running the same Android version
- Both had the animations disabled system-wide
- The same hacks to disable edge-case animations were applied 🤡
- Both had hardware acceleration enabled

It was hard to find the root of the flakiness. But the fact that the failing tests were all on a screen with a ViewPager, reminded me that I've ran into some issues in the past. Apparently, Espresso doesn't idle when switching ViewPager tabs.

## The solution

I'm sharing this piece of kotlin code that should be part of the UI Testing Swiss Knife of Android devs.
Registering this `ViewPager2IdlingResource` in the `IdleRegistry` will make sure that the test steps will only be executed once the ViewPager scroll is settled.
```kotlin
class ViewPager2IdlingResource(viewPager: ViewPager2) : IdlingResource {  
  
    companion object {  
        private const val NAME = "viewPagerIdlingResource"  
    }  
  
    private var isIdle = true // Default to idle since we can't query the scroll state.  
    private var resourceCallback: IdlingResource.ResourceCallback? = null  

    init {  
        viewPager.registerOnPageChangeCallback(object : ViewPager2.OnPageChangeCallback() {  
            override fun onPageScrollStateChanged(state: Int) {  
                // Treat dragging as idle, or Espresso will block itself when swiping  
                isIdle = (state == ViewPager2.SCROLL_STATE_IDLE || state == ViewPager2.SCROLL_STATE_DRAGGING)  
  
                if (isIdle && resourceCallback != null) {  
                    resourceCallback!!.onTransitionToIdle()  
                }  
            }  
        })  
    }  
  
    override fun getName() = NAME  
  
    override fun isIdleNow() = isIdle  
  
    override fun registerIdleTransitionCallback(resourceCallback: IdlingResource.ResourceCallback) {  
        this.resourceCallback = resourceCallback  
    }  
}
```
Then just use it in the test code like this: 
```kotlin
// before interacting with the ViewPager
val viewPager2IdlingResource = ViewPager2IdlingResource(EspressoHelper.getCurrentActivity()!!.findViewById(R.id.viewPager))  
IdlingRegistry.getInstance().register(viewPager2IdlingResource)

(...)

// cleanup
IdlingRegistry.getInstance().unregister(viewPager2IdlingResource)
```
Here's the code for `EspressoHelper.getCurrentActivity()` which allows us to get the instance of the current Activity during the test execution: 
```kotlin
fun getCurrentActivity(): Activity? {  
    var currentActivity: Activity? = null  
    getInstrumentation().runOnMainSync { run { currentActivity = ActivityLifecycleMonitorRegistry.getInstance().getActivitiesInStage(Stage.RESUMED).elementAtOrNull(0) } }  
    return currentActivity  
}
```

Happy testing with reduced flakiness 🚀
