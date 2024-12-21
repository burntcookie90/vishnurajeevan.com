---
title: APIs In The Real World
---

Anytime we host guests, its almost guaranteed we'll get the following question: "Where's y'all's recycling?". We have a dual bin combo recycling/trash can and it seems that its just enough indirection to be confusing in practice. The recycling side has a blue bag and the trash side is white. For my wife and I, this is enough to maintain our habits without confusion. However, for guests, folks have opened up the bin and asked to confirm "left side is recycling right?". 

```kotlin
interface TrasherAndRecycler {
  fun trash(obj: Any)

  fun recycle(obj: Any)
}
```

After a few times of that, I decided to print out labels that you can see when you open the lid. _Even then_ people open the lid with their foot and look over at me "which side is recycling?".

```kotlin
interface TrasherAndRecycler {

  /**
  * Use this for garbage
  */
  fun trash(obj: Any)

  /**
  * Use this for recycling
  */
  fun recycle(obj: Any)
}
```

I left it at that for years, just allowing myself to live in the world of responding "left side of the bin, yep!". 

Last night, we were prepping for hosting a party and I decided to make both sides of the bin trash, and put white bags in both and labeled both sides as "garbage". I then put a blue bin next to it, as well as a blue bag on the deck for recycling. This morning I realized that, not only did I not get asked even once where the recycling is, the blue bags and bins had _only_ recycling in them!

```kotlin
interface Trasher {
  /**
  * Use this for garbage
  */
  fun trash(obj: Any)
}
```

```kotlin
interface Recycler {
  /**
  * Use this for recycling
  */
  fun recycle(obj: Any)
}
```

The design of our trash bin obscures the "functionality" of the two containers within it. New guests step on the pedal to open the bin but then don't notice the blue bag, or the labeling and end up having to ask for confirmation.

![](/assets/image/trash-bin.jpg)