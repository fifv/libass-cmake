cmake_minimum_required(VERSION 3.30.0)
project(libass-cmake VERSION 0.1.0)

set(CMAKE_COLOR_DIAGNOSTICS ON)

add_link_options($<$<CONFIG:RELEASE>:-s>)

#####
# 1. fribidi
# 2. harfbuzz
# 3. freetype
# 4. unibreak
#
# fontconfig is opt-out


### note: emscripten has builtin freetype and harfbuzz
add_subdirectory(freetype 3rd/freetype)
add_subdirectory(harfbuzz 3rd/harfbuzz)

### cmake doesn't support use env and pkg-config...
# set(ENV{PKG_CONFIG_PATH} /home/fifv/c/trylibass/fribidi/builddir/dist/usr/local/lib/pkgconfig)
# set(ENV{PKG_CONFIG_SYSROOT_DIR} /home/fifv/c/trylibass/fribidi/builddir/dist)
# find_package(PkgConfig REQUIRED)
# pkg_check_modules(FRIBIDI REQUIRED fribidi)

###### libass ######

add_library(ass STATIC
    libass/libass/ass_utils.c
    libass/libass/ass_filesystem.c
    libass/libass/ass_cache.c
    libass/libass/ass.c
    libass/libass/ass_library.c
    libass/libass/ass_parse.c
    libass/libass/ass_string.c
    libass/libass/ass_strtod.c
    libass/libass/ass_font.c
    libass/libass/ass_fontselect.c
    libass/libass/ass_outline.c
    libass/libass/ass_shaper.c
    libass/libass/ass_drawing.c
    libass/libass/ass_bitmap.c
    libass/libass/ass_blur.c
    libass/libass/ass_rasterizer.c
    libass/libass/ass_render.c
    libass/libass/ass_render_api.c
    libass/libass/ass_bitmap_engine.c
    libass/libass/c/c_rasterizer.c
    libass/libass/c/c_blend_bitmaps.c
    libass/libass/c/c_be_blur.c
    libass/libass/c/c_blur.c
    # libass/libass/ass_fontconfig.c # just not use this file to disable fontconfig, or don't define CONFIG_FONTCONFIG in config.h
)
target_include_directories(ass PUBLIC
    configs/libass # config.h
    libass/libass
)
target_link_directories(ass PRIVATE

)
target_link_libraries(ass PRIVATE
    freetype
    harfbuzz
    fribidi
    unibreak
)

###### libunibreak ######

add_library(unibreak STATIC
    libunibreak/src/unibreakbase.c
    libunibreak/src/unibreakdef.c
    libunibreak/src/linebreak.c
    libunibreak/src/linebreakdata.c
    libunibreak/src/linebreakdef.c
    libunibreak/src/eastasianwidthdef.c
    libunibreak/src/emojidef.c
    libunibreak/src/graphemebreak.c
    libunibreak/src/wordbreak.c
)

###### fribidi ######


set(fribidi_gentab_dir ${CMAKE_SOURCE_DIR}/fribidi/gen.tab)
set(fribidi_output_dir ${CMAKE_BINARY_DIR}/3rd/fribidi)

add_library(fribidi STATIC
    fribidi/lib/fribidi.c
    fribidi/lib/fribidi-arabic.c
    fribidi/lib/fribidi-bidi.c
    fribidi/lib/fribidi-bidi-types.c
    fribidi/lib/fribidi-char-sets.c
    fribidi/lib/fribidi-char-sets-cap-rtl.c
    fribidi/lib/fribidi-char-sets-cp1255.c
    fribidi/lib/fribidi-char-sets-cp1256.c
    fribidi/lib/fribidi-char-sets-iso8859-6.c
    fribidi/lib/fribidi-char-sets-iso8859-8.c
    fribidi/lib/fribidi-char-sets-utf8.c
    fribidi/lib/fribidi-deprecated.c
    fribidi/lib/fribidi-joining.c
    fribidi/lib/fribidi-joining-types.c
    fribidi/lib/fribidi-mirroring.c
    fribidi/lib/fribidi-brackets.c
    fribidi/lib/fribidi-run.c
    fribidi/lib/fribidi-shape.c
)
target_include_directories(fribidi PUBLIC
    fribidi/lib
    configs/fribidi
    ${fribidi_output_dir}
)
target_compile_definitions(fribidi PRIVATE
    HAVE_CONFIG_H
)
add_dependencies(fribidi fribidi-unicode-version-h fribidi-gen-i)


### target: fribidi-gen-i
### 1. build some tools on host
### 2. use built tools to generate .i files
### 3. add them to the target
add_custom_target(fribidi-gen-i)
function(fribidi_gen generator_src_base input_filebases output)

    ### cmake doesn't have .map() , so we need this ugly loop to expand list
    set(input_full_paths "")
    foreach(input_file ${input_filebases})
        list(APPEND input_full_paths "${fribidi_gentab_dir}/${input_file}")
    endforeach()



    add_custom_command(
        OUTPUT ${fribidi_output_dir}/${output}
        WORKING_DIRECTORY ${fribidi_gentab_dir}

        COMMAND clang
        ${generator_src_base}.c
        ${fribidi_gentab_dir}/packtab.c
        -o ${fribidi_output_dir}/${generator_src_base}
        -I../lib
        -I../../configs/fribidi
        -I${fribidi_output_dir}
        -DHAVE_CONFIG_H

        COMMAND ${fribidi_output_dir}/${generator_src_base}
        2 # COMPRESSION=2 in meson.build
        ${input_full_paths}
        ${generator_src_base}
        > ${fribidi_output_dir}/${output}

        DEPENDS
        ${fribidi_gentab_dir}/${generator_src_base}.c
        ${input_full_paths}
        fribidi-unicode-version-h

        COMMENT "Generating ${output} with host tool"
    )
    add_custom_target(${output}
        DEPENDS ${fribidi_output_dir}/${output}
    )
    add_dependencies(fribidi-gen-i ${output})
endfunction()

fribidi_gen(gen-bidi-type-tab "unidata/UnicodeData.txt" bidi-type.tab.i)
fribidi_gen(gen-joining-type-tab "unidata/UnicodeData.txt;unidata/ArabicShaping.txt" joining-type.tab.i)
fribidi_gen(gen-arabic-shaping-tab "unidata/UnicodeData.txt" arabic-shaping.tab.i)
fribidi_gen(gen-mirroring-tab "unidata/BidiMirroring.txt" mirroring.tab.i)
fribidi_gen(gen-brackets-tab "unidata/BidiBrackets.txt;unidata/UnicodeData.txt" brackets.tab.i)
fribidi_gen(gen-brackets-type-tab "unidata/BidiBrackets.txt" brackets-type.tab.i)

add_custom_command(
    OUTPUT ${fribidi_output_dir}/fribidi-unicode-version.h
    WORKING_DIRECTORY ${fribidi_gentab_dir}

    COMMAND clang
    ${fribidi_gentab_dir}/gen-unicode-version.c
    -o ${fribidi_output_dir}/gen-unicode-version
    -I../lib
    -I../../configs/fribidi
    -DHAVE_CONFIG_H

    COMMAND ${fribidi_output_dir}/gen-unicode-version
    ${fribidi_gentab_dir}/unidata/ReadMe.txt
    ${fribidi_gentab_dir}/unidata/BidiMirroring.txt
    ${fribidi_gentab_dir}/gen-unicode-version
    > ${fribidi_output_dir}/fribidi-unicode-version.h

    DEPENDS
    ${fribidi_gentab_dir}/gen-unicode-version.c
    ${fribidi_gentab_dir}/unidata/ReadMe.txt
    ${fribidi_gentab_dir}/unidata/BidiMirroring.txt

    COMMENT "Generating fribidi-unicode-version.h with host tool"
)
add_custom_target(fribidi-unicode-version-h
    DEPENDS ${fribidi_output_dir}/fribidi-unicode-version.h
)

###### libass's test.c ######

# note this need libpng and zlib, so it will fail in emscripten

add_executable(ass_test
    libass/test/test.c
)
target_link_libraries(ass_test PUBLIC ass)
