# WebXR Front-facing Camera API - explainer

## Introduction

Some XR device form factors, most notably smartphones, have multiple cameras that can be used to power an immersive (generally AR) experience. The mobile AR frameworks, such as ARCore & ARKit, allow developers to configure the AR session by selecting the camera to be used; however, this configuration knob is currently unavailable in WebXR. The front-facing camera API changes that by enabling the sites to express their preference to use a front-facing camera when creating immersive sessions.

### Note on terminology

“Front-facing camera” is equivalent to “user-facing camera”. On smartphones, this is the camera that the users would usually use for capturing selfies. “Back-facing camera” is equivalent to “environment-facing camera”. On smartphones, this is the camera that would usually be capturing the users’ environment. This is also the camera that we expect implementations currently use by default when a user enters an immersive augmented reality session on a supported smartphone. For a more visual description, see [VideoFacingModeEnum](https://w3c.github.io/mediacapture-main/#dom-videofacingmodeenum).

## Use cases

The front-facing camera API allows the sites to enter Augmented Reality mode using a camera that will normally be facing the user. This should allow the sites to use other existing WebXR APIs to interact with the user’s environment, and potentially directly with the user.

The use cases for the API are going to be focused around creating experiences where a site can interact with the user by leveraging existing APIs (like hit-test, depth, lighting estimation). This can be for example:
- Games where users can interact with virtual objects (most likely by leveraging tracking, depth sensing and/or hit-test APIs): freestyle football using the head,fruit ninja.
- E-commerce (leveraging tracking, depth sensing, and/or hit-test APIs): virtual object placement behind / alongside the user (e.g. for size comparison with furniture or to align the user with virtual objects like clothing), redecorating / repainting the home with the user visible in the preview.
- Real-time communications with information about the user’s environment: background blur (using depth information), augmenting camera feed by adding virtual objects (parrot on a shoulder, [funny hats](https://www.w3.org/TR/webrtc-nv-use-cases/#funnyhats)).

### Non-goals

Some experiences listed above may become easier with the existence of face tracking API. This module is solely about creating a session that uses the front-facing camera. Other existing and upcoming APIs (e.g. face-tracking, expression tracking) are explicit non-goals and will be developed in separate modules, but should not conflict with this API. Additionally, this API does not intend to allow simultaneous access to multiple cameras on the device.

## How to use the API

The entry point to the feature is at session creation. The site can express desire to create an AR session using front-facing camera by adding a `“front-facing”` feature descriptor to the list of required or optional features:

```js
const session = await navigator.xr.requestSession( "immersive-ar", {
  requiredFeatures: ["front-facing"],
});
```

By default, the camera image feed will contain mirrored images (both when returning them from the raw camera API, and when XR compositor composes them for display to the user), and the projection matrix returned from `XRView.projectionMatrix` will include a horizontal flip. This default is in place as it represents the common user expectation (e.g. selfie camera preview shows mirrored image). This places additional burden on the site developers, as they need to ensure that the winding order of the triangles they render is taking into account the horizontal flip encoded in the projection matrix (e.g. by configuring the front-facing polygons by calling `glFrontFace()` with appropriate parameter) and that any text rendering / textures used are taking into account the flip.

The default behavior of the API can be changed by explicitly configuring it at session creation. This can be achieved by passing extra information to `requestSession()` call:

```js
const session = await navigator.xr.requestSession("immersive-ar", {
  requiredFeatures: ["front-facing"],
  frontFacing: {
    displayOrientation: "unmirrored",  // default is "mirrored"
  }
});
```

Note that this configuration knob may not be strictly necessary to launch in the initial version of the API - we should be able to add it afterwards after getting signals from developers related to the API ergonomics. Such an addition should not cause a breaking change.

Since the front-facing camera feature’s API surface is minimal and only applies at session creation, there is no way to find out whether the feature is enabled or not after the session has been created. This poses a problem when it was requested as an optional feature, as the session creation can succeed even if the front-facing camera feature is not supported. To learn whether the feature is active on a session, the site can inspect the `session.enabledFeatures` collection:

```js
const is_front_facing_enabled = session.enabledFeatures.includes("front-facing");
```

### Note on session creation

When fulfilling a session creation request, the user agent is free to select a device that does not support some (or all) optional features requested by the site. This choice is based, among other things, on whether available hardware and underlying XR frameworks support the requested features. Since requesting a front-facing camera feature may cause other features to become unavailable, care must be taken by the site to ensure that the collections of required and optional features are adequately conveying the requirements of the application. For example, let’s consider a hypothetical device that supports depth-sensing in rear-facing camera mode, and also supports front-facing camera without depth-sensing. This leads to a couple of possible cases:

```js
const session = await navigator.xr.requestSession("immersive-ar", {
  requiredFeatures: ["front-facing", "depth-sensing"]});  // #1, fails

const session = await navigator.xr.requestSession("immersive-ar", {
  requiredFeatures: ["front-facing"]});  // #2, succeeds, will use front-facing camera

const session = await navigator.xr.requestSession("immersive-ar", {
  requiredFeatures: ["depth-sensing"]});  // #3, succeeds, will use rear-facing camera

const session = await navigator.xr.requestSession("immersive-ar", {
  requiredFeatures: ["depth-sensing"],
  optionalFeatures: ["front-facing"]});  // #4, succeeds, will use rear-facing camera

const session = await navigator.xr.requestSession("immersive-ar", {
  optionalFeatures: ["front-facing", "depth-sensing"]});  // #5, can succeed, will use ???
```

The last example (#5) from above is notable, since there is currently no tie-breaking mechanism specified when a session creation can satisfy multiple optional features, but not at the same time. A conformant implementation could for example prioritize features occurring earlier in the `optionalFeatures` sequence (with the assumption that the ones occurring earlier in the sequence are more important to the site), or attempt to create a session that maximizes the number of optional features enabled.

Open question: should core spec mandate tie-breaking mechanism?

## Rationale / alternatives considered

This section is mostly relevant for future implementers, as it focuses on the design of the API. It is expected that this (and the rest of the explainer) will evolve as the feature is discussed in the Immersive Web CG/WG.

### Session creation

The most important design goal was to ensure that extending the WebXR API by adding the front-facing camera feature won’t cause us to collide with already existing, or yet-to-come features. Most notably, there is currently a related feature being discussed in the Immersive Web CG/WG (expression tracking), and there have been some previous attempts at enabling face tracking API through WebXR. The addition of a new feature descriptor would mean that:
- If we ever add a feature that directly conflicts with the front-facing camera feature (naive example: `“back-facing”`), there should be no problem introduced (i.e. this would not be a breaking change as there should be no sites that used a non-existent feature descriptor, and the session won’t be created with conflicting features requested).
- If we ever add a feature that also requires the use of a front-facing camera of the device (e.g. face tracking / expression tracking), this should also be fine - those features won’t be conflicting with each other. The main risk here is that a site could use a hypothetical `“face-tracking”` feature as a proxy to force us to use a front-facing camera, and not because it actually wanted to use face tracking capabilities.
- If there are existing features that conflict with the front-facing camera feature, the WebXR Device API already provides a mechanism to resolve this. Unsatisfiable set of required features will be rejected, and an unsatisfiable set of optional features should cause the user agent to still create a session, albeit with unspecified features (which the site should tolerate, since it listed those features as not critical to the experience).

The above discussion also serves as the argument for treating `“front-facing”` as a new feature descriptor (as opposed to a configuration knob), as we want it to be able to participate in the feature resolution algorithm. Were it a configuration knob (either exposed at session creation, or afterwards), it could force the site to re-try session creation attempt if front-facing camera is only optionally needed (example #4 from [above](note-on-session-creation) would not be expressible).

Open question: should we also add a new feature descriptor for back-facing camera? Given that there is an existing feature incubation for expression tracking, which could theoretically be implemented for smartphone AR in front-facing camera mode, the sites could want to force the use of an environment-facing camera. Alternatives would be to never use a front-facing camera when the sites didn’t explicitly ask for it via the `”front-facing”` feature descriptor (i.e. we won’t be able to add a feature that’d imply the use of front-facing camera), or treat `”front-facing”` feature descriptor as implicitly present when requesting such features. Implicitly adding the feature descriptor comes with its own set of problems though (i.e. it’d force all the sites to always check whether they use the camera they needed, which could be a back-compat issue).

#### Simultaneous access to cameras

In the future, it may be possible to leverage multiple cameras to power the AR sessions on smartphones, and the underlying AR frameworks may make it possible to access the data coming from multiple cameras. The API shape proposed above should at the very least not conflict with this capability. One way to implement simultaneous access to cameras would be to treat the `”front-facing”` feature descriptor as a modifier that only influences the camera that will be selected for the immersive experience (i.e. camera selection for primary views), and treat other camera streams as secondary views, which should already be covered by `”secondary-views”` [feature](https://www.w3.org/TR/webxr/#secondary-view-secondary-views).

#### Alternative #1 - camera facing is a configuration knob

```webidl
enum XRCameraFacing {
  "back",
  "front",
};

dictionary XRCameraConfigInit {
  optional XRCameraFacing facing = "back",
  optional XRCameraDisplayOrientation displayOrientation = "mirrored",
};

partial dictionary XRSessionInit {
  XRCameraConfigInit cameraConfig;
};
```


Pros:
- Makes it explicit which camera will be used if the session got created.
- Requested features do not influence the choice of the camera.

Cons:
- In case the front-facing camera is requested optionally and not supported, forces the site to attempt to request the session again.

#### Alternative #2 - introduce new XRSessionMode

We can also introduce a new `XRSessionMode` instead of relying on the existing ones and adding a new feature descriptor. One option would be to introduce `”smartphone-ar”` session mode as some features (such as front-facing camera) may be more tailored for smartphones. Another option would be to introduce a `”front-facing-ar”` session mode (name TBD).

Pros:
- Makes it explicit which camera will be used if the session got created.
- Requested features do not influence the choice of the camera.

Cons:
- In case the front-facing camera is requested optionally and not supported, forces the site to attempt to request the session again.
- Introduces additional burden on the developers to create experiences that could target different session modes - some developers may decide to only target the new session mode.

### Display orientation configuration
When exposing front-facing camera API, we face the choice of using the camera images as-is, or mirroring them. Users will expect to see a mirrored camera stream when in selfie camera mode (citation needed 🙂). This should be a reasonable default, and should not negatively affect the use cases (if the site needs to access the camera image, it can invert the mirroring itself). This assumes that the site is fine with mirrored camera stream being used for compositing - in cases where that is undesirable, it would have to use raw camera access API to access the image, and render the un-mirrored version on top of the image rendered by XR compositor. To address this potential problem, a `frontFacing` dictionary was added to session initialization parameters. This dictionary then allows us to expose a configuration knob to tweak the orientation of the camera image used by the XR compositor (and by raw camera access API), and also influences the `XRView.projectionMatrix` that will be returned (by not introducing a horizontal flip).

## Appendix: Proposed WebIDL

```webidl
// New feature descriptor: "front-facing".

enum XRCameraDisplayOrientation {
  "mirrored",
  "unmirrored", // name TBD
};

dictionary XRFrontFacingCameraInit {
  optional XRCameraDisplayOrientation displayOrientation = "mirrored",
};

partial dictionary XRSessionInit {
  XRFrontFacingCameraInit frontFacing;
};
```

## Appendix: References

ARKit front-facing camera support:
https://developer.apple.com/documentation/arkit/choosing_which_camera_feed_to_augment.

ARCore camera facing enum, which can be used to configure the camera: 
https://developers.google.com/ar/reference/java/com/google/ar/core/CameraConfig.FacingDirection
