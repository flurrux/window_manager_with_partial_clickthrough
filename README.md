

## About

This is a fork of the [window_manager package](https://github.com/leanflutter/window_manager) that adds support for **rectangular click-through regions** on Windows. Instead of making the entire window click-through, you can now specify a single rectangular area that passes mouse events to windows below it, while the rest of your application remains fully interactive.

__Full disclosure__:  
Most of the changes were authored by a coding agent since i am not well versed in c++ and windows APIs.

My use case is a custom screen-capture tool where i need the recording area to be click-through, but the rest of the application needs to be interactive to stop the recording.

It works, with the only caveat being that the window's frame style switches to a very unappealing default style. This affects only the window's titlebar (including maximize & close buttons) and its borders. I've tried alternative solutions, but nothing worked, so i'm begrudgingly accepting this flaw.

For more info, please read [CLICK_THROUGH_REGION.md](./CLICK_THROUGH_REGION.md) (also authored by copilot).
