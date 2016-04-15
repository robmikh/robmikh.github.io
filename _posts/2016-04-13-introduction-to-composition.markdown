---
layout: post
title:  "Introduction to Composition"
date:   2016-04-13 19:48:00 -0700
categories: xaml uwp composition
---

Welcome to the Composition API! If you've watch some of the [//build talks](https://channel9.msdn.com/Events/Build/2016/B818) the past two years you may have seen some members of our team show off the new API. While the API is certainly new for Windows 10, the Composition team has actually been around for awhile...

<!--more-->

**UPDATE 04-14-2016: [@NicoVermerir](https://www.twitter.com/nicovermeir) made a great [suggestion](https://twitter.com/NicoVermeir/status/720864213992755200) involving the use of `nameof`, check it out bellow.**

<h2>History Lesson</h2>

Since Windows Vista, the DWM has been powering the Windows experience. Everything you see on the screen we almost always have some hand in drawing. Developers were able to (and still are!) leverage the system compositor using [Direct Composition](https://msdn.microsoft.com/en-us/library/windows/desktop/hh437371%28v=vs.85%29.aspx). DComp, as we call it, was a way to position elements (that we call Visuals) on the screen in a tree like structure. For any of you that know the concept of the Visual Tree in XAML, this should sound familiar. The API was C++ only, and when Windows 8 came around there wasn't a way to use the API in the new modern app world. This is where the Windows.UI.Composition API came into existence with Windows 10, as a way to leverage the DWM to make fast and beautiful user interfaces without breaking the perf bank.

For a more in-depth look at Composition as a whole, our friend [Kenny Kerr](https://twitter.com/kennykerr) wrote a great article that you can check out [here](https://msdn.microsoft.com/en-us/magazine/mt590968).

<h2>Getting Started</h2>

While it is possible to create a "pure" Composition app, many of you will be delighted to know that you can actually consume the API from within XAML! This is a way to manipulate your pre-existing applications to add rich new experiences without much code. The very first thing you'll want to do is to grab the `Visual` that represents a `UIElement`. For many of you, this method will become your best friend:

{% highlight c# %}
var visual = ElementCompositionPreview.GetElementVisual(this);
{% endhighlight %}

<h1>What exactly is a visual?</h1>

A Visual is a node in the Visual Tree, which means they can be the children of another Visual. However, there are different types of Visuals. First is `ContainerVisual` which inherits from `Visual`. `ContainerVisual` is different from `Visual` in that it can have children of its own. `SpriteVisual` inherits from `ContainerVisual`, which means that it can have children of its own but also host its own content. This content is applied using the `Brush` property on `SpriteVisual`. For the most part, expect to be working with `SpriteVisual`. Just like with XAML's Visual Tree, Visuals are always relatively positioned by their parent. This also means that anything you attach to a `UIElement` will be placed *on top* of it.

![Visual Hierarchy](/assets/visualhierarchy.JPG){: .center-image }

Comming back to the world of XAML, `ElementCompositionPreview.GetElementVisual` retreives the Visual that represents a given `UIElement`. Any `UIElement`, which even includes the Frame (this makes for some pretty neat animation tricks and transitions that we'll cover in a later post)! Some things you should be aware of when getting a `Visual` from XAML:

* GetElementVisual returns a `Visual`, so you can't assign content to it. You are however, free to add a child `SpriteVisual` that has content (or event an entire subtree by attaching a `ContainerVisual` with many children!) by using the `ElementCompositionPreview.SetElementChildVisual` method.
* Using the Composition API requires the November 2015 update to Windows 10, otherwise known as build 10586.
* You can move entire subtrees or single visuals anywhere in the tree with ease, but you always need to detach it from its parent first. Currently, Visuals returned by `ElementCompo sitionPreview.GetElementVisual` do not report their parent (even though they have one) and thus cannot be reparented somewhere else without doing it in XAML. The rule of thumb is usually: when you've created it you've got complete control of it. You can, however, change properties or animate Visuals returned by XAML.

<h2>Baby's First Visual</h2>

*If you'd like to follow along, the entire sample we'll be building is available [here](https://github.com/robmikh/blog.samples/tree/master/2016.04.13/Hello%20World).*

The easiest thing to do is to create a `SpriteVisual` with a solid color, so we'll start with that. After creating a blank XAML app, go to MainPage.xaml.cs and try something like this:

{% highlight c# %}
var compositor = ElementCompositionPreview.GetElementVisual(this).Compositor;
var visual = compositor.CreateSpriteVisual();

visual.Size = new Vector2(150, 150);
visual.Offset = new Vector3(50, 50, 0);
visual.Brush = compositor.CreateColorBrush(Colors.Red);

ElementCompositionPreview.SetElementChildVisual(this, visual);
{% endhighlight %}

What does this do? It first gets the `Compositor` object from the Page's `Visual`. The `Compositor` object is the second most important concept in the Composition API. Nearly every object related to the Composition API is created through this object. Usually your first plan of attack should be to get ahold of this object. Luckily for you, nearly every Composition object has a reference back to the `Compositor` that created it. Within your XAML application, there will only be one `Compositor` so it doesn't matter where you get it from.

After we've retrieved the `Compositor` we are able to create a `SpriteVisual`. After that we set the visual's size and give it an Offset (notice that you'll need to be using the System.Numerics namespace for this). Finally we create a brush that just contains the color red and assign it to the visual (we'll cover image loading in a future post next week). The only thing left to do is to attach it to the tree by using `SetElementChildVisual`. Just like with XAML, if it's not in the tree it won't draw.

All of that should get you this:

![Hello World!](/assets/helloworld1.jpg){: .center-image }

So far, so good. But I think we can do better. One of the strengths of the Composition API are the animations. Let's try one of those next:

<h2>Fun with animations</h2>

*You may be wondering why we've jumped straight to animations, a seemingly advanced concept. Have no fear, as it's actualy fairly easy using the Composition API! In this section you'll learn how to create animations, and attach these animations not only to your own visuals but to XAML elements as well.*

One of the animations available to us is the `KeyFrameAnimation`, which we can create by using the `Compositor`:

{% highlight c# %}
var animation = compositor.CreateScalarKeyFrameAnimation();
{% endhighlight %}

Next we'll want to add some key frames:

{% highlight c# %}
var easing = compositor.CreateLinearEasingFunction();

animation.InsertKeyFrame(0.0f, 0.0f);
animation.InsertKeyFrame(1.0f, 360.0f, easing);
{% endhighlight %}

The first parameter signifies a normalized time for that key frame, 0.0f being the beginning and 1.0f being the end (thus, you can't assign something more than 1.0f for the first parameter). The second parameter is the actual value that the animation will set on a property. Depending on what type of `KeyFrameAnimation` you created the type of the second parameter will vary. Here we are animating a single float value. The third parameter is optional, but it allows you to specify what type of easing function you'd like. Heare we just want a simple linear function, but you can define other functions like a cubic Bezier curve. 

Next we are going to set how long the animation will run:

{% highlight c# %}
animation.Duration = TimeSpan.FromMilliseconds(3000);
animation.IterationBehavior = AnimationIterationBehavior.Forever;
{% endhighlight %}

Now all that's left is to attach it to the visual:

{% highlight c# %}
visual.StartAnimation(nameof(visual.RotationAngleInDegrees), animation);
visual.CenterPoint = new Vector3(visual.Size.X / 2.0f, visual.Size.Y / 2.0f, 0);
{% endhighlight %}

Notice that starting an animation has two parameters, the first being the name of the property as a `string` and the second being the animation itself.

**UPDATE 04-14-2016: Sometimes you're up late and `string`s can be hard. [@NicoVermerir](https://www.twitter.com/nicovermeir) made a great [suggestion](https://twitter.com/NicoVermeir/status/720864213992755200) to use the `nameof` operator when referencing a visual's property for use in an animation or expression. This is a great approach as it still gets you IntelliSense and is replaced at compile time, meaning that there shouldn't be any difference in performance. To learn more about the `nameof` operator, check out this [MSDN page](https://msdn.microsoft.com/en-us/library/dn986596.aspx). Thanks Nico!**

That last line is just specifying what the center point of our visual will be. The `CenterPoint` property will determine the point at which the visual will rotate around. Since we want this point to be in the center of the visual, we set it to half the visual's width and height. This way the visual will rotate around its center. And that's it! The visual should now be rotating forever! But we can do even better than that. In the next bit we'll be covering two concepts at once: XAML interop and reusing animations.

<h2>XAML Interop</h2>

*XAML interop describes the act of mixing XAML and Composition. Here we'll be taking the same animation we've already built and apply it to a XAML control.*

In our XAML for the page, I'm going to add a `Button`:

{% highlight xml %}
<Grid Background="{ThemeResource ApplicationPageBackgroundThemeBrush}">
    <Button x:Name="AnimatingButton" 
            Content="Hello, World!" 
            HorizontalAlignment="Right" 
            Margin="0, 50, 50, 0" />
</Grid>
{% endhighlight %}

Next is to set up our animation for our new button:

{% highlight c# %}
var buttonVisual = ElementCompositionPreview.GetElementVisual(AnimatingButton);

buttonVisual.CenterPoint = new Vector3(
    (float)AnimatingButton.ActualWidth / 2.0f, 
    (float)AnimatingButton.ActualHeight / 2.0f, 
    0.0f);

buttonVisual.StartAnimation(nameof(buttonVisual.RotationAngleInDegrees), animation);
{% endhighlight %}

One thing to keep in mind is that because we're using the button's `ActualWidth` and `ActualHeight` properties we need to make sure the control is loaded before we try this out, otherwise it will report a size of 0. The easiest way is to move all of the code we've written to `AnimatingButton`'s loaded event handler. Here's how it should look in the end:

<iframe src='https://gfycat.com/ifr/GreedySoupyDiplodocus' frameborder='0' scrolling='no' width='640' height='390.2439024390244' allowfullscreen></iframe>{: .center-image }

And that's it! That's how easy it is to use the Composition API and have it interop with XAML. 

You can find the source code for this project here: {% include icon-github.html username="robmikh" %}/[blog.samples](https://github.com/robmikh/blog.samples)/[2016.04.13/Hello World](https://github.com/robmikh/blog.samples/tree/master/2016.04.13/Hello%20World)

If you'd like to learn more about animations, you can find that [here](https://msdn.microsoft.com/en-us/windows/uwp/graphics/composition-animation).

Tune in next week where we'll be covering how to use images with your visuals and how to apply effects! Make sure you don't miss a thing by subscribing to the [RSS feed](/feed.xml). If you have any questions or feedback, feel free to reach me on [twitter](https://www.twitter.com/robmikh) or by [email](mailto:robmikh@robmikh.com). For Composition questions you're also free to contact the team on [twitter](https://www.twitter.com/WindowsUI) or file an issue on our [GitHub page](https://github.com/Microsoft/WindowsUIDevLabs).