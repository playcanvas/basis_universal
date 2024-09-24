cmake_minimum_required(VERSION 3.0)

project(basisu_transcoder_js)

# KTX2 support (and Zstandard) in the transcoder is purely optional. Use "cmake -DKTX2=FALSE" to completely disable KTX2 support (which also disables Zstandard).
# If this is FALSE you'll only be able to transcode .basis files.
option(KTX2 "KTX2" TRUE)

# If KTX2 is enabled, Zstandard is needed to support decompressing supercompressed UASTC files. 
# If you know you'll never be transcoding KTX2 files using this feature you can set KTX2_ZSTANDARD to 0.
option(KTX2_ZSTANDARD "KTX2_ZSTANDARD" TRUE)

message("KTX2=${KTX2}")
message("KTX2_ZSTANDARD=${KTX2_ZSTANDARD}")

if (EMSCRIPTEN)
  set(CMAKE_CXX_STANDARD 11)

  function(build_target name flags enabled)
    set(SRC_LIST
  	  ../../transcoder/basisu_transcoder.cpp
	    basis_wrappers.cpp
    )
  
    if (KTX2)
	    set(KTX2_DEFINITION BASISD_SUPPORT_KTX2=1)
    else()
  	  set(KTX2_DEFINITION BASISD_SUPPORT_KTX2=0)
    endif()
  
    if (KTX2_ZSTANDARD)
 	    set(SRC_LIST ${SRC_LIST}
		    ../../zstd/zstddeclib.c
	    )
	    set(ZSTD_DEFINITION BASISD_SUPPORT_KTX2_ZSTD=1)
    else()
  	  set(ZSTD_DEFINITION BASISD_SUPPORT_KTX2_ZSTD=0)
    endif()
  
    add_executable(${name}.js ${SRC_LIST})

    set(allTypes UASTC_HDR UASTC ASTC ATC BC7 DXT1 DXT5A ETC2_EAC_A8 ETC2_EAC_RG11 FXT1 PVRTC1 PVRTC2)
    set(typeList "")

    foreach(type ${allTypes})
      string(FIND ${enabled} ${type} found)
      if(${found} EQUAL -1)
        list(APPEND typeList "BASISD_SUPPORT_${type}=0")
      else()
        list(APPEND typeList "BASISD_SUPPORT_${type}=1")
      endif()
    endforeach()

    target_compile_definitions(${name}.js PRIVATE NDEBUG ${typeList} ${KTX2_DEFINITION} ${ZSTD_DEFINITION})
    target_compile_options(${name}.js PRIVATE -O3 -fno-strict-aliasing)
    target_include_directories(${name}.js PRIVATE ../../transcoder)

    set_target_properties(${name}.js PROPERTIES
        OUTPUT_NAME "${name}"
        SUFFIX ".js"
        LINK_FLAGS "${flags} --bind -s ALLOW_MEMORY_GROWTH=1 -O3 -s ASSERTIONS=0 -s MALLOC=emmalloc -s MODULARIZE=1 -s EXPORT_NAME=BASIS")
  endfunction()

  function(build name enabled)
    build_target("${name}.wasm" "-s WASM=1" ${enabled})
    build_target("${name}" "-s WASM=0 -s SINGLE_FILE=1" ${enabled})
  endFunction()

  build("basis" "UASTC_HDR|UASTC|ASTC|DXT1|DXT5A|ETC2_EAC_A8|ETC2_EAC_RG11")
endif()