---
layout: post
title:  "Changes to XAML-Composition Interop"
date:   2016-07-16 12:00:00 -0700
categories: uwp xaml composition
---

This week I came across [Mike Taultly](https://twitter.com/mtaulty)'s blog post where he spotted a behavior change between the November Update and the upcoming Anniversary Update in regards to XAML-Composition interop. You can find that blog post [here](https://mtaulty.com/2016/07/15/windows-10-anniversary-update-14388-uwp-visual-layer-offsets-on-preview-sdk-14388/). Initially this behavior was thought to be a bug by others on twitter, and admittedly this behavior might seem a bit cryptic (I was tripped up by it as well!). This change in behavior is actually a change in the model for XAML-Composition interop which you can read about [here](https://github.com/Microsoft/WindowsUIDevLabs/wiki/XAML-Composition-Property-Synchronization-Changes), but we're going to go into greater depth here.

<!--more-->

So to recap, what exactly is the behavior that has changed? Let's say that we have a `Button` positioned somewhere on our page that is not the default position. Maybe something like this:

{% highlight xml %}
<Button x:Name="TestButton" Content="Click Me!" 
                            HorizontalAlignment="Right" 
                            VerticalAlignment="Bottom" 
                            Margin="15" />
{% endhighlight %}

Here the button is positioned in the lower right corner of the window with a margin of 15. Now let's say that on the `Page`'s `Loaded` event handler we try to move our `Button` using the Composition API.

{% highlight c# %}
public void OnLoaded(object sender, RoutedEventArgs args)
{
	var visual = ElementCompositionPreview.GetElementVisual(TestButton);
	visual.Offset = new Vector3(-100, -100, 0);
}
{% endhighlight %}

In the November Update, we would expect this to move the visual up and to the left by 100 in each direction. However, in the Anniversary Update this doesn't happen! So why is this happening? Or rather, why are things *not* happening? For that we need to return to the mechanics of XAML-Composition interop and discuss why it was that the previous behavior worked as it did.

As we covered last time, the Composition API is used by XAML under the covers, and every `UIElement` has a collection of `Visual`s that represent it. When you call `ElementCompositionPreview.GetElementVisual`, you get a `Visual` that represents the `UIElement`. But where in the `UIElement`'s sub-tree is that visual located? In the November Update a control's sub-tree looked something like this:

![XAML Tree in the November Update](/assets/novemberupdatexamltree.jpg){: .center-image }

The `Visual` that is returned to you (Visual B) is a child of the root visual of the control. XAML controls the root of this tree (Visual A) for its own layout, meaning that anything you change on Visual B is purely additive. This is why if you were to try our experiment in the November Update the Button would not be sitting at an absolute (-100, -100), but rather (-100, -100) relative to where XAML had positioned the `Button`. This is because your offset is always relative to the position of your parent. With this model, XAML and the developer never touched the same visual.

In the Anniversary Update the model was changed to allow some more complex scenarios, namely Implicit Animations. Now a `UIElement`'s sub-tree looks something like this:

![XAML Tree in the Anniversary Update](/assets/anniversaryupdatexamltree.jpg){: .center-image }

Now the `Visual` returned by `ElementCompositionPreview.GetElementVisual` is the root visual for the control, which also means it is the same control that XAML touches to do layout. This makes features like Implicit Animations doable, as you can set the animation directly on the offset that XAML will change when it recomputes layout. However, this also means that there is a risk of XAML clobbering the value set by the developer. This is why setting the offset doesn't do anything, as the offset is getting clobbered by XAML's layout positioning. Let's say that we changed our code to change the offset when the user clicks the `Button`. Because this is after XAML has computed layout, and doesn't have a reason to recompute it, the value is not clobbered. However, here we would see another difference between the November Update and the Anniversary Update. The `Button` looks like it disappeared! This is because you're changing the real offset of the control now, which means that the button is actually at (-100, -100) since there is no positioning information that it is inheriting from its parent. 

In short, visual properties that you could modify on a `UIElement` were **additive** in the November Update and are **absolute** in the Anniversary Update.

To help out developers, we've identified the properties that you effectively "share" with XAML:

* Offset
* Size
* Opacity
* TransformMatrix
* Clip
* CompositeMode

So what's the policy behind the way XAML updates these values? XAML keeps track of (ie, caches) the last value it set on a property. During a layout calculation, if it calculates a value that is different from the last value XAML decided to set, then it will set a new value. Conversely, if the value calculated by XAML is the same as what it had set before (ie, its baseline) then it will not set a new value. This holds true *even if you have changed the value yourself*, as XAML only checks against what it knows it had set itself. This also includes when the control is first created, where in this case the default values are used for the baseline.

Let's go back to our experiment and change the button to look like this:


{% highlight xml %}
<Button x:Name="TestButton" Content="Click Me!" />
{% endhighlight %}

By removing the HorizontalAligment, VerticalAlignment, and Margin, XAML will not set the offset of our `Button`. The default value for its offset property is (0, 0, 0), and after XAML creates the `Button` it will calculate layout and see that the `Button` should have an offset of (0, 0, 0). Since this value matches the baseline, it does **not** touch the visual. This way if you were to change the offset of the `Button` it wouldn't be clobbered by XAML.

However, if you were to add a Margin to the `Button` later on, then XAML would calculate layout and see that the new offset it calculated is different from the baseline, and then set the offset property on the visual to the new value. This would clobber whatever you had set before. This would also detach any animations or expressions you may have set (with the exception of Implicit Animations by their nature). 

If you wish to emulate the behavior from the November Update, then you could introduce a "parent visual" yourself. While you can't change the tree structure, you can indirectly influence it by creating other controls. If you were to wrap the `Button` in something like a `Grid` or a `Border`, you could work around this behavior:

{% highlight xml %}
<Border HorizontalAlignment="Right" VerticalAlignment="Bottom" Margin="15">
	<Button x:Name="TestButton" Content="Click Me!" />
</Border>
{% endhighlight %}

Because the `Button`'s offset will be seen as the same as the baseline during creation, XAML won't change the offset property on the root visual. This allows you to change them in peace. Additionally, the positioning of the `Button` is now relative to the parent `Border` control. This is one way to emulate the November Update behavior. 

Admittedly this can take some time to get used to, so feel free to ask me questions on [twitter](https://twitter.com/robmikh) or by [email](mailto:robmikh@robmikh.com). Also feel free to come by the team's [GitHub page](https://github.com/Microsoft/WindowsUIDevLabs) to ask questions as well. Make sure you’re subscribed to the [RSS feed](http://blog.robmikh.com/feed.xml) so that you never miss a post! I haven't forgotten about the video library application, it's just taking me longer than I'd like. I may postpone it until after the Anniversary Update comes out, as working from within my Insider VM is kind of a bummer since I can't even use a full monitor. Additionally I'm working on another project I may start blogging about as well!