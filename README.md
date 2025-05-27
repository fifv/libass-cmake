# libass-cmake

Build [libass](https://github.com/libass/libass) and its dependencies, all with CMake

The purpose is to cross-compile to WebAssembly via emscripten

### Usage

1. Add this project to your project and in `CMakeLists.txt`
    ~~~cmake
    ### add this project in your source, by clone or submodule
    add_subdirectory(libass-cmake)
    ### link your app with libass, the library target `ass` is provided by this project
    add_executable(your-app somesources.c)
    target_link_libraries(your-app PRIVATE ass)
    ~~~
2. Use libass, see its `libass/test/test.c` as a reference usage
    ~~~c
    #include <ass.h>
    int main() {
        ass_library_init();
        /* ... */
    }
    ~~~

### Submodules
1. [libass](https://github.com/libass/libass) - the main library
2. [fribidi](https://github.com/fribidi/fribidi) - "Unicode Bidirectional Algorithm", required by libass, built with autotools, partitally port to CMake
3. [harfbuzz](https://github.com/harfbuzz/harfbuzz) - Text Shaping, required by libass, support CMake natively
4. [freetype](https://github.com/freetype/freetype) - Text Rendering, required by libass, support CMake natively
5. [libunibreak](https://github.com/adah1972/libunibreak) - "Unicode line breaking", required by libass, built with autotools, port to CMake


### TODO
- [ ] currently fribidi and libass' configurations are skipped and I put the generated `config.h` directly in `./configs/`, that's expected as I want control the configurations. But it may be broken someday, so it will be better to invent some method address this issue

- [ ] actually build this with emscripten and built a backend in typescript which import this project to test functionality


