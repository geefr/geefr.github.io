---
layout: post
title: "Porting VSGVR to the Oculus Quest"
date: 2022-11-27 13:00
categories: blog
---

This post runs through the steps I took to add Oculus Quest support to [VSGVR](https://github.com/geefr/vsgvr), my project to add OpenXR integration to the [VSG](https://github.com/vsg-dev/VulkanSceneGraph) rendering engine.

The starting point of all of this is that VSGVR already exists, and has been tested on various desktop OpenXR runtimes. The changes here are specifically for Android, and use a version of the [VulkanSceneGraph Android Example](https://github.com/vsg-dev/vsgExamples/tree/master/examples/platform/vsgandroidnative).

The full code for the VSGVR Quest example should be available in the VSGVR repository after some cleanup, the changes aren't quite merged in just yet.

# Android Project Setup

So starting from an Android native-activity example, we start by pulling in both VSG and VSGVR to the CMakeLists.txt

```cmake
add_subdirectory(D:/dev/vsg/VulkanSceneGraph vsg)
add_subdirectory(D:/dev/vsg/vsgvr vsgvr)

target_include_directories(vsgnative PRIVATE
    ${ANDROID_NDK}/sources/android/native_app_glue
        D:/dev/vsg/vsgvr/vsgvr/include)

# add lib dependencies
target_link_libraries(vsgnative
    vsg::vsg vsgvr
    android
    native_app_glue
    log
    )

# Required to expose android-specific XR types
# (In vsgvr this is already defined for us by `vsgvr/OpenXRCommon.h`)
target_compile_definitions(vsgnative PRIVATE XR_USE_PLATFORM_ANDROID)
```

And set up the AndroidManifest.xml with the required Quest / VR parameters (See the [Oculus developer docs](https://developer.oculus.com/documentation/native/android/mobile-openxr/?locale=en_GB))

```xml
  <!-- Set permissions and features -->
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
  <uses-feature
    android:name="android.hardware.vulkan.version"
    android:version="0x400003"
    android:required="true" />

  <!-- This .apk has no Java code itself, so set hasCode to false. -->
  <application
      android:allowBackup="false"
      android:fullBackupContent="false"
      android:icon="@mipmap/ic_launcher"
      android:label="@string/app_name"
      android:hasCode="false">

      <meta-data android:name="com.oculus.intent.category.VR" android:value="vr_only"/>
      <meta-data android:name="com.oculus.supportedDevices" android:value="quest|quest2|cambria"/>

    <!-- Our activity is the built-in NativeActivity framework class.
         This will take care of integrating with our NDK code. -->
    <activity android:name="android.app.NativeActivity"
              android:label="@string/app_name"
              android:launchMode="singleTask"
              android:resizeableActivity="false"
              android:screenOrientation="landscape"
              android:configChanges="density|keyboard|keyboardHidden|navigation|orientation|screenLayout|screenSize|uiMode"
              android:theme="@android:style/Theme.Black.NoTitleBar.Fullscreen">
        <!-- Tell NativeActivity the name of our .so -->
        <meta-data android:name="android.app.lib_name" android:value="vsgnative" />
        <intent-filter>
          <action android:name="android.intent.action.MAIN" />
          <category android:name="com.oculus.intent.category.VR" />
          <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
  </application>

  <uses-feature android:name="android.hardware.vr.headtracking" android:required="true" android:version="1" />
```

main.cpp is then updated to migrate from VSG to VSGVR - I won't elaborate on that for now as VSGVR is a prototype, but it's almost the same code as the desktop VSGVR example; Primarily this involves creating a `vsgvr::OpenXRViewer` instead of `vsg::Viewer`, and some changes to the rendering loop to comply with OpenXR's application model.

And once that's ready our app should be good to go right? - We've build for android, marked it as a VR app as needed by the quest, and we know the OpenXR code already works as we tested it dilligently on desktop first.

No of course not, when we call `xrCreateInstance` we get XR_ERROR_INITIALIZATION_FAILED. There's no real hint as to why, but this is because we haven't initialised the OpenXR runtime correctly for Android.

# OpenXR Android Requirements

If we read the spec closer, it turns out some platforms require special initialisation for the OpenXR loader. Specifically in our XrInstanceCreateInfo structure "next must be NULL or a valid pointer to the next structure in a structure chain".

Good thing the OpenXR spec is so extensive, let's take a look at [XrInstanceCreateInfoAndroidKHR](https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#XrInstanceCreateInfoAndroidKHR)

```c++
// Provided by XR_KHR_android_create_instance
typedef struct XrInstanceCreateInfoAndroidKHR {
    XrStructureType    type;
    const void*        next;
    void*              applicationVM;
    void*              applicationActivity;
} XrInstanceCreateInfoAndroidKHR;
```

A bunch of `void*` pointers, okay. So first the spec says we have to:
* Enable the `XR_KHR_android_create_instance` extension
* Set the structure `type` and `next` fields, as with all OpenXR calls
* Set `applicationVM` and `applicationActivity`

That's fairly straightforward then, and for the Android values we'll just need to query through the native glue. Here's a quick test that the code will at least compile:
```c++
void android_main(struct android_app* app)
{
    XrInstanceCreateInfoAndroidKHR createInfo;
    createInfo.type = XR_TYPE_INSTANCE_CREATE_INFO_ANDROID_KHR;
    createInfo.next = nullptr;
    // applicationVM is a pointer to the JNI’s opaque JavaVM structure, cast to a void pointer
    createInfo.applicationVM = static_cast<void*>(app->activity->vm);
    // applicationActivity is a JNI reference to an android.app.Activity that will drive
    // the session lifecycle of this instance, cast to a void pointer.
    createInfo.applicationActivity = static_cast<void*>(app->activity->clazz);
}
```

Now that we know what's needed, let's quickly patch those changes into vsgvr. For now with some basic `#ifdef ANDROID` blocks in `OpenXRInstance.cpp`. To pass in the native handles that'll have to be through the `OpenXRTraits` structure.

Naturally this exploded into a larger rework of vsgvr to allow us to have an `OpenXRAndroidTraits` as well, but in principle we're just passing 2 `void*` handles through a C++ API in a portable way (An interesting subject, for another time).

Anyway, after those updates and ensuring the Android handles are passed through does it work? ..Of course not, we still get `XR_ERROR_INITIALIZATION_FAILED` when trying to create an instance - We've missed another important step.

# Android OpenXR Loader Initialisation

Taking a look at an [OpenXR example we know works](https://github.com/artyomd/Quest-XR) there's something else we missed - A strange bunch of function lookups and calls to initialise an android loader.

Once again we return to the OpenXR spec (which really does contain everything we need, if we look carefully) and find the extension [XR_KHR_loader_init_android](https://registry.khronos.org/OpenXR/specs/1.0/html/xrspec.html#XR_KHR_loader_init).

This states that "On Android, some loader implementations need the application to provide additional information on initialization. This extension defines the parameters needed by such implementations. If this is available on a given implementation, an application must make use of it." - So assuming that the Quest does require a loader to be called, we're still not doing that correctly.

Setup of this is roughly similar to the `xrCreateinstance` changes, though in this case we don't opt-in to the extension; If it's present we need to use it, otherwise we don't. That works out as something like:
```c++
// Before any XR functions may be called we need to check if additional values are needed
// for runtime intiailisation. See XR_KHR_loader_init.
// Presence of this extension / requirement is denoted by a getProcAddress lookup.
auto fn = (PFN_xrInitializeLoaderKHR)xr_pfn_noexcept(XR_NULL_HANDLE, "xrInitializeLoaderKHR");
if( fn )
{
    auto androidTraits = _xrTraits.cast<vsgvr::OpenXrAndroidTraits>();
    if( androidTraits && androidTraits->vm && androidTraits->activity )
    {
        XrLoaderInitInfoAndroidKHR loaderInitInfo;
        loaderInitInfo.type = XR_TYPE_LOADER_INIT_INFO_ANDROID_KHR;
        loaderInitInfo.next = nullptr;
        loaderInitInfo.applicationVM = static_cast<void*>(androidTraits->vm);
        loaderInitInfo.applicationContext = static_cast<void*>(androidTraits->activity);
        fn(reinterpret_cast<const XrLoaderInitInfoBaseHeaderKHR*>(&loaderInitInfo));
    }
    else
    {
        throw Exception({"On Android an OpenXRAndroidTraits structure must be provided, with valid JNI/Activity pointers."});
    }
}
```

And after that does it work? Again no, that's slightly annoying..

But this time, calling `xrCreateInstance` returns `XR_ERROR_RUTIME_UNAVAILABLE` which is progress. Of course googling for such an error returns few results past the sections of the OpenXR specification. After a little searching however I find:
* Various github issues where android applications can't start, but none related to the quest
* Hints that when the screen / headset is off we'll get XR_ERROR_RUNTIME_UNAVAILABLE
* Hints that when the runtime has been stopped / contexts lost that we should then get XR_ERROR_RUNTIME_UNAVAILABLE

# Quest-specific OpenXR Loader

In the end this new error boils down to "The loader was unable to find or load a runtime", and if we pick through the [Oculus developer docs](https://developer.oculus.com/documentation/native/android/mobile-openxr/?locale=en_GB) we can find a one-liner saying "Meta provides a custom Android loader as the standard OpenXR Android loader is currently not supported on Meta Quest devices".

So while our API calls appear to be correct, we're using the wrong OpenXR loader / runtime. Perhaps your application wouldn't encounter this issue, but since vsgvr's cmake includes the loader by doing a `add_subdirectory(deps/openxr)` we're currently using the stock (Android) runtime rather than the Oculus-specific one.

So to correct this we need to:
* Download the [Oculus OpenXR SDK](https://developer.oculus.com/downloads/package/oculus-openxr-mobile-sdk/)
* Update vsgvr's CMake to not-use the default OpenXR loader, and instead import this one

That's a little frustrating as ti means we'll need separate binaries for the Quest and other Android-based setups, but since we're building vsgvr from source in the Android application anyway that's not a huge concern. The most complex part here is preserving support for the desktop / generic OpenXR builds, which we can do with a handy CMake option.

```cmake
if( VSGVR_XR_PLATFORM STREQUAL "OPENXR_GENERIC" )
  add_subdirectory( deps/openxr )
elseif( VSGVR_XR_PLATFORM STREQUAL "OPENXR_OCULUS_MOBILE" )
  if(NOT ANDROID)
    message(FATAL_ERROR "Oculus Mobile SDK is only supported when building for Android")
  endif()
  set( VSGVR_OCULUS_OPENXR_MOBILE_SDK "${CMAKE_CURRENT_SOURCE_DIR}/../ovr_openxr_mobile_sdk")
  if(NOT EXISTS ${VSGVR_OCULUS_OPENXR_MOBILE_SDK})
    message(FATAL_ERROR "Failed to find Oculus Mobile SDK, please download from https://developer.oculus.com/downloads/package/oculus-openxr-mobile-sdk/ and set VSGVR_OCULUS_OPENXR_MOBILE_SDK")
  endif()

  add_library(openxr_loader SHARED IMPORTED)
  set_property(TARGET openxr_loader PROPERTY IMPORTED_LOCATION
    "${VSGVR_OCULUS_OPENXR_MOBILE_SDK}/OpenXR/Libs/Android/${ANDROID_ABI}/${CMAKE_BUILD_TYPE}/libopenxr_loader.so")
  target_include_directories(openxr_loader INTERFACE 
    "${VSGVR_OCULUS_OPENXR_MOBILE_SDK}/OpenXR/Include"
    "${VSGVR_OCULUS_OPENXR_MOBILE_SDK}/3rdParty/khronos/openxr/OpenXR-SDK/include")
endif()
```

And does it work? Yes, sortof:
* The user isn't positioned at the origin (vsgvr is lacking in this aspect anyway)
* Controllers are in the wrong place, where they would be if the user was at the origin
* Controller rotation and translation are inverted somehow, despite that working on desktop OpenXR
* Pressing the Oculus button on the right controller triggers an illegal instruction

# Oculus Button Illegal Instruction

![Illegal Instruction on Input](/assets/img/2022-11-27/6-sigill-on-vr-input.png)

This happens only when the Oculus button on the right controller is pressed, and not for any other button on either controller, or headset.

But thankfully [someone else](https://github.com/guiltydogprods/VrCubeWorld_NativeActivityVS) has had the same issue, and:
* It only happens while debugging
* If you continue past the illegal instruction everything is fine again

`¯\_(ツ)_/¯` I guess we just won't press the Oculus button while debugging, slightly annoying for taking screenshots but oh well.

# Summary

While there's outstanding issues with controllers being in the wrong place, vsgvr is now able to run on the Quest 2, and the port only took around 4 hours to complete. As always the OpenXR specification is fairly clear and straightforward (though a little verbose), and other than having to use a different `openxr_loader.so` the development setup for Oculus isn't too bad.

That seems like a good place to leave it for now - The only bugs appear to be weird Android debugging quirks, and software errors within VSGVR itself. I expect things like controller positioning are programming errors in the application, since running with the desktop Oculus OpenXR runtime sees things operating correctly.

[Demo video](https://www.youtube.com/shorts/Pt-tV3IkL94)
