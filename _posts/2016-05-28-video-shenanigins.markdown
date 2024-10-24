---
layout: post
title:  "Video Shenanigins"
date:   2016-05-28 12:00:00 -0700
categories: uwp composition
---

Last time we covered expressions and a little more of XAML interop. This time we will be covering something pretty exciting... video! We'll be going through how play a video using the Windows.Media.Playback API along with Composition. Because we will be displaying the video through Composition, we'll also have some fun with applying what we've learned before about animations and effects. It's important to note that this topic is limited to those on the Anniversary Update for Windows 10, which at the time of writing you can get through the [Windows Insider Program](https://insider.windows.com/).

<!--more-->

Before we start, I do want to give you an update on the post about page transitions. In the interest of properly covering the material, I'll be chopping the topic up into multiple parts and breaking it down to its core concepts. Part of the delay has also been because while I was writing the blog post I also got some inspiration into making a lot of the techniques used in the post into a library. So bare with me on the delay, and I do apologize for that! You should look forward to more regular posts (hopefully not any longer than every other week). 

Back to the fun stuff.

<h2>MediaPlayer</h2>

New to the Anniversary update to Windows 10 is the `MediaPlayer` object in the `Windows.Media.Playback` namespace. I'm personally a huge fan of this addition, as it comes with a lot of goodies that we'll cover in the future. For now, however, we'll just be covering how this class relates to the Composition API. Before we can place a video in a visual, however, we need to be playing a video first!

As always, I've set up a project you can look at [here](https://github.com/robmikh/blog.samples/tree/master/2016.05.28/VideoDemo) to follow along. 

You'll notice that the first thing we'll be doing is creating a local `MediaPlayer` in our page's C# code. Unlike `MediaElement`, `MediaPlayer` is not a XAML control. It is possible to use the `MediaPlayerElement` if you want the new functionality but you also want something that goes into your XAML tree. We'll be focusing solely on `MediaPlayer`. All we have to do is create one and we're already halfway to playing video.

{% highlight c# %}
private MediaPlayer _player;

public MainPage()
{
    this.InitializeComponent();

    _player = new MediaPlayer();
}
{% endhighlight %}

Next is to select the video we'll be playing. You'll notice that I've added some buttons to the page's `CommandBar`, with one of them being an "Open Video" button. In the event handler for that button, we'll allow the user to pick a video of their choice by using a `FileOpenPicker`. First we need to crate our picker and configure it to start at the user's video library and to set the filter to only show video files that we specify.

{% highlight c# %}
var picker = new FileOpenPicker();
picker.SuggestedStartLocation = PickerLocationId.VideosLibrary;
picker.ViewMode = PickerViewMode.Thumbnail;
picker.FileTypeFilter.Add(".mp4");
picker.FileTypeFilter.Add(".avi");
{% endhighlight %}

After that we'll show the picker to the user. If the user successfully selects a video, we'll turn it into a `MediaSource` which we will use to create a `MediaPlaybackItem`. 

{% highlight c# %}
var file = await picker.PickSingleFileAsync();
if (file != null)
{
    var source = MediaSource.CreateFromStorageFile(file);
    PlaySource(source);
}
{% endhighlight %}

Inside of `PlaySource`, we create our `MediaPlaybackItem` and set that to our `MediaPlayer`'s current source. Once we wet the video as the source, we also tell the `MediaPlayer` to begin playing.

{% highlight c# %}
var mediaItem = new MediaPlaybackItem(source);

_player.Source = mediaItem;
_player.Play();
{% endhighlight %}

However, if you were to run the application in its current state, you would be able to hear the audio... but nothing would show up! For that we'll be switching over to the Composition API. To get ready, we'll create a `EnsureComposition` method that will make sure our `Compositor` and `SpriteVisual` are ready.

{% highlight c# %}
private void EnsureComposition()
{
    if (_compositor == null)
    {
        _compositor = ElementCompositionPreview.GetElementVisual(this).Compositor;
        _visual = _compositor.CreateSpriteVisual();

        _visual.Size = new Vector2((float)ActualWidth, (float)ActualHeight);
        SizeChanged += (s, a) =>
        {
            _visual.Size = new Vector2((float)ActualWidth, (float)ActualHeight);
        };

        ElementCompositionPreview.SetElementChildVisual(this, _visual);
    }
}
{% endhighlight %}

<h2>Videos and Swapchains</h2>

As we've covered in the past, `Sprite Visual` is the type of `Visual` that holds content through its `Brush` property. We've also covered that there are 3 main types of brushes: `CompositionColorBrush`, `CompositionSurfaceBrush`, and `CompositionEffectBrush`. Today we'll be interested in the last two. The basic building block we'll be using the most is `CompositionSurfaceBrush` (because once you have that, you can feed that into an effect as we covered [in the past](http://blog.robmikh.com/uwp/composition/2016/04/21/images-and-effects.html)). To create a `CompositionSurfaceBrush` we need to have an object that implements `ICompositionSurface`.

So if we can get a video as a surface, we can display it using the Composition API. At its core, playing a video is flipping a swapchain, which you can put into a surface for use with the Composition API. However, that's an advanced topic that we would need to cover in a future post (and primarily would leverage the C++ media APIs along with DirectX). For now if you want to know how to roll your own that way, you can check out [this overview](https://msdn.microsoft.com/en-us/windows/uwp/graphics/composition-native-interop) of interoping DirectX with the Composition API, and particularly you can check out the [ICompositorInterop](https://msdn.microsoft.com/en-us/library/windows/apps/mt620068.aspx) interface which provides both `CreateCompositionSurfaceForHandle` and `CreateCompositionSurfaceForSwapChain`. At this point I'm sure you're asking: "But surely there is an easier way, right?"

Have no fear! Our `MediaPlayer` comes equipped with a `GetSurface` method, where you can pass in your `Compositor` and get back a surface. We only need to do this once for our `MediaPlayer`, so we'll put this into a method called `EnsureVideoSurface`. To get a surface that represents your video, all you have to do is the following:

{% highlight c# %}
_mediaSurface = _player.GetSurface(_compositor);
var surface = _mediaSurface.CompositionSurface;
{% endhighlight %}

It's that easy! After we pass in our `Compositor` object to `MeidaPlayer`, it will generate a swapchain surface and hand it back to us. From there all we have to do is create a `CompositionSurfaceBrush` and attach it directly to our `SpriteVisual`!

{% highlight c# %}
_videoBrush = _compositor.CreateSurfaceBrush(surface);
_visual.Brush = _videoBrush;
{% endhighlight %}

Now if you run the application you should see the video playing!

![Simple Video Playback](/assets/videodemo1.jpg){: .center-image }

Because this is a regular `SpriteVisual`, you can do anything you want with it! Rotate it, animate it, move it, place an expression.... anything! One of my favorites is to apply an effect to our video.

<h2>Effects and Video</h2>

If you remember effects from our last discussion of the topic, then the rest of this will look really familiar. Essentially none of the steps are different, and all we need to do is supply our brush that contains our video to the effect. Inside of `EnsureComposition` we'll create our effect factory and effect brush:

{% highlight c# %}
var graphicsEffect = new HueRotationEffect
{
    Name = "HueRotation",
    Angle = 0.0f,
    Source = new CompositionEffectSourceParameter("Video")
};

var effectFactory = _compositor.CreateEffectFactory(
    graphicsEffect, 
    new string[] { "HueRotation.Angle" });

var effectBrush = effectFactory.CreateBrush();

_visual.Brush = effectBrush;
{% endhighlight %}

Next, we'll remove the line where we assigned our video brush to our visual and instead set it as a parameter to our effect.

{% highlight c# %}
_videoBrush = _compositor.CreateSurfaceBrush(surface);
//_visual.Brush = _videoBrush;
_effectBrush.SetSourceParameter("Video", _videoBrush);
{% endhighlight %}

Finally we'll add an `AppBarToggleButton` and an event handler for it. If the button is active, we'll animate the HueRotation effect. If it's not active we'll just set the `Angle` to 0 to turn off the effect. We've also disabled the button if the user hasn't loaded a video yet.

{% highlight c# %}
if (EffectButton.IsChecked == true)
{
    var animation = _compositor.CreateScalarKeyFrameAnimation();
    animation.InsertKeyFrame(0.0f, 0.0f);
    animation.InsertKeyFrame(1.0f, 2.0f * (float)Math.PI);
    animation.Duration = TimeSpan.FromMilliseconds(5000);
    animation.IterationBehavior = AnimationIterationBehavior.Forever;

    _effectBrush.StartAnimation("HueRotation.Angle", animation);
}
else
{
    _effectBrush.Properties.InsertScalar("HueRotation.Angle", 0.0f);
}
{% endhighlight %}

Now you should be able to turn on and off the effect at will!

![Video with an effect](/assets/videodemo2.jpg){: .center-image }

If you want to play with more effects on videos, feel free to check out the [Video Playground sample](https://github.com/Microsoft/WindowsUIDevLabs/tree/master/SampleGallery/Samples/SDK%20Insider/VideoPlayground) I wrote on the [WindowsUIDevLabs](https://github.com/Microsoft/WindowsUIDevLabs) GitHub page!

As always, make sure you're [subscribed](http://blog.robmikh.com/feed.xml) so you never miss a post! Additionally let me know ~~on twitter or~~ by [email](mailto:robmikh@robmikh.com) if you have any questions or feedback. Next time we'll be discussing the anatomy of a XAML application from the perspective of the Composition API, which should make XAML/Composition interop easier to understand.