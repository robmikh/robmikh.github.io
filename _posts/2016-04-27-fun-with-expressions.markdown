---
layout: post
title:  "Fun with Expressions"
date:   2016-04-27 18:15:00 -0700
categories: xaml uwp composition
---

Last week we covered the basics of loading image for use with the Composition API, as well as the effect system. Additionally we covered how easy it is to build on what you already know by applying what we learned about animations and applied an animation to our effect. This week we'll be heading back into the world of XAML to do some interop with the Composition API. Particularly we will be looking at the expression engine.

<!--more-->

<h2>What are expressions?</h2>

Expressions in the Composition API are powered by the Expression Engine within the system compositor. Essentially, expressions are a means to express a property of a `CompositionObject` (the base object of all objects in the Composition API) in terms of other properties from other `CompositionObject`s. What does that mean, exactly? This means you could have a `HueRotationEffect`'s `Angle` property be relative to a given visual's `Offset` property. That would allow you to change the color of an image perfectly in sync with a visual moving around in a circle.

Generally, expressions are at their best when used in tandem with either other animations or expressions, or together with user input. For example, you could subtly move an image in relation to a user scrolling on the page. The best part is that because expressions are computed in the system compositor, you never have to worry about your UI thread to make these high performance changes. After all, the last thing you would want is for your change to the UI to be out of sync or lag behind the user's input, breaking the illusion.

<h2>The basics of an expression</h2>

Expressions share a lot in common with animations in that if you can use an animation on a property you should be able to use an expression as well. Expressions and animations are also applied to a `CompositionObject` using the same function:

{% highlight c# %}
visual.StartAnimation(nameof(visual.Offset), expression);
{% endhighlight %}

You've also probably figured out that expressions have similar characteristics to animations, in that attaching a new expression or animation to the same property will stop the old animation/expression. Additionally, if you manually set the property it will halt the current animation or expression that is assigned to that property. You can also stop an expression or animation with a function call:

{% highlight c# %}
visual.StopAnimation("Offset");
{% endhighlight %}

Let's say that we wanted a visual (Visual A) to have the same offset as another visual (Visual B). We can set up an expression that will automatically update for us when we change the offset of Visual B either through setting the `Offset` property or through an animation. That expression would look something like this:


{% highlight c# %}
var expression = compositor.CreateExpressionAnimation();
expression.Expression = "visual.Offset";
expression.SetReferenceProperty("visual", visualB);

visualA.StartAnimation(nameof(visualA.Offset), expression);
{% endhighlight %}


Now no matter what we do, Visual A will have the same offset as Visual B. You can use whatever name you want for reference parameters, but you have to "register" them with the expression by calling `SetReferenceProperty`. You can also name and use objects or types that do not inherit from `CompositionObject` by using `Set*Parameter` (for example, `SetScalarParameter`). However, a type that does not inherit from `CompositionObject` won't automatically be updated, as the system compositor doesn't know the value has changed. We'll cover how to get around this later in the post.

However, you can do more than just set one visual's offset to be the same as another's. Let's say that Visual B will animate from (0, 0, 0) to (300, 0, 0), and we want Visual A to move in the opposite direction relative to Visual B. All we have to do is tweak the expression:


{% highlight c# %}
expression.Expression = "Vector3(300, 0, 0) - visual.Offset";
{% endhighlight %}


Now whenever Visual B's offset changes the compositor will subtract it from the offset (300, 0, 0), which should place Visual A opposite of Visual B. 

<iframe src='https://gfycat.com/ifr/CookedPassionateArchaeopteryx' frameborder='0' scrolling='no' width='640' height='359.5505617977528' allowfullscreen></iframe>{: .center-image }

You might of noticed that I defined an consumed a `Vector3` from within the expression. The expression engine gives you access to many helper functions in order to properly do your calculations. You can find a complete list of these functions [here](https://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.composition.expressionanimation.aspx).

Even with these simple tools you can do a lot with the expression system in the Composition API. However, there's a concept that should allow you to do a lot more...

<h1>Property sets</h1>

You may have noticed that all objects that derive from `CompositionObject` have a `Properties` property, which is of type `CompositionPropertySet`. Using a property set will allow you to set properties for that `CompositionObject`, just the same as the normal properties the derived class may expose. This is also the means by which you change adjustable properties in a `CompositionEffectBrush`. More importantly for this week, however, is that property sets also allow us to leverage the expression engine and the system compositor even in cases where we wish to define something in terms of an object or type that is not a `CompositionObject`. 

Creating a `CompositionPropertySet` is pretty straight forward, like with most objects in the Composition API, we use the `Compositor` to create one:

{% highlight c# %}
var propertySet = compositor.CreatePropertySet();
{% endhighlight %}

After creating a `CompositionPropertySet`, you can manipulate its values by calling the appropriate `Insert*` function. If the named value does not already exist, it will create it. If it does exist it will change the value to the new value.

{% highlight c# %}
propertySet.InsertScalar("SomeMadeUpName", 0.5f);
{% endhighlight %}

When you want to reference a value from a property set in an expression, it's the same as any other `CompositionObject`:

{% highlight c# %}
var expression = compositor.CreateExpressionAnimation();
expression.Expression = "propertySet.SomeMadeUpName";
expression.SetReferenceProperty("propertySet", propertySet);

visual.StartAnimation(nameof(visual.Opacity), expression);
{% endhighlight %}

By using a property set, the system compositor knows when we change a value, which allows the expression to stay up to date. This way you can use the same value in a property set in multiple expressions and only have to update a value once in one place, instead of updating every expression individually. 

While in these examples we created a `CompositionPropertySet`, there are ways to retrieve them from certain XAML elements. Next stop, XAML interop with Composition expressions!

<h2>Using a ScrollViewer as an input to an expression</h2>

As of Windows 10 build 10586, the first place where the ability to retrieve a `CompositoinPropertySet` from a `UIElement` is up and running is for a `ScrollViewer`. While that sounds like just one place, keep in mind that `ListView`, `GridView`, and more all have `ScrollViewer` somewhere inside them. If you're able to get a hold of it, you'll be able to retrieve a `CompositionPropertySet`.

Just like the last time we looked at using XAML and the Composition API together, we'll be using the helpers found in `ElementCompositionPreview`. In this scenario we'll be using `GetScrollViewerManipulationPropertySet`. You'll get back a `CompositionPropertySet` which has the following values:

* Translation (Vector3)
* CenterPoint (Vector3)
* Scale (Vector3)
* Matrix (Matrix4x4)


This week we'll be focusing on the `Translation` property, which tells us how far the `ScrollViewer` is scrolled in a given direction. Now it's time to put these new skills to use.

<h2>Adding parallax to a ListView</h2>

*If you'd like to follow along, you can find the finished project [here](https://github.com/robmikh/blog.samples/tree/master/2016.04.27/ExpressionsDemo).*

We'll be doing things a little differently this week to better highlight not only the capabilities of expressions and XAML interop with the Composition API, but also to highlight how it is pretty easy to add these techniques to existing applications. I have a small demo application, which we'll be adding some parallax effects to the `ListView`. Currently the application looks and behaves like this:

<iframe src='https://gfycat.com/ifr/FaithfulTightDodo' frameborder='0' scrolling='no' width='320' height='581.818181818' allowfullscreen></iframe>{: .center-image }

As you can tell, the experience is kind of static. By adding a splash a motion, we'll be able to create a better user experience without much of a hit to performance or added complexity. Currently our XAML is laid out like this:

![App Layout](/assets/applayout.JPG){: .center-image }

What we'll be doing is having the image in the background move a little bit when the user scrolls, making a cool parallax effect. We'll also do something about that text that is used as the page's header. Because we need to get the `ScrollViewer` from a `ListView`, we'll need to use some common extensions that take advantage of the `VisualTreeHelper` class. I won't spend time covering them here, as they're pretty well known, but if you'd like to see the code for the extensions we'll be using for this project you can find that [here](https://github.com/robmikh/blog.samples/blob/master/2016.04.27/ExpressionsDemo/VisualTreeExtensions.cs).

Additionally, since we want to get the `ScrollViewer` from a `ListView`, we have to make sure that the `ListView` is done loading. Thus we'll be doing all the work to set this up in the `ListView`'s `Loaded` event handler. Our first step will be to get a `Compositor` and get the `CompositionPropertySet` that corresponds to our `ListView`:

{% highlight c# %}
var compositor = ElementCompositionPreview.GetElementVisual(this).Compositor;
var scrollViewer = MainListView.GetChildOfType<ScrollViewer>();

var scrollerPropertySet = ElementCompositionPreview.
    GetScrollViewerManipulationPropertySet(scrollViewer);
{% endhighlight %}

Next we want to create our expression, which will define our background image's offset in terms of the `ScrollViewer`'s `Translation.Y` property. However, in order to achieve the parallax effect we need to ensure that the content in the `ListView` moves at a different speed (preferably faster) than the background. Since we clearly can't make the content move faster (as it is controlled by the user's finger or scroll wheel), we'll have to make the background move slower. To do that we'll just multiply the offset with a value that is less than 1.

{% highlight c# %}
var expression = compositor.CreateExpressionAnimation();

expression.Expression = "scroller.Translation.Y * parallaxFactor";
expression.SetScalarParameter("parallaxFactor", 0.3f);
expression.SetReferenceParameter("scroller", scrollerPropertySet);
{% endhighlight %}

Now that we have our expression set up, we need to apply it to our background image.

{% highlight c# %}
var backgroundVisual = ElementCompositionPreview.GetElementVisual(BackgroundImage);
backgroundVisual.StartAnimation("Offset.Y", expression);
{% endhighlight %}

However, we have a slight problem! If you take the app for a spin now, you'll notice that if you scroll to far down the background image will completely disappear! This makes the other elements unreadable, as well as making for a confusing experience to the user. To get around this we'll use the `Clamp` helper function to make sure the offset doesn't go bellow -608 (the image has a height of 658, so this leaves us with at least 50 on the screen at all times). We'll set the upper bound to 999, just so that the bouncing mechanic still works. Realistically this shouldn't go beyond 100 or so, but we'll use 999 just to make it clear that this number isn't really needed and arbitrary.


{% highlight c# %}
expression.Expression = "Clamp(scroller.Translation.Y * parallaxFactor, -608, 999)";
{% endhighlight %}

Now if you try the app it should make a nice banner with a height of 50, even when you scroll all the way to the bottom. The next item on the agenda is then to change the text. We'll have the text start out halfway down the image and then move up to its spot as the header with a similar parallax effect. We first need to set up our expression:

{% highlight c# %}
var textVisual = ElementCompositionPreview.GetElementVisual(HeaderTextBlock);
String progress = "Clamp(visual.Offset.Y / -100.0, 0.0, 1.0)";

var textExpression = compositor.CreateExpressionAnimation()
textExpression = "Lerp(Vector3(0, 200, 0), Vector3(0, 0, 0), " + progress +")";
textExpression.SetReferenceParameter("visual", backgroundVisual);

textVisual.StartAnimation("Offset", textExpression);
{% endhighlight %}

First we take the visual that represents our `TextBlock`, and we define a `string` that we'll use later. This is just to demonstrate that the `Expression` property is really just a `string`, so you can use your favorite string building techniques with expressions. After that we create the expression, which we use to express our page header's offset in terms of the background image. We use the `Lerp` function to interpolate between a starting value and an ending value. The progress string we defined earlier will spit out a value between 0 and 1 that represents how far the background has moved. Note that we don't check this against the entire height of the image, but rather just part of it. After that we attach the expression to our `TextBlock` and we're done!

The finished product should look like this:

<iframe src='https://gfycat.com/ifr/SharpYearlyCowbird' frameborder='0' scrolling='no' width='320' height='581.818181818' allowfullscreen></iframe>{: .center-image }

For homework, you should to try replace the background image with a `SpriteVisual` that has an effect. Then try to tie in one of your effect's properties with the `Translation.Y` property from the `ScrollViewer` using an effect! Applying a saturation effect or a hue rotation should make for a pretty interesting project to get used to using expressions. If you really want to get fancy and are on a Windows Insider build (or your reading this when the Anniversary Update has been released), you could tie the `Translation.Y` property to a blur effect! We'll cover blur in a later post once the Anniversary Update has released (although I may change my mind and we could cover it sooner).

As always, make sure you're [subscribed](http://blog.robmikh.com/feed.xml) so you never miss a post! Additionally let me know on [twitter](https://twitter.com/robmikh) or by [email](mailto:robmikh@robmikh.com) if you have any questions or feedback. Next week we will be covering some cool page transitions using the Composition API along with XAML!
