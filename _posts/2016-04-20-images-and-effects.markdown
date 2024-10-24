---
layout: post
title:  "Images and Effects"
date:   2016-04-20 19:15:00 -0700
categories: uwp composition
---

Last week we covered the basics of the Composition API along with the basics of having composition interop with XAML, particularly with animations. This week we're going to deal more with the Composition side of the equation, and as promised we will be covering the use of images and the Composition Effect system.

<!--more-->

However, I'd first like to do some housekeeping. Last week I hinted at the performance benefits of the Composition API, but I wanted to touch more on the subject, especially after getting some questions about it. You may also wonder what is the benefit of using something like the Composition API vs XAML or even DirectX. Or in other words...

<h2>Why even use Composition?</h2>

While it's true that you can achieve many of the same things we do using DirectX directly (especially since at the end of the day that's what we use to draw!), and that XAML has its own Storyboard animations, not all of it is a direct comparison. When you use DirectX or XAML you do this inside your application's process. If your application is hung or is completing some big chunk of work on the UI thread, your UI will update slowly or not at all. 

You can make sure you avoid these disruptions by structuring your application differently, but it's not entire avoidable. If you make some heavy dependent animations in XAML, XAML has no choice but to do all of these on the UI thread since the animations influence the layout of your elements. Other times what you want to achieve is just too complex in just XAML alone, or you don't want to bring out the big hammer of DirectX just to display a couple of images.  

One of the largest strengths of the Composition API is the fact that we do the work outside of the application process. If you remember our discussion about the system compositor, the DWM, last week this should start to make some sense. The Composition API is a way to describe what you want your application to look like to the DWM, and there we can draw it in an efficient and performant manner. Because we're responsible for drawing everything, we can update your animations at 60Hz even if your UI thread is blocked! The Composition APIs can also provide powerful tools that you can't get from XAML directly without the need to drop down to DirectX and setting up an entire pipeline.

However, you should consider the Composition API as a tool in the same way XAML and DirectX are. While the Composition API is great, there are some problems you'll want to solve using other tools. For example, while we're working hard to support more and more effects, there are some effects that we still don't support through the Composition API (we'll discuss this bellow in the effects section) that you could achieve through Win2D. Or if you're building some 3D simulation or game, DirectX is the obvious choice.  In other scenarios you might want to leverage the layout capabilities of XAML, so doing something through XAML would make more sense. 

Do know that we're working hard to make sure that the decision of what tool to use gets easier and easier as we work with other teams, but you should always consider all your options.

Now we can get back to our regularly scheduled programming.

<h2>Image Loading</h2>

Last week we used the `CompositionColorBrush` to create a red rectangle, and that's a good starting point for our introduction into Surface and Brush objects. As we discussed last time, `SpriteVisual`s are capable of having content assigned to them, particularly through their Brush property. In order to assign the color red as the content of our visual, we created a `CompositionBrush`, particularly a `CompositionColorBrush`, and assigned it to our visual. However, there are also other types of brushes, namely `CompositionSurfaceBrush` and `CompositionEffectBrush`. A surface, or rather a `ICompositionSurface`, is some kind of content that you can assign to a brush. This can be a normal image or even a swap chain. For today we'll just be covering normal images.

To populate a surface, we use a `CompositionDrawingSurface`. You can always drop down to DirectX and C++ to draw your image as shown [here](https://msdn.microsoft.com/en-us/windows/uwp/graphics/composition-native-interop). However, we also worked with the Win2D team, and you can use Win2D to draw into a `CompositionDrawingSurface`. Although I've taken it another step further and created a library you can download through nuget to manage all your image loading needs. It's built with C# and utilizes Win2D to create `CompositionDrawingSurfaces`, and you can download it [here](https://www.nuget.org/packages/Robmikh.Util.CompositionImageLoader) and check out the project on its [GitHub page](https://github.com/robmikh/compositionimageloader). 

In the next section we're going to discuss what it would take to do the image loading and drawing all by yourself using Win2D, so if you want you can skip straight to the example using CompsoitionImageLoader. However, there is one concept you should be aware of, and that's the concept of a lost graphics device. 

<h1>Lost Devices</h1>

When a device is lost, you traditionally need to recreate your graphics device and all of the resources you created with that device. XAML usually handles this for you, but you need to be aware of it when using the Composition APIs. 

When can a device be lost? It could be anything from a graphics driver update, to a device that is switching to a different GPU (like with the Surface Book or a laptop disconnecting from something like a Razer Core), or a set of incorrect calls to the GPU that puts it in a bad state and it needs to reset (we call this [Timeout Detection and Recovery](https://msdn.microsoft.com/en-us/library/windows/hardware/ff570087(v=vs.85).aspx), otherwise known as TDR). To learn more about lost devices, check out this [MSDN article](https://msdn.microsoft.com/en-us/library/windows/desktop/bb324479(v=vs.85).aspx).

While it is not strictly required to handle a lost device, it is greatly recommended that your application is resilient to it. Otherwise when a user detaches their Surface Book from its base with a GPU, your application might lose all of its surfaces! So what do you have to do? If you use CompositionImageLoader you should be set. Just know that if you use `LoadImageFromUri` or `LoadImageFromUriAsync`, you're not protected from a lost graphics device. The `ImageLoader` does have a `DeviceReplacedEvent` event you can listen to and redraw your surfaces if you want. However, there's an even easier way, and that's by using `ManagedSurface`s. If you use `CreateManagedSurface` or `CreateManagedSurfaceAsync` and keep the object alive, you won't have to worry about lost graphics devices (same goes for methods that create a `TextSurface`). `ManagedSurface` is the recommended way to go.

One last thing about the `ImageLoader`, because it creates a graphics device you should try to only keep one around. Creating many of them can become expensive and waste memory. Store it somewhere where you can have good access to it and reuse it. That's when the `ImageLoader` can do its best. The `ImageLoader` makes all of its drawing calls asynchronously (even with the non async methods, the surface creation is synchronous but the `ImageLoader` draws to it later on) and under a lock, so you shouldn't have to worry too much about threading. 

<h1>Drawing with Win2D</h1>

The first step is to setup your graphics device for use with the Composition API. We'll use CompositionImageLoader as an example of how to set up a device. All you need is a compositor and a Win2D canvas device (using the shared device is fine):

{% highlight c# %}
private void CreateDevice()
{
    if (_compositor != null)
    {
        if (_canvasDevice == null)
        {
            _canvasDevice = CanvasDevice.GetSharedDevice();
            _canvasDevice.DeviceLost += DeviceLost;
        }

        if (_graphicsDevice == null)
        {
            _graphicsDevice = CanvasComposition.CreateCompositionGraphicsDevice(_compositor, _canvasDevice);
            _graphicsDevice.RenderingDeviceReplaced += RenderingDeviceReplaced;
        }
    }
}
{% endhighlight %}

Here you can see the `ImageLoader` gets the shared canvas device and subscribes to its `DeviceLost` event. After that it sets up a `CompositionGraphicsDevice` with the aid of the `CanvasComposition` helper. `CanvasComposition` should be your best friend if you're working with both the Composition API and Win2D. Here, `CanvasComposition` has a helper function that creates a `CompositionGraphicsDevice` given a `Compositor` and a `CanvasDevice`. Here's what we do when the `CanvasDevice` raises its `DeviceLost` event:

{% highlight c# %}
private void DeviceLost(CanvasDevice sender, object args)
{
    Debug.WriteLine("CompositionImageLoader - Canvas Device Lost");
    sender.DeviceLost -= DeviceLost;

    _canvasDevice = CanvasDevice.GetSharedDevice();
    _canvasDevice.DeviceLost += DeviceLost;

    CanvasComposition.SetCanvasDevice(_graphicsDevice, _canvasDevice);
}
{% endhighlight %}

When you use `CanvasComposition` and call `SetCanvasDevice` to replace the underlying device for a `CompositionGraphicsDevice`, that `CompositionGraphicsDevice` will raise its `RenderingDeviceReplaced` event. It is on this event that you should redraw/recreate any of your resources created by that device.

After that, drawing with Win2D is pretty easy, it essentially boils down to the following steps:

1. Create a `CompositionDrawingSurface`
2. Resize the surface if necessary
3. Draw into the surface

You can see this demonstrated below:

{% highlight c# %}
var surface = _graphicsDevice.CreateDrawingSurface(
    new Size(0, 0), 
	DirectXPixelFormat.B8G8R8A8UIntNormalized, 
	DirectXAlphaMode.Premultiplied);

using (var canvasBitmap = await CanvasBitmap.LoadAsync(_canvasDevice, uri))
{
    CanvasComposition.Resize(surface, canvasBitmap.Size);
	
    using (var session = CanvasComposition.CreateDrawingSession(surface))
    {
        session.Clear(Color.FromArgb(0, 0, 0, 0));
        Rect rect = new Rect(0, 0, canvasBitmap.Size.Width, canvasBitmap.Size.Height);
        // Since our surface is the same size as our image,
        // the source rect and destination rect are the same.
        session.DrawImage(canvasbitmap, rect, rect);
    }
}
{% endhighlight %}

First we create a new `CompositionDrawingSurface`, and we use the `B8G8R8A8UIntNormalized` pixel format along with a premultiplied alpha. Notice that we specify a size of (0, 0), this is mainly because we don't know how big our surface is going to be, so we'll wait until we know how big it is and then resize the surface. Next we create a `CanvasBitmap` and use a Uri to specify where we want to load our image from. After that we resize our surface and call `CreateDrawingSession`. 

Now we can draw whatever we want to the surface using any of your favorite Win2D calls. And that's it! It is important to note that drawing to your surface is done in your process, but once we have the bits we just reuse them. 

<h1>CompositionImageLoader</h1>

*You can follow along [here](https://github.com/robmikh/blog.samples/tree/master/2016.04.19/ImageLoading).*

While Win2D is pretty easy, using the CompositionImageLoader is even easier:

{% highlight c# %}
var managedSurface = await imageLoader.CreateManagedSurfaceFromUriAsync(
    new Uri("ms-appx:///Assets/tripphoto1.jpg"));
{% endhighlight %}

As we noted before, as long as we keep the `ManagedSurface` alive (and of course, the `ImageLoader` itself as well) we don't have to worry about lost graphics devices. After you have your surface, regardless if you drew to it using Win2D or had the `ImageLoader` do it, the next step is to create a `CompositionSurfaceBrush` and assign it as the `Brush` property on a visual.

{% highlight c# %}
var brush = compositor.CreateSurfaceBrush(managedSurface.Surface);

visual.Brush = brush;
{% endhighlight %}

That should look like this:

![Image Loading](/assets/imageloading1.jpg){: .center-image }

Something you should know is that you can assign the same brush to multiple visuals. This also applies to using that brush as the input to an effect, as we'll discuss bellow. 

If you'd like to see this project, you can find it [here](https://github.com/robmikh/blog.samples/tree/master/2016.04.19/ImageLoading).

Those of you poking around in the package will notice that ImageLoader can actually measure and draw text for you as well. Go ahead and check it out! Let me know if you find any issues or have any questions over on the project's [GitHub page](https://github.com/robmikh/compositionimageloader).

<h2>Composition Effects</h2>

*The final version is available [here](https://github.com/robmikh/blog.samples/tree/master/2016.04.19/EffectsDemo).*

Loading an image is great and all, and a log of the time it might be all you need to do, but sometimes you want to add something to it or make a change to your image without creating many different version of the same asset. Or if we want to get even fancier, animate an effect on an image live. This is where effects come into play. 

These are the basic steps for using effects (we'll be going through each step individually):

1. Create an effect definition.
2. Create a `CompositionEffectFactory` (and thus compile the effect).
3. Create a new `CompositionEffectBrush` using your new effect factory.
4. Assign the brush to the visual you desire.

<h1>Creating an effect definition</h1>

The Composition API borrows its effect definitions from the Win2D library, however not all Win2D effects are supported. You can find a complete list, along with what is and is not supported by the Composition API, [here](http://microsoft.github.io/Win2D/html/N_Microsoft_Graphics_Canvas_Effects.htm). While not everything is supported by the Composition API, there's still a lot to choose from (and we're always trying to add more support). For starters we'll be using an invert effect:

{% highlight c# %}
IGraphicsEffect graphicsEffect = new InvertEffect
{
    Source = new CompositionEffectSourceParameter("image")
};
{% endhighlight %}

We first name the effect and then define the Source property, this is what we'll want to invert. Here we use the `CompositionEffectSourceParamter` and call the parameter 'image'. What this means is that once we compile the effect, we can create the resulting factory to create many effect brushes that all use the same effect definition. However, each effect brush can take in a different `CompositionSurfaceBrush` as a source (which we'll cover in another section). 

<h1>Create a CompositionEffectFactory</h1>

After you create an effect definition, pass it into the `Compositor` and create a `CompositionEffectFactory`:

{% highlight c# %}
_effectFactory = _compositor.CreateEffectFactory(graphicsEffect);
{% endhighlight %}

What this does is take your definition and compile it into a shader that we'll use in the DWM. Because of this, you can chain many effect definitions into each other, but once you compile an effect brush you can feed it into another effect. We'll cover chaining in a bit, but keep that in mind.

It's usually a good idea to keep around your effect factory, as it can create many effect brushes that follow the same definition. 

<h1>Create a CompositionEffectBrush</h1>

Now that we have our shiny new effect factory, we'll put it to use to create a `CompositionEffectBrush`.

{% highlight c# %}
var effectBrush = _effectFactory.CreateBrush();
{% endhighlight %}

However, we need to assign a `CompositionSurfaceBrush` to the 'image' parameter we specified when we created the definition. All you have to do is call SetSourceParameter on the effect brush:

{% highlight c# %}
effectBrush.SetSourceParameter("image", imageBrush);
{% endhighlight %}

Now this effect brush will invert our image! But in order to see that in action, we need to assign the brush to our visual.

<h1>Assign an effect brush to a SpriteVisual</h1>

Assigning a `CompositionEffectBrush` is just as easy as assigning any other brush. Just set it to the `SpriteVisual`'s `Brush` property like so:

{% highlight c# %}
visual.Brush = effectBrush;
{% endhighlight %}

That should look something like this:

![Effects Demo 1](/assets/effectsdemo1.jpg){: .center-image }

As you can see, applying an effect to an image is pretty easy. However, sometimes you want to apply multiple effects at once to an image. In order to do that you would chain the effect definitions. Let's try that next.

<h1>Chaining Effects</h1>

Chaining effects is pretty easy. Just set the source of one effect to be another effect! Let's apply a circle mask to our inverted image. To do that we change the `Source` parameter of our `InvertEffect` to a `CompositeEffect`.

{% highlight c# %}
IGraphicsEffect graphicsEffect = new InvertEffect
{
    Source = new CompositeEffect
    {
        Mode = Microsoft.Graphics.Canvas.CanvasComposite.DestinationIn,
        Sources =
        {
            new CompositionEffectSourceParameter("image"),
            new CompositionEffectSourceParameter("mask")
        }
    }
};

// --- Code from earlier ----

var managedMaskSurface = await _imageLoader.CreateManagedSurfaceFromUriAsync(
    new Uri("ms-appx:///Assets/CircleMask.png"));
var maskBrush = _compositor.CreateSurfaceBrush(managedMaskSurface.Surface);

effectBrush.SetSourceParameter("mask", maskBrush);
{% endhighlight %}

Our mask image is a transparent png with a while circle in the middle. We want our image to be masked so that we only see a circle cutout of the original image. We'll use the `DestinationIn` mode to basically take the overlapping section of both images when they're put on top of each other. You can read more about the composite effect [here](https://msdn.microsoft.com/en-us/library/windows/desktop/hh706320.aspx). Then we set two `CompositionEffectSourceParamters` as our sources to the composite effect. 

We then load the circle image and feed the `CompositionSurfaceBrush` into our effect brush. Now our output should look like this:

![Effects Demo 2](/assets/effectsdemo2.jpg){: .center-image }

Things are shaping up nicely! But I still think we can do better. Last week we finished with an animation, and we'll do so again here. Animating a property of an effect is just as easy as animating a property on a visual! To demonstrate this, we'll chain a `HueRotationEffect` and then animate its `Angle` property.

<h1>Animating Effects</h1>

Before you animate an effect, there is some set up you have to do. First you have to make sure that you declare the a property as animatable when creating your effect factory. You also have to give a name to your effect so that you can reference it when setting up the animation. 

Here, we'll first update our effect definition to include a `HueRotationEffect` and make sure we assign something to the `Name` property.

{% highlight c# %}
IGraphicsEffect graphicsEffect = new HueRotationEffect
{
    Name = "hueEffect",
    Angle = 0.0f,
    Source = new InvertEffect
    {
        Source = new CompositeEffect
        {
            Mode = Microsoft.Graphics.Canvas.CanvasComposite.DestinationIn,
            Sources =
            {
                new CompositionEffectSourceParameter("image"),
                new CompositionEffectSourceParameter("mask")
            }
        }
    }
};
{% endhighlight %}

After that we just need to add a new parameter to our `CreateEffectFactory` call. This parameter will contain an array of `string`s that contain all of the adjustable effect properties our definition contains. The format is the name of the effect followed by the name of the property: 

{% highlight c# %}
_effectFactory = _compositor.CreateEffectFactory(
    graphicsEffect, 
    new string[] { "hueEffect.Angle" });
{% endhighlight %}

You can also apply a name to the nested effects and adjust their properties as well, just add them to the array and make sure each effect has its own name.

Finally, all we have to do is animate the `Angle` property of the effect. This should look very similar to last week's animation section:

{% highlight c# %}
var animation = _compositor.CreateScalarKeyFrameAnimation();
var easing = _compositor.CreateLinearEasingFunction();

animation.InsertKeyFrame(0.0f, 0.0f);
animation.InsertKeyFrame(1.0f, (float)(2.0 * Math.PI), easing);

animation.IterationBehavior = AnimationIterationBehavior.Forever;
animation.Duration = TimeSpan.FromMilliseconds(4000);

effectBrush.StartAnimation("hueEffect.Angle", animation);
{% endhighlight %}

The Angle property's range is from 0 to 2 * Pi, and we have it going on forever at 4 second intervals. We also set a linear easing function again, as it works better for the effect we're going for. We start the animation on our effect brush, and we make sure that we reference the property's full name in our `string`. And now it should look like this:

<iframe src='https://gfycat.com/ifr/PracticalSameAkitainu' frameborder='0' scrolling='no' width='640' height='460.43165467625903' allowfullscreen></iframe>{: .center-image }

As you can tell, once you learn the building blocks of the API building on it becomes very easy! If you'd like to see the source for our finished product, you can find that [here](https://github.com/robmikh/blog.samples/tree/master/2016.04.19/EffectsDemo).

As always, make sure you're [subscribed](http://blog.robmikh.com/feed.xml) so you never miss a post! Next week we will be covering expressions! As always let me know ~~on twitter or~~ by [email](mailto:robmikh@robmikh.com) if you have any questions or feedback.
