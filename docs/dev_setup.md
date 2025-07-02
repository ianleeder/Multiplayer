# Development Environment Setup

This document details how to set up a RimWorld Multiplayer development environment.

> **Note:** This guide is written for Visual Studio Code in Windows.  If you want to use another IDE or OS, YMMV.

## (Optional) Duplicated game installations

If you wish to have the option to play a "pristine" copy of the game, it is best to have a separate copy for development.

1. Navigate to your `steamapps\common` directory.
1. Copy the existing installation directory from `RimWorld` to `RimWorldDev`

> **Note:** You can do this multiple times, for example if you wanted a v1.6 Rimworld and a v1.5 Rimworld to do development in both environments.

This way the Steam copy in `RimWorld` is still pristine and can be used for standard play. And you have another copy in `RimWorldDev` that you can develop in.

## (Optional) Separate game settings

You may want different settings depending on whether you're playing with friends, or doing development.  With friends you likely want high-res, full-screen.  During development you might like the game to be in Window mode, as small as possible (1024x768), and with Development mode enabled!

These instructions allow you to maintain separate game environments between "pristine" and "dev"

1. The game ships with a README file (`steamapps\common\RimWorld\Readme.txt`)
1. This file details a command-line switch to specify to the game where to store save files _and config_.
    - **Note:** The file also details a `-quicktest` option to fast-load into a tiny map.
1. Create a shortcut to the executable (`RimWorldWin64.exe`), and edit the shortcut details.
1. After the target add the following text: ` -savedatafolder=DevSaveData`
    - The target should now look like:  
    ![Screenshot of shortcut properties](https://i.imgur.com/y32RP4f.png)
    - This specifies to save the config and games in a directory named `DevSaveData` in the same folder as the executable, eg `steamapps\common\RimWorldDev\DevSaveData`


## Clone Multiplayer Mod

> **Note:** It's recommended to fork the source code to your own account/repository first, and clone that.

1. Navigate to your local game mods directory (eg `steamapps\common\RimWorldDev\Mods`)
1. Clone your code repository here (eg `RimWorldDev\Mods\Multiplayer`)  
    > **Note:** The build steps assume this relative location and copy binaries accordingly. Otherwise paths will need to be updated and/or binaries moved after each build.
1. Contrary to the [documentation](https://github.com/rwmt/Multiplayer/blob/master/CONTRIBUTORS.md), __do not__ base your work off the `development` branch. It is well and truly out of date.
1. Copy the `Languages` folder from an original mod installation (eg `steamapps\workshop\content\294100\2606448745\1.5\Languages`) to your mod folder (`steamapps\common\RimWorldDev\Mods\Multiplayer\Languages`)

If you navigate to the `Multiplayer\Source` directory you should be able to build the solution now.

The `Source\Client\Multiplayer.csproj` specifies to copy the compiled files back to the Mod directories `Assemblies` and `AssembliesCustom` as a build action.

Technically you can edit the code to make changes, build and RimWorld will respect these changes.  The next section will detail how to perform debugging.

## Debugging

Some guides specify to install a specific `mono-2.0-bdwgc.dll`, but I found them to all be out of date. Below is an alternative that I got working.

### dnSpy

This will allow you to inspect and eventually debug vanilla RimWorld code.

1. Install [dnSpyEx](https://github.com/dnSpyEx/dnSpy).

### Doorstop

[Rimworld Doorstop](https://github.com/pardeike/Rimworld-Doorstop) allows you to enable the RimWorld Unity debugger. Follow the instructions on their page. Below is a summary.

1. Download the latest [UnityDoorstop Release](https://github.com/NeighTools/UnityDoorstop/releases) (select the release for your platform, eg `doorstop_win_release_4.4.0.zip`)
    - Unzip and place the files `doorstop_config.ini` and `winhttp.dll` into the __RimWorld__ root directory (not mod directory)
1. Download the latest [Rimworld Doorstop Release](https://github.com/pardeike/Rimworld-Doorstop/releases) (select the `Doorstop.dll`)
    - Place the `Doorstop.dll` file into the __RimWorld__ root directory (not mod directory)
1. Modify the `doorstop_config.ini` as per the [Rimworld Doorstop instructions](https://github.com/pardeike/Rimworld-Doorstop)
    - Ensure you update the following settings:  
        ```ini
        debug_enabled=true
        debug_address=127.0.0.1:55555
        ```
    - The port you choose only affects the debugger you are using. Visual Studio will default to port 56000, dnSpy defaults to port 55555.

### Start a debug session

1. Open dnSpy.
1. Select File > Open, and select `\RimWorldWin64_Data\Managed\Assembly-CSharp.dll`
1. Set a breakpoint
    - For example, Assembly-CSharp > RimWorld > ActiveDropPod > PodOpen
1. From here you have two ways to debug:
    1. Launch the game first and connect the debugger
        - Run the game (`\RimWorldWin64.exe`)
        - From dnSpy, select Debug > Start Debugging
            - Debug engine: Unity (Connect)
            - IP Address: `127.0.0.1`
            - Port: <whichever port was selected in `doorstop_config.ini` above>  
            > __Hint__: if you are using dnSpy and have the port set to the default 55555, then you don't need to enter any values.  Simply select "Unity (Connect)" and click OK/hit enter.
    1. Allow dnSpy to launch the game to connect the debugger
        - From dnSpy, select Debug > Start Debugging
        - Debug engine: Unity
        - Executable: `\RimWorldWin64.exe`
1. Load the Multiplayer DLL into dnSpy
    1. From dnSpy select Debug > Windows > Modules (this will only be present while connected)
        - This will list all `.dll` files used by `RimWorld.exe`.
    1. Look for `Multiplayer.dll` and open it, but note that it **may not appear right away**.
        - If you can't find `Multiplayer.dll`, enter "data-" into the search bar
        - Double-click any module with names like `data-0000022CFEF538E0`. It usually has a version "0.5"
        - Check the left-side **Assembly Explorer** to see if `Multiplayer.dll` appears there.
1. Play the game! You can run multiple instances of the same `\RimWorldWin64.exe` and connect over LAN.  
    Ensure you trigger an appropriate event for the breakpoint set:  
    ![](https://i.imgur.com/Up2tJ9P.png)