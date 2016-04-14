---
layout: post
title:  "Introduction to Composition"
date:   2016-04-13 19:48:00 -0700
categories: xaml uwp composition
---

First off, welcome to the Composition API! If you've watch some of the [//build talks](https://channel9.msdn.com/Events/Build/2016/B818) the past two years you may have seen some members of our team show off the new API. While the API is certainly new for Windows 10, the Composition team has actually been around for awhile...

<h2>History Lesson</h2>

Since Windows Vista, the DWM has been powering the Windows experience. Everything you see on the screen we almost always have some hand in drawing. Developers were able to (and still are!) leverage the system compositor using [Direct Composition](https://msdn.microsoft.com/en-us/library/windows/desktop/hh437371%28v=vs.85%29.aspx). DComp, as we call it, was a way to position elements (that we call Visuals) on the screen in a tree like structure. For any of you that know the concept of the Visual Tree in XAML, this should sound familiar. The API was C++ only, and when Windows 8 came around there wasn't a way to use the API in the new modern app world. This is where the Windows.UI.Composition API came into existence with Windows 10, as a way to leverage the DWM to make fast and beautiful user interfaces without breaking the perf bank.

For a more in depth look at Composition as a whole, our friend Kenny Kerr wrote a great article that you can check out [here](https://msdn.microsoft.com/en-us/magazine/mt590968).

<h2>Getting Started</h2>

While it is possible to create a "pure" Composition app, many of you will be delighted to know that you can actually consume the API from within XAML! This is a way to manipulate your pre-existing applications to add rich new experiences without much code. The very first thing you'll want to do is to grab the `Visual` that represents a `UIElement`. For many of you, this method will become your best friend:

{% highlight c# %}
var visual = ElementCompositionPreview.GetElementVisual(this);
{% endhighlight %}

<h1>What exactly is a visual?</h1>

A Visual is a node in the Visual Tree, which means they can be the children of another Visual. However, there are different types of Visuals. Currently, the two big types are `ContainerVisual` and  `SpriteVisuals` (in fact, the latter inherits from the former). A `ContainerVisual` can have children Visuals, while a `SpriteVisual` can have children Visuals *and* content. For the most part, expect to be working with `SpriteVisual`. Just like with XAML's Visual Tree, Visuals are always relatively positioned by their parent. This also means that anything you attach to a `UIElement` will be placed *on top* of it.

Comming back to the world of XAML, `ElementCompositionPreview.GetElementVisual` retreives the Visual that represents a given `UIElement`. Any `UIElement`, which even includes the Frame (this makes for some pretty neat animation tricks and transitions that we'll cover in a later post)! Some things you should be aware of when getting a `Visual` from XAML:

* GetElementVisual returns a `Visual`, so you can't assign content to it. You are however, free to add a child `SpriteVisual` that has content (or event an entire subtree by attaching a `ContainerVisual` with many children!) by using the `ElementCompositionPreview.SetElementChildVisual` method.
* Using the Composition API requires the November 2015 update to Windows 10, otherwise known as build 10586.
* Visuals returned by `ElementCompositionPreview.GetElementVisual` cannot be reparented, as they are already in the tree and you are not allowed to remove them from their XAML element. However, you are free to attach and remove any Visuals you create.

<h2>Baby's First Visual</h2>

The easiest thing to do is to create a `SpriteVisual` with a solid color, so we'll start with that. After creating a blank XAML app, go to MainPage.xaml.cs and try something like this:

{% highlight c# %}
var compositor = ElementCompositionPreview.GetElementVisual(this).Compositor;
var visual = compositor.CreateSpriteVisual();

visual.Size = new Vector2(150, 150);
visual.Offset = new Vector3(50, 50, 0);
visual.Brush = compositor.CreateColorBrush(Colors.Red);

ElementCompositionPreview.SetElementChildVisual(this, visual);
{% endhighlight %}

What does this do? It first gets the `Compositor` object from the Page's `Visual`. The `Compositor` object is the second most important conecpt in the Composition API. Nearly every object related to the Composition API is created through this object. Usually your first plan of attack should be to get ahold of this object. Luckly for you, nearly every Composition object has a reference back to the `Compositor` that created it. Within your XAML application, there will only be one `Compositor` so it doesn't matter where you get it from.

After we've retreived the `Compositor` we are able to create a `SpriteVisual`. After that we set the visual's size and give it an Offset (notice that you'll need to be using the System.Numerics namespace for this). Finally we create a brush that just contains the color red and asign it to the visual. The only thing left to do is to attach it into the tree by using `SetElementChildVisual`. Just like with XAML, if it's not in the tree it won't draw.

All of that should get you this:

![Hello World!](/assets/helloworld1.jpg)

So far, so good. But I think we can do better. One of the strengths of the Composition API are the animations. Let's try one of those next:

<h2>Fun with animations</h2>

One of the animations available to us is the `KeyFrameAnimation`, which we can create by using the `Compositor`:

{% highlight c# %}
var animation = compositor.CreateScalarKeyFrameAnimation();
{% endhighlight %}

Next we'll want to add some key frames:

{% highlight c# %}
var easing = compositor.CreateLinearEasingFunction();

animation.InsertKeyFrame(0.0f, 0.0f, easing);
animation.InsertKeyFrame(1.0f, 360.0f, easing);
{% endhighlight %}

The first paramter signifies a normalized time for that key frame, 0.0f being the begining and 1.0f being the end. The second paramter is the actual value that the animation will set on a property. Depending on what type of `KeyFrameAnimation` you created, there will be different accepted types for the second paramter. Here we are animating a single float value. The third paramter is optional, but it allows you to specify what type of easing function you'd like. Heare we just want a simple linear function, but you can define other functions like a cubic Bezier curve. 

Next we are going to set how long the animation will run:

{% highlight c# %}
animation.Duration = TimeSpan.FromMilliseconds(3000);
animation.IterationBehavior = AnimationIterationBehavior.Forever;
{% endhighlight %}

Now all that's left is to attach it to the visual:

{% highlight c# %}
visual.StartAnimation("RotationAngleInDegrees", animation);
visual.CenterPoint = new Vector3(75, 75, 0);
{% endhighlight %}

That last line is just specifying what the center point of our visual will be. This way the visual will rotate around its center. And that's it! The visual should now be rotating forever! But we can do even better than that. In the next bit we'll be covering two concepts at once: XAML interop and resuing animations.

<h2>XAML Interop</h2>

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

buttonVisual.StartAnimation("RotationAngleInDegrees", animation);
{% endhighlight %}

One thing to keep in mind is that because we're using the button's `ActualWidth` and `ActualHeight` properties we need to make sure the control is loaded before we try this out, otherwise it will report a size of 0. The easiest way is to move all of the code we've written to `AnimatingButton`'s loaded event handler. Here's how it should look in the end:

![Hello World, Part Two!](/assets/helloworld2.gif)

And that's it! That's how easy it is to use the Composition API and have it interop with XAML. 

You can find the source code for this project here: {% include icon-github.html username="robmikh" %}/[blog.samples](https://github.com/robmikh/blog.samples)/[2016.04.13/Hello World](https://github.com/robmikh/blog.samples/tree/master/2016.04.13/Hello%20World)

If you'd like to learn more about animations, you can find that [here](https://msdn.microsoft.com/en-us/windows/uwp/graphics/composition-animation).