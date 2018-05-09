# I want a good code-editor for Zephyr RTOS development
Back and forth between commandline and Source code editor with partly working includes... it's not cool!

So I'll be trying to set up VSCode with proper include paths and build support for the Zephyr RTOS.
Zephyr is moving towards beeing fully cross-platform (see end of file for details of wich Version was up to date when
I wrote this) and VSCode is so, too, so all of this should apply to Linux, Win and Mac. I'm using ubuntu Linux while
writing this and will also try it on Windows.

## Part 1: Zephyr working on the Commandline
Zephyr brings quite a lot of helper scripts and tools with it, so before integrating it into the editor try it out at the commandline!
- Follow the Zephyr [GettingStarted](http://docs.zephyrproject.org/getting_started/getting_started.html) (use the right Version! if you're using Zephyr v1.11 then go to [GettingStartedv1.11](http://docs.zephyrproject.org/1.11.0/getting_started/getting_started.html)
- Play around with it, try building, flashing a app or running it in the emulator

## Part 2: Cmake integration in VSCode
Zephyr's build system is based on a tool called cmake. There's an extension for VSCode, that provides quite good integration. In VSCode press `Ctrl+P` and type:
```
ext install vector-of-bool.cmake-tools
```
The extension has not yet been adapted to VSCodes multi-root-workspaces yet, but it works well, as long, as the app (your main CMakeLists.txt file) is in your first workspace folder.
So here is how I set my workspace up:
- `File -> Open Folder -> [App Folder, e.g. /path/to/zephyr/samples/hello_world`
- `File -> Add Folder to Workspace... -> /path/to/zephyr/`, for easy browsing of the Zephyr kernel sources
- Add all additional project dependencies to the workspace
- create a build directory somewhere (I prefer out-of-tree builds) and add it to teh workspace
- save the workspace

The cmake-tools extension relies on so called kits to define the toolchain for each project. Since zephyr configures it's own toolchain we just set a dummy value in cmake-kits.json (`Ctrl+Shift+P -> CMake: edit user-local CMake kits) to satisfy the cmake extension.
 ```json
 [
  {
    "name": "Zephyr",
    "toolchainFile": ""
  }
 ]
 ```

 When using zephyr on the commandline, we need to set a few environment-variables to configure the which toolchain to use. We need to do the same inside VSCODE, but we don't source the zephyr_env.sh script, but instead directly define the ZEPHYR_BASE variable. In the future there might be more functionality packed into that script, but right now everything works fine like this.
 - open workspace settings (`Ctrl+, -> select WORKSPACE SETTINGS`)
 ```json
{
	"settings": {
		"cmake.environment": {
			"ZEPHYR_TOOLCHAIN_VARIANT": "zephyr",
            "ZEPHYR_SDK_INSTALL_DIR": "/path/to/zephyr-sdk",
            "ZEPHYR_BASE": "/path/to/zephyr"
        }
	},
}
```

**Continue here!**


## Part 3: Syntax highlghting, goto definition etc.
VSCodes capabilities for context aware syntax highligting, goto definitioon etc. are normally provided through the ms-vscode.cpptools extension. I treid, but didn't get it to work in a way any where close to acceptable. They seem to be switching the way they store and handle those relationships, so this might change soon. 


**Continue here: cquery**




## Appendix
Time-of-Writing: 2018-04-26
Referring to last release of Zephyr RTOS: v1.11