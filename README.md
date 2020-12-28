# NetBSD-Qt-Notes

Qt5 (5.15) packages can simply be installed on NetBSD via the package manager (pkgin) but other platform specific installations need to be installed manually. These are my notes on what I achieved and a cheatsheet for myself. I'll try to improve it to make a guide of this page. Feel free to share your experience to make this description better.

Contents:
* Qt Emscripten
* Qt Android
* Qt Creator

# Installing Emscripten for Qt

-install clang, llvm, lld (llvm linker) packages
-see https://github.com/WebAssembly/binaryen#building

`git clone https://github.com/WebAssembly/binaryen.git`

`git checkout version_91`

`cmake . && make`

-build fastcomp: https://emscripten.org/docs/building_from_source/building_fastcomp_manually_from_source.html
`mkdir fastcomp`

`cd fastcomp`

`git clone https://github.com/emscripten-core/emscripten-fastcomp`

`cd emscripten-fastcomp`

`git clone https://github.com/emscripten-core/emscripten-fastcomp-clang tools/clang`

-NOTE: You must clone it into a directory named clang as shown, so that Clang is present in tools/clang!

`mkdir build`

`cd build`

`cmake .. -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD="host;JSBackend" -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_TESTS=OFF -DCLANG_INCLUDE_TESTS=OFF`

-Call make to build the sources, specifying the number of available cores:
`make -j1`

-Next thing is to download and install emsdk: https://emscripten.org/docs/getting_started/downloads.html

`git clone https://github.com/emscripten-core/emsdk.git`

`cd emsdk`

-installing qt wasm according to: https://doc.qt.io/qt-5/wasm.html
-for some additional info see: https://wiki.qt.io/Qt_for_WebAssembly

-Install the specified version listed for qt-5.15:
`./emsdk install 1.39.8`

`./emsdk activate 1.39.8`

-WARNING: 'source emsdk/emsdk_env.sh' does NOT set the envvars correctly
-check emsdk/.emscripten and create a script (which needs to be run via source command) to set them via exports like (replacing user name with yours):
`export NODE_JS=node`

`export LLVM_ROOT=/usr/pkg/bin`

`export BINARYEN_ROOT=/homer/r0ller/binaryen`

`export EMSCRIPTEN_ROOT=/home/r0ller/emsdk/upstream/emscripten`

`export TEMP_DIR=/home/r0ller/emsdk/tmp`

`export COMPILER_ENGINE=$NODE_JS`

`export JS_ENGINES=[$NODE_JS]`

-to see if it works fine run:
`emcc -v`

-Later this file shall replace ~/.emscripten and emsdk/.emscripten without the export commands and apostrophing the values like: NODE_JS='node'

-when it comes to downloading the qt5 source: https://wiki.qt.io/Building_Qt_5_from_Git#Getting_the_source_code

`git clone git://code.qt.io/qt/qt5.git`

`cd qt5`

`git checkout 5.15`

`perl init-repository`

-NOTE: set emscripten envvars
`./configure -xplatform wasm-emscripten -nomake examples -prefix $PWD/qtbase`

-once finished, build required modules:
`make module-qtbase`

-or along with others like:
`make module-qtbase module-qtdeclarative`

-if it gets stuck at qtlibraryinfo_final.o it can be fixed according to this hint: https://git.sailfishos.org/mer-core/qtbase/commit/52d64fca662d0e488801fc40dffdc0a732cfdbd5

-modify the corresponding makefile:
`vi qtbase/qmake/Makefile`

-so qtlibraryinfo_final.o target looks like this:
qlibraryinfo_final.o: $(SOURCE_PATH)/src/corelib/global/qlibraryinfo.cpp $(BUILD_PATH)/src/corelib/global/qconfig.cpp	$(CXX) -c -o $@ $(CXXFLAGS) $(SOURCE_PATH)/src/corelib/global/qlibraryinfo.cpp $<

# Installing Android for Qt

-qt prerequisites: https://wiki.qt.io/Building_Qt_5_from_Git

`git clone git://code.qt.io/qt/qt5.git`

`cd qt5`

`git checkout 5.15`

`perl init-repository`

-useful sites with additional info:

https://doc.qt.io/qt-5/android.html

https://doc.qt.io/qt-5/android-building.html

https://doc.qt.io/qt-5/android-building.html#using-manual-installation

-based on this latter page, try this:
`./configure -xplatform android-clang --disable-rpath -nomake tests -nomake examples -android-ndk <path/to/sdk>/ndk-bundle/ -android-sdk <path/to/sdk> -no-warnings-are-errors -android-abis arm64-v8a`

-when first tried I had to change this but when tried later I did not need the next two lines so I think they can be skipped:
-used an uname symlink to an uname.sh in /h0me/r0ller/bin to mock Linux system info
-in qtbase/mkspecs/linux-g++/qplatformdefs.h replace: #include <features.h> with #include "/usr/include/g++/parallel/features.h"

-search for "not supported by Android" in qtbase/configure.pri and paste the lines relevant for Linux host in the appropriate condition in the if branch of that else section.
-gmake
-according to configure, 'gmake install' is a must but as I did not set any prefix, by default it will install to /usr/local so that needs to be created as root before gmake install. I cloned my qt source in the Android folder hoping that no install will be required in the end, so that turned out to be a mistake. Conclusion: SET A PREFIX
-set: ln -s /usr/pkg/bin/bash /bin/bash

-Configuring qt kit for android is pretty difficult and I only managed to get it work for NDK but the next steps describe what I tried.
-set the compiler used for compiling qt for android e.g.:
/home/r0ller/Android/Sdk/ndk-bundle/toolchains/llvm/prebuilt/linux-x86_64/bin/clang++

-pages used for the configuration:

https://doc.qt.io/qt-5/android-getting-started.html

https://doc.qt.io/qtcreator/creator-developing-android.html#specifying-android-device-settings

-as qt for Android requires a specific NDK version, you need to get the different components somehow for which I used this page:
https://androidsdkoffline.blogspot.com/p/android-sdk-10-api-29-q-direct-download.html

-essential packages listed by creator:
"build-tools;29.0.2" "ndk;21.1.6352462" "platform-tools" "platforms;android-29" "cmdline-tools;latest"


-setting up via sdkmanager:
`export JAVA_HOME=/usr/pkg/java/openjdk8` //openjdk11 won't work!

-start sdkmanager in Android/Sdk/tools/bin, issue from any location: `sdkmanager --sdk_root=/home/r0ller/Android/Sdk --version`

-copy the specified android-ndk-r21b directory CONTENT, after unzipping it must be copied into the install directory which shall look like: /home/r0ller/Android/Sdk/ndk-bundle/

-android app build issues: type_traits.h not found
-set in the corresponding files <type_traits> to "llvm/Support/type_traits.h" and add INCLUDEPATH += /usr/pkg/include in project .pro file

-additional info: https://wiki.qt.io/Android

-may worth a try: set INCLUDEPATH if qt creator complains about the arm compiler manually set and wants the qmake compilere: /home/r0ller/Android/qt5/qtbase/mkspecs/android-clang

-set ANDROID_NDK_ROOT to your NDK root
-could not read qmake configuration file so I had to set it manually: /home/r0ller/Android/qt5-install/mkspecs/android-clang/qmake.conf

-used this as inspiration but no clue for what:
https://stackoverflow.com/questions/28684647/developing-a-qt-app-for-android-from-the-command-line/28692281

-openssl had to be installed, adding openssl libs to project still needs to be done -> not at all, api >=21 does not need openssl

-compiling from creator does not work maybe due to qt version seen erroneous by creator and seems that wrong qmake is picked
-BUT: if the command copied from effective qmake call (found on project build page) is issued in the debug-build directory (e.g. /home/r0ller/TW/gettere3/build-debug-Android/) produces the arm libs (after getting rid of system bin dirs in PATH):

`~/Android/qt5-install/bin/qmake ~/TW/getttere3/getttere3/getttere3.pro -spec /home/r0ller/Android/qt5-install/mkspecs/android-clang CONFIG+=debug && /home/r0ller/Android/Sdk/ndk-bundle/prebuilt/linux-x86_64/bin/make`

-create the directory defined in .pro for ANDROID_PACKAGE_SOURCE_DIR in the project directory (not the build directory) otherwise make apk will fail with "cannot find android sources"

-Another approach to build an android app is as follows:
-Set ENVIRONMENT VARIABLES like JAVA_HOME, add java path to PATH, ANDROID_SDK_ROOT, ANDROID_NDK_ROOT, etc. see: https://doc.qt.io/qt-5/deployment-android.html
`make apk`

-it generated androiddeployqt like this is in the Makefile at the apk target: ~/Android/qt5-install/bin/androiddeployqt --input android-gettere3-deployment-settings.json --output /home/r0ller/TW/getttere3/build-getttere3-Android-Debug --android-platform android-29 --gradle

-gradle will fail due to unknown platform: netbsd error
-cd to the android-build directory within the project build directory and execute: gradlew tasks --stacktrace
-see for additional info: https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/gradle-core/src/main/java/com/android/build/gradle/internal/res/Aapt2MavenUtils.kt

-other pages about deployment:

https://doc.qt.io/qt-5/deployment-android.html

https://doc.qt.io/qtcreator/creator-deploying-android.html
