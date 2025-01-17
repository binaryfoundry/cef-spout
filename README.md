# CEF with Spout Output

Upgraded to Visual Studio 2022 and CEF 110.

This application is heavily based on daktronics' CEF-Mixer application (see https://github.com/daktronics/cef-mixer). In its core, it is a Chromium-based web content rendering engine (a "headless" browser), rendering its output also as a texture to a Spout (https://github.com/leadedge/Spout2/) data channel for further use in other Spout-enabled applications. 

This project was originally intended as a tool to work with HTML/JS/SVG-based visuals in other applications like MadMapper for creative coding and music/art installations (see other repos), but can be universally used to fetch live graphical output of a web browser for any purpose.

See the original cef-mixer readme/description below for build instructions and basic usage. PLEASE NOTE: It is recommended to use the CMAKE GUI together with the original cmake Files (and to set CEF_ROOT manually then), and not the provided .bat because of additional dependencies that might or might not work with the latter.


## CEF Offscreen-Rendering (OSR) Mixer Demo

A sample application to demonstrate how to use the proposed `OnAcceleratedPaint()` callback when using CEF for HTML off-screen rendering.  This application uses D3D11 shared textures for CEF which improves the OSR rendering performance.

## Build Instructions

1. If you don't have it already - install CMake and Visual Studio 2022
    * VS 2022 Community Edition is fine - just make sure to install C/C++ development tools

2. Download latest CEF to create a custom build or use an example binary distribution
    * Sample distributions support **Chromium 110**
    * [x64 sample binary distribution][x64_build] (Release build only)
    * [x86 sample binary distribution][x86_build] (Release build only)

    
3. From a command prompt set the environment variable **CEF_ROOT** to the location of your CEF binary distribution.  Then run the gen_vs.bat script.

```
> set CEF_ROOT=<path\to\cef\binary-distribution>
> gen_vs2017.bat
```

> Note: if you're building for x86 you will need to modify gen_vs2017.bat to specify the correct generator for CMake

4. Open the build/cefmixer.sln solution in Visual Studio

> If using one of the sample binary distributions from step 2 - make sure to change the build configuration to **Release** since the distributions above do not contain **Debug** versions

5. Build the **ALL_BUILD** project

6. Run the **cefmixer.exe** application

## Usage
Once the cefmixer.exe is built, it can be run without any arguments - in which case it will automatically navigate to https://webglsamples.org/aquarium/aquarium.html

In addition to rendering an HTML view off-screen, the demo application will also create an overlay layer using a PNG image file (the red DEMO graphic in the screenshots below).

The following screenshot was taken when running on a gaming monitor at 144Hz:

![VSync On][demo1]

The url for the HTML layer can be specified as a command line argument: (width x height for the window size are also supported on the command-line)

```
cefmixer.exe https://threejs.org/examples/webgl_animation_keyframes_json.html --width=960 --height=540
```
Pressing `Ctrl+V` will allow the HTML view to run unthrottled with no v-sync:

![VSync Off][demo2]

Obviously, there are not many use cases to render frames completely unthrottled - but the point is to let the integrating application control all timing aspects. This demo application uses the new `SendExternalBeginFrame` method to issue BeginFrame requests to Chromium to synchronize HTML updates with its render loop.

### Multiple Views

The application can tile a url into layers arranged in a grid to test multiple HTML browser instances.  Each layer is an independent CEF Browser instance.  The following example uses the `--grid` command-line switch to specify a 2 x 2 grid:

```
cefmixer.exe http://webglsamples.org/dynamic-cubemap/dynamic-cubemap.html --grid=2x2
```

![Grid][demo3]

### Custom Layering

The command-line examples above work to get something running quickly.  However, it is also possible to define the layers using a simple JSON file.

For example, if the following is saved to a file called `composition.json` :

```json
{
  "width":960,
  "height":540,
  "layers": [
     {
       "type":"web",
       "src":"http://webglsamples.org/spacerocks/spacerocks.html"
     },
     {
       "type":"web",
       "src":"file:///C:/examples/overlay.svg",
       "left":0.5,
       "top":0.5,
       "width":0.5,
       "height":0.5			
     }
  ]
}
```

> Note: layer positions are in normalized 0..1 units where 0,0 is the top-left corner and 1,1 is the bottom-right corner.

We can run `cefmixer` using the above JSON layer description:

```
cefmixer.exe c:\examples\composition.json
```

![JSON][demo4]

The application uses the handy utility method `CefParseJSON` in CEF to parse JSON strings.

## Integration
The update to CEF proposes the following changes to the API for application integration.

1. Enable the use of shared textures when using window-less rendering (OSR).

```c
CefWindowInfo info;
info.SetAsWindowless(nullptr);
info.shared_texture_enabled = true;
```

2. Override the new `OnAcceleratedPaint` method in a `CefRenderHandler` derived class:

```c
void OnAcceleratedPaint(
		CefRefPtr<CefBrowser> browser,
		PaintElementType type,
		const RectList& dirtyRects,
		void* share_handle) override
{
}
```

`OnAcceleratedPaint` will be invoked rather than the existing `OnPaint` when `shared_texture_enabled` is set to true and Chromium is able to create a shared D3D11 texture for the HTML view.

3. Optionally enable the ability to issue BeginFrame requests

```c
CefWindowInfo info;
info.SetAsWindowless(nullptr);
info.shared_texture_enabled = true;
info.external_begin_frame_enabled = true;
```

At an interval suitable for your application, make the following call (see [web_layer.cpp](https://github.com/daktronics/cef-mixer/blob/master/src/web_layer.cpp) for a full example) :

```c
browser->GetHost()->SendExternalBeginFrame();
```

When using `SendExternalBeginFrame`, the default timing in CEF is disabled and the `windowless_frame_rate` setting is ignored.


## Room for Improvement
A future update could include the following 
 * ~~Allow the client application to perform SendBeginFrame by adding a new method to CEF's public interface.~~
     * ~~Chromium already supports an External BeginFrame source - CEF currently does not expose it directly.~~
     * **Update** this is now supported in the latest revision
 * Update `OffscreenBrowserCompositorOutputSurface` class to handle both the Reflector and a shared texture
     * This was attempted originally but ran into issues creating a complete FBO on the Reflector texture
     * Not a big deal for CEF applications, since CEF does not use the Reflector concept in Chromium anyway.
 * Take the Chromium changes directly to the Chromium team
     * We can get the job done with the patching system built into CEF to apply Chromium changes, but rather the shared texture FBO probably makes more sense as a pull request on Chromium itself.  Seems only reasonable applications that use Headless-mode in Chromium could also benefit from shared textures.

[demo1]: https://user-images.githubusercontent.com/2717038/37864646-def58a70-2f3f-11e8-9df9-551fe65ae766.png "Cefmixer Demo"
[demo2]: https://user-images.githubusercontent.com/2717038/37864824-a02a0648-2f41-11e8-9265-be60ad8bf8a0.png "No VSync"
[demo3]: https://user-images.githubusercontent.com/2717038/37864648-ea76954c-2f3f-11e8-90d6-4130e56086f4.png "Grid"
[demo4]: https://user-images.githubusercontent.com/2717038/37930171-9850afe0-3107-11e8-9a24-21e1b1996fa5.png "JSON"
[x64_build]: https://cef-builds.spotifycdn.com/cef_binary_110.0.25%2Bg75b1c96%2Bchromium-110.0.5481.78_windows64_minimal.tar.bz2 "x64 Distribution"
[x86_build]: https://cef-builds.spotifycdn.com/cef_binary_110.0.25%2Bg75b1c96%2Bchromium-110.0.5481.78_windows32_minimal.tar.bz2 "x86 Distribution"
[pr158]: https://bitbucket.org/chromiumembedded/cef/pull-requests/158/support-external-textures-in-osr-mode/diff "Pull Request"
[changes]: https://github.com/daktronics/cef-mixer/blob/master/CHANGES.md "Walkthrough"

