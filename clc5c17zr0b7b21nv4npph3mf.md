# How to improve performance of the Android Emulator on Windows

![How to improve performance of the Android Emulator on Windows](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091371315/Z2j7SFEh4y.jpeg)

In this article I'm going to share a quick tip about how to improve performance of the Android Emulator when running in Windows environment on a machine with a NVidia graphics card.

What we are going to do is instruct the NVIDIA driver to use the GPU to render the Android Emulator instead of leaving this task to the integrated graphics.

Requirements
------------

In order to use this tweak, you must have the following:  
1\. A Windows machine equipped with a NVIDIA graphics card. I'm using a Dell XPS 15 (9560).  
2\. The latest graphics drivers installed.  
3\. A recent version of the Android Emulator.

Steps
-----

**1\. Start the emulator**  
In order to ensure the emulator is found by the driver, you need to run it before executing the next steps  
**2\. Open NVIDIA Control Panel**  
Right click your desktop and select NVIDIA Control Panel. If the option is missing, update your graphics drivers.  
![How to improve performance of the Android Emulator on Windows](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091373244/GaooGl1Ko.png) **3\. Select `Manage 3D Settings` on the left menu**  
**4\. Switch from `Global Settings` to `Program Settings` on the right side panel**  
**6\. Manually add the emulator to the list of customizable programs**  
Click the _Add_ button then locate your emulator executable. It should appear as something like `{path_to_android_skd}\emulator\quemu\windows-x86_64\quemu-system-i386.exe`. Click _Add selected program_.  
**5\. Select the emulator in the _Select a program to customize_ dropdown list**  
**6\. Select _High Performance NVIDIA processor_ as the preferred graphics processor for this program**  
**7\. (optional) Tweak other settings for the graphics driver**  
Optionally you may want to change other settings for the program. For example, I keep _Power Management Mode_ to _Adaptive_ to prevent the fans from spinning for the emulator alone.  
**8\. Apply**  
**9\. Restart Emulator**

Bonus: GeForce Experience features
----------------------------------

To test that this works you can use the `GeForce Experience` features with the emulator running. For example you can enable the frame rate monitor on the top right or record clips of the "action". You can for that matter even broadcast your emulator on Twitch if you feel inclined to do so :) Access these features by pressing Alt+Z while the emulator is running - assuming GeForce Experience is installed of course.

Conclusion
----------

By default, the android emulator does not take advantage of a discrete graphics card. While the increase in framerate is moderate, I could find no downside to switching. If you try it, let me know your experience in the comments.