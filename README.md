# Firefox Native Controls

Simple source code patches that re-enable native controls on Firefox. Requires modifying Firefox installation files, but you don't need to use a custom build of Firefox altogether.

It is still recommended that you use ESR and disable automatic updates. Unfortunately, updating is just a sacrifice you have to make to theming. However, I may write a custom updater with this mod in consideration, eventually.

The only potential caveat: Widevine support. I don't know if this is controlled by `xul.dll` or not, but there is a file beside it called `xul.dll.sig` in official Firefox builds that has information regarding a Widevine certificate (and is otherwise seemingly completely unused). If you are based, this will not matter to you.

At the moment, it is required that you still manually disable non-native controls in order to use this.

## Change documentation

- [Scrollbars](docs/scrollbars.md)

## Installing prebuilt versions

In the releases section on the right, there are pre-compiled versions of `xul.dll` for Firefox ESR.

Check your Firefox version and download the right version for the version that you have, and then replace `xul.dll` in the installation path with the file that you downloaded.

If you don't trust the versions I built myself, then you can mix the provided patches with the Firefox source code (see the following section). The prebuilt versions are just provided for convenience, because the Firefox source code is pretty big and takes a while to compile.

Step-by-step:

1. Download the release on the right side of the page. If your version is unsupported, then you will have to build the changes from source.
2. Replace `C:\Program Files\Mozilla Firefox\xul.dll` with the downloaded `xul.dll` file.
3. Make sure that `widget.non-native-theme.enabled` is false in `about:config`.
4. Close all Firefox processes in Task Manager and restart the browser.

## Building from source

[Clone Firefox for yourself](https://firefox-source-docs.mozilla.org/setup/index.html), and then mix in the patches as needed. Note that when you clone, it will put in the `mozilla-central` branch. You probably don't want this as these correspond to the latest Nightly builds, which will mean that the produced `xul.dll` binary will likely be incompatible with the current build environment.

After cloning, navigate to `.hg/hgrc` in the source directory and change the default path to the branch of your preferred build. For example, I changed it to `https://hg.mozilla.org/releases/mozilla-esr115` in order to access specific tags for the ESR 115 release. You can usually then find specific tags relating to subversions here: https://hg.mozilla.org/releases/mozilla-esr115/tags ("tags" tab in the Mercurial web viewer).

Since Firefox uses Mercurial for version control, it may be a little unfamiliar to you. You "update", rather than "checkout", different branches. So after you make those changes, I think you just do something like `hg pull -u -r FIREFOX_115_3_1esr_RELEASE` to switch to the specific branch. `pull` makes it get the specific version from remote, and `-u` means to update.

Run `./mach build` to build Firefox from source and then `./mach run` to test it. Here is a [Windhawk](//windhawk.net) mod that I use to quickly test classic theme for Firefox only (where I make a copy of `firefox.exe` named `firefoxa.exe`):

```cpp
// ==WindhawkMod==
// @id              firefoxa-classic-theme-test
// @name            [Testing] firefox classic theme
// @description     The best mod ever that does great things
// @version         0.1
// @author          ephemeralViolette
// @include         firefoxa.exe
// @compilerOptions -luxtheme
// ==/WindhawkMod==

#include <uxtheme.h>

BOOL Wh_ModInit()
{
    SetThemeAppProperties(0);
    return TRUE;
}
```

Also, currently there is a bug you need to be aware of. After disabling `widget.non-native-theme.enabled`, you must restart Firefox completely, or else the scrollbars will simply not render. I do not know why this happens exactly (maybe the native theme isn't loaded upon disabling the property? I haven't looked at other controls).

After building, the `xul.dll` file can be found in somewhere like `obj-x86_64-pc-windows-msvc/dist/bin` in the source code root, which looks just like `C:\Program Files\Mozilla Firefox`.

### Distribution method

Also because Firefox uses Mercurial rather than Git, I found it would be more trouble than it's worth to attempt to post a modified codebase onto GitHub. I initially thought to fork [mozilla/gecko-dev](//github.com/mozilla/gecko-dev), but the commit identifiers for this repository do not at all align with their Mercurial revision identifiers, so it is less than worthless. Also, I found I couldn't even find certain tags which were useful to access in the Mercurial version.

As a result, I just `hg export` the patches I make, which makes them pretty easy to bring back into the codebase later.

## Will you do other controls?

Probably. My main focus with the initial release was restoring native scrollbars. I didn't see other controls as important, since their native appearances could be better replicated with CSS.