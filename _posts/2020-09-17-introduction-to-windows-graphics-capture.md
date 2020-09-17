---
layout: post
title:  "Introduction to Windows.Graphics.Capture"
date:   2020-09-16 19:15:00 -0700
categories: uwp wgc capture
---

Some introduction

<!--more-->

## Specifying what you want to capture

If you want to get pixels back from something using the Windows.Graphics.Capture API, you need to find a way to represent it using a `GraphicsCaptureItem`. Later on we’ll go into more detail how to get a `GraphicsCaptureItem`, but for now know that currently a `GraphicsCaptureItem` can represent anything from a `Visual`, a window, or a display. Of course, this list could grow in the future!
A `GraphicsCaptureItem` contains pieces of information that you may find useful:

```csharp
class GraphicsCaptureItem
{
    // The display name can come from a few places:
    //    - A Visual’s Comment property
    //    - A Window’s title
    //    - A simple display description (e.g. Display 1)
    string DisplayName { get; }
    // This is a snapshot of the size this item describes.  Note that
    // in the case of a Visual, this is simply the Size property. It 
    // does not take RelativeSizeAdjustment into account.
    Windows.Graphics.SizeInt32 Size { get; }
    // This will tell you when what you’re capturing has gone away. 
    // Either the window was closed, the monitor unplugged, or the 
    // visual destroyed.
    event TypedEventHandler<GraphicsCaptureItem, object> Closed;
}
```

## Creating a frame pool

To receive frames from the system compositor, you'll need to create a `Direct3D11CaptureFramePool`. There are two ways to create one based on how to you want to handle your threading when receiving frames. 

The first option is by using the static `Create` method. Calling `Create` requires that you have a `DispatcherQueue` for your current thread (and you're pumping messages, of course). This will make the `FrameArrived` callback fire on the calling thread. If you're developing a UWP application, your main thread will do; although, you may not want to (more on that later). If you're developing a Win32 application, you'll need to call [`CreateDispatcherQueueController`](https://docs.microsoft.com/en-us/windows/win32/api/dispatcherqueue/nf-dispatcherqueue-createdispatcherqueuecontroller) and pump messages on that thread. Or for either type of application, you can create a new thread with it's own `DispatcherQueue` by calling [`DispatcherQueueController.CreateOnDedicatedThread`](https://docs.microsoft.com/en-us/uwp/api/windows.system.dispatcherqueuecontroller.createondedicatedthread?view=winrt-19041#Windows_System_DispatcherQueueController_CreateOnDedicatedThread). This is a similar requirement to using the `Windows.UI.Composition` APIs.

The second option is by using the static `CreateFreeThreaded` method. Calling `CreateFreeThreaded` does not require the calling thread to have a `DispatcherQueue`. However, it means that the `FrameArrived` callback will fire on the frame pool's internal worker thread. Make sure you're accessing resources in a thread-safe matter if you wish to use this.

Unless you're only capturing a single frame, I would not recommend calling `Create` on your UI thread. The `FrameArrived` event will typically fire at the refresh rate of your primary monitor.

After you've picked which way you want to create your `Direct3D11CaptureFramePool`, the parameters are the same:

```csharp
class Direct3D11CaptureFramePool : IDisposable // Or IClosable if you're using C++
{
    static Direct3D11CaptureFramePool Create(
        Windows.Graphics.DirectX.Direct3D11.IDirect3DDevice device,
        Windows.Graphics.DirectX.DirectXPixelFormat format,
        int numberOfBuffers,
        Windows.Graphics.SizeInt32 size);
    
    static Direct3D11CaptureFramePool CreateFreeThreaded(
        Windows.Graphics.DirectX.Direct3D11.IDirect3DDevice device,
        Windows.Graphics.DirectX.DirectXPixelFormat format,
        int numberOfBuffers,
        Windows.Graphcis.SizeInt32 size);

    /* ... */
}
```

You'll need to provide a Direct3D 11 device. To create a `IDirect3DDevice` from a `ID3D11Device`, check out the [`CreateDirect3DDevice`](https://docs.microsoft.com/en-us/windows/win32/api/windows.graphics.directx.direct3d11.interop/nf-windows-graphics-directx-direct3d11-interop-createdirect3ddevice) method. Note that `CanvasDevice` from the Win2D package implements `IDirect3DDevice`.

You'll also need to provide a pixel format. Currently we only support `B8G8R8A8UIntNormalized` for a SDR capture, and `R16G16B16A16Float` for HDR capture. Keep in mind that HDR capture has some known issues, mainly that there isn't a way to specify a boost value for SDR content. This will be addressed in a future update to Windows. Currently if you know that the window you want to capture entirely has HDR content (e.g. a game), then go for it. Otherwise, I'd capture in SDR.

The last two parameters specify the number of buffers in the frame pool and the size of each buffer respectively.

## Starting a capture

To initiate a capture, you'll need to create a `GraphicsCaptureSession`. This object represents the capture itself, and has some properties that you can tweak (e.g. turn cursor rendering on/off). To create one, call the `CreateCaptureSession` method on the frame pool and provide your `GraphicsCaptureItem`:

```csharp
var device = new CanvasDevice();
var item = /* We'll cover this later */;
var framePool = Direct3D11CaptureFramePool.Create(
    device,
    DirectXPixelFormat.B8G8R8A8UIntNormalized,
    2,
    item.Size());

var session = framePool.CreateCaptureSession(item);
```

To begin a capture, call the `StartCapture` method. To end your capture, call `Dispose` if you're using C# or `Close` if you're using C++. C++ callers can also release all references to the session to stop the capture as well.

```csharp
session.StartCapture();
/* Some time passes */
session.Dispose();
```

## Receiving frames

It's now time to get some pixels back! The way to get a new frame is to call the `TryGetNextFrame` method on the `Direct3D11CaptureFramePool`. When you create the frame pool, all buffers that are created are available to the system compositor. Whenever the system compositor deciedes to draw, it will attempt to retreive a buffer from the pool. If none is available, the frame is skipped. If a frame is available, the system compositor will render it and then add it to the frame pool's internal queue. Calling `TryGetNextFrame` tries to retreive a frame from the front of this queue. If there are no rendered frames waiting, `TryGetNextFrame` will return null.

Of course, we don't recommend that you poll the `Direct3D11CaptureFramePool`, but instead subscribe to the `FrameArrived` event. When the event fires, `TryGetNextFrame` will be guaranteed to have a new frame for you (assuming somebody hasn't beat you to it).

```csharp
framePool.FrameArrived += (sender, args) => 
{
    using (var frame = sender.TryGetNextFrame())
    {
        /* Do something with the pixels */
    }
};
```

You'll notice that I used a `using` statement for the returned `Direct3D11CaptureFrame`. Calling `Dispose` or `Close` will return the frame back to the frame pool and make it available to the system compositor. You should only access the underlying Direct3D texture when you have a live `Direct3D11CaptureFrame`. Don't cache it!

The `Direct3D11CaptureFrame` also contains some information about when the frame was rendered. It also contains information about the size of the content you captured. For windows this will be the size of the window, for displays it will be the resolution of the display. Use this to determine if you want to resize your frame pool by calling `Recreate`. We'll cover that more later.

```csharp
class Direct3D11CaptureFrame : IDisposable // Or IClosable if you're using C++
{
    Windows.Graphics.DirectX.Direct3D11.IDirect3DSurface Surface { get; }
    Windows.Graphics.SizeInt32 ContentSize { get; }
    TimeSpan SystemRelativeTime { get; }
}
```

