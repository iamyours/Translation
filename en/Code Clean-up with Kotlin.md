> title: Code Clean-up with Kotlin. 
> source: [https://proandroiddev.com/code-clean-up-with-kotlin-19ee1c8c0719](https://proandroiddev.com/code-clean-up-with-kotlin-19ee1c8c0719)
> aurthor: [Gabor Varadi](https://proandroiddev.com/@Zhuinden)

I haven’t written a full-fledged article since [that article on Dagger in November!](https://medium.com/@Zhuinden/that-missing-guide-how-to-use-dagger2-ef116fbea97) Clearly this can’t go on like this forever, can it?  

However, being swamped with work to do and being busy is generally a good thing — especially if what you’re paid to work on is written in 100% Kotlin! After a while, things you take for granted in Java just seem like meaningless clutter that can be refactored with Kotlin language features — in ways that Java’s language specifications wouldn’t allow.  

I’ve seen a lot of people (previously, me included) who felt that “usage of Kotlin directly results in an unreadable mess”; but apparently with some discipline it really is possible to write more concise and more readable code.  

So without further ado, here are some tips and tricks to reduce clutter compared to what one would do in Java, while keeping it readable instead of just short.  

### 1.) Ability to use control flow keywords as part of an assigment
While this might seem basic, I often see newly written Kotlin samples that do not utilize this feature at all.  

In Java, one could easily write code like this:  
``` java
String name;
if(something) {
    ...
    name = "Something";
} else {
    name = "Other thing";
}
```
In Kotlin, we can reduce this like so:  
``` kotlin
val name = if(something) {
    ...
    "Something" 
} else "Other thing"
```
Personally, I’m not a fan of abandoning the `{`and `}` of an if statement, I even set the auto-formatter to force the braces. We can make this nicer using `when`, allowing us to nicely encapsulate this block of code.  

``` kotlin
val name = when {
    something -> {
        ...
        "Something"
    }
    else -> "Other thing"
}
```

What’s nice is that we can take this further one level — we could even write something like `return when {...}`.

### 2.) Using inline generic extension functions to reduce duplication
#### a.) apply
Sometimes, we have some setup code that we just can’t reduce at all with Java. A good example would be the static factory method `newInstance()` that people tend to define for Fragments.  
``` java
public class CatFragment extends Fragment {
    public static CatFragment newInstance(String catId) {
        CatFragment catFragment = new CatFragment();
        Bundle bundle = new Bundle();
        bundle.putString("catId", catId);
        catFragment.setArguments(bundle);
        return catFragment;
    }
}
```
Then we have another fragment, initialized in a very similar way:  
``` java
public class DogFragment extends Fragment {
    public static DogFragment newInstance(String dogId) {
        DogFragment dogFragment = new DogFragment();
        Bundle bundle = new Bundle();
        bundle.putString("dogId", dogId);
        dogFragment.setArguments(bundle);
        return dogFragment;
    }
}
```
Now with Kotlin, we could keep this exact same logic:  
``` kotlin
class CatFragment: Fragment() {
    companion object {
        fun newInstance(catId: String): CatFragment {
            val catFragment = CatFragment()
            val bundle = Bundle()
            bundle.putString("catId", catId)
            catFragment.arguments = bundle
            return catFragment
        }
    }
}
```
In fact, if you check samples online, this is what you often find.  

However, **all those local variables** I defined with `val`? They **are completely unnecessary**. I can use [the standard generic inline function apply {](https://github.com/JetBrains/kotlin/blob/9ffd0db4a87116a0c825f2f0163b8ad3eb1a922f/libraries/stdlib/src/kotlin/util/Standard.kt#L45-L49), and move these instantiations in the lambda.  
``` kotlin
class CatFragment: Fragment() {
    companion object {
        fun newInstance(catId: String) = CatFragment().apply {
             arguments = Bundle().apply {
                 putString("catId", catId)
             }
        }
    }
}

class DogFragment: Fragment() {
    companion object {
        fun newInstance(dogId: String) = DogFragment().apply {
             arguments = Bundle().apply {
                 putString("dogId", dogId)
             }
        }
    }
}
```
We’ve eliminated a lot of duplication and unnecessary local variables we wrote “just to make things work”.   

However, one could argue that we’re nesting a lot. And we’re duplicating the `apply { arguments = `stuff. Couldn’t this be made nicer? Sure could be, if we define our own extension function for every fragment.   

We can write this code:  
``` kotlin
class CatFragment: Fragment() {
    companion object {
        fun newInstance(catId: String) = CatFragment().withArgs {
             putString("catId", catId)
        }
    }
}
class DogFragment: Fragment() {
    companion object {
        fun newInstance(dogId: String) = DogFragment().withArgs {
             putString("dogId", dogId)
        }
    }
}
```
Where `withArgs` is:  
``` kotlin
inline fun <T: Fragment> T.withArgs(
                             argsBuilder: Bundle.() -> Unit): T = 
    this.apply {
        arguments = Bundle().apply(argsBuilder)
    }
```
With that, our code is more readable, and with the help of Kotlin’s `inline` keyword, this simplification doesn’t have any performance cost.  

If the signature of `withArgs` seems somewhat complicated, I assure you — reading this stuff becomes first-hand nature after working with Kotlin for a bit. After all, we’re just passing a lambda, and mess around a bit with who’s `this`! I initially learned about it from [here](https://academy.realm.io/posts/kau-jake-wharton-testing-robots/).  

#### b.) let
Oftentimes you check through some Kotlin code and it looks like this:  
``` kotlin
activity?.childFragmentManager
        ?.beginTransaction()
        ?.setCustomAnimations(...)
        ?.replace(...)
        ?.addToBackStack(...)
        ?.commit()
```
So many `?`s! Clearly we can make this nicer?  
``` kotlin
activity?.let { activity ->
    activity.childFragmentManager
            .beginTransaction(...)
            .setCustomAnimations(...)
            .replace(...)
            .addToBackStack(...)
            .commit()
}
```
By using a single `?.let`, we could remove all the safe call operators! Also, one more things to mention, of course — instead of letting `let` rename the argument to `it`, I can just specify an actual name for it.  

This is probably not a surprise to long-time Kotlin users, but still — you often find examples like this:   
``` kotlin
placeAutocompleteResult.predictions.forEach {
    placeList.add(PlaceModel(it.description, it.placeId))
    placeNameList.add(it.description)
}
```
So you look at `it` and **you need to find out from the context what it is.**  

I’d prefer to write:  
``` kotlin
placeAutocompleteResult.predictions.forEach { prediction ->
    placeList.add(PlaceModel(prediction))
    placeNameList.add(prediction.description)
}
```
More characters? Yes. More readable? Also yes.  

I try to avoid the usage of `it` almost wherever possible. It definitely solves the “nested `it`” problem.  

#### c.) takeIf
In some cases like this:  
``` kotlin
if(something != null && something.thingy) {
    ...
} else {
    ...
}
```
You can replace that logic with.  
``` kotlin
something?.takeIf { it.thingy }?.let {
   ...
} ?: {
   ...
}
```
Which can be handy in some assignments. I don’t use it that often though.   

### 3.) Killing Android View boilerplate with anko / kotlin-android-extensions
First thing first: [anko](https://github.com/Kotlin/anko) is a modular library, which was written with the intention to help Android development. It is also completely independent from `kotlin-android-extensions`.  

A lot of **anko** is a double-edged sword: for example, I sure wouldn’t use `anko-layouts`, nor `anko-sqlite`, and probably not `anko-coroutines` either (but I haven’t delved that deep in `suspend`, personally).  

However, **anko** has two useful things: `anko-commons` and `anko-listeners`.  
``` gradle
implementation “org.jetbrains.anko:anko-commons:0.10.4” 
implementation “org.jetbrains.anko:anko-sdk15-listeners:0.10.4”
```
Once we’ve added these, and we also apply 

``` gradle
apply plugin: ‘kotlin-android-extensions’
```

We can now replace all that view binding  
``` kotlin
@BindView(R.id.my_button)
lateinit var myButton: Button
@OnClick(R.id.my_button)
fun onButtonClick() {
    ...
}
override void onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.my_activity)
    ButterKnife.bind(this)
}
```
With  
``` kotlin
import kotlinx.synthetic.my_activity.myButton
override void onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.my_activity)
    myButton.onClick {
       ...
    }
}
```
I used to favor ButterKnife, but as no one (including me) wrote an annotation processor that would generate page objects for Espresso tests based on `@BindView` and `@OnClick` annotations, I guess the transition can be worth it.   

(EDIT from the future: In fact, check out my library [espresso-helper](https://github.com/Zhuinden/espresso-helper) for leveraging the power of extension functions!)  

Don’t forget to use `LayoutContainer` in RecyclerView’s ViewHolders, though.  

Also, please use `camelCase` for your view IDs if you use `kotlin-android-extensions`.   

#### 4.) Killing Parcelable boilerplate with @Parcelize
Implementing Parcelable manually is a pain. Generating a Parcelable implementation is easy, but maintenance after it is hard. `auto-value-parcel` is nice, but what if we could make it easier?  

Sure enough, thanks to `kotlin-android-extensions`, we can now use `@Parcelize` to turn something parcelable.  
``` kotlin
androidExtensions { 
    experimental = true 
}
```
Then we can do:  
``` kotlin
@SuppressLint("ParcelCreator")
@Parcelize
data class CatKey(val clazz: String) : Parcelable {
    constructor() : this("CatKey")
    ...
}
```
`@Parcelize` uses the primary constructor for determining what to save out as Parcelable. For additional configuration, it is possible to override the default behavior with `object: Parceler<T>`, and we can also provide `@TypeParceler` if needed. You can also ignore properties with `@IgnoredOnParcel`.  

It’s all described on the [official page](https://kotlinlang.org/docs/tutorials/android-plugin.html), though. Sometimes the documentation is more helpful than Stack Overflow (as I couldn’t find any questions about it — the documentation describes things perfectly well, though!).  

#### 5.) A note on DI frameworks
I’d generally ignore Kodein for now, or at least [until 5.0 is out of beta](https://salomonbrys.github.io/Kodein/?5.0/migration-4to5). It seems that it’ll make the DSL nicer.  

However, in my opinion, [Dagger isn’t as hard as people tend to say](https://medium.com/@Zhuinden/that-missing-guide-how-to-use-dagger2-ef116fbea97), it’s still the safest bet when it comes to all these emerging “DI” libraries (or service locators) like Koin and Kodein.  

``` kotlin
@Singleton class MyRepository @Inject constructor(
    private val service: MyService,
    private val dao: MyDao
) { ...
```
Dagger should work just fine, as long as you’re using it with the JVM world.  

### Conclusion
Hopefully, this article helped show some ways to utilize Kotlin and its extensions to simplify our code, while keeping it still readable.  

I haven’t really mentioned lazy delegates nor tuples/decomposition, those also have their uses from time to time. I also haven’t mentioned the newly released [android-ktx either](https://github.com/android/android-ktx), there’s lots of ideas to gather from its source code!  

For what it’s worth, the Kotlin ecosystem is growing, and it truly does have features (`apply`, `when`, inline extension functions and higher order functions) which make everyday problems easier to solve.  


























