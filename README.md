# NetBSD-Qt-Notes

Note: rewriting in progress to qt-6.7.2

Qt6 (6.7.2) packages can simply be installed on NetBSD via the package manager (pkgin) but other platform specific installations need to be installed manually. These are my notes on what I achieved and a cheatsheet for myself. I'll try to improve it to make a guide of this page. Feel free to share your experience to make this description better.

Contents:
* Qt Emscripten
* Qt Android
* Qt Creator
* Current state

# Installing Emscripten for Qt

-install clang, binaryen, llvm, lld (llvm linker), nodejs, npm packages, python3, perl, qt6 (having the same version of qt source for wasm and the installed native is important) via pkgin

-download and install emsdk: https://emscripten.org/docs/getting_started/downloads.html

`git clone https://github.com/emscripten-core/emsdk.git`

`cd emsdk`

-installing qt wasm according to: https://doc.qt.io/qt-6/wasm.html

-edit emsdk.py and change the check for the LINUX system (to set a variable called LINUX to true) to let NetBSD pass the condition: `if not MACOS and (platform.system() == 'NetBSD')`

-Install the specified version listed for qt-6.7:
`./emsdk install 3.1.50`

`./emsdk activate 3.1.50`

-usually a linux specific nodejs gets installed as well in the node directory and the prebuilt binaries are for linux in emsdk/upstream/bin, so rename the prebuilt bin directory to emsdk/upstream/o_bin and set a symlink to the native node binaries in /usr/pkg/bin as emsdk/upstream/bin and also rename emsdk/node/20.18.0_64bit/bin to /usr/pkg/bin. No clue why setting (see later) LLVM_ROOT and NODE_JS are not enough to override looking up the binaries in there but that's how it seems to work.

-WARNING: 'source emsdk/emsdk_env.sh' does NOT set the envvars correctly

-check emsdk/.emscripten and create a script (which needs to be run via source command) to set them via exports like (replacing user name with yours):
`export NODE_JS=node`

`export LLVM_ROOT=/usr/pkg/bin`

`export BINARYEN_ROOT=/homer/r0ller/binaryen`

`export EMSCRIPTEN_ROOT=/home/r0ller/emsdk/upstream/emscripten`

`export TEMP_DIR=/home/r0ller/emsdk/tmp`

`export COMPILER_ENGINE=$NODE_JS`

`export JS_ENGINES=[$NODE_JS]`

`export EMSDK=/home/r0ller/emsdk-3.1.50`

-to see if it works fine run:
`emcc -v`

-Later this file shall replace ~/.emscripten and emsdk/.emscripten without the export commands and apostrophing the values like: NODE_JS='node'

-when it comes to downloading the qt6 source (don't worry about cloning qt5 in the next step, qt itself mentions this strange stuff): https://wiki.qt.io/Building_Qt_6_from_Git

`git clone git://code.qt.io/qt/qt5.git qt6`

`cd qt6`

`git switch 6.7.2`

`perl init-repository`

-NOTE: set emscripten envvars, (and here it turns out that having the same version of qt source and the installed native is important) then:

`./configure -qt-host-path /usr/pkg/qt6 -xplatform wasm-emscripten -nomake examples -prefix $PWD/qtbase`

-once finished, build required modules:
`make module-qtbase`

-names of other supported modules (https://doc.qt.io/qt-6/wasm.html#supported-qt-modules) can be found here (https://wiki.qt.io/Qt_for_WebAssembly) and can be built in parallel like:
`make module-qtbase module-qtdeclarative module-qtquickcontrols2`

-if it gets stuck at qtlibraryinfo_final.o it can be fixed according to this hint: https://git.sailfishos.org/mer-core/qtbase/commit/52d64fca662d0e488801fc40dffdc0a732cfdbd5

-modify the corresponding makefile:
`vi qtbase/qmake/Makefile`

-so qtlibraryinfo_final.o target looks like this:  
`qlibraryinfo_final.o: $(SOURCE_PATH)/src/corelib/global/qlibraryinfo.cpp $(BUILD_PATH)/src/corelib/global/qconfig.cpp	$(CXX) -c -o $@ $(CXXFLAGS) $(SOURCE_PATH)/src/corelib/global/qlibraryinfo.cpp $<`

# Set up WebAssembly as target in Qt Creator

-set webassembly option: Help/About Plugins/Device Support

-set Emscripten SDK path in Tools/Options/Devices/WebAssembly

-in Tools/Options/Kits/Compilers set up a custom (using Add/Custom instead of Add/Emscripten) C/C++ compiler for emcc/em++ for the target: asmjs-unknown-unknown-emscripten-32bit with error parser Clang

-in Tools/Options/Kits/Qt Versions add the wasm compiled Qt qmake binary to configure a Qt Version

-in Tools/Options/Kits/Kits add a kit for the wasm Qt selecting the WebAssembly runtime as device type and specifying the custom emcc/em++ compilers for the wasm Qt version configured previously

-Create an empty Qt Quick project and build it (you may need to specify in the build step after qmake that QTC should use /usr/bin/make). It'll take long as emscripten compiles libc.a for the first time besides the project. Load the generated html in a browser. In case of chrome and firefox you'll get a CORS error which can be fixed (in firefox at least) by setting security.fileuri.strict_origin_policy to false in about:config.

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

# Installing Qt Creator

-I installed Qt Creator from git current so my version is 4.14.0-beta1 (4.13.82)

-the steps I followed are described here: https://wiki.qt.io/Building_Qt_Creator_from_Git

`git clone --recursive https://code.qt.io/qt-creator/qt-creator.git`

`export LLVM_INSTALL_DIR=/path/to/llvm (/usr/pkg without /bin)`

`mkdir qt-creator-build`

`cd qt-creator-build`

`qmake ../qt-creator/qtcreator.pro`

`gmake qmake_all`

`gmake -j<number-of-cpu-cores+1>`

-there are some changes I hadd to apply which are listed below:

-this page gave the hint to set fno-rtti: http://clang-developers.42468.n3.nabble.com/undefined-reference-to-typeinfo-for-clang-ASTFrontendAction-td1848336.html

-this page gave the hint to set the following variable: https://bugreports.qt.io/browse/QTCREATORBUG-17876?focusedCommentId=351818&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel
QTC_NO_CLANG_LIBTOOLING=true

-finally, I set the -fno-rtti in the qtcreator.pro file according to this page: https://doc.qt.io/qt-5/qmake-variable-reference.html
CONFIG+=rtti_off

-set in qt-creator/src/src.pro:
QTC_NO_CLANG_LIBTOOLING = true

-set in qt-creator/src/tools/clangrefactoringbackend/clangrefactoringbackend.pro:
QMAKE_CXXFLAGS += -fno-rtti
(added after first QMAKE_CXXFLAGS)

-set in qt-creator/src/tools/clangpchmanagerbackend/clangpchmanagerbackend.pro:
QMAKE_CXXFLAGS += -fno-rtti
(added after first QMAKE_CXXFLAGS)

-if gmake stops, just restart it. it needs several turns to compile, mostly plugin builds fail

-LD_LIBRARY_PATH needs to be set if you don't install qtcreator

-I got an error on starting qtcreator(.sh) when I built qt5 myself but not when qt5 was installed via pkgin:
qt.qpa.plugin: Could not find the Qt platform plugin "xcb" in ""
reason: libqxcb.so is missing from /usr/pkg/plugins/platforms

-for all build steps I used gmake

-in case you want to install qt creator (probably gmake goes here as well):
`make install INSTALL_ROOT=$INSTALL_DIRECTORY`

-configuring kits in Qt creator to build android or emscripten targets is pretty difficult and I mostly managed to get some partial results by try and error so I have no specific notes on that. However, for emscripten there is a specific flag in qt creator settings that needs to be checked and I guess I found that described here at the Enabling the Webassembly plugin section: https://doc.qt.io/qtcreator/creator-setup-webassembly.html

# Current state

-Native: everything I tried (both QtWidgets and QML) works fine except debugging but I have not even tried configuring it for the Qt Creator.

-Android: as mentioned eralier I cannot build an android app from qt creator as it seems Qt Creator does not pick the correct qmake but when I copy the effective qmake call and issue it on the command line then I get a successful build of the libraries. Unfortunately, I have not yet succeeded in building an apk of them.

-Emscripten: everything what I built as native could be built by emscripten as well.

-After getting tired of building an Android apk I decided to try to use an emscripten build of a QML UI with a native C++ backend on Android. Finally, I sketched a crossplatform architecture for Android, Node JS/Browser and native use cases. You can find a post about it on the Qt forum and the repo here:

https://forum.qt.io/topic/121354/crossplatform-desktop-android-browser-nodejs-architecture-sketch

https://github.com/r0ller/qwa

