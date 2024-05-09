---
title: Attempting To Create A Window-resize Shortcut
date: 2024-04-28 2:30:00 +0000
categories: [Software Engineering]
tags: [operating systems, windows, developer experience]
img_path: /assets/img
---
The monitors at my workplace have an abnormal configuration, one is 1080p @ 27", another 1440p @ 32", and the last 4k 15". Using multiple displays of varying DPIs can be a strange experience (at least on Windows), particularly dragging windows from one display to another, it may shrink or grow, and appear at a different position than you anticipated. I don't think there is an intuitive way to handle this, as each user may expect different behaviour.

![An image showing a desk, with a laptop, one portrait monitor, and another landscape monitor, of varying DPIs.](windowsscrollresize-monitorsexample.jpeg)
_Example of a problematic setup, from [Terence Eden's blog](https://shkspr.mobi/blog/2020/09/how-can-i-get-consistent-sizes-with-physically-different-monitors-on-linux/)_

I can live with most of the quirks, but there's one that really gets to me. I'm very particular about where I place my windows and how they're sized, and after moving a window from one display to another the size is not how I want it to be in its new position. Changing the size of a window is quite a tricky task, as you need to get your mouse in just the right invisible area in order to begin the drag, this is UX that we have all become accustomed to, but I don't think is particularly smooth when you're moving around the desktop at the speed of light, jumping from app to app.

One day it hit me, for a shortcut that seemed so intuitive that I was surprised it didn't already exist within Windows, let alone via some third party app. What if; while dragging a window you could scroll your mouse wheel up or down, in order to resize the window. Sounds pretty smooth right? It wouldn't conflict with any existing functionality/shortcuts, and you could even do something neat too like size it according to its current aspect ratio, or hold shift to only resize the width rather than the height.

It stuck in my head for a week or two until I finally thought, well I am a software engineer after all, let's try it out!

## Starting Off
To pull off this functionality I knew I'd need to be hooking into the operating system somehow in order to listen for global events such as window dragging and scroll wheel actions. Thanks to some previous funky stunts in WinForms over the years I know the place to start looking for this sort of functionality is the Win32 API. The APIs are defined in C dlls that come as part of Windows, we can use them with C# by using some techniques you don't usually see in day to day C# usage.

Let's start by seeing if we can listen globally for a window being dragged. First we need to setup a hook, so that whenever the event happens, a function of ours is called. For this we need to use the [SetWinEventHook function](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwineventhook), of which one argument is a [WinEventProc](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nc-winuser-wineventproc). Here's an excerpt of the key parts.

```
HWINEVENTHOOK SetWinEventHook(
  [in] DWORD        eventMin,
  [in] DWORD        eventMax,
  [in] HMODULE      hmodWinEventProc,
  [in] WINEVENTPROC pfnWinEventProc,  ──────────────┐
  [in] DWORD        idProcess,                      │
  [in] DWORD        idThread,                       │
  [in] DWORD        dwFlags                         │
);                                                  │
                                                    │
void WinEventProc(   <──────────────────────────────┘
  HWINEVENTHOOK hWinEventHook,
  DWORD event,
  HWND hwnd,
  LONG idObject,
  LONG idChild,
  DWORD idEventThread,
  DWORD dwmsEventTime
)
```

This may look scary to begin with, but it's really just methods, like in C#, with convoluted names to confuse newcomers. There is official documentation explaining the types here: [Windows Data Types](https://learn.microsoft.com/en-us/windows/win32/winprog/windows-data-types). We can map these definitions to C# equivalents.
* DWORD: Unsigned 32 bit integer `UInt32` (`uint`).
* LONG: Signed 32 bit integer `Int32` (`int`).
* HWND: A handle to a window (a `void*` pointer), for our managed code, we can just use `IntPtr`.
* HMODULE: Another `void*` pointer, we can use `IntPtr` for.
* WinEventProc: A function shape or in C# we could call a `delegate`.

We can start defining these in our C# code:

```csharp
public IntPtr SetWinEventHook(uint eventMin, uint eventMax, IntPtr hmodWinEventProc, WinEventProc pfnWinEventProc, uint idProcess, uint idThread, uint dwFlags);
public delegate void WinEventProc(IntPtr hWinEventHook, uint @event, IntPtr hwnd, int idObject, int idChild, uint idEventThread, uint dwmsEventTime);
```

One step is missing though, for our `SetWinEventHook` method, we must indicate that it is going to call out to an external DLL, this can be done with the [DllImport attribute](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.dllimportattribute?view=net-8.0), and using the `static` and [`extern`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/extern) modifiers, like so:

```csharp
[DllImport("user32.dll")]
public static extern IntPtr SetWinEventHook(uint eventMin, uint eventMax, IntPtr hmodWinEventProc, WinEventProc pfnWinEventProc, uint idProcess, uint idThread, uint dwFlags);
```

When we call the SetWinEventHook method, the first 2 arguments `eventMin` and `eventMax` specify a range of events to listen to, these are all defined in the [Event Constants documentation](https://learn.microsoft.com/en-us/windows/win32/winauto/event-constants). We can find in the documentation EVENT_SYSTEM_MOVESIZESTART and EVENT_SYSTEM_MOVESIZEEND of which the values are 0x000A and 0x000B respectively.

```csharp
private const uint EVENT_SYSTEM_MOVESIZESTART = 0x000A;
private const uint EVENT_SYSTEM_MOVESIZEEND = 0x000B;

static void Main()
{
    var hook = Win32.SetWinEventHook(EVENT_SYSTEM_MOVESIZESTART, EVENT_SYSTEM_MOVESIZEEND, nint.Zero, winEventDelegate, 0, 0, 0);

    Win32.UnhookWinEvent(hook);
}

// Matching the 'WinEventProc' delegate.
private static void WindowMoveSizeCallback(IntPtr hWinEventHook, uint @event, IntPtr hwnd, int idObject, int idChild, uint idEventThread, uint dwmsEventTime)
{
    Debug.WriteLine($"Hwnd {hwnd} event: {eventType}");
}
```

I like to put these definitions in a static class `Win32` just so they're all neatly defined together. Now we can try and use the API. We need to define a method that matches that delegate.

## Notes
Change all mentions of `IntPtr` to `nint` and explain why - https://learn.microsoft.com/en-us/dotnet/api/system.intptr?view=net-8.0 (read remarks).

