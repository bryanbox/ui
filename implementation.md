# ui library implementation information

All platform-specific functionality is isolated in a mega-type `sysData` that stores OS toolkit handles and provides methods that do the work for the frontend API. The file `sysdata.go` defines a type `cSysData` that contains everything all platforms have in common and dummy definitions of the `sysData` functions that panic. The platform-specific `sysData` embeds `cSysData`.

The key `sysData` function is `sysData.make()`. It takes two arguments: the initial text of the control (if any), and a pointer to a `sysData` that represents the window that holds the control. If this pointer is `nil`, we are creating a window instead, so any window-specific actions are taken here.

`cSysData` contains the control type as a number, an `event` channel where the object's standard event (close button for `Window`, click for `Button`, etc.) is stored, and a `resize()` function variable that is called to resize child controls on window resize.

Controls need only two functions: a `make()` function that builds the control, and a `setRect()` function that positions the control in its parent window. `make()` takes a single argument: the parent window. This window's `sysData` field gets passed to the control's `sysdata.make()` to indicate that it is a child control.

To keep things thread safe, all UI functions should be run on a separate thread whose OS thread is locked. This thread is goroutine `ui()`, presently started by `init()` (but it may be started by a dedicated function later). This goroutine is defined per platform, but takes the general form of
``` go
func ui(initErrors chan error) {
    runtime.LockOSThread()
    initErrors <- doPlatformInit()    // inform init() of success or failure
    for {
        select {
        case message <- uitask:
            do(message)
        default:
            platformEventLoopIteration()
        }
    }
}
```
`uitask` is a channel that transmits messages. These messages indicate platform-specific functions to call and their arguments.

All `sysData` methods except `sysData.setRect()` dispatch through `uitask`. Control resizing is handled within the UI goroutine itself, so `sysData.setRect()` (and thus `sysData.resize()`) call resizing functions directly. (The GTK+ backend broke spectacularly otherwise.)

## Windows
On Windows, all controls are windows, window classes are used to define their type, and messages are used to perform actions on windows and dispatch(different word? TODO) events. The data that we need to store, then, is the class name, initial styles, and combobox/listbox messages.

For `Window`s, a new window class is created for each window that you open. This window class is only different by its message handling function, or window procedure/WndProc. The WndProc is generated as a closure, so that it can safely absorb the window's `sysData` (so we don't need t look it up manually). This is all in `stdwndclass_windows.go`.

For controls, Windows uses an ID number to identify controls to the parent window, rather than passing around the window handles directly. The `sysData` for a window takes care of all this.

`uitask` transmits structures of type `uimsg`, which contain three things:
<ul><li> A `syscall.LazyProc` to call
<li> The parameters, as a `[]uintptr` (just like `syscall.LazyProc.Call()`)
<li> A channel to transmit `system.LazyProc.Call()`'s return values on</ul>
The `sysData` functions create the return channel, wait for results on it, then destroy the channel. As all DLL dispatch is done through `syscall.LazyDLL`, the Windows backend does not use cgo.

## Unix (except Mac OS X)
GTK+ is strange: there are constructor functions that return `GtkWidget *`, but anything that actually accesses a control requires the `GtkWidget *` to be cast to the appropriate control type. Fortunately, we can do this at time of call and just store `GtkWidget *`s for everything. And as most of these control methods take the same form, we can just store a list of functions to call for each control type.

`uitask` is a channel that takes `func()` literals. These are closures generated by the `sysData` functions that contain the GTK+ calls and a channel send for return values (like with Windows above). If no return value is needed, the channel send just sends a `struct{}`. The UI goroutine merely calls these functions.

As the GTK+ main loop system does not quite run in a sane way (it allows recursion, and the `gtk_main_loop_iteration_do` function only onperates on the innermost call), we cannot use the `for`/`select` template for `ui()`. Fortunately, we can hook into the GDK main loop (that the GTK+ main loop uses) to run our `uitask` dispatches whenever the GDK main loop is idle. The only catch is that the `uitask` receives have to be non-blocking for this to work properly, so we wrap them in a `select` with a null `default`.

GTK+ layout managers are not used since the UI library's layout managers are coded in a portable way. (`GtkFixed` is used instead.) This isn't ideal, but it works for now.

The only major snag with the GTK+ implementation is the implementation of `Listbox`; see `listbox_unix.go` for details.

## Mac OS X
The Mac OS X implementation has yet to be written. (My Mac is presently out of comission; I'm waiting for a replacement PSU to arrive.) It will use Cocoa and call the Objective-C runtime manually (by using cgo to link to libobjc and calling `objc_msgSend`, etc.). The `uitask` channel will likely behave as it does on Windows.