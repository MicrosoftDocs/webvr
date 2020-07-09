---
title: Input
description: Input is used to convert a cinematic experience into an compelling interactive one by allowing a user to alter the state of the world around them. 
author: leweaver
ms.author: leweaver
ms.date: 08/01/2017
ms.topic: article
keywords: WebVR Input
---

# Input
Input is one of the most difficult things to get right in virtual reality (VR), but is essential to creating a compelling interactive experience. Input is used to convert a cinematic experience into an interactive one by allowing a user to alter the state of the world around them. Interaction could be as simple as playing/pausing a video by gazing at a button, to a complex drag action requiring two tracked controllers.

In [WebVR 1.1](https://w3c.github.io/webvr/spec/1.1/), a site will most likely expect visitors to be limited to using only one of the following input methods due to their device.

- Mouse
- Gamepad
  - Traditional (Xbox Controller)
  - 3DOF (Pointing & buttons)
  - 6DOF (Pointing, Position and buttons)

Let's dig in to how to handle each input device and see how all these devices can be gathered from at once, providing a robust, multi-platform input handler.


## Mouse
Since a person wearing a head mounted display (HMD) can’t actually see what they are doing with an unrestricted mouse, it would be very easy for them to accidently click on things they don’t intend to i.e the browser close button, other windows, or the operating system interface. For this reason, Microsoft Edge disables unrestricted mouse input when an immersive VR session is started and the headset presence sensor is covered (you’ll see the Win+Alt+I banner visible on your monitor when this is the case). This is the behavior for all apps in Windows Mixed Reality and is not unique to WebVR.

The safest way to handle mouse input when in an immersive WebVR session is to use the [pointerlock API](https://developer.mozilla.org/en-US/docs/Web/API/Pointer_Lock_API). There are two events in the WebVR spec that are specifically designed to make this easy – [`vrdisplaypointerrestricted`](https://msdn.microsoft.com/en-us/library/mt801974) and [`vrdisplaypointerunrestricted`](https://msdn.microsoft.com/en-us/library/mt801975). They are fired at the appropriate times when the page enters VR, the HMD presence sensor detects changes, or any number of other reasons input focus would change.

To remain compatible with browsers that don't fire those events, it's still a good idea to also call [`requestPointerLock`](https://developer.mozilla.org/en-US/docs/Web/API/Element/requestPointerLock) just prior to calling [`requestPresent`](https://developer.mozilla.org/en-US/docs/Web/API/VRDisplay/requestPresent) for the user safety reasons mentioned above. 
 

The below snippet shows an example of safely using pointerlock to handle mouse down events. Here we see the pointerlock being requested just before calling `requestPresent` and in response to a `vrdisplayrestricted` event which is called whenever mouse input context changes from HMD to Monitor (such as uncovering the presence sensor on a Windows Mixed Reality headset).

<iframe height='300' scrolling='no' title='Pointerlock webvr' src='//codepen.io/MicrosoftEdgeDocumentation/embed/xXKagP/?height=300&theme-id=23761&default-tab=result&embed-version=2' frameborder='no' allowtransparency='true' allowfullscreen='true' style='width: 100%;'>See the Pen <a href='https://codepen.io/MicrosoftEdgeDocumentation/pen/xXKagP/'>Pointerlock webvr</a> by Microsoft Edge Docs (<a href='https://codepen.io/MicrosoftEdgeDocumentation'>@MicrosoftEdgeDocumentation</a>) on <a href='https://codepen.io'>CodePen</a>.
</iframe>

```javascript
// Globals that you would need to initialize properly in a real app - out of scope of this example.
let webglCanvas = null, vrDisplay = null;

/* Called once after page load. You would also initialize your WebGL context and resources here as well as . */
function initialize() {
  // Subscribe to event handlers as you require (handle clicks for experience logic, etc)
  webglCanvas.addEventListener('mousedown', (e) => console.log('Mouse Down'));

  // Register for the pointer restriction events, for browsers that disable unrestricted mouse when a headset has taken input
  window.addEventListener('vrdisplaypointerrestricted', () => webglCanvas.requestPointerLock(), false);
  window.addEventListener('vrdisplaypointerunrestricted', document.exitPointerLock(), false);

  // Fallback for browsers that do not fire unrestricted events: exit pointer lock when we stop presenting, as it is safe to use the mouse again.
  window.addEventListener('vrdisplaypresentchange', () => !vrDisplay.isPresenting && document.exitPointerLock(), false);
}

/* This method would be invoked when the user clicks the Enter VR button */
function tryRequestPresent() {
    
    // Request pointerlock manually for browsers that do not raise vrpointerrestricted events.
    // This method is not foolproof, since browsers reserve the right to eject the user from pointlock at any time.
    if (typeof(window.onvrdisplaypointerrestricted) === 'undefined')
      webglCanvas.requestPointerLock();

    vrDisplay.requestPresent([{source: webglCanvas}])
      .then(
        () => console.log('In VR'),
        () => console.error('Failed to enter VR')
      );
}
```

## Gamepad
Traditional, 0DOF, 3DOF, and 6DOF controllers are all exposed via the [`navigator.getGamepads`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/getGamepads) array. Since the [gamepad spec](https://w3c.github.io/gamepad/#dom-navigator-getgamepads) doesn't state anything about a minimum or maximum length for this array, nor the order in which elements appear, it's safest to assume all browsers implement it differently (and guess what, they do!).


It's especially important when dealing with gamepads to make no assumptions on the type, ordering, or number of devices that will be returned in the array. For example, Windows Mixed Reality users may be using [motion controllers](https://developer.microsoft.com/en-us/windows/mixed-reality/motion_controllers) or a gamepad to navigate the browser in VR; they will expect those controllers to continue working once they enter a WebVR immersive session. 

Your experience can determine the capability of all of the plugged in devices using the [`pose`](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad/pose) property of the gamepad. It's bad practice to look at the [`gamepad.id`](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad/id) property for this purpose.

```javascript
if (gamepad.mapping === 'standard')
  'Standard Gamepad';
else if (gamepad.pose && gamepad.pose.hasOrientation && gamepad.pose.hasPosition)
  '6DOF: Pointing and position';
else if (gamepad.pose && gamepad.pose.hasOrientation)
  '3DOF: Pointing only';
else
  '0DOF Clicker, or other';
```

### Controller buttons

Traditional gamepad button mappings are [documented](https://www.w3.org/TR/gamepad/#remapping) in the W3C specification. You can make sure a gamepad is using this configuration by reading the [`Gamepad.mapping`](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad/mapping) property:

```javascript
if (gamepad.mapping === 'standard') console.log(gamepad.id + ' is a traditional gamepad');
```

An example with traditional gamepad mapping can be seen in the following Babylon.js WebVR example. Pressing the `[0]` button of a gamepad starts the game, which corresponds to the `A` button of an Xbox controller.


<iframe height='300' scrolling='no' title='Babylon.js dino game using Babylon.GUI - WebVR' src='//codepen.io/MicrosoftEdgeDocumentation/embed/preview/RjgpJd/?height=300&theme-id=31247&default-tab=result&embed-version=2&editable=true' frameborder='no' allowtransparency='true' allowfullscreen='true' style='width: 100%;'>See the Pen <a href='https://codepen.io/MicrosoftEdgeDocumentation/pen/RjgpJd/'>Babylon.js dino game using Babylon.GUI - WebVR</a> by Microsoft Edge Docs (<a href='https://codepen.io/MicrosoftEdgeDocumentation'>@MicrosoftEdgeDocumentation</a>) on <a href='https://codepen.io'>CodePen</a>.
</iframe>


Things get interesting with so many types of VR controllers available since each one has different hardware configuration: number of buttons, thumbstick, touchpad etc. While there are no mappings defined in the spec for any of the VR controllers, we can base our approach on the knowledge that the [specification states](https://www.w3.org/TR/gamepad/#dfn-buttons): 

>  It's recommended that buttons appear in decreasing order of importance such that the primary button, secondary button, tertiary button, and so on appear as elements 0, 1, 2, ... in the buttons array

It's convention that the trigger on many 6DOF controllers is the primary, "most important" button. However, controllers in WebVR 1.1 always place the trigger at index `1` if it's present. Controllers that lack a trigger correctly place the "most important" button at index 0.

With the WebVR 1.1 API we can't reliably determine whether or not a controller has a trigger. For [point-and-commit and gaze-and-commit](https://developer.microsoft.com/en-us/windows/mixed-reality/gestures#gaze-and-commit]), it's recommended to listen for a primary "select" action on both button index `0` and `1`. 

>[!NOTE]
> Be careful with the button array size. The controller may not have a button defined at index `1`.

Known controllers and their mappings:

</br> | Windows Mixed Reality | Oculus Touch | Vive controller | Oculus Remote |
--- | --- | --- | --- | --- |
| **Control type** | **6DOF** | **6DOF** | **6DOF** | **3DOF** |
| **buttons[0]** | Thumbstick | Thumbstick | Touchpad | Inner Ring * |
| **buttons[1]** | Select * | Trigger * | Trigger * | Back |
| **buttons[2]** | Grasp | Grip | Grips | Outer Ring - Up |
| **buttons[3]** | Menu | A/X | Menu | Outer Ring - Down |
| **buttons[4]** | Touchpad | B/Y | | Outer Ring - Left |
| **buttons[5]** | | Surface | | Outer Ring - Right |
| **buttons[6]** | | Menu |
| **axes[0]** | Thumbstick X | Thumbstick X | Touchpad X |
| **axes[1]** | Thumbstick Y | Thumbstick Y | Touchpad Y |
| **axes[2]** | Touchpad X |
| **axes[3]** | Touchpad Y |

</br> | Daydream Controller | Oculus Go / GearVR controller | GearVR headset |
--- | --- | --- | --- |
| **Control type** | **3DOF** | **3DOF** | **3DOF (Head-mounted)** |
| **buttons[0]** | Touchpad * | Touchpad * | Touchpad * |
| **buttons[1]** | | Trigger |
| **axes[0]** | Touchpad X | Touchpad X |  Touchpad X |
| **axes[1]** | Touchpad Y | Touchpad Y |  Touchpad Y |

* __*__ _Primary Button_

The easiest way for your experience to support all gamepads is to use a simple gaze-and-commit interaction style, listening for button presses in either index `0` or `1` for the commit. Remember to be careful with array size - the controller may not actually have more than 1 button! Also adding support for point-and-commit will gives users even more of a choice. Supporting both point-and-commit and gaze-and-commit gives users choice of input device.

### Rendering controller models
The only time the [`gamepad.id`](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad/id) property is useful is for rendering a model of a your controller. You could, for example, use the string returned by that field to look up a .glTF file. If you take this approach, remember to include a generic default or fallback model in the event that a controller with an unforeseen id appears.

## Putting it all together

The following CodePen shows one way to listen for a primary action across multiple input sources. This could be incorporated into a wider gaze-and-commit system. 


The canvas will pulse a different colour depending on what input source the action came from:
 - Grey: Deactivated: No action detected
 - Red: Pointer (Mouse Click or Touchpad on the canvas element)
 - Blue: Keyboard Space Bar
 - Green: Gamepad

<iframe height='300' scrolling='no' title='Multi input detection' src='//codepen.io/MicrosoftEdgeDocumentation/embed/oGvPpM/?height=300&theme-id=23761&default-tab=result&embed-version=2' frameborder='no' allowtransparency='true' allowfullscreen='true' style='width: 100%;'>See the Pen <a href='https://codepen.io/MicrosoftEdgeDocumentation/pen/oGvPpM/'>Multi input detection</a> by Microsoft Edge Docs (<a href='https://codepen.io/MicrosoftEdgeDocumentation'>@MicrosoftEdgeDocumentation</a>) on <a href='https://codepen.io'>CodePen</a>.
</iframe>

If we take a closer look at the JavaScript behind the example, we see the APIs used to handle our inputs.

```javascript
function init() {
  
  // Defines what happens when user input is detected.
  myAction = new Action(
    function(source, active) { 
      // This is executed on any mouse button down, gamepad button down, or keyboard 'space' key down. 
      setState(active ? source : null); 
    });  
  
  // Register for mouse restriction events, so we can request pointerlock.
  // See https://codepen.io/MicrosoftEdgeDocumentation/pen/oGvPpM for more details on this pointerlock implementation
  window.addEventListener('vrdisplaypointerrestricted', () => webglCanvas.requestPointerLock(), false);
  window.addEventListener('vrdisplaypointerunrestricted', () => document.exitPointerLock(), false);  
  window.addEventListener('vrdisplaypresentchange', () => { if (!vrDisplay.isPresenting) document.exitPointerLock() }, false);
  
  // Register for mouse (any button) event handlers
  webglCanvas.addEventListener('mousedown', () => myAction.activate('mouse'));
  webglCanvas.addEventListener('mouseup', () => myAction.deactivate('mouse'));
  // Register for keyboard (space bar) 
  window.addEventListener("keydown", (event) => { if (event.key === ' ') { myAction.activate('keyboard'); event.preventDefault(); }});
  window.addEventListener("keyup",   (event) => { if (event.key === ' ') { myAction.deactivate('keyboard'); event.preventDefault(); }});
}

// Simple helper function; simply checks if either button at index 0 or 1 is pressed on the given gamepad.
function isSelectButtonPressed(gamepad) {
  if (!gamepad.buttons || !gamepad.buttons.length) return false;
  if (gamepad.buttons[0].pressed) return true;
  if (gamepad.buttons.length > 1 && gamepad.buttons[1].pressed) return true;
  return false;
}

// This method is called one per frame, we need to read gamepad state here since the Gamepad API relies on Polling.
function tick() {
  // You should make a new call to getGamepads each frame. Here we use feature detection to gracefully handle absence of gamepad support.
  var gamepads = navigator.getGamepads ? navigator.getGamepads() : [];
  var activate = false;
  for (var i = 0; i < gamepads.length; i++) {
    if (gamepads[i] && isSelectButtonPressed(gamepads[i])) {
      activate = true;
      break;
    }
  }

  if (activate) myAction.activate('gamepad'); 
  else myAction.deactivate('gamepad');
}
```