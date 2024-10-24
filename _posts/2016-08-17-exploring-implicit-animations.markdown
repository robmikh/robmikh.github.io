---
layout: post
title:  "Exploring Implicit Animations"
date:   2016-08-17 19:15:00 -0700
categories: uwp xaml composition
---

A while back I put up a poll on [Twitter](https://twitter.com/robmikh/status/755932610740654080) asking you all what you would like to see me cover next. The people have spoken and we'll be covering Implicit Animations! Don't worry, we'll also be covering those other topics later! Implicit Animations were added in the Windows 10 Anniversary Update as part of the Composition API. We'll first cover the basic concept behind implicit animations, then we'll cover the need for implicit animations, and finally we'll put together a little demo that shows off the new feature!

<!--more-->

<h2>What are Implicit Animations?</h2>

*If you'd like to follow along, you can find this demo on [GitHub](https://github.com/robmikh/blog.samples/tree/master/2016.08.17/ImplicitAnimationsSimpleDemo)*

In the past, we've covered using the [animation capabilities in the Composition API](http://blog.robmikh.com/xaml/uwp/composition/2016/04/14/introduction-to-composition.html). These animations are constructed and can be applied at any time to `CompositionObject`s that have animatable properties. However, to actually fire the animation you were required to specify the target property as well as supply the animation to use in the object's `StartAnimation` function. If you were to change the property manually in the middle of an animation, the animation would be canceled. In this way, these animations are considered "explicit", in that you must call the `StartAnimation` function to start the animation.

Implicit Animations, on the other hand, are animations that are triggered by changes in the target property. This means that if you were to set the `Offset` property of a `Visual` that had an implicit animation defined, then all subsequent changes to `Offset` will cause an animation between the old value and the new value that was set.

So how do you create an implicit animation? At the core we still use the same animation objects that we would previously:

{% highlight c# %}
var offsetAnimation = _compositor.CreateVector3KeyFrameAnimation();
offsetAnimation.Target = nameof(Visual.Offset);
offsetAnimation.InsertExpressionKeyFrame(1.0f, "this.FinalValue");
offsetAnimation.Duration = TimeSpan.FromMilliseconds(1500);
{% endhighlight %}

Right off the bat, you'll notice some differences. The first is that there is a new `Target` property to our animation object. I'll explain why this is needed for now, but just know that we specify the property we intend the property to be used for. Next we use `InsertExpressionKeyFrame` (rather than `InsertKeyFrame`) to specify a key frame. When using implicit animations you don't need to set a key frame for the beginning of the animation, as that is implied. We do specify, however, the end of the animation. Here we use an expression key frame so that we can reference the final value of the property (remember, these animations will run because the target property is set by something/someone). The way we reference this final value is by using the `string` "this.FinalValue". After that we specify how long this animation should run for. Next, we set up our `ImplicitAnimationCollection`.

{% highlight c# %}
var implicitAnimations = _compositor.CreateImplicitAnimationCollection();
implicitAnimations[nameof(Visual.Offset)] = offsetAnimation;
{% endhighlight %}

You'll see that the `ImplicitAnimationCollection` is a dictionary of sorts, and that the key is the property that will trigger the animations we assign to the entry. After we do that, we then assign this `ImplicitAnimationCollection` to our visual, using the new `ImplicitAnimations` property:

{% highlight c# %}
_visual.ImplicitAnimations = implicitAnimations;
{% endhighlight %}

Now, any time we set the `Offset` property of our visual, it will animate to its new location using the animation we defined above. To illustrate that, I've added some code that sets the `Offset` property whenever a `Button` is clicked:

{% highlight c# %}
private void OffsetButton_Click(object sender, RoutedEventArgs e)
{
    if (!_isOnRight)
    {
        _visual.Offset = new Vector3(350.0f, 50.0f, 00.0f);
    }
    else
    {
        _visual.Offset = new Vector3(50.0f, 50.0f, 0.0f);
    }

    _isOnRight = !_isOnRight;
}
{% endhighlight %}

All of that should look something like this:

<iframe src='https://gfycat.com/ifr/ComposedRemorsefulBandicoot' frameborder='0' scrolling='no' width='640' height='452' allowfullscreen></iframe>{: .center-image }


Pretty neat, right? Well, we're not done yet. Remember that `Target` property that was on our key frame animation object? You may be asking why that's even there. What's the point if we're just going to specify it on the `ImplicitAnimationCollection` as well? Great question! That's because you're not limited to animating the value that is changing, and that you're only triggering the animation from the property change. If you wanted to, you could animate the `RotationAngle` every time the `Offset` was changed. Or even better, you could do both!

<h1>Animation Groups</h1>

Let's spin our visual around while it's moving. To do this, we'll need to create a new key frame animation:

{% highlight c# %}
var rotationAnimation = _compositor.CreateScalarKeyFrameAnimation();
rotationAnimation.Target = nameof(Visual.RotationAngleInDegrees);
rotationAnimation.InsertKeyFrame(0.0f, 0.0f);
rotationAnimation.InsertKeyFrame(1.0f, 360.0f);
rotationAnimation.Duration = TimeSpan.FromMilliseconds(1500);
{% endhighlight %}

Notice that we didn't do anything special for this key frame animation. You're not required to specify the "this.FinalValue" value, and can supply any animation you want. Next we create the `CompositionAnimationGroup` object and add our animations:

{% highlight c# %}
var animationGroup = _compositor.CreateAnimationGroup();
animationGroup.Add(offsetAnimation);
animationGroup.Add(rotationAnimation);
{% endhighlight %}

The `CompositionAnimationGroup` can then be used in place of our single animation:

{% highlight c# %}
implicitAnimations[nameof(Visual.Offset)] = animationGroup;
{% endhighlight %}

Now our visual spins as we move it!

<iframe src='https://gfycat.com/ifr/BouncyHideousEmu' frameborder='0' scrolling='no' width='640' height='452' allowfullscreen></iframe>{: .center-image }

At this point, you may be saying to yourself "that's nice and all, but I could also do that explicitly, what do I gain with Implicit Animations?" It's true that this example could of easily applied both animations on the button press, instead of relying on Implicit Animations. Generally speaking, when you have complete control over your visuals, you may not need to use Implicit Animations. But what if you're not the only one setting values?

<h2>Why use Implicit Animations?</h2>

You might recall that we made some changes to the way XAML-Composition interop worked, and we covered those changes [last time](http://blog.robmikh.com/uwp/xaml/composition/2016/07/16/changes-to-xaml-composition-interop.html). Starting in the Anniversary Update to Windows 10, the visual that is returned by `ElementCompositionPreview.GetElementVisual` is the same visual that XAML uses to do layout. This means that with Implicit Animations, you can have your `UIElement`s animate between their offset values every time XAML changes their position when it computes layout. One example of this is during a resize. 

I'm sure you can imagine having the elements of a `GridView` animate to their new position every time you resized your window, instead of just snapping into place. In short, Implicit Animations are the most useful when you don't have full control of the visual you wish to animate. This way you can guarantee an animation without being the one to consciously kick of the animation each time.

Before we jump into an example, it's important to go through some of the limitations of the Implicit Animation system:

* Implicit Animations are currently only available to `Visual`s, and not (yet) to other `CompositionObject`s.
* Implicit Animations will not trigger on the same commit that they are created.

The first one is pretty straight forward, and we're continually working to expand the support of the Implicit Animation system. However the second one might seem a bit foreign to some of you, which menas it's time for a history lesson.

<h1>Direct Composition and Commits</h1>

As we've covered in the past, the Composition API shares a lot with its predecessor, [Direct Composition](https://msdn.microsoft.com/en-us/library/windows/desktop/hh437371(v=vs.85).aspx) (also known as DComp). Both are based on the same [kernel communication](https://msdn.microsoft.com/en-us/library/windows/desktop/hh437350(v=vs.85).aspx) model with the DWM. When a change is made to an object from either the Composition API or from Direct Composition, the changes are not immediately relayed to the DWM. Instead, these changes a book-kept in the kernel and then sent over to the DWM as a "batch." This mechanism is controlled by calling `Commit` (on a DComp Device, which is analogous to the `Compositor` object in the Composition API) in Direct Composition, which instructs the kernel to send the current changes to the DWM. 

In the Composition API, we opted to move to a model with an implicit commit. In this model, `Commit` is regularly called on behalf of the developer. For the most part, you won't have to worry about commits, as we've design the API for commits to be largely invisible. However, they do tangentially come up when using the Implicit Animation system. We don't honor implicit animations that have been set on the first commit that they've been created/assigned. What does that mean? It means that you can't set the implicit animation on one line, set a property on the next line, and then expect to see the animation happen. Why is that? It's because we expect the implicit animations to be set at the same time a visual is created, and that could lead to (depending on the order of the calls) properties animating from their default value (0 in most cases) to their "real" first value during initialization. 

Imagine all of the items of your `GridView` animating to their position from the top left corner of their parent to their first position every time you load your app. Yikes! To prevent that, we decided to ignore implicit animations on the first commit they are assigned to a visual. What does that mean for you? Ideally, not much. We expect the Implicit Animation system to be leveraged to animate values in (direct or indirect) response to user input, like resizing a window, which would happen well after the first commit if you set them up during initialization. 

**My personal advice on this is to not worry too much about it.** I just thought I'd share this in case someone comes across this while trying to use the Implicit Animation system in some ~~weird~~ interesting way! If you use implicit animations to trigger layout animations, you should be just fine! You can read up on commits (specifically in relation to the Composition API) [here](https://msdn.microsoft.com/en-us/library/windows/apps/mt592877.aspx) on MSDN. If for whatever reason you would like to know when your current "batch" will be committed, you can use the `CompositionCommitBatch`, which you can read up about [here](https://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.composition.compositioncommitbatch.aspx).

Now with that out of the way, we can finally get onto our XAML-Composition interop sample using Implicit Animations!

<h2>Using Implicit Animations for layout animations</h2>

<!-- Bonus cut content:
"Because Pokémon GO is all the rage, let's say that we have an app with a `GridView` that displays the [Digimon](http://orig07.deviantart.net/41a2/f/2010/189/d/f/trollface_by_deniskapwnz.png) of the Digidestined from Digimon Adventure 1."

Just typing that caused me to watch Our War Game again. Toei... pls https://www.youtube.com/watch?v=14juI1oNg6Y
-->

*If you'd like to follow along, you can find this demo on [GitHub](https://github.com/robmikh/blog.samples/tree/master/2016.08.17/SimpleColorPicker)*

Let's say that we have a collection of colors arranged in a `GridView`, a sort of simple color picker:

![Simple Color Picker](/assets/colorpicker.png){: .center-image }

The first thing we'll want to do is hook into our `GridView`'s `ContentChanging` event:

{% highlight c# %}
private void MainGridView_ContainerContentChanging(
    ListViewBase sender, 
    ContainerContentChangingEventArgs args)
{
    var elementVisual = 
        ElementCompositionPreview.GetElementVisual(args.ItemContainer);
    if (args.InRecycleQueue)
    {
        elementVisual.ImplicitAnimations = null;
    }
    else
    {
        EnsureImplicitAnimations();
        elementVisual.ImplicitAnimations = _implicitAnimations;
    }
}
{% endhighlight %}

Here we are getting the visual that represents the current item container. Given that item containers can be recycled, we want to remove the implicit animations from these containers. One they are ready to be reused, we'll see them again without the `InRecycleQueue` flag. If an item container doesn't have the `InRecycleQueue` flag, we'll attach our implicit animation to it. Notice that we're going with an "ensure pattern" for our implicit animation. Technically we would be fine initializing our implicit animation during our page's initialization, but to avoid a race altogether we used this pattern:

{% highlight c# %}
private void EnsureImplicitAnimations()
{
    if (_implicitAnimations == null)
    {
        var compositor = 
            ElementCompositionPreview.GetElementVisual(this).Compositor;

        var offsetAnimation = compositor.CreateVector3KeyFrameAnimation();
        offsetAnimation.Target = nameof(Visual.Offset);
        offsetAnimation.InsertExpressionKeyFrame(1.0f, "this.FinalValue");
        offsetAnimation.Duration = TimeSpan.FromMilliseconds(400);

        var rotationAnimation = compositor.CreateScalarKeyFrameAnimation();
        rotationAnimation.Target = nameof(Visual.RotationAngle);
        rotationAnimation.InsertKeyFrame(.5f, 0.160f);
        rotationAnimation.InsertKeyFrame(1f, 0f);
        rotationAnimation.Duration = TimeSpan.FromSeconds(400);

        var animationGroup = compositor.CreateAnimationGroup();
        animationGroup.Add(offsetAnimation);
        animationGroup.Add(rotationAnimation);

        _implicitAnimations = compositor.CreateImplicitAnimationCollection();
        _implicitAnimations[nameof(Visual.Offset)] = animationGroup;
    }
}
{% endhighlight %}

Now our color squares should fly around when we resize the window!

<iframe src='https://gfycat.com/ifr/DazzlingOddballArgusfish' frameborder='0' scrolling='no' width='640' height='360' allowfullscreen></iframe>{: .center-image }

However, there's one more thing I'd like to cover before we wrap up. Now that there has been 3 major releases of Windows 10 (Original Release, November Update, and Anniversary Update), I think it's time to talk about how to make sure your apps still run on older versions of Windows 10 while still taking advantage of new features. You'll notice that this demo project is targeting the Anniversary Update (build 14393), but it has a minimum version of the first release of Windows 10 (build 10240). So how do we make sure that this still works without blowing up when an unsupported API is called? The solution is the `ApiInformation.IsTypePresent` function. It will return true if the API is available. It's important to note that this isn't just for different versions of Windows 10, but also to check if features are supported on the device you're on. Not all APIs are available to all platforms, and while that list shrinks with every release, please check MSDN to see what calls you should wrap with `ApiInformation` calls.

As long as you don't call code that isn't supported, it's ok for the code itself to "exist" in the app. In our case, since both the initialization and application of our implicit animations are centralized in the `ContentChanging` event handler, we can add our call to `ApiInformation` there:

{% highlight c# %}
private void MainGridView_ContainerContentChanging(
    ListViewBase sender, 
    ContainerContentChangingEventArgs args)
{
    if (ApiInformation.IsTypePresent(
            typeof(ImplicitAnimationCollection).FullName))
    {
        var elementVisual = 
            ElementCompositionPreview.GetElementVisual(args.ItemContainer);
        if (args.InRecycleQueue)
        {
            elementVisual.ImplicitAnimations = null;
        }
        else
        {
            EnsureImplicitAnimations();
            elementVisual.ImplicitAnimations = _implicitAnimations;
        }
    }
}
{% endhighlight %}

Now if you run this app on builds 10240 or 10586 (the November Update) the app won't crash when you run it! It won't have the implicit animation, but your users will appreciate still being able to use your app. 

And that does it for this time. Thanks for reading! If you'd like to keep up to date with the blog, make sure you’re subscribed to the [RSS feed](http://blog.robmikh.com/feed.xml). As always, feel free to ask me questions ~~on twitter or~~ by [email](mailto:robmikh@robmikh.com). Also feel free to come by the team's [GitHub page](https://github.com/Microsoft/WindowsUIDevLabs) to ask questions as well. See you next time!