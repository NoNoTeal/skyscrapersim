# Building SBS on arm64 (Apple M1 chip) macOS Machines - by xteal

If unsure which chip you have (which you SHOULD already know), run this in your terminal:
sysctl -n machdep.cpu.brand_string

You can use these instructions to build on an x86_64 (Intel) chip, such as macs made prior to late 2020 given you know how to edit the instructions accordingly.


Prerequisites
===

Install CMake
---
https://cmake.org

Install XCode under /Applications/Xcode.app
---
https://xcodereleases.com/ (Apple ID Required, pick the latest version)

# Compiling SBS/OGREBullet Dependencies

Install wxWidgets 3.1.5
===
## 1. Download wxWidgets
- https://github.com/wxWidgets/wxWidgets/releases/download/v3.1.5/wxWidgets-3.1.5.tar.bz2

Getting ready to Compile
---
Run the following commands (adjust the SDK version if needed):
```bash
./configure CXXFLAGS="-stdlib=libc++ -std=c++11" --enable-universal-binary=x86_64,arm64 --with-cocoa --with-macosx-version-min=12.0 CPPFLAGS="-stdlib=libc++" LIBS=-lc++ CXXCPP="/usr/bin/gcc -E" --disable-debug_flag --disable-sys-libs
```

## 2. Run make:
- on the regular M1 chip probably: `make -j8`
- on the regular M1 Pro chip: `make -j10`
- on the regular M1 Max chip probably: `make -j16`

### Troubleshooting the build:
 - **make: /path/to/wxWidgets-3.1.5: Permission denied:** remove all spaces from the wxwidgets folder name, reconfigure and remake.
 - **undefined symbols for architecture arm64:** wipe your homebrew formulae. If there is a package with libtiff, or something else installed by x86_64 homebrew or vice versa, then you will get undefined symbols for the opposite architecture. 
 

## 3. Install:
 Now run `sudo make install` and wxWidgets should install under `/usr/local`.

Install OGRE 1.9.1
===

## 1. Download OGRE and the dependencies
- https://github.com/OGRECave/ogre/archive/refs/tags/v1.9.1.tar.gz
- https://github.com/NoNoTeal/skyscrapersim/releases/download/Dependencies/dependencies.zip
 - These precompiled static(contains all functions, needed or not) dependencies should include the following (save you time from figuring out how to compile each library):
  - boost:atomic
  - boost:chrono
  - boost:date_time
  - boost:filesystem
  - boost:system
  - boost:thread
  - wgOIS
  - zziplib
  - zlib
  - freetype
  - freeimage
  - FMOD
- http://developer.download.nvidia.com/cg/Cg_3.1/Cg-3.1_April2012.dmg (x86_64 ONLY, if you aren't interested in clouds ignore this)

## 2. Install the dependencies:

### Dependencies available on Intel Brew:
---
```sh
ibrew install libxaw pkg-config cppunit zlib libzzip freetype freeimage
```

### OGRE Dependencies.zip:
---
Unzip the ogre dependencies and COPY (not move) the files under /usr/local. When prompted, click the merge button and delete the dependencies folder or else Ogre will pick up that folder over /usr/local.

### NVIDIA's Cg toolkit.dmg:
---
Open the dmg, and it is OK if the app won't open for you. Right click on the app and click on Show Package Contents, then unzip `Contents > Resources > Installer Items > NVIDIA_Cg.tgz` and move `./Library/Frameworks/Cg.framework to /usr/local/lib`. Skyscraper NEEDS this. You then need to go into the headers folder under the framework, usually `~/Cg.framework/Headers` and copy it / create a directory to `/usr/local/include/Cg`.

## 3. Edit the following files:
---
## ~/Ogre/RenderSystems/GL/CMakeLists.txt
```cmake
# Find this near line 112
set_target_properties(RenderSystem_GL PROPERTIES LINK_FLAGS "-framework Cocoa -framework Carbon -framework OpenGL")

# Replace it with:
set_target_properties(RenderSystem_GL PROPERTIES LINK_FLAGS "-framework AGL -framework Cocoa -framework Carbon -framework OpenGL")
```

## ~/Ogre/OgreMain/CMakeLists.txt
```cmake
# Find this near line 295
list(APPEND LIBRARIES "-latomic")

# Replace it with:
list(APPEND LIBRARIES "-lboost_atomic -lbz2")
# Alternatively, you can ignore this step if you just rename libboost_atomic.a to libatomic.a (NOT TESTED)
# bz2 is for M1.
```

## ~/Ogre/OgreMain/src/OgrePlatformInformation.cpp
```cpp
// Find this near line 190
        __asm__
        (
            "cpuid": "=a" (result._eax), "=b" (result._ebx), "=c" (result._ecx), "=d" (result._edx) : "a" (query)
        );
        
// Replace it with:

        register Ogre::uint __a asm("r0") = result._eax;
        register Ogre::uint __b asm("r1") = result._ebx;
        register Ogre::uint __c asm("r2") = result._ecx;
        register Ogre::uint __d asm("r3") = result._edx;

        __asm__
        (
            "cpuid": "=r" (__a), "=r" (__b), "=r" (__c), "=r" (__d) : "r" (query)
        );
// I don't even know what this does, it is ASM x86_64 code so it errors when trying to build for arm64.
```

## ~/Ogre/RenderSystems/GL/src/OSX/OgreOSXWindow.cpp
```cpp
# Find this near line 326
cgErr = CGLSetParameter(mCGLContextObj, kCGLCPSwapInterval, &swapInterval);

# Replace it with:
cgErr = (CGError)CGLSetParameter(mCGLContextObj, kCGLCPSwapInterval, &swapInterval);
# Don't be scared of the CPP replacement! There are only a few other small modifications you will perform under the SBS source code.
```

Getting ready to compile
---
 
Run the following commands (adjust the SDK path if needed):
```cmake
mkdir -p /usr/local/lib/macosx/release

cmake -DOGRE_CONFIG_DOUBLE=ON -DOGRE_BUILD_SAMPLES=OFF -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" -DCMAKE_INSTALL_PREFIX=/usr/local -DCMAKE_CXX_FLAGS="-stdlib=libc++ -std=c++11 -Wno-c++11-narrowing" -DOGRE_STATIC=NO
```

## 4. Run make:
- on the regular M1 chip probably: `make -j8`
- on the regular M1 Pro chip: `make -j10`
- on the regular M1 Max chip probably: `make -j16`
 
### Troubleshooting the build:
 - You may run into an error **(isysroot, arch, etc.)**, just run the cmake command again then the make command.
 - **ld: warning: directory not found for option '-L/usr/local/lib/macosx/release'**: Create this directory: `/usr/local/macosx/release`
 - **ld: can't write output file...**: Try `sudo make -j(cores)`
 - When in doubt, delete `~/Ogre/CMakeFiles` and `~/Ogre/CMakeCache.txt`, then run the CMake command twice, then `make -j(cores)`
 

## 5. Install:
 Now run `sudo make install` and OGRE should install under `/usr/local`.
 
 The frameworks will install under `/usr/local/lib/macosx/release`. Move everything from that directory to just under `usr/local/lib`. Delete `/usr/local/lib/macosx`.

Install the Custom Caelum Source
===

## 1. Download Caelum
- https://skyscraper.tliquest.net/downloads/dev/other_apps/caelum.tar.gz

## 2. Edit the following files:
---
## ~/Caelum/main/CMakeLists.txt
```cmake
# Find this near line 19
target_link_libraries(${LIBNAME} ${Ogre_LIBRARIES})
# And replace it with:
target_link_libraries(${LIBNAME} "-L/usr/local/lib;-lboost_system;-lpthread;-F/usr/local/lib;-framework Ogre")
# When you compiled OGRE, it came out as a framework. Caelum thinks OGRE is static or a dylib. The pkgconfig file that comes with the OGRE build also tells Caelum that it should treat OGRE like a library, this is a more invasive way of changing it...
```

Getting ready to compile
---
Run the following command:
```bash
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" -DCMAKE_CXX_FLAGS="-stdlib=libc++ -std=c++11" -DCAELUM_SCRIPT_SUPPORT=0
```

## 2. Run make & Install:
- on the regular M1 chip probably: `make -j8`
- on the regular M1 Pro chip: `make -j10`
- on the regular M1 Max chip probably: `make -j16`
- Then run `sudo make install`

Install the Custom Bullet Source
===
## 1. Download Bullet-SVN
- https://skyscraper.tliquest.net/downloads/dev/other_apps/bullet.tar.bz2
 - Backup link if bullet goes down: 

## 2. Edit the following files:
---
## ~/Bullet/src/LinearMath/btVector3.h
```cpp
// Find this near line 42
define BT_SHUFFLE(x,y,z,w) ((w)<<6 | (z)<<4 | (y)<<2 | (x))
// And replace it with:
define BT_SHUFFLE(x, y, z, w) (((w) << 6 | (z) << 4 | (y) << 2 | (x)) & 0xff)
// Why? Errors. I don't know how to explain this.
```

Getting ready to compile
---

Run the following commands:
```cmake
cmake -DCMAKE_C_FLAGS=-fPIC -DCMAKE_CXX_FLAGS=-fPIC -DINSTALL_EXTRA_LIBS=ON -DBUILD_DEMOS=ON -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" -DBUILD_SHARED_LIBS=NO -DFRAMEWORK=NO
```

## 3. Run make & Install:
- on the regular M1 chip probably: `make -j8`
- on the regular M1 Pro chip: `make -j10`
- on the regular M1 Max chip probably: `make -j16`
- Then run `sudo make install`

# Compiling SBS/OGREBullet
## 1. Download this repository's source code, and open it up somewhere.

## 2. Make any changes, here is what I did prior.
---
## ~/SBS/CMakeLists.txt
```cmake
# Find and delete this near line 55
file(GLOB SVNREV_FILES
	svnrev/*.c
)
# And delete this near line 91
#svnrev
add_executable(svnrev/svnrev ${SVNREV_FILES})
if(UNIX)
	add_custom_target(MakeRev ALL ./svnrev.sh)
	add_dependencies(MakeRev svnrev/svnrev)
endif()
# svn repo might be dead.
```
## ~/SBS/src/SBS/sbs.cpp
```cpp
// Find and delete line 57:
#include "revsbs.h"

// Then around line 65, find this and remove SVN_REVSTR and add anything else:
version = "0.11.0." + std::string(SVN_REVSTR);

version = "0.11.0." + std::string("xteal");
// errors occur if you do not do this.
```

## ~/SBS/src/SBS/camera.cpp
```cpp
// Find line 90:
    cfg_zoomspeed = sbs->GetConfigFloat("Skyscraper.SBS.Camera.ZoomSpeed", 0.2);
// change to
    cfg_zoomspeed = sbs->GetConfigFloat("Skyscraper.SBS.Camera.ZoomSpeed", 0.2) * 25;
// This is so zooming is in bigger increments.
    
// Find and change line 551-552:
    Real x = (float)mouse_x / (float)width;
    Real y = (float)mouse_y / (float)height;
// to
    Real x = (float)mouse_x / ((float)width/MainCamera->getViewport()->getWidth());
    Real y = (float)mouse_y / ((float)height/MainCamera->getViewport()->getHeight());
// When I was building this with wxWidgets 3.0.5, the user interface looked quite ugly in macOS dark mode. Skyscrapersim was not intended to be built in 2018 or around the release of macOS Mojave (debut of dark theme), so I switched to wxWidgets 3.1.5 which did support dark theme. However, the tradeoff was the game appearing in the lower left of the screen, this addresses it.
```

## ~/SBS/src/frontend/mainscreen.cpp
```cpp
// Make a new line and add this at line 76:
SetBackgroundStyle(wxBG_STYLE_PAINT);
// This makes it so you don't crash when trying to load the actual bulk of the simulator.
```

## ~/SBS/src/frontend/skyscraper.cpp
```cpp
// Remove line 55:
#include "revmain.h"

// Feel free to change line 102 to anything else that is a string value:
version_rev = SVN_REVSTR;
// to
version_rev = "idk";

// Make a new line and add this to line 421:
mRenderWindow->resize(mViewport->getActualWidth(), mViewport->getActualHeight());
// See the camera.cpp comment for more details.

// Replace line 601 with this:
mViewport = mRenderWindow->addViewport(mCamera);
// to
float scale = (float) window->GetContentScaleFactor();
mViewport = mRenderWindow->addViewport(mCamera, 0, 0.0F, -(scale/2.0F), scale, scale);
// Also see camera.cpp for more details.
```
            

Getting ready to compile
---

Run the following commands:
```cmake
cmake -DCMAKE_CXX_FLAGS="-stdlib=libc++ -std=c++11" -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64"
```

## 3. Run make & Install:
- on the regular M1 chip probably: `sudo make -j8`
- on the regular M1 Pro chip: `sudo make -j10`
- on the regular M1 Max chip probably: `sudo make -j16`
Yes, sudo is needed. 

Then you can run `make install`.

Closing Notes & Optional Tutorials
===

Yes, your Skyscraper build is somewhat glitchy. Here is a list of drawbacks that are out of my control. I have put in enough time into this project for someone else to better build Skyscraper. Does making an arm64 build matter? Somewhat. Apple, sooner or later will cut off Rosetta 2 support. This repository is a proof of concept to migrate Skyscraper to arm64 before Rosetta 2 is dead. Either way, this game is not resource intensive so FPS doesn't matter.

Limitations:
- On arm64, you can't use gravity. Turning on gravity and touching the floor teleports you to `0,0,0`, or the origin of the "room". You spawn in the default position but you are immediately touching the ground, so it appears that you spawn in `0,0,0`, but it's actually `0,5,0` -> touch ground -> `0,0,0`. To counteract this, open skyscraper with Rosetta.
- The lack of clouds from Caelum. It's more for asthetics than for essential usage. However, you WILL crash when exiting a level and entering another one. This is because Cg is used by Caelum to generate clouds and pops up an error when loading a level about clouds missing. To counteract this, build everything ONLY for x86_64, and have the Cg.framework file on hand. Ogre needs Cg to make a plugin, and Caelum needs Cg to make clouds. Then, go to plugins.cfg under your skyscraper build, and erase the # before `Plugin=Plugin_CgProgramManager`. Or if you don't want to deal with this... disable Caelum in `skyscraper.ini`. 

Building the optional dependencies (untested)
===

Install libOIS 1.5.1
---
## 1. Download OIS (or the latest version)
- https://github.com/wgois/OIS/archive/refs/tags/v1.5.1.zip

## Getting ready to Compile
Run the following commands (adjust the SDK version if needed):
```bash
cmake -DOIS_BUILD_SHARED_LIBS=OFF -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" -DCMAKE_CXX_FLAGS="-stdlib=libc++ -std=c++11"
```

## 2. Run make:
- on the regular M1 chip probably: `make -j8`
- on the regular M1 Pro chip: `make -j10`
- on the regular M1 Max chip probably: `make -j16`

## 3. Install:
 Now run `sudo make install` and libOIS should install under `/usr/local`.

Install Homebrew
---
```sh
arch -x86_64 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# If you want to build for arm64, or make a universal binary with some tradeoffs (no caelum clouds), remove the arch -x86_64 part.
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

(You should also add this to your .zprofile, .bash_profile or whatever shell you're using to run Intel Homebrew easily)

```sh
alias ibrew="arch -x86_64 /usr/local/bin/brew"
# Using ibrew will install the x86_64 version of packages.
```

```sh
# Install both on your Intel Brew and Native Brew
ibrew install freeimage freetype zlib libzzip bz2
brew install freeimage freetype zlib libzzip bz2
```

Honestly, if you were making Skyscraper for one or the other; install the appropriate homebrew, and install the packages and move on.

But universal binaries complicate everything:

1. Get the hash of each package you install, so `reinstall` after installing.

2. Go to ~/Library/Caches/Homebrew/downloads and find the file named after the hash

3. Copy the archives to someplace and unzip

4. Locate packages, and run `lipo -create -arch arm64 <path to arm64 lib> -arch x86_64 <path to x86_64 lib> -output <wherever you want>

And if I remember correctly, some of these libraries are actually dynamic so you would need to track down the source for some of these libraries, compile from source as a static / non shared library, then stitch it together. 

Install FMOD
---
Create an FMOD account, then access https://www.fmod.com/download. 

1. Download FMOD Engine, NOT FMOD Studio. Download the latest version for macOS as it contains (v2) universal binaries. 
2. Run commands under /fmoddir/api/core (You can do this for the OLDEST version as well, it is /fmoddir/api/lowlevel instead of core, just without arm64 support.)

```sh
mkdir /usr/local/include/fmod
cp -a inc/* /usr/local/include/fmod
cp -a lib/* /usr/local/lib
```

zziplib
---
```bash
cmake -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" -DBUILD_STATIC_LIBS="ON" -DBUILD_SHARED_LIBS="OFF"
cmake --build .
```

Building Boost
---
You may have seen Boost mentioned as a dependency, in this context, we use Boost for multithreading but it is also a pain in the ass to compile. Here are the best steps I have derived in building Boost. You need to build the compiler for the internal Boost libraries, then build each individual library needed with special exceptions. You can now see why I chose to bundle Boost with the other dependencies.

Create a new terminal at `~/Boost/tools/build/v2` and run `./bootstrap.sh cxxflags="-arch x86_64 -arch arm64 -stdlib=libc++ -std=c++11" cflags="-arch x86_64 -arch arm64" linkflags="-arch x86_64 -arch arm64"`

Copy the `b2` executable to `/usr/local/bin`.

Here is a list of the libraries that Skyscraper, Caelum, or Ogre rely on:
- atomic
- chrono
- date_time
- filesystem
- system
- thread

## Edit the following files
---

For some reason, Boost doesn't let you set the c++ versioning, so you 

~/boost/lib/atomic/build/Jamfile.v2
```jam
# line 11
project boost/atomic
    : requirements
# add this
      <cxxflags>-stdlib=libc++
      <cxxflags>-std=c++11
```

~/boost/lib/chrono/build/Jamfile.v2
```jam
# line 14
project boost/chrono    
      : source-location ../src    
      : requirements
# add this
      <cxxflags>-stdlib=libc++
      <cxxflags>-std=c++11
```

~/boost/lib/date_time/build/Jamfile.v2
```jam
# line 23
      <link>static:<define>BOOST_DATE_TIME_STATIC_LINK
# add this
      <cxxflags>-stdlib=libc++
      <cxxflags>-std=c++11
```

~/boost/lib/filesystem/build/Jamfile.v2
```jam
# line 11
      : usage-requirements # pass these requirement to dependents (i.e. users)
# add this
      <cxxflags>-stdlib=libc++
      <cxxflags>-std=c++11
```

~/boost/lib/system/build/Jamfile.v2
```jam
# line 15
      <link>shared:<define>BOOST_SYSTEM_DYN_LINK=1
      <link>static:<define>BOOST_SYSTEM_STATIC_LINK=1
# add this
      <cxxflags>-stdlib=libc++
      <cxxflags>-std=c++11
```

~/boost/lib/thread/build/Jamfile.v2
```jam
# line 44
      : requirements <threading>multi
# add this
      <cxxflags>-stdlib=libc++
      <cxxflags>-std=c++11
```

## Generating universal binaries (Compiling)

Do this for every library listed previously, I recommend you start with the system library first.

In advance for the day that someone finds a method that lets you compile SBS for the arm64 architecture...

Create a new terminal at your `~/boost/lib/<library>/build` then run the following commands.

1. `b2 link=static toolset=darwin cxxflags="-arch x86_64" -a`

2. After running this, copy the `.a` file under `~/boost/bin.v2/libs/<lib_name>/build/darwin<...>/debug/link-static/` to a safe place, delete the darwin folder that contains the files, and run the next command, repeating this statement.

3. `b2 link=static toolset=darwin cxxflags="-arch arm64" -a`

4. `lipo -create -arch arm64 <dir_to_arm64_archive> -arch x86_64 <dir_to_x86_64_archive> -output <dir_to_desired_output>`

If you're really interested in if the archives have merged, run `lipo -info <dir_to_universal_archive>`.

Ignore steps 2-4 if you aren't interested in making arm64 files. If you're making one or the other, just don't combine them / edit architectures accordingly.
