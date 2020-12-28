# NetBSD-Qt-Notes

Qt5 (5.15) packages can simply be installed on NetBSD via the package manager (pkgin) but other platform specific installations need to be installed manually. These are my notes on what I achieved and a cheatsheet for myself. I'll try to improve it to make a guide of this page. Feel free to share your experience to make this description better.

Contents:
* Qt Emscripten
* Qt Android
* Qt Creator

# Installing Emscripten for Qt:

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
qlibraryinfo_final.o: $(SOURCE_PATH)/src/corelib/global/qlibraryinfo.cpp $(BUILD_PATH)/src/corelib/global/qconfig.cpp	$(CXX) -c -o $@ $(CXXFLAGS) $(SOURCE_PATH)/src/corelib/global/qlibraryinfo.cpp $<`

