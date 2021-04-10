---
title: "Win10 Multi-monitor DPI Adjustment"
date: 2021-03-07T18:21:11+08:00
---

__Note: This article is mainly translated by Google translator, please turn to the Chinese version for more accurate expression if possible.__

Win10 currently has poor support for multiple different dpi displays. In the display settings, the canvas size of the display is determined by the resolution of the display rather than the physical size of the display. This leads to annoying misalignment when a window spans different monitors, and the mouse cannot be aligned when the mouse is moved across different monitors. At present, most of the tutorials are aligned by modifying the zoom and layout in the display settings, but this method does not solve the problem most of the time. One is that the optional zoom value is only a few integers, and the other is that the window will still be displayed when the window is spanned. Render at the native resolution of another monitor.

<center><embed src="uh.svg" style="width:500px;max-width:100%;" type="image/svg+xml" /></center>

There are various combinations of actual display sizes and resolutions. There are various problems with adjusting the zoom through display settings. Fortunately, there is a way to solve this problem: modify the display resolution through the graphics card. Although this will cause the monitor to not necessarily work point-to-point and display blurry, it is much better than the uncomfortable window scaling problem.

The following are the functions of NVIDIA graphics cards, and AMD graphics cards have not been tried. First, right-click on the desktop -> NVIDIA Control Panel, find Display -> Change Resolution on the left, and Customize -> Create Custom Resolution on the right to add a new resolution option for the monitor.

![fix](fix.png)

How should the resolution be calculated? First select a main screen, and set the other secondary screens so that everyone's dpi is equal. Assuming that the diagonal length of the main screen is $x$, the resolution width is $a$, and the diagonal length of the secondary screen to be adjusted is $y$, then the resolution width of the secondary screen is $b=a\times\frac{y}{x }$. The high resolution can be calculated according to the screen ratio. After calculating the equivalent dpi resolution of the secondary screen, add the resolution in the graphics card settings, then select the new resolution in the Win10 display settings, and then adjust the zoom to the same as the main screen, you can enjoy the pleasure of alignment.

<center><embed src="fine.svg" style="width:500px;max-width:100%;" type="image/svg+xml" /></center>

Unable to point-to-point resolution will cause the font display of the secondary screen to be blurred. Search for "clear type" in the start menu and find "Adjust Clear Type text". Do a correction on the secondary screen to alleviate this problem.
