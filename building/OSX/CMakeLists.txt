# Copyright 2023 DreamWorks Animation LLC
# SPDX-License-Identifier: Apache-2.0

# mkdir build && cd build && cmake ..
# cmake --build . --target help
# ... all (the default if no target is provided)
# ... clean
# ... depend
# ... edit_cache
# ... rebuild_cache

# mkdir build && cd build && cmake ..

# cmake --build . --target python
# cmake --build . --target blosc
# cmake --build . --target boost
# cmake --build . --target jsoncpp
# cmake --build . --target lua
# cmake --build . --target microhttpd
# cmake --build . --target opensubdiv
# cmake --build . --target openexr
# cmake --build . --target tbb
# cmake --build . --target openvdb
# cmake --build . --target log4cplus
# cmake --build . --target cppunit

# cmake --build . --target random123
# ls random123-prefix/src/random123

# cmake --build . --target ispc
# cmake --build . --target embree
# cmake --build . --target opencolorio
# cmake --build . --target pybind11
# cmake --build . --target openimageio
# cmake --build . --target openimagedenoise
# cmake --build . --target materialx
# cmake --build . --target usd


# ... python
# ... blosc
# ... boost
# ... jsoncpp
# ... lua
# ... microhttpd
# ... opensubdiv
# ... openexr
# ... tbb
# ... openvdb
# ... log4cplus
# ... cppunit
# ... random123
# ... ispc
# ... embree
# ... opencolorio
# ... pybind11
# ... openimageio
# ... openimagedenoise
# ... materialx
# ... usd


cmake_minimum_required (VERSION 3.23.1)

project(osx_openmoonray_deps)

set(SET_INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/install)
message("CMAKE_CURRENT_SOURCE_DIR: "${CMAKE_CURRENT_SOURCE_DIR})
message("SET_INSTALL_DIR:          "${SET_INSTALL_DIR})

include(ExternalProject)

# For Univseral
#-DCMAKE_OSX_ARCHITECTURES=x86_64;arm64

set(BUILD_AS_DEPENDENCIES 1)
message("BUILD_AS_DEPENDENCIES:    "${BUILD_AS_DEPENDENCIES})

# ----------------------------
# [0] Python
# ----------------------------
set(_PYTHON_VERSION 3.9.12)
set(_PYTHON_VERSION_SHORT 3.9)
set(_PYTHON_BUILD 3)
set(_PYTHON_ROOT ${SET_INSTALL_DIR}/${_PYTHON_VERSION}-r${_PYTHON_BUILD})
set(_PYTHON_BUILD_CONFIG
  --enable-shared
  --with-pymalloc)

if(UNIX)

  if(APPLE)

    # on macOS we depend on homebrew's openssl@1.1 and zlib
    execute_process(
      COMMAND brew --prefix openssl@1.1
      OUTPUT_VARIABLE sslPrefixDir
      OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(
      COMMAND brew --prefix zlib
      OUTPUT_VARIABLE zlibPrefixDir
      OUTPUT_STRIP_TRAILING_WHITESPACE)
    list(APPEND _PYTHON_BUILD_CONFIG
      --with-openssl=${sslPrefixDir}
      ZLIB_CFLAGS="-I${zlibPrefixDir}/include"
      ZLIB_LIBS="-L${zlibPrefixDir}/lib -lzlib")

  endif(APPLE)

  ExternalProject_Add(python
    URL "https://www.python.org/ftp/python/${_PYTHON_VERSION}/Python-${_PYTHON_VERSION}.tgz"
    INSTALL_DIR ${_PYTHON_ROOT}
    # PLATFORM BUILD --------
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=<INSTALL_DIR> ${_PYTHON_BUILD_CONFIG}
    # UNIVERSAL OSX BUILD (needs special configure)
    #CONFIGURE_COMMAND <SOURCE_DIR>/configure --enable-universalsdk --with-universal-archs=universal2 --prefix=<INSTALL_DIR> ${_PYTHON_BUILD_CONFIG}
    BUILD_IN_SOURCE 1
    EXCLUDE_FROM_ALL 1)

endif(UNIX)

set_target_properties(python
  PROPERTIES
  OM_PYTHON_VERSION ${_PYTHON_VERSION}
  OM_PYTHON_BUILD ${_PYTHON_BUILD}
  OM_PYTHON_PATH ${_PYTHON_ROOT})



set(PYTHON_LIBRARY )

# ----------------------------
# [1] Blosc 
# ----------------------------
ExternalProject_Add(blosc
    GIT_REPOSITORY https://github.com/Blosc/c-blosc
    GIT_TAG a44e961498cdca2eb893fa897cd15dd007fad496 # 1.20.1
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
)

# ----------------------------
# [2] Boost
# ----------------------------
set(PYTHON_VER 3.9)
set(PYTHON_ROOT ${_PYTHON_ROOT})
ExternalProject_Add(boost
    #URL https://sourceforge.net/projects/boost/files/boost/1.73.0/boost_1_73_0.tar.gz
    URL https://sourceforge.net/projects/boost/files/boost/1.70.0/boost_1_70_0.tar.gz
    UPDATE_COMMAND ./bootstrap.sh --prefix=${SET_INSTALL_DIR} cxxflags="-arch arm64"
    CONFIGURE_COMMAND ""
    #BUILD_COMMAND ./b2 install -j64 --user-config=${CMAKE_CURRENT_SOURCE_DIR}/custom_python_config.jam --with_python variant=release toolset=gcc link=shared threading=multi
    BUILD_COMMAND ./b2 install -j64 --user-config=${CMAKE_CURRENT_SOURCE_DIR}/custom_python_config.jam --with_python variant=release toolset=clang link=shared threading=multi cxxflags="-arch x86_64 -arch arm64"
    BUILD_IN_SOURCE 1
    INSTALL_COMMAND ""
    #DEPENDS blosc
)

# ----------------------------
# [3] JsonCpp
# ----------------------------
message("_PYTHON_ROOT: "${_PYTHON_ROOT})
# import sysconfig
# sysconfig.get_paths()

set(PYTHON_INCLUDE_DIR ${_PYTHON_ROOT}/include/python3.9)

ExternalProject_Add(jsoncpp
    GIT_REPOSITORY https://github.com/open-source-parsers/jsoncpp.git
    GIT_TAG 15949af09856076b584c7b432f065386ccd267a9 # 0.7.0
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DJSONCPP_LIB_BUILD_SHARED:BOOL=ON
        -DPYTHON_INCLUDE_DIR=${_PYTHON_ROOT}/include/python3.9
        -DPYTHON_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
        -DJSONCPP_WITH_TESTS=OFF
    #DEPENDS boost
)


# ----------------------------
# [4] Lua
# ----------------------------
ExternalProject_Add(lua
    URL https://www.lua.org/ftp/lua-5.3.5.tar.gz
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND ""
# 'make linux' uses the readline library : use 'make generic' to
# avoid this dependency
    BUILD_COMMAND make generic MYCFLAGS=-fPIC MYLIBS=-ldl
    INSTALL_COMMAND cp src/lua src/luac ${SET_INSTALL_DIR}/bin
            COMMAND cp src/lua.h src/luaconf.h src/lualib.h src/lauxlib.h src/lua.hpp ${SET_INSTALL_DIR}/include
            COMMAND cp src/liblua.a ${SET_INSTALL_DIR}/lib
    #DEPENDS jsoncpp
)


# ----------------------------
# [5] MicroHttpd
# ----------------------------
ExternalProject_Add(microhttpd
    URL https://ftp.gnu.org/gnu/libmicrohttpd/libmicrohttpd-0.9.37.tar.gz
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND ./configure --prefix ${SET_INSTALL_DIR}
    BUILD_COMMAND ${MAKE}
    INSTALL_COMMAND make install
    #DEPENDS lua
)


# ----------------------------
# [6] OpenSubdiv
# ----------------------------
ExternalProject_Add(opensubdiv
    GIT_REPOSITORY https://github.com/PixarAnimationStudios/OpenSubdiv
    GIT_TAG 82ab1b9f54c87fdd7e989a3470d53e137b8ca270 # v3_4_3
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
      -DCMAKE_BUILD_TYPE=Release
      -DNO_PTEX=1 -DNO_OMP=1 -DNO_TBB=1 -DNO_CUDA=1 -DNO_GLFW_X11=1
      -DNO_OPENCL=1 -DNO_CLEW=1 -DNO_REGRESSION=1 -DNO_EXAMPLES=1 -DNO_TUTORIALS=1 
      -DPython_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
    #DEPENDS microhttpd
)


# ----------------------------
# [7] OpenEXR
# ----------------------------
ExternalProject_Add(openexr
	COMMAND ${_PYTHON_ROOT}/bin/pip3 install numpy
    GIT_REPOSITORY https://github.com/AcademySoftwareFoundation/openexr
    GIT_TAG 8bc3741131db146ad08a5b83af9e6e48f0e94a03 # v2.5.7
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DCMAKE_OSX_ARCHITECTURES=arm64
        -DBUILD_SHARED_LIBS=OFF
        -DPython3_INCLUDE_DIR=${_PYTHON_ROOT}/include/python3.9
        -DPython3_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
        -DPYILMBASE_ENABLE=OFF
        -DOPENEXR_VIEWERS_ENABLE=OFF
        -DBUILD_TESTING=OFF
    #DEPENDS opensubdiv
)


# ----------------------------
# [8] TBB
# ----------------------------
# if Windows():
#     TBB_URL = "https://github.com/oneapi-src/oneTBB/releases/download/2019_U6/tbb2019_20190410oss_win.zip"
#     TBB_ROOT_DIR_NAME = "tbb2019_20190410oss"
# elif MacOS():
#     # On MacOS Intel systems we experience various crashes in tests during
#     # teardown starting with 2018 Update 2. Until we figure that out, we use
#     # 2018 Update 1 on this platform.
#     TBB_URL = "https://github.com/oneapi-src/oneTBB/archive/refs/tags/2019_U6.tar.gz"
#     TBB_INTEL_URL = "https://github.com/oneapi-src/oneTBB/archive/refs/tags/2018_U1.tar.gz"
# else:
#     TBB_URL = "https://github.com/oneapi-src/oneTBB/archive/refs/tags/2019_U6.tar.gz"

ExternalProject_Add(tbb
	# ---- ORIG OPENMOONRAY TBB ----
    URL "https://github.com/oneapi-src/oneTBB/archive/refs/tags/2019_U6.tar.gz"
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND ""
    #BUILD_COMMAND ${MAKE}
    BUILD_COMMAND make arch=arm64
    INSTALL_COMMAND bash -c "cp build/*_release/libtbb*.* ${SET_INSTALL_DIR}/lib"
            COMMAND bash -c "cp -r include/tbb ${SET_INSTALL_DIR}/include"
    #DEPENDS openexr
)


# ----------------------------
# [9] OpenVDB
# ----------------------------
ExternalProject_Add(openvdb
    GIT_REPOSITORY https://github.com/AcademySoftwareFoundation/openvdb
    GIT_TAG 88ae51a34ca70a23442dc60bba83894f35f7f728 # v8.2.0
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS 
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DBLOSC_ROOT=${SET_INSTALL_DIR}
    #DEPENDS tbb
)



# ----------------------------
# [10] Log4CPlus
# ----------------------------
ExternalProject_Add(log4cplus
    GIT_REPOSITORY https://github.com/log4cplus/log4cplus
    GIT_TAG bd2dfe9ef0817277a7d41fadecd84addaf943fe3 # REL_1_1_2
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS 
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
    #DEPENDS openvdb
)

# ----------------------------
# [11] CppUnit
# ----------------------------
ExternalProject_Add(cppunit
	# needed for arm64
    URL http://dev-www.libreoffice.org/src/cppunit-1.15.1.tar.gz
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND ./configure --prefix ${SET_INSTALL_DIR}
    BUILD_COMMAND ${MAKE}
    INSTALL_COMMAND make install
    #DEPENDS log4cplus
)

# ----------------------------
# [12] Random123 (for nvidia?)
# ----------------------------
ExternalProject_Add(random123
    GIT_REPOSITORY https://github.com/DEShawResearch/random123
    GIT_TAG 726a093cd9a73f3ec3c8d7a70ff10ed8efec8d13 # v1.14.0
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND make install-include prefix=${SET_INSTALL_DIR}
    #DEPENDS cppunit
)

# ----------------------------
# [13] ISPC
# ----------------------------
ExternalProject_Add(ispc
    #URL https://github.com/ispc/ispc/releases/download/v1.14.1/ispc-v1.14.1-linux.tar.gz
    URL https://github.com/ispc/ispc/releases/download/v1.19.0/ispc-v1.19.0-macOS.arm.tar.gz
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND cp bin/ispc ${SET_INSTALL_DIR}/bin
    #DEPENDS random123
)

# ----------------------------
# [14] Embree
# ----------------------------
ExternalProject_Add(embree
    # GIT_REPOSITORY https://github.com/embree/embree
    # GIT_TAG 69bd4c272f1ed608494f233ecfff3feec516880b # v3.12.1
    # PATCH_COMMAND patch -p1 < ${SET_INSTALL_DIR}/embree_missing_type.patch

    # For MacOS we use version 3.13.3 to include a fix from Intel
	# to build on Apple Silicon.
    URL "https://github.com/embree/embree/archive/v3.13.3.tar.gz" # from USD
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS 
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DEMBREE_ISPC_EXECUTABLE=<INSTALL_DIR>/bin/ispc
        -DEMBREE_TBB_ROOT=<INSTALL_DIR>
        -DEMBREE_IGNORE_INVALID_RAYS=ON
        -DEMBREE_RAY_MASK=ON
        -DEMBREE_MAX_ISA=AVX
        -DEMBREE_TUTORIALS=OFF
        -DBUILD_SHARED_LIBS=ON
        -DEMBREE_TUTORIALS=OFF # from USD
        #-DEMBREE_ISPC_SUPPORT=OFF # from USD
        -DEMBREE_MAX_ISA=NEON # from USD
        -DEMBREE_ISA_NEON=ON # from USD
    #DEPENDS ispc
)


# ----------------------------
# [15] OpenColorIO
# ----------------------------

ExternalProject_Add(opencolorio
    GIT_REPOSITORY https://github.com/AcademySoftwareFoundation/OpenColorIO
    GIT_TAG 056b7b0cb0d087961e9dba75104820e44faf52a1 # v2.0.2
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
    	-DHalf_LIBRARY=${SET_INSTALL_DIR}/lib/libHalf-2_5.a
    	-DHalf_INCLUDE_DIR=${SET_INSTALL_DIR}/include
    	-DCMAKE_OSX_ARCHITECTURES:STRING=arm64
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DBUILD_SHARED_LIBS=ON
        -DOCIO_BUILD_STATIC=OFF
        -DCMAKE_CXX_STANDARD=17
        -DPython_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
        -DPython_INCLUDE_DIR=${_PYTHON_ROOT}/include/python3.9
        #-DImath_ROOT=${SET_INSTALL_DIR}
        #-DImath_DIR=${SET_INSTALL_DIR}/lib/cmake/OpenEXR
        #-DImath_INCLUDE_DIR=${SET_INSTALL_DIR}/include
        #-DImath_LIBRARY=${SET_INSTALL_DIR}/lib/libHalf-2_5.a
    #DEPENDS embree
)

# ${SET_INSTALL_DIR}/lib/libHalf-2_5.a



# ----------------------------
# [15.5] pybind11
# ----------------------------
ExternalProject_Add(pybind11
	GIT_REPOSITORY "https://github.com/pybind/pybind11.git"
	GIT_TAG "v2.10.4"
	INSTALL_DIR ${SET_INSTALL_DIR}
	CMAKE_ARGS
		-DCMAKE_BUILD_TYPE=Release 
		-DPYBIND11_TEST=OFF 
		-DPYTHON_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
		-DCMAKE_INSTALL_PREFIX=${SET_INSTALL_DIR} 
		# -DPYBIND11_PYTHON_VERSION=${_PYTHON_VERSION}
	# CONFIGURE_COMMAND ""
	# COMMAND       ${CMAKE_COMMAND} -DCMAKE_BUILD_TYPE=Release -DPYBIND11_TEST=OFF -DPYTHON_EXECUTABLE=${_PYTHON_ROOT}/bin/python3 -DCMAKE_INSTALL_PREFIX=${_PYBIND11_ROOT} -DPYBIND11_PYTHON_VERSION=${_PYTHON_VERSION}
	# INSTALL_COMMAND make install
	# BUILD_IN_SOURCE 1
	# EXCLUDE_FROM_ALL 1
)



# ----------------------------
# [16] OpenImageIO
# ----------------------------
ExternalProject_Add(openimageio
    GIT_REPOSITORY https://github.com/OpenImageIO/oiio
    GIT_TAG 55ebad1f77f06527b23b4fe9d0da81fe8d0ecdc8 # 2.2.15.1
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
    	-DCMAKE_OSX_ARCHITECTURES:STRING=arm64
    	-DBOOST_ROOT=${SET_INSTALL_DIR}
        -DOpenEXR_ROOT=<INSTALL_DIR>
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DUSE_QT=0
        -DUSE_PYTHON=1
        -DENABLE_FFmpeg=0
        -DPython_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
        -DTBB_ROOT=${SET_INSTALL_DIR}
        -DOIIO_BUILD_TESTS=OFF
    #DEPENDS opencolorio
)

# ----------------------------
# [17] OpenImageDenoise
# ----------------------------
ExternalProject_Add(openimagedenoise
    URL https://github.com/OpenImageDenoise/oidn/releases/download/v1.4.3/oidn-1.4.3.src.tar.gz
    # GIT_REPOSITORY https://github.com/OpenImageDenoise/oidn
    # GIT_TAG d959bac5b7130b31c41095811ddfbe58c4cf03f4 # 1.4.3
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DISPC_EXECUTABLE=<INSTALL_DIR>/bin/ispc
        -DTBB_ROOT=<INSTALL_DIR>
        -DOIDN_APPS=OFF
    #DEPENDS openimageio
)


# ----------------------------
# [18] MaterialX
# ----------------------------
ExternalProject_Add(materialx
	#GIT_REPOSITORY https://github.com/AcademySoftwareFoundation/MaterialX/archive/v1.38.4.zip
	URL https://github.com/AcademySoftwareFoundation/MaterialX/releases/download/v1.38.7/MaterialX-1.38.7.tar.gz
    #GIT_REPOSITORY https://github.com/materialx/MaterialX/archive/v1.38.4.zip
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DMATERIALX_BUILD_SHARED_LIBS=ON
        -DMATERIALX_BUILD_OIIO=ON
        -DMATERIALX_OIIO_DIR=${SET_INSTALL_DIR}
        -DMATERIALX_BUILD_TESTS=ON
        -DMATERIALX_BUILD_VIEWER=ON
        -DMATERIALX_BUILD_GRAPH_EDITOR=ON
        -DMATERIALX_BUILD_PYTHON=ON
        -DMATERIALX_PYTHON_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
    #DEPENDS openimageio
)

# ----------------------------
# [19] USD
# ----------------------------
# ExternalProject_Add(setup_python_usd
# 	URL ""
# 	INSTALL_DIR ${SET_INSTALL_DIR}
#     CONFIGURE_COMMAND ""
#     BUILD_COMMAND ${_PYTHON_ROOT}/bin/pip3 install -U pip
# 	COMMAND     ${_PYTHON_ROOT}/bin/pip3 install pyside6==6.5.0 pyopengl jinja2
# )



ExternalProject_Add(usd
    GIT_REPOSITORY https://github.com/PixarAnimationStudios/USD
    # GIT_TAG 7a5f8c4311fed3ef2271d5e4b51025fb0f513730 # v21.08
    #GIT_TAG 3b097e3ba8fabf1777a1256e241ea15df83f3065 # v22.05
    # --------------------------------------------------
    # arm64 requires PySide6 which is introduced here
    GIT_TAG "v23.05" 
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
        -DCMAKE_PREFIX_PATH=<INSTALL_DIR>
        -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DPXR_USE_PYTHON_3=ON
        -DPXR_ENABLE_PYTHON_SUPPORT=ON
        -DPython3_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
        -DPython3_LIBRARY=${_PYTHON_ROOT}/lib/libpython3.9.dylib
        -DPython3_INCLUDE_DIR=${_PYTHON_ROOT}/include/python3.9
        -DTBB_USE_DEBUG_BUILD=OFF
        -DPXR_BUILD_TESTS=OFF
        -DPXR_BUILD_EXAMPLES=OFF
        -DPXR_BUILD_TUTORIALS=OFF
        -DPXR_BUILD_USD_TOOLS=OFF
        -DPXR_ENABLE_PTEX_SUPPORT=OFF
        -DPXR_ENABLE_OPENVDB_SUPPORT=OFF
        -DPXR_BUILD_USDVIEW=OFF
        -DBoost_NO_BOOST_CMAKE=ON 
        -DBoost_NO_SYSTEM_PATHS=ON
)

ExternalProject_Add(usd_larger
    GIT_REPOSITORY https://github.com/PixarAnimationStudios/USD
    # GIT_TAG 7a5f8c4311fed3ef2271d5e4b51025fb0f513730 # v21.08
    #GIT_TAG 3b097e3ba8fabf1777a1256e241ea15df83f3065 # v22.05
    # --------------------------------------------------
    # arm64 requires PySide6 which is introduced here
    GIT_TAG "v23.05" 
    INSTALL_DIR ${SET_INSTALL_DIR}
    CMAKE_ARGS
    	-DCMAKE_OSX_ARCHITECTURES:STRING=arm64
    	-DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
        -DCMAKE_PREFIX_PATH=<INSTALL_DIR>
        -DPXR_USE_PYTHON_3=ON
        -DPXR_ENABLE_PYTHON_SUPPORT=ON
        -DPython3_EXECUTABLE=${_PYTHON_ROOT}/bin/python3.9
        -DPython3_LIBRARY=${_PYTHON_ROOT}/lib/libpython3.9.dylib
        -DPython3_INCLUDE_DIR=${_PYTHON_ROOT}/include/python3.9
        -DTBB_USE_DEBUG_BUILD=OFF
        -DPXR_BUILD_TESTS=OFF
        -DPXR_BUILD_EXAMPLES=OFF
        -DPXR_BUILD_TUTORIALS=OFF
        -DPXR_BUILD_USD_TOOLS=ON
        -DPXR_BUILD_USDVIEW=OFF
        #-DPXR_BUILD_USDVIEW=ON
        -DPXR_ENABLE_PTEX_SUPPORT=OFF
        #-DPXR_PREFER_SAFETY_OVER_SPEED=ON
        #-DPXR_ENABLE_OPENVDB_SUPPORT=ON
        #-DPXR_BUILD_EMBREE_PLUGIN=ON
        -DPXR_BUILD_OPENIMAGEIO_PLUGIN=ON
        -DPXR_BUILD_OPENCOLORIO_PLUGIN=ON
        #-DPXR_BUILD_ALEMBIC_PLUGIN=ON
        #-DPXR_ENABLE_MATERIALX_SUPPORT=ON
        #-DPXR_BUILD_PYTHON_DOCUMENTATION=ON
        # -DPYSIDE_AVAILABLE=True
        # -DPYSIDEUICBINARY=${_PYTHON_ROOT}/bin/pyside6-uic
        # -DBoost_NO_BOOST_CMAKE=ON 
        # -DBoost_NO_SYSTEM_PATHS=ON
        # -DBOOST_ROOT=${SET_INSTALL_DIR}
        # -DBoost_INCLUDE_DIR=${SET_INSTALL_DIR}/include
        # -DBoost_LIBRARY_DIR=${SET_INSTALL_DIR}/lib
        -DBoost_INCLUDE_DIR=${SET_INSTALL_DIR}/include
        -DBoost_ROOT=${SET_INSTALL_DIR}
        -DBoost_NO_BOOST_CMAKE=On
        -DBoost_NO_SYSTEM_PATHS=True
        
    #DEPENDS materialx
    #DEPENDS openimagedenoise
)


# set(_USD_BUILD_CONFIG
#   --ptex
#   #--openimageio
#   --opencolorio
#   #--prman
#   #--prman-location /anim/toolsrels/renderman/RenderManProServer-24.3
#   --materialx
#   #--openvdb
#   --alembic
#   --hdf5
#   --no-tests
#   #--force OpenImageIO
#   --force-all
#   #--build-args=OpenImageIO,-DUSE_PYTHON=OFF
#   --build-target arm64
#   --build-args=USD,"-DPXR_BUILD_OPENIMAGEIO_PLUGIN=ON -DOIIO_LOCATION=/Users/dillonbailey/DEV/oiio/5.9.23/oiio/dist"

#   )

# if(UNIX)
  
#   ExternalProject_Add(usd
#     # We have a small patch for non-framework builds of python in our repo
#     # which means the git tag we're checking out contains that patch on
#     # top of the specified release.

#     #GIT_REPOSITORY "git@github.com:digitalfish/usd-opensource.git"
#     #GIT_TAG "df-pipeline-ci"
#     GIT_REPOSITORY "https://github.com/PixarAnimationStudios/USD.git"
#     GIT_TAG "v23.05"
#     INSTALL_DIR ${_USD_ROOT}
#     CONFIGURE_COMMAND ""
#     BUILD_COMMAND ${_PYTHON_ROOT}/bin/pip3 install -U pip
#     COMMAND       ${_PYTHON_ROOT}/bin/pip3 install pyside6==6.5.0 pyopengl jinja2
#     COMMAND       ${CMAKE_COMMAND} -E env VERBOSE=1 PATH=${_PYTHON_ROOT}/bin:$ENV{PATH} python3 build_scripts/build_usd.py --src <TMP_DIR>/src --build <TMP_DIR>/build ${_USD_BUILD_CONFIG} ${_USD_ROOT}
#     INSTALL_COMMAND ""
#     BUILD_IN_SOURCE 1
#     EXCLUDE_FROM_ALL 1)

# endif(UNIX)

# =================================
# set(BUILD_RF_FOR_UE 1)

# if(BUILD_RF_FOR_UE)
# 	target_compile_definitions( ${PROJECT_NAME} PUBLIC DF_REFLEX_UNREAL=1)
# 	message("build this for UE")
# endif(BUILD_RF_FOR_UE)

#set(CMAKE_OSX_ARCHITECTURES=arm64;x86_64

# if(APPLE)
# 	message("hello apple")
# execute_process(COMMAND uname -m
# OUTPUT_VARIABLE CMAKE_OSX_ARCHITECTURES
# OUTPUT_STRIP_TRAILING_WHITESPACE)
# endif(APPLE)

# set(CMAKE_REQUIRED_LINK_OPTIONS "-arch;x86_64")
# check_c_compiler_flag("-arch x86_64" x86_64Supported)
# message("x86_64Supported=${x86_64Supported}")

# set(CMAKE_REQUIRED_LINK_OPTIONS "-arch;arm64")
# check_c_compiler_flag("-arch arm64" arm64Supported)
# message("arm64Supported=${arm64Supported}")



