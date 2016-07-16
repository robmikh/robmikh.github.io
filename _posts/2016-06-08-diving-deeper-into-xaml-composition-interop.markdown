---
layout: post
title:  "Diving deeper into XAML-Composition Interop"
date:   2016-06-08 19:15:00 -0700
categories: uwp xaml composition
---

As we continue to explore the Composition API, we’ll be tackling increasingly advanced topics. In order to cover these topics effectively, we’ll need to cover some base concepts in more detail. Additionally, this exploration from time to time will lead us into the world of Composition and XAML interop, so understanding how XAML’s visual tree relates to the Composition visual tree is crucially important. We’ll be covering concepts like drawing order and also cover some neat tricks.

<!--more-->

*For those of you looking for the basics, don’t worry! We’ll continue to cover more fundamental topics as well.*

However, before we dive into how Composition and XAML interact with each other, we’ll be covering some specifics about Composition.

<h2>Draw Order and VisualCollection</h2>

I’ve mentioned the visual tree in the past, but we haven’t fully covered what this structure means for you as an app developer. We have mentioned that children will draw “on top” of their parents, in the sense that they are drawn last. But what about siblings? Who would draw first? You’ll notice that for the majority of the visual tree construction we’ve been doing in the samples on the blog, we’ve only really used the `InsertAtTop` method off of the `Children` property, but there are others as well. 

* **InsertAtTop** inserts the given visual into the tree as the child of the parent visual, and it is placed “on top” of all other siblings. This means that this new child will be drawn last among its siblings and as a result will draw on top of its siblings’ subtrees. 
* **InsertAtBottom** is very similar to `InsertAtTop`, but instead inserts the new visual behind all of its siblings. This way the new visual will draw first among its peers’ subtrees and will be underneath.
* **InsertAbove** takes two arguments, the first being the new visual and the second being the sibling that this new visual will be in front of.
* **InsertBelow** also takes two arguments, and will place the visual behind the given sibling.

It’s important to note that we’re speaking in terms of subtrees. If Visual A is behind Visual B in the Children collection, Visual A will not only draw underneath Visual B but also all of Visual B’s children. This shouldn’t be too out there, as this is also the same way XAML’s visual tree operates with its controls.

However, the importance of the draw/visual order is in how the tree is constructed in a XAML app.

<h2>GetElementVisual and SetElementChildVisual</h2>

So we know that every `UIElement` has a corresponding Visual, meaning that the visuals that represent child controls are also children of the parent visual. But then if you add a visual to a `UIElement` using `SetElementChildVisual`… what does the tree look like? The Visual that is returned by `GetElementVisual` is always the topmost visual representing the control. This means that if you apply an Opacity of 0.8 to a `Button` through its `Visual`, then not only the button will be effected but also any visuals that have been attached to it as well. When you add a visual to a `UIElement` using `SetElementChildVisual`, you always are placed as the topmost sibling relative to the control’s normal content.

For example, if you were to attach a visual with a red square as its content to a button, it would draw on top of the button. This visual would also draw on top of all of the button’s children as well. So if you had an `Image` control as a child (or content) of a `Button`, the red square would draw on top of both controls. 

Keeping this ordering in mind is important, as it will affect the way you will want to build your visual tree. Additionally, you can take advantage of this fact to do some really neat things in your app. We’ll be covering one of my favorite tricks in a moment, but first we’ll take a look at how a XAML app is typically built.

<h2>Anatomy of a XAML app</h2>

Like all UWP (Universal Windows Platform) apps, a XAML app starts with a `CoreWindow`. All UWP apps have a `CoreWindow` of some sort. If you’re a XAML app, XAML keeps a reference to your `CoreWindow` in `Window.Current.CoreWindow`. If you’re a DirectX app, you’ll be given your `CoreWindow` by the system, which you then use to create your swapchain for your current “view”. You can also write a “frameworkless” app much like a DirectX app, but instead you would be using the Composition API without any XAML. We’ll be covering frameworkless Composition apps in the future!

Regardless of the type of app you have, the `CoreWindow` is present and represents your app’s “window”. This holds true even if your app is fullscreen (like on a mobile or IOT device). Additionally, you would set your content root as the `CoreWindow`’s content. XAML handles this for you, but also gives you some control in the form of the `Window.Current.Content` property. If you’ve ever cracked open `App.xaml.cs` from the blank app template, you’ll see that the content of the window is set to a `Frame` control.


{% highlight c# %}
protected override void OnLaunched(LaunchActivatedEventArgs e)
{
    Frame rootFrame = Window.Current.Content as Frame;

    // Do not repeat app initialization when the Window already has content,
    // just ensure that the window is active
    if (rootFrame == null)
    {
        // Create a Frame to act as the navigation context and navigate to the
        // first page
        rootFrame = new Frame();

        rootFrame.NavigationFailed += OnNavigationFailed;

        if (e.PreviousExecutionState == ApplicationExecutionState.Terminated)
        {
            //TODO: Load state from previously suspended application
        }

        // Place the frame in the current Window
        Window.Current.Content = rootFrame;
    }

    if (e.PrelaunchActivated == false)
    {
        if (rootFrame.Content == null)
        {
            // When the navigation stack isn't restored navigate 
            // to the first page, configuring the new page by passing 
            // required information as a navigation parameter
            rootFrame.Navigate(typeof(MainPage), e.Arguments);
        }
        // Ensure the current window is active
        Window.Current.Activate();
    }
}
{% endhighlight %}


A typical app will look something like this:


<iframe src='https://gfycat.com/ifr/FewConcernedGossamerwingedbutterfly' frameborder='0' scrolling='no' width='640' height='359.5505617977528' allowfullscreen></iframe>{: .center-image }


Note that this isn’t always true, and you can set `Window.Current.Content` to just about anything. But for the most part, a frame is used as the “root content” for the `CoreWindow` in a XAML app. As you can see above, the `Frame` stays constant while your page changes. If you’ve ever wanted for content to persist no matter what page you’re on, or even between page changes, this is a very powerful piece of information that you can take advantage of…

<h2>Taking Advantage of the Visual Tree</h2>

Now here’s where the fun begins. We stated before that calling `SetElementChildVisual` will place your visual as the topmost child of the `UIElement`. Luckily for us, Frame is a `UIElement`. This means that we can place content on the frame that will persist no matter what page is being displayed! This works because the page hosted by a `Frame` is considered a child of the `Frame`, meaning that any visual you attach to the `Frame` will be drawn on-top of the page content. You can take advantage of this in a page transition scenario. Let’s say that you have some image that you want to stay on the screen when you change pages, and even animate into its new position on the next page.

To do that you would pull the visual off of the current page and re-parent it to the `Frame`. Then you would tell the `Frame` to navigate to a new page. When the navigation completes, animate the visual to its new location and then pull the visual back off the `Frame` and re-parent it to the new page. That’s all it takes! I’ve created the following visualization to help you understand the flow:


<iframe src='https://gfycat.com/ifr/DevotedBonyAurochs' frameborder='0' scrolling='no' width='640' height='359.5505617977528' allowfullscreen></iframe>{: .center-image }


You’ll notice that because we’re doing some re-parenting in this scenario, you can't directly use the Visual you receive from `GetElementVisual`. You’ll have to use a visual that you’ve created. In the following example (which you can find [here](https://github.com/robmikh/blog.samples/tree/master/2016.06.08/FunWithFrames) on GitHub), we’ll take a rotating visual and take it from one page to another. The first page looks like this:


![Page 1](/assets/page1.jpg){: .center-image }


When you click the button, it will take you to the second page which looks like this:


![Page 2](/assets/page2.jpg){: .center-image }


On the first page we’ll create a simple rotating visual with a solid color as its content and set it as the child of the page:


{% highlight c# %}
private void InitComposition()
{
    var compositor = 
        ElementCompositionPreview.GetElementVisual(this).Compositor;

    var visual = compositor.CreateSpriteVisual();
    visual.Size = new Vector2(75.0f, 75.0f);
    visual.Offset = new Vector3(50.0f, 50.0f, 0.0f);
    visual.CenterPoint = new Vector3(visual.Size / 2.0f, 0.0f);
    visual.Brush = compositor.CreateColorBrush(Windows.UI.Colors.Red);

    var easing = compositor.CreateLinearEasingFunction();
    var animation = compositor.CreateScalarKeyFrameAnimation();
    animation.InsertKeyFrame(0.0f, 0.0f);
    animation.InsertKeyFrame(1.0f, 360.0f, easing);
    animation.IterationBehavior = AnimationIterationBehavior.Forever;
    animation.Duration = TimeSpan.FromMilliseconds(3000);

    visual.StartAnimation(nameof(Visual.RotationAngleInDegrees), animation);

    ElementCompositionPreview.SetElementChildVisual(this, visual);
}
{% endhighlight %}


Next, we’ll modify the event handler for the button. We’ll set the page’s child visual to null, and then set our visual as the child of our `Frame` (in this case we’ll use `Window.Current.Content`, which is our `Frame`). 


{% highlight c# %}
private void Button_Click(object sender, RoutedEventArgs e)
{
    var visual = ElementCompositionPreview.GetElementChildVisual(this);
    ElementCompositionPreview.SetElementChildVisual(this, null);
    ElementCompositionPreview.SetElementChildVisual(
        Window.Current.Content, 
        visual);

    Frame.Navigate(typeof(SecondaryPage));
}
{% endhighlight %}


Finally, on the second page’s `OnNavigatedTo` method, we’ll take the visual away from the `Frame` and re-parent it to our current page. We'll also animate the visual to a new position.


{% highlight c# %}
protected override void OnNavigatedTo(NavigationEventArgs e)
{
    base.OnNavigatedTo(e);

    var visual = 
        ElementCompositionPreview.GetElementChildVisual(
            Window.Current.Content);
    ElementCompositionPreview.SetElementChildVisual(
        Window.Current.Content, 
        null);
    ElementCompositionPreview.SetElementChildVisual(this, visual);

    var compositor = visual.Compositor;
    var animation = compositor.CreateVector3KeyFrameAnimation();
    animation.InsertKeyFrame(0.0f, visual.Offset);
    animation.InsertKeyFrame(
        1.0f, 
        new Vector3(
            (float)Frame.ActualWidth - 150, 
            visual.Offset.Y, 
            visual.Offset.Z));
    animation.Duration = TimeSpan.FromMilliseconds(1500);

    visual.StartAnimation(nameof(Visual.Offset), animation);
}
{% endhighlight %}


I also went back and added something similar for when the button takes you back on the second page back to the first. And that’s it! It should look like this:


<iframe src='https://gfycat.com/ifr/SafePlayfulCatbird' frameborder='0' scrolling='no' width='640' height='481.203007518797' allowfullscreen></iframe>{: .center-image }


You don’t have to only use one `Visual`; you can have entire subtrees that you re-parent! You also can use more complex content. This is very similar to how the `ConnectedAnimationService` works in the Anniversary Update, except that you can do this right now (We’ll be covering how to use this technique for page transitions in a future post)! But that’s not all, remember how we played a video inside of a visual [last time](http://blog.robmikh.com/uwp/composition/2016/05/28/video-shenanigins.html)? Using this technique along with video playback inside a `SpriteVisual` is a great way to implement seamless “picture-in-picture” functionality!

Using this knowledge, you should be able to do very creative things with your UI inside your apps. As always, these simple building blocks are powerful tools to achieve much more complex tasks. To showcase this, we’ll be going through our first case study next time! We’ll be creating a simple video library app! Make sure you’re subscribed to the [RSS feed](http://blog.robmikh.com/feed.xml) so that you never miss a post! Additionally feel free to ask me questions or give me feedback on [twitter](https://twitter.com/robmikh) or by [email](mailto:robmikh@robmikh.com). 
