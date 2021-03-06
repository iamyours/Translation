> 原文: [Managing visibility in Flutter - How hiding widgets can also hide inefficiency in your code.](https://medium.com/flutter/managing-visibility-in-flutter-f558588adefe)     
> 作者: [Luke Freeman](https://medium.com/@lukeaf)  
> Markdown:[原文](https://github.com/iamyours/Translation/blob/master/en/1.Your%20android%20libraries%20should%20not%20ask%20for%20an%20application%20context.md) | [译文](https://github.com/iamyours/Translation/blob/master/zh/1.%E4%BD%A0%E7%9A%84Android%E5%BA%93%E6%98%AF%E5%90%A6%E8%BF%98%E5%9C%A8Application%E4%B8%AD%E5%88%9D%E5%A7%8B%E5%8C%96%EF%BC%9F.md). 

![By Marian (https://www.flickr.com/photos/lescientist/8430282209) ](https://miro.medium.com/max/3072/1*q-Y9wu-ZE_4YHLYcPpjgGA.jpeg)
**Update:** Flutter now has a built in Visibility widget so it’s better you use that. However, it’s still a good read on widget design and implementation so give it a read and a clap!  

I thought I’d write a quick snippet / tutorial on how you can easily write a generic widget to manage visibility in your Flutter application. It’s a pretty common task and it’s something that, once you’ve written it, you’ll probably re-use across all your projects. But, before we begin, let’s talk about what visibility means.  

For most use cases there are going to be four specific visibility states:  

### Visible
This is self explanatory.

### Invisible
Rendered in place. Is not visible and is not interactive. You will probably use this state in conjunction with another constrained widget (e.g: Container) because Invisible widgets will still occupy space on the screen even though they’re not interactive and not visible. For example, you may have a Container with a background that has an Image that is invisible until it loads. You don’t want the UI jumping around in this case.  

### Offscreen
Rendered off screen. Is not visible and is not interactive. It will still incur a cost but it is generally OK because it is useful for doing widget size calculations where you don’t need to show the widget all of the time. For example, if you need to calculate the size of a widget for the end state of an animation where the widget is only visible right at the end.  

### Gone
Your widget is not rendered at all. Ideally the widget is never included in the widget tree. It should incur a zero or close to zero cost.  

Regarding cost, you should consider the following: **It is always going to be better for your app to never include the widget in the first place.** This way Flutter doesn’t have to worry about widgets that aren’t ever going to be displayed. If it’s something that you may include based on some state then you can still use this approach as well but it may help to simplify things by using a more generic mechanism. In general, your approach will be a bit like:  
``` dart
List<Widget> views = [];
if (shouldBeIncluded) {
  views.add(myView);
}
// use views later
```

When deciding what strategy to use, ask yourself the following:  
- **Will the condition for displaying my widget never be met for certain cases?** An example is when you have a first time experience, on-boarding, etc. If you have a condition where you only show it the first time the app starts then you won’t want to include it ever again. So, this one is easy. In this case you should not include your widget in the first place. You can use a Gone strategy.

- **Do I need to keep the widget in place or should the rest of the UI adapt to fill the missing space?** If you need to keep it in place and do a swap or something similar then you can use an Invisible strategy and toggle the visibility as needed. If you need to eliminate the “white space”, then you can use Offscreen.  

- **Do I need to still factor in the size of my widget even if it isn’t visible?** Use Offscreen but read the above point first :)

So, keeping in mind all of this information, is it possible to develop a re-usable widget to manage visibility? Yes. **With caveats.**  

So, what does the code look like? Like this: 
``` dart
import 'package:flutter/widgets.dart';
import 'package:meta/meta.dart';

enum VisibilityFlag {
  visible,
  invisible,
  offscreen,
  gone,
}

class Visibility extends StatelessWidget {
  final VisibilityFlag visibility;
  final Widget child;
  final Widget removedChild;

  Visibility({
    @required this.child,
    @required this.visibility,
  }) : this.removedChild = Container();

  @override
  Widget build(BuildContext context) {
    if (visibility == VisibilityFlag.visible) {
      return child;
    } else if (visibility == VisibilityFlag.invisible) {
      return new IgnorePointer(
        ignoring: true,
        child: new Opacity(
          opacity: 0.0,
          child: child,
        ),
      );
    } else if (visibility == VisibilityFlag.offscreen) {
      return new Offstage(
        offstage: true,
        child: child,
      );
    } else {
      // If gone, we replace child with a custom widget (defaulting to a
      // [Container] with no defined size).
      return removedChild;
    }
  }
}
```
A quick note on how it works before we get into the caveats:  

- For **Visible**: we don’t do anything special.  
- For **Invisible**: we wrap the widget in an IgnorePointer widget and an Opacity widget with the value zero. This limits your ability to interact with the widget and hides it but will keep it in place. So, you will see empty space wherever the widget is inserted.
- For **Offscreen**: we wrap the widget in an Offstage widget which will render the widget off screen so that it won’t be visible and it won’t take up any screen real estate. However, it is still being rendered so it won’t be zero cost. Beware.
- For **Gone**: we provide a way for you to render a different widget (defaulting to a Container with no size). If you want to render a placeholder, an icon or something else, this is your place to do it. It still isn’t zero cost, but it shouldn’t be too awful for most cases. For true Gone, use the strategies described above.  

So a couple of quick caveats:  

- As mentioned above, there are real costs associated with this approach so be careful and where possible just don’t include your original widget. I sound like a broken record but you shouldn’t forget it. There is no tree shaking here.  
- Opacity widgets can be super expensive so use **Invisible** sparingly.

Aaaand that’s it. This widget is still pretty useful but, be smart and write efficient code. Your visibility approach will/should always take into consideration your specific needs so be sure to always think about what is the best approach.  



