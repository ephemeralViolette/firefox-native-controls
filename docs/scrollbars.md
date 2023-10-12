# How I ported native scrollbars to modern Firefox

"Native" scrollbars (really, they were never native, just themed to look native) were disabled in Firefox 97 and residual code was removed in Firefox 98. This means that the last version of Firefox that they were accessible in was Firefox 96, which was released a pretty long time ago by now.

While it would be ideal to theme the scrollbars natively (i.e. with CSS), there are a few issues with this plan that make it somewhat unviable:

- Installation would require a userChrome.js script (i.e. an autoconfig browser chrome script loader) in order to modify the scrollbar CSS at all. This is because the scrollbars exist in no-mans-land, inaccessible by even the browser chrome inspector and userChrome.css.  
- CSS properties on the scrollbar elements, being CSS, are very limited. I could not figure out a way to even adequately replicate the drawing of the native Windows scrollbars from XP to 7, so I gave up rather fast. Since the scrollbars are XUL custom elements, you can't even use pseudoelements on them for glyphs.
- Sites would randomly override the custom CSS if they used any custom scrollbar styles, which I also found very undesirable. I could not figure out how to fix that.

It took me about 4 days straight to figure my way around the Firefox codebase and solve some pretty weird, pretty obvious bugs with this. I am currently maining my `xul.dll` build with native scrollbars and I think it works better than I expected.

My original plan was to do this using Windhawk, which would have been a nicer solution, if only I weren't completely misguided in the beginning, and if Windhawk could actually hook Firefox's symbols. Even though Mozilla has a symbol server for all release builds of Firefox, the symbols for `xul.dll` are so large (~1.19 GB) that Windhawk seemingly fails to load it. I assume it infinitely looped or whatever.

Now, I think a source code modification is the cleanest and best approach to patching this, in any case. Even if it's only a few functions that needed to be patched realistically, it would have still gotten pretty unmanagable to be hooking everything from foreign code needlessly.

## Don't get lost

ScrollbarDrawing (`widget/`) and its related classes are completely pointless to the implementation of native-styled scrollbars. They are
only responsible for drawing routines of the non-native ones seen in regular use.

Scrollbar drawing proper is done by `layout/generic/nsGfxScrollFrame.cpp`, with backing from `layout/xul/nsScrollbarFrame.cpp`. Themes
are reported by the application theme, which is `nsNativeThemeWin` when non-native controls are disabled (presumedly).

The primary file you will be editing is `widget/win/nsNativeThemeWin.cpp`.

## Important change history:

1. Sometime before the scrollbar changes, rendering code was restructured, such that
nsBasicNativeTheme, previously platform-specific, became `Theme` (`widgets/Theme.cpp`).
This doesn't really matter except in tracking change history, which may be quite useful
to do. (commit: https://github.com/mozilla/gecko-dev/commit/0468798e53b23f65d6538d985c0d3fef08cd5816)

2. As of ESR 115, scrollbar sizing code was greatly simplified. Previously, there was a `ScrollbarSizes`
struct which was used internally. This allowed for scrollbars to have independent horizontal and
vertical sizes, albeit this was never used. To account for this change, anything extending from
`nsITheme` (like `nsNativeThemeWin`) must change:

    ```
    virtual ScrollbarSizes GetScrollbarSizes(nsPresContext *, StyleScrollbarWidth, Overlay);
    ```

    to:

    ```
    virtual LayoutDeviceIntCoord GetScrollbarSize(const nsPresContext *, StyleScrollbarWidth, Overlay);
    ```

    Do note that the `const` before the `nsPresContext` argument is also very important.

    (commit: https://github.com/mozilla/gecko-dev/commit/604d8268b2e83340b8fce766bed3a0302c65cac4)

    **This was important to me in an early iteration, however, I figured out you don't need this function in the first place.** I simply use `GetMinimumWidgetSize` and `ClassicGetMinimumWidgetSize` since the initial public release of this.

3. Similarly, XUL layout was deprecated for the scrollbars by ESR 115. You will need to modify how
scrollbar size calculation works in order for it to appear at all (in debug builds, there will be an assertion
fail). I originally did this by modifying `ScrollbarMinSize` in `layout/xul/nsScrollbarFrame.cpp`, but later changed it to a simpler approach by modifying
`GetMinimumWidgetSize` in `widget/win/nsNativeThemeWin.cpp`, which meant modifying one fewer file. (commit: https://github.com/mozilla/gecko-dev/commit/38b10eafda0c7d5959deab3b7f212eddaba5cc57)

4. At some point, `EventState` was merged into `ElementState`. You can just replace all "`EventState`" with
"`ElementState`" and "`NS_EVENT_STATE_`" with "`ElementState::`", if this is applicable.

5. At some point (I don't know when), `nsNativeThemeWin::GetWidgetBorder` broke with scrollbar thumbs. This may need to be fixed, which can be done by redirecting the call to `nsNativeThemeWin::ClassicGetWidgetBorder`.

In the provided diff file, I basically already made all of these necessary changes as of ESR 115.

## General instructions

Here are the general steps for restoring native scrollbars. This is the approach I used to restore them to ESR 115,
and these instructions may be very useful for applying the patch to previous or later versions of Firefox.


1. Generally undo these commits:
    - ["Always draw scrollbars using the non-native theme on Windows."](https://github.com/mozilla/gecko-dev/commit/36215bf43067b04058cc397a12ef78b4f864d2a6)
    - ["Remove dead windows scrollbar drawing code."](https://github.com/mozilla/gecko-dev/commit/98c3acf4c210d194f0e48f9d6e87813a002c390b)

    Note that there have been a few changes in types and code organisation since then, so you will need to make
    some modifications to the code. One such example includes changing `GetScrollbarSizes` to the simpler `GetScrollbarSize`
    (described above).

2. Fix up layout code if necessary (important change history #2 and #3).

3. In nsNativeThemeWin, make `GetWidgetBorder` call `ClassicGetWidgetBorder` for thumbs:
    - `StyleAppearance::ScrollbarthumbVertical`
    - `StyleAppearance::ScrollbarthumbHorizontal`
    
    This is necessary in order for the horizontal thumbs to appear at all with themes for some versions of Firefox.

## Files I changed

- `widget/windows/nsNativeThemeWin.cpp` (most things)
- `widget/windows/nsUXThemeData.cpp` & `.h` (old UxTheme theme classes)