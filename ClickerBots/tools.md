# Tools

## Programming Language

[**AutoIt**](https://www.autoitscript.com/site/autoit) is one of the most popular [**scripting programming languages**](https://en.wikipedia.org/wiki/Scripting_language) for writing clicker bots. It has a lot of features that facilitate development of automation scripts:

1. Easy to learn syntax.
2. Detailed on-line documentation and large community-based support forums.
3. Smooth integration with [**WinAPI**](https://en.wikipedia.org/wiki/Windows_API) functions and third-party libraries.
4. Built-in source code editor.

AutoIt is an excellent tool to start with programming. If you already have some experience with another programming language like C++, C#, Python, etc, you can use this language to implement examples from this chapter. Relevant WinAPI functions that are used by AutoIt will be mentioned.

[**AutoHotKey**](http://ahkscript.org) is a second scripting programming language that can be recommended for starting with game bots development. It has most of AutoIt features but the syntax of this language is more unique. Some things will be simpler to implement with AutoHotKey than with AutoIt. But AutoHotKey language may be slightly more difficult to learn.

There are a lot of examples and guides about development of game bots with both AutoIt and AutoHotKey languages on the Internet. Thus, you are free to choose a tool that you prefer. We will use AutoIt language in this chapter.

## Image Processing Libraries

AutoIt language itself has a powerful support of the image analyzing methods. But there are two third-party libraries that will be extremely helpful for this domain:

1. [**ImageSearch**](https://www.autoitscript.com/forum/topic/148005-imagesearch-usage-explanation) library allows you to search a specified picture sample in the window of a game application.

2. [**FastFind**](https://www.autoitscript.com/forum/topic/126430-advanced-pixel-search-library/) library provides advanced methods for searching regions in the window of a game application. You can specify a number of [**pixels**](https://en.wikipedia.org/wiki/Pixel) of the given color. FastFind library returns the regions that contains closest number of these pixels. Also the library allows you to find a pixel of the given color that is closest to the given point.

## Image Analyzing Tool

Possibility to check parameters of an image (like pixel color or pixel coordinates) is very helpful for developer of clicker bots. This task appears when you try to debug a bot application and check the correctness of image processing algorithms.

There are plenty of tools that allow you to take color of pixels from the screen and to get current coordinates of mouse cursor.  You can easily find these tools with Google. I use the [**ColorPix**](https://www.colorschemer.com/colorpix_info.php) application that solves debugging tasks perfectly.

## Source Code Editors

AutoIt language is distributed with the customized version of SciTE editor. It is great editor for programming and debugging AutoIt scripts. But more universal editors like [**Notepad++**](https://notepad-plus-plus.org) are more suitable if you use another programming language like Python or AutoHotKey. [**Microsoft Visual Studio**](https://www.visualstudio.com/en-us/products/visual-studio-express-vs.aspx) is the best choice for developers who prefer C++ and C# languages.

## API Hooking

We will develop example applications using high level AutoIt language. The language encapsulates calls of WinAPI functions in the simplified interface. But it is necessary to know which WinAPI functions have been actually used by internals of AutoIt. This allows you to understand algorithms better. Moreover, when you know exact WinAPI function that is used, you can interact with it directly using your favorite programming language.

There are a lot of tools that provide WinAPI calls [**hooking**](https://en.wikipedia.org/wiki/Hooking). I use freeware [**API Monitor v2**](http://www.rohitab.com/apimonitor) application. It allows you to filter all hooked calls, to gather information of the process, to decode input and output parameters of called functions and to view process memory. Full list of features is available on developers website.
