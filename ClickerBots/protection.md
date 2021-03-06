# Protection Approaches

This section covers approaches to develop protection systems against clicker bots. Most effective protection systems are separated into two parts. One part is launched on client-side. This allows us to control points of interception and embedding data, which are related to devices, OS and a game application. Server-side part of the protection system allows us to control a communication between a game application and a game server. Most algorithms for clicker bots detection are able to work on client-side only.

Main purpose of the protection system is to detect a fact of the bot application usage. There are several variants of reaction on this detection:

1. Write a warning message about the suspicious player account to the server-side log file.
2. Interrupt current connection between the suspicious player and the game server.
3. Ban an account of the suspicious player and prevent its future connection to the game server.

We will focus on bots detection algorithms only. Also ways to overcome these algorithms will be discussed. Reaction on the bots detection will not considered here.

## Test Application

We will test our examples of protection systems on Notepad application. The tested example should detect an AutoIt script that types a text in the Notepad window. Our examples are implemented in AutoIt language too. This approach helps us to make source code shorter and clear to understand. But C++ language is used to develop real protection systems in most cases.

This is a [`SimpleBot.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/SimpleBot.au3) script that types "a", "b" and "c" letters consistently in the Notepad window:
```AutoIt
$hWnd = WinGetHandle("[CLASS:Notepad]")
WinActivate($hWnd)
Sleep(200)

while true
    Send("a")
    Sleep(1000)
    Send("b")
    Sleep(2000)
    Send("c")
    Sleep(1500)
wend
```
Let us assume that each letter represents some action of bot in game application window. Now you can launch Notepad and the `SimpleBot.au3` script. The script will start to type letters in the Notepad window in an infinite loop. This is a start point for our research of protection systems. Purpose of each example protection system is to detect the launched `SimpleBot.au3` script. These examples should distinguish legal user actions and simulated by a bot actions.

## Analysis of Actions

There are several obvious regularities in the `SimpleBot.au3` script. Our protection system can analyze the performed actions and make conclusion about usage of a bot. First regularity is time delays between the actions. User does not have a possibility to repeat his actions with very precise delays. Protection algorithm can measure delays between the actions of one certain type. There is very high probability that the actions are simulated by a bot in case the delays between them are less than 100 milliseconds. Now we will implement protection algorithm, which is based on this time measurement.

The protection system should solve two tasks: capture user's actions and analyze them. This is a code snippet to capture pressed keys:
```AutoIt
global const $gKeyHandler = "_KeyHandler"

func _KeyHandler()
    $keyPressed = @HotKeyPressed

    LogWrite("_KeyHandler() - asc = " & asc($keyPressed) & " key = " & $keyPressed)
    AnalyzeKey($keyPressed)

    HotKeySet($keyPressed)
    Send($keyPressed)
    HotKeySet($keyPressed, $gKeyHandler)
endfunc

func InitKeyHooks($handler)
    for $i = 0 to 256
        HotKeySet(Chr($i), $handler)
    next
endfunc

InitKeyHooks($gKeyHandler)

while true
    Sleep(10)
wend
```
We use the [`HotKeySet`](https://www.autoitscript.com/autoit3/docs/functions/HotKeySet.htm) AutoIt function here to assign a **handler** or **hook** for pressed keys. This assignment is happen in the `InitKeyHooks` function. The `_KeyHandler` function is a handler that is called, when any key with ASCII codes from 0 to 255 is pressed. There are several actions in the `_KeyHandler`:

1. Pass the pressed key to the `AnalyzeKey` function. The pressed key is available by `@HotKeyPressed` macro.
2. Disable the `_KeyHandler` by the `HotKeySet($keyPressed)` call. This is needed to send the captured key to the Notepad window.
3. Send the pressed key to the Notepad window by the `Send` function.
4. Enable the `_KeyHandler` again by the `HotKeySet($keyPressed, $gKeyHandler)` call.

This is a code of the `AnalyzeKey` function:
```AutoIt
global $gTimeSpanA = -1
global $gPrevTimestampA = -1

func AnalyzeKey($key)
    local $timestamp = (@SEC * 1000 + @MSEC)
    LogWrite("AnalyzeKey() - key = " & $key & " msec = " & $timestamp)
    if $key <> 'a' then
        return
    endif

    if $gPrevTimestampA = -1 then
        $gPrevTimestampA = $timestamp
        return
    endif

    local $newTimeSpan = $timestamp - $gPrevTimestampA
    $gPrevTimestampA = $timestamp

    if $gTimeSpanA = -1 then
        $gTimeSpanA = $newTimeSpan
        return
    endif

    if Abs($gTimeSpanA - $newTimeSpan) < 100 then
        MsgBox(0, "Alert", "Clicker bot detected!")
    endif
endfunc
```
Time spans between pressing of the "a" key are measured here. We can use a **trigger action** term to name this pressing. There are two global variables for storing current state of the protection algorithm:

| Name | Description |
| -- | -- |
| `gPrevTimestampA` | [**Timestamp**](https://en.wikipedia.org/wiki/Timestamp) of the last happening trigger action |
| `gTimeSpanA` | Time span between last two trigger actions |

Both these variables have `-1` value on a startup. This matches to the uninitialized state. This algorithm is able to make conclusion about bot usage after minimum three trigger actions. The first action is needed for the `gPrevTimestampA` variable initialization:
```AutoIt
    if $gPrevTimestampA = -1 then
        $gPrevTimestampA = $timestamp
        return
    endif
```
Timestamp of the second action is used to calculate the `gTimeSpanA` variable. This variable equals to a subtraction between timestamps of the current and previous actions:
```AutoIt
    local $newTimeSpan = $timestamp - $gPrevTimestampA
    $gPrevTimestampA = $timestamp

    if $gTimeSpanA = -1 then
        $gTimeSpanA = $newTimeSpan
        return
    endif
```
The third action is used to calculate the new time span and compare it with the previous one, which is stored in the `gTimeSpanA` variable:
```AutoIt
    if Abs($gTimeSpanA - $newTimeSpan) < 100 then
        MsgBox(0, "Alert", "Clicker bot detected!")
    endif
```
We have measured two time spans here:

1. Time span between the first and the second trigger actions.
2. Time span between second and third trigger actions.

If subtraction of these two time spans is less than 100 milliseconds, user is able to repeat his actions with precision of 100 milliseconds. It is impossible for human but absolutely normal for a bot. Therefore, the protection system concludes that these actions have been simulated by a bot. The message box with "Clicker bot detected!" text is displayed in this case.

This is full code of the [`TimeSpanProtection.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/TimeSpanProtection.au3) script with skipped content of `_KeyHandler` and `AnalyzeKey` functions:
```AutoIt
global const $gKeyHandler = "_KeyHandler"
global const $kLogFile = "debug.log"

global $gTimeSpanA = -1
global $gPrevTimestampA = -1

func LogWrite($data)
    FileWrite($kLogFile, $data & chr(10))
endfunc

func _KeyHandler()
    ; SEE ABOVE
endfunc

func InitKeyHooks($handler)
    for $i = 0 to 256
        HotKeySet(Chr($i), $handler)
    next
endfunc

func AnalyzeKey($key)
    ; SEE ABOVE
endfunc

InitKeyHooks($gKeyHandler)

while true
    Sleep(10)
wend
```
We can improve our `SimpleBot.au3` script to avoid the considered protection algorithm. The simplest way is to add random delays between bot actions. This is fixed version of the bot with the [`RandomDelayBot.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/RandomDelayBot.au3) name:
```AutoIt
SRandom(@MSEC)
$hWnd = WinGetHandle("[CLASS:Notepad]")
WinActivate($hWnd)
Sleep(200)

while true
    Send("a")
    Sleep(Random(800, 1200))
    Send("b")
    Sleep(Random(1700, 2300))
    Send("c")
    Sleep(Random(1300, 1700))
wend
```
Combination of the `SRandom` and the `Random` AutoIt functions is used here to calculate delay time. You can launch `TimeSpanProtection.au3` script and then `RandomDelayBot.au3` one. The protection system is not able to detect the bot in this case.

The bot has the second regularity, which allow us to detect the `RandomDelayBot.au3` script. The regularity is simulated actions itself. The bot repeats actions "a", "b" and "c" cyclically. There is very low probability that an user will repeat these actions in the same order constantly.

This is a code snippet of the [`ActionSequenceProtection.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/ActionSequenceProtection.au3) script with the new version of `AnalyzeKey` function. This function checks repeating sequence of the captured actions:
```AutoIt
global const $gActionTemplate[3] = ['a', 'b', 'c']
global $gActionIndex = 0
global $gCounter = 0

func Reset()
    $gActionIndex = 0
    $gCounter = 0
endfunc

func AnalyzeKey($key)
    LogWrite("AnalyzeKey() - key = " & $key);

    $indexMax = UBound($gActionTemplate) - 1
    if $gActionIndex <= $indexMax and $key <> $gActionTemplate[$gActionIndex] then
        Reset()
        return
    endif

    if $gActionIndex < $indexMax and $key = $gActionTemplate[$gActionIndex] then
        $gActionIndex += 1
        return
    endif

    if $gActionIndex = $indexMax and $key = $gActionTemplate[$gActionIndex] then
        $gCounter += 1
        $gActionIndex = 0

        if $gCounter = 3 then
            MsgBox(0, "Alert", "Clicker bot detected!")
            Reset()
        endif
    endif
endfunc
```
This is a list of global variables and constants that are used in this algorithm:

| Name | Description |
| -- | -- |
| `gActionTemplate` | List of actions in the sequence that should be unique for a bot script |
| `gActionIndex` | Index of the last captured action according to the `gActionTemplate` list |
| `gCounter` | Number of repetitions of the actions sequence |

The `AnalyzeKey` function processes three cases of matching current captured action and elements of the `gActionTemplate` list. First case happens when the captured action does not match any element of the `gActionTemplate` list:
```AutoIt
    $indexMax = UBound($gActionTemplate) - 1
    if $gActionIndex <= $indexMax and $key <> $gActionTemplate[$gActionIndex] then
        Reset()
        return
    endif
```
We call the `Reset` function in this case. This function resets to zero both `gActionIndex` and `gCounter` variables. 

Second case of the `AnalyzeKey` function happens when the captured action matches to an element of the `gActionTemplate` list. Also this element is not last one of the list and element's index equals to the `gActionIndex` variable:
```AutoIt
    if $gActionIndex < $indexMax and $key = $gActionTemplate[$gActionIndex] then
        $gActionIndex += 1
        return
    endif
```
Value of the `gActionIndex` variable is incremented in this case. 

Last `if` condition of the `AnalyzeKey` function checks the case when the captured action equals to the last element of the `gActionTemplate` list:
```AutoIt
    if $gActionIndex = $indexMax and $key = $gActionTemplate[$gActionIndex] then
        $gCounter += 1
        $gActionIndex = 0

        if $gCounter = 3 then
            MsgBox(0, "Alert", "Clicker bot detected!")
            Reset()
        endif
    endif
```
The `gCounter` is incremented and `gActionIndex` reset to zero in this case. After these actions our algorithm is ready to analyze next sequence of the player's actions. When the predefined in the `gActionTemplate` list sequence of actions is detected three times, the protection system concludes that player uses a bot application. The `gCounter` variable equals to 3 and a message box with the "Clicker bot detected!" text is displayed in this case. Then `Reset` function is called and  the protection system becomes ready to detect a bot again.

You can launch the `ActionSequenceProtection.au3` and `RandomDelayBot.au3` scripts. New protection system is able to detect the bot with random delays between simulated actions. 

The described approach with analysis of actions sequence can lead to false positives. This means that protection system detects a bot application when actually the player repeats the same actions without any bot. To reduce a possibility of false positives you can increase the maximum allowable value of the `gCounter` in this `if` condition:
```AutoIt
        if $gCounter = 3 then
            MsgBox(0, "Alert", "Clicker bot detected!")
            Reset()
        endif
```
Also you can improve the considered protection approach and analyze actions without a predefined actions sequence. Protection system can accumulate all user's actions and search frequently repeated regularities. These regularities warn about a possible usage of a clicker bot.

We can improve our bot script further to avoid the protection systems that searches the actions regularities. This is a [`RandomActionBot.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/RandomActionBot.au3) script:
```AutoIt
SRandom(@MSEC)
$hWnd = WinGetHandle("[CLASS:Notepad]")
WinActivate($hWnd)
Sleep(200)

while true
    Send("a")
    Sleep(1000)
    if Random(0, 9, 1) < 5 then
        Send("b")
        Sleep(2000)
    endif
    Send("c")
    Sleep(1500)
wend
```
The idea of this improvement is to perform simulated actions irregularly. The action "b" is simulated by this bot with 50% probability. This break the conditions of the `AnalyzeKey` function of the protection system. Thus, the `ActionSequenceProtection.au3` script is not able to detect the `RandomActionBot.au3` one.

## Process Scanner

Another approach to detect clicker bots is to analyze a list of the launched processes. If you know a name of the bot application, you can find it in this list.

This is a [`ProcessScanProtection.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/ProcessScanProtection.au3) script, which scans the list of launched processes:
```AutoIt
global const $kLogFile = "debug.log"

func LogWrite($data)
    FileWrite($kLogFile, $data & chr(10))
endfunc

func ScanProcess($name)
    local $processList = ProcessList($name)

    if $processList[0][0] > 0 then
        LogWrite("Name: " & $processList[1][0] & " PID: " & $processList[1][1])
        MsgBox(0, "Alert", "Clicker bot detected!")
    endif
endfunc

while true
    ScanProcess("AutoHotKey.exe")
    Sleep(5000)
wend
```
List of the launched processes is available via [`ProcessList`](https://www.autoitscript.com/autoit3/docs/functions/ProcessList.htm) AutoIt function. This function has optional input parameter with a process name to search. The `AutoHotKey.exe` process name is passed in our example. The `ProcessList` function returns a two dimensional array. This is description of elements in this array:

| Element | Description |
| -- | -- |
| `$processList[0][0]` | The number of processes in the array |
| `$processList[1][0]` | Process name |
| `$processList[1][1]` | Process ID (PID) |

If the `$processList[0][0]` element is greater than zero, the `AutoHotKey.exe` process is launched now.

Why we are looking for the `AutoHotKey.exe` process instead of the `AutoIt.exe` one? There is a problem with testing the `ProcessScanProtection.au3` script. This script is written in the AutoIt language. Therefore, the `AutoIt.exe` process of AutoIt [**interpreter**](https://en.wikipedia.org/wiki/Interpreted_language) is started when you launch the script. This means that the protection system detects self instead of the `SimpleBot.au3` script. But we can implement the bot algorithm in AutoHotKey language. This is the [`SimpleBot.ahk`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/SimpleBot.ahk) script:
```AutoHotKey
WinActivate, Untitled - Notepad
Sleep, 200

while true
{
    Send, a
    Sleep, 1000
    Send, b
    Sleep, 2000
    Send, c
    Sleep, 1500
}
```
You can compare this script with the `SimpleBot.au3` one. These scripts look very similar. There are minor differences in the syntax to call functions. You should specify input parameters of the function after a comma in AutoHotKey. Names of used functions are the same as AutoIt ones.

Now we are ready to test our protection system example. These are the steps to do it:

1. Launch the Notepad application.
2. Launch the `ProcessScanProtection.au3` script.
3. Launch the `SimpleBot.ahk` script. Check that AutoHotKey interpreter is installed in your system.
4. Wait until the protection system detects the `SimpleBot.ahk` script.

You see the message with "Clicker bot detected!" text when the bot script is detected.

It is very simple to avoid this kind of protection systems. The most straightforward way is to use AutoHotKey compiler. This compiler allows you to get executable binary file from the specified AutoHotKey script. The name of bot process differs from the `AutoHotKey.exe` one if you compile the script and launch it.

These are steps to create `SimpleBot.exe` executable file from the `SimpleBot.ahk` script:

1. Launch the AutoHotKey compiler application. Path of this application is `C:\Program Files (x86)\AutoHotkey\Compiler\Ahk2Exe.exe` by default.
2. Select the `SimpleBot.ahk` script as a "Source (script file)" parameter in the "Required Parameters" panel.
3. Leave the "Destination (.exe file)" parameter empty in the "Required Parameters" panel. This means that resulting executable file will be created in the directory of the source script.
4. Press the "> Convert <" button.

This is a screenshoot of the AutoHotKey compiler window:

![AutoHotKey Compiler](ahk2exe.png)

You will get a message box with the "Conversion complete" text when compilation is finished.

Now you can launch the generated `SimpleBot.exe` file instead of the `SimpleBot.ahk` script. The `ProcessScanProtection.au3` system is not able to detect the bot anymore. This happens because now there is a process with `SimpleBot.exe` name instead of the `AutoHotKey.exe` one.

How we can improve the `ProcessScanProtection.au3` script to detect compiled version of the bot? It is very simple to change a name of binary file. But it is more difficult to change its content. There are a lot of ways to distinguish a file by its content. These are just several ideas to do it:

1. Calculate a [**hash sum**](https://en.wikipedia.org/wiki/Checksum) for content of the files and compare it with the predefined value.
2. Check a sequence of bytes in the specific place of the file.
3. Search a specific byte sequence or string in the file.

This is a [`Md5ScanProtection.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/Md5ScanProtection.au3) script that calculates a [**MD5**](https://en.wikipedia.org/wiki/MD5) hash sum for executable files of all launched processes and detects the bot:
```AutoIt
#include <Crypt.au3>

global const $kLogFile = "debug.log"
global const $kCheckMd5[2] = ["0x3E4539E7A04472610D68B32D31BF714B", _
                              "0xD960F13A44D3BD8F262DF625F5705A63"]

func LogWrite($data)
    FileWrite($kLogFile, $data & chr(10))
endfunc

func _ProcessGetLocation($pid)
    local $proc = DllCall('kernel32.dll', 'hwnd', 'OpenProcess', 'int', _
                          BitOR(0x0400, 0x0010), 'int', 0, 'int', $pid)
    if $proc[0] = 0 then 
        return ""
    endif
    local $struct = DllStructCreate('int[1024]')
    DllCall('psapi.dll', 'int', 'EnumProcessModules', 'hwnd', $proc[0], 'ptr', _
            DllStructGetPtr($struct), 'int', DllStructGetSize($struct), 'int_ptr', 0)

    local $return = DllCall('psapi.dll', 'int', 'GetModuleFileNameEx', 'hwnd', _
                            $proc[0], 'int', DllStructGetData($struct, 1), 'str', _
                            '', 'int', 2048)
    if StringLen($return[3]) = 0 then
        return ""
    endif
    return $return[3]
endfunc

func ScanProcess()
    local $processList = ProcessList()
    for $i = 1 to $processList[0][0]
        local $path = _ProcessGetLocation($processList[$i][1])
        local $md5 = _Crypt_HashFile($path, $CALG_MD5)
        LogWrite("Name: " & $processList[$i][0] & " PID: " & $processList[$i][1] & _
                 " Path: " & $path & " md5: " & $md5)

        for $j = 0 to Ubound($kCheckMd5) - 1
            if $md5 == $kCheckMd5[$j] then
                MsgBox(0, "Alert", "Clicker bot detected!")
            endif
        next
    next
endfunc

while true
    ScanProcess()
    Sleep(5000)
wend
```
Bot detection algorithm is implemented in the `ScanProcess` function. Now the `ProcessList` AutoIt function is called without a parameter. Therefore, the resulting `processList` array contains a list of all running processes. When we get this list, we can retrieve a path of the executable files, which start each process.

Process is a set of [**modules**](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684232%28v=vs.85%29.aspx). Each module matches to one executable or DLL file, which is loaded to the process memory. Module contains full information about its file. The `_ProcessGetLocation` function retrieves the path to module's file. Next step is to calculate a hash sum of this file with the [`_Crypt_HashFile`](https://www.autoitscript.com/autoit3/docs/libfunctions/_Crypt_HashFile.htm) AutoIt function. When the hash sum is calculated, we compare it with elements of the `kCheckMd5` array. This array contains hash sums of the `SimpleBot.exe` and 'AutoHotKey.exe' executable files in our case. This allows the protection system to detect both the `SimpleBot.ahk` script and the `SimpleBot.exe` application.

This is an algorithm of the `_ProcessGetLocation` function:

1. Call the [`OpenProcess`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684320%28v=vs.85%29.aspx) WinAPI function to get a handle of specified process.
2. Call the [`EnumProcessModules`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682631%28v=vs.85%29.aspx) WinAPI function to get a list of process' modules.
3. Call the [`GetModuleFileNameEx`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms683198%28v=vs.85%29.aspx) WinApi function to get a full path of module's file. First module in the list, which is returned by `EnumProcessModules` function, always matches to the executable file and all others modules match to DLLs.

You can launch the `Md5ScanProtection.au3` script and check that both `SimpleBot.ahk` script and `SimpleBot.exe` executable file are detected successfully. In case the `SimpleBot.ahk` script is not detected, you use a version of AutoHotKey application, which differs from mine. To fix it, you should read the correct MD5 sum of the `AutoHotKey.exe` executable in the `debug.log` file and change the `kCheckMd5` array accordingly.

There are several ways to improve our bot to avoid the `Md5ScanProtection.au3` script. All these ways focus on change a content of the executable file. This is a list of possible changes:

1. Perform a minor change of the `SimpleBot.ahk` script for example in the delay value. Then compile a new version of the script with `Ahk2Exe.exe` application.

2. Patch a header of the `AutoHotKey.exe` executable file with binary files editor (for example with [**HT editor**](http://hte.sourceforge.net))

It is dangerous to make changes in the executable file. These can broke the file and the application will crash at startup. But [**COFF**](https://en.wikipedia.org/wiki/COFF) header of the executable file contains several standard fields, which values do not affect behaviour of the application. Time of the file creation is one of these fields. This is an algorithm to change it with HT editor:

1. Launch the HT editor application with the administrator privileges. It will be convenient to copy the editor into the directory with the `AutoHotKey.exe` file.
2. Press *F3* key to pop up the "open file" dialog.
3. Press *Tab* to switch to the "files" list. Then select the `AutoHotKey.exe` file. Press *Enter* to open this file.
4. Press *F6* key to open the "select mode" dialog with the list of available modes. Choose the "- pe/header" mode. Now you see a list of executable file headers.
5. Choose the "COFF header" and press *Enter*. Select the "time-data stamp" field of the header.
6. Press *F4* key to edit the timestamp value. Change the value.
7. Press *F4* and choose "Yes" option in the "confirmation" dialog to save changes.

This is a screenshot of HT editor application at the changing timestamp step:

![HT Editor](ht-editor.png)

You get a new `AutoHotKey.exe` executable file, which content differs from the original file. Therefore, a MD5 hash sum of the new file differs from the hash sum of the original one. Now `Md5ScanProtection.au3` script is not able to detect launched AutoHotKey process.

Possible way to improve the protection system is to use advanced techniques to analyze a content of executable files. You can check a sequence of bytes in specific place of the file if you calculate a hash sum for these bytes only.

## Keyboard State Check

Windows OS provides kernel level mechanism to distinguish simulated keystrokes. 

First of all you should capture all low-level keyboard input events. The [`SetWindowsHookEx`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms644990%28v=vs.85%29.aspx) WinAPI function allows you to set a hook procedure, which is called each time when the specified event happens. First parameter of this function defines a type of hook procedure, i.e. which events will be captured. The `WH_KEYBOARD_LL` hook type allows you to capture keyboard input events. 

The hook procedure receives the [`KBDLLHOOKSTRUCT`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms644967%28v=vs.85%29.aspx) structure, which contains a detailed information about the captured event. All keyboard events, which are produced by `SendInput` and `keybd_event` WinAPI functions, have the `LLKHF_INJECTED` flag in the `KBDLLHOOKSTRUCT` structure. Keyboard events, which are produced by a keyboard driver, do not have this flag. This flag is set on Windows kernel level and it is impossible to disable this feature on WinAPI level.

This is the [`KeyboardCheckProtection.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/KeyboardCheckProtection.au3) script, which checks the `LLKHF_INJECTED` flag to detect a clicker bot:
```AutoIt
#include <WinAPI.au3>

global const $kLogFile = "debug.log"
global $gHook

func LogWrite($data)
    FileWrite($kLogFile, $data & chr(10))
endfunc

func _KeyHandler($nCode, $wParam, $lParam)
    if $nCode < 0 then
        return _WinAPI_CallNextHookEx($gHook, $nCode, $wParam, $lParam)
    endIf

    local $keyHooks = DllStructCreate($tagKBDLLHOOKSTRUCT, $lParam)

    LogWrite("_KeyHandler() - keyccode = " & DllStructGetData($keyHooks, "vkCode"));

    local $flags = DllStructGetData($keyHooks, "flags")
    if $flags = $LLKHF_INJECTED then
        MsgBox(0, "Alert", "Clicker bot detected!")
    endif

    return _WinAPI_CallNextHookEx($gHook, $nCode, $wParam, $lParam)
endfunc

func InitKeyHooks($handler)
    local $keyHandler = DllCallbackRegister($handler, "long", "int;wparam;lparam")
    local $hMod = _WinAPI_GetModuleHandle(0)
    $gHook = _WinAPI_SetWindowsHookEx($WH_KEYBOARD_LL, _
                                      DllCallbackGetPtr($keyHandler), $hMod)
endfunc

InitKeyHooks("_KeyHandler")

while true
    Sleep(10)
wend
```
Algorithm of this script is similar to one, which is used in the `TimeSpanProtection.au3` and `ActionSequenceProtection.au3` scripts. We use the `InitKeyHooks` function to install the `_KeyHandler` hook procedure. This procedure captures all low-level keyboard input events. This is an algorithm to install the procedure:

1. Register a `_KeyHandler` user function as a callback by the [`DllCallbackRegister`](https://www.autoitscript.com/autoit3/docs/functions/DllCallbackRegister.htm) AutoIt function. This operation allows you to pass `_KeyHandler` to other WinAPI functions.
2. Get handle of the current module by the [`GetModuleHandle`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms683199%28v=vs.85%29.aspx) WinAPI function.
3. Install the `_KeyHandler` function into a hook chain by the `SetWindowsHookEx` WinAPI function. We should pass to this function the handle of module where the `_KeyHandler` is defined.

There is an algorithm to check the `LLKHF_INJECTED` flag in the `_KeyHandler` function. These are steps of this algorithm:

1. Check a value of the `nCode` parameter. In case the value is less than zero, the captured keyboard event is passed to the next hook in the chain without any processing. Both `wParam` and `lParam` parameters do not contain actual information about the keyboard event in this case.
2. Create the `KBDLLHOOKSTRUCT` structure from the `lParam` input parameter by the `DllStructCreate` function.
3. Get the `flags` field of the `KBDLLHOOKSTRUCT` structure by `DllStructGetData` function.
4. Check if the `LLKHF_INJECTED` flag is present. Captured keyboard event is simulated by a clicker bot if there is the flag.

You can launch the `KeyboardCheckProtection.au3` script, Notepad application and the `SimpleBot.au3` script to test this protection approach. When the bot simulates the first key, you see the "Clicker bot detected!" message.

There are several ways to avoid this kind of protection systems. All of them focus on simulation keyboard events at level that is lower than WinAPI. These are list of these ways:

1. [**Virtual machine**](https://en.wikipedia.org/wiki/Virtual_machine) (VM) trick.
2. Use a keyboard driver instead of WinAPI functions to simulate keyboard events. [InpOut32](http://www.highrez.co.uk/downloads/inpout32/) project is an example of this kind of drivers.
3. [Emulate keyboard and mouse devices](../ExtraTechniques/input-device-emulation.md).

We can use VM to avoid protection systems, which check the `LLKHF_INJECTED` flag. VM has [**virtual device drivers**](https://en.wikipedia.org/wiki/Device_driver#Virtual_device_drivers). These drivers solve two tasks: emulate hardware devices and provide access to real hardware. All events from emulated or real hardware are passed via virtual device drivers. This means that Windows OS inside the VM cannot distinguish source of hardware events. If you keypress in the VM window, this action is legal in point of view OS in VM. If bot keypress in this window, this action is still legal in point of view OS. This happens because virtual device drivers process both these events in the same way.

This is an algorithm of VM trick:

1. Install one of VM applications ([Virtual Box](https://www.virtualbox.org/wiki/Downloads), [VMWare](http://www.vmware.com/products/player/playerpro-evaluation.html) or [Windows Virtual PC](https://www.microsoft.com/en-us/download/details.aspx?id=3702)).
2. Install Windows OS inside the VM.
3. Launch Notepad application and the `KeyboardCheckProtection.au3` script inside the VM.
4. Launch the `VirtualMachineBot.au3` script outside the VM i.e. on the host system.

This is a [`VirtualMachineBot.au3`](https://ellysh.gitbooks.io/video-game-bots/content/Examples/ClickerBots/ProtectionApproaches/VirtualMachineBot.au3) script:
```AutoIt
Sleep(2000)

while true
    Send("a")
    Sleep(1000)
    Send("b")
    Sleep(2000)
    Send("c")
    Sleep(1500)
wend
```
There is only one difference between this script and `SimpleBot.au3` one. The window of Notepad application is not activated at startup here. There is a two second delay instead at the startup. You should activate the window of VM application during this delay. Then script starts to work. The protection system cannot detect it.

## Summary

We have considered approaches to protect game application against clicker bots. There are ways to avoid these approaches. You can find suitable way easy only if you know algorithm of the protection system. There are ideas how to explore this algorithm:

1. Hook WinAPI calls of protection system process. You can do it with API Monitor or similar application.
2. [**Reverse**](https://en.wikipedia.org/wiki/Reverse_engineering) an executable and DLL file of the protection system.
3. Consequently try several ways to avoid this protection system. Then you can assume which protection algorithms are used.

Most of modern client-side protection systems combine several protection algorithms. Therefore, effective clicker bot should combine several methods to avoid these systems.
