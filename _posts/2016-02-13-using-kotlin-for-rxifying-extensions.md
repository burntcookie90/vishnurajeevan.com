---
title:  Using Kotlin Extensions for Rx-ifying 
---

Kotlin and Reactive Extensions (Rx) are the new hotness in Android development, and not without reason. Both technologies are loved for being concise, expressive and powerful. This is especially useful in the Android world where APIs can be long-winded and filled with ceremony.

One of the main features that I've fallen in love with while using Kotlin is class extensions (docs). This feature comes in handy quite often when working with APIs that are not yet RxJava compatible in a project that is using RxJava extensively.

Let's take a look at a simple example when using the SlidingUpPanel library.
Basic API Usage

One of the things you might want to do when using this library is listen for panel events such as expanding and collapsing of the panel.

When doing this without RxJava you get the following codeblock

```kotlin
slidingUpPanel.setPanelSlideListener(object: SlidingUpPanelLayout.PanelSlideListener {
  override fun onPanelExpanded(p0 : View?) {
  }

  override fun onPanelSlide(p0 : View?, p1 : Float) {
  }

  override fun onPanelCollapsed(p0 : View?) {
  }

  override fun onPanelHidden(p0 : View?) {
  }

  override fun onPanelAnchored(p0 : View?) {
  }
})
```

This is a required implementation when you might only care about the expanded and collapsed state.
Kotlin Extension

So, lets move this into a Kotlin extension:

We're going to start with a data class to model the panel events:

```kotlin
enum class PanelEvent { COLLAPSED, EXPANDED, HIDDEN, ANCHORED, SLIDE }
data class PanelData(val event : PanelEvent, val panel : View?, val slideOffset : Float? = null)
```

Now the actual class extension:

```kotlin
fun SlidingUpPanelLayout.panelSlides() : Observable<PanelData> {
  return Observable.defer<PanelData> {
    Observable.create {
      if (!it.isUnsubscribed) {
        setPanelSlideListener(object : SlidingUpPanelLayout.PanelSlideListener {
          override fun onPanelSlide(panel : View?, slideOffset : Float) = it.onNext(PanelData(PanelEvent.SLIDE, panel, slideOffset))
          override fun onPanelExpanded(panel : View?) = it.onNext(PanelData(PanelEvent.EXPANDED, panel))
          override fun onPanelCollapsed(panel : View?) = it.onNext(PanelData(PanelEvent.COLLAPSED, panel))
          override fun onPanelHidden(panel : View?) = it.onNext(PanelData(PanelEvent.HIDDEN, panel))
          override fun onPanelAnchored(panel : View?) = it.onNext(PanelData(PanelEvent.ANCHORED, panel))
        })

        it.add(object : MainThreadSubscription() {
          //we should null the listener when the subscriber unsubscribes
          override fun onUnsubscribe() = setPanelSlideListener(null)
        })
      }
  }}
}
```

And the calling code:

```kotlin
slidingUpPanel.panelSlides()
  .map { it.event }
  .filter {it == PanelEvent.COLLAPSED || it == PanelEvent.EXPANDED }
  .subscribe {
    Timber.d("panel state changed")
  }
```

Now you have the power of RxJava's composibility and concise syntax availble to you, without losing the context of the action itself.
Java Interop

Because of Kotlin and Java's interoperability we can also use this extension from Java code as a "Util" class.

Add the following to the top of the extension file (otherwise it'll use <FileName>Kt as the classname)

```kotlin
@file:JvmName("SlidingUpPanelLayoutUtils")
```

And now from java code you can do the following

```kotlin
SlidingUpPanelLayoutUtils.panelSlides(slidingUpPanelLayout)
      .map(new Func1<PanelData, PanelEvent>() {
        @Override
        public PanelEvent call(PanelData panelData) {
          return panelData.getEvent();
        }
      })
      .filter(new Func1<PanelEvent, Boolean>() {
        @Override
        public Boolean call(PanelEvent panelEvent) {
          return panelEvent == PanelEvent.COLLAPSED || panelEvent == PanelEvent.EXPANDED;
        }
      })
      .subscribe(new Action1<PanelEvent>() {
        @Override
        public void call(PanelEvent panelEvent) {
          Timber.d("panel state changed");
        }
      });
```

For more examples, checkout RxBinding which has extensions for Android framework classes.
