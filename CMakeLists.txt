cmake_minimum_required(VERSION 3.5 FATAL_ERROR)

# Print a message that using the Makefiles is recommended.
message(NOTICE: " The CMakeLists.txt is unmaintained. Use the Makefile if possible.")

# Enforce "Debug" as standard build type.
if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

# CMake project configuration.
project(yquake2 C)

# Cmake module search path.
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/stuff/cmake/modules ${CMAKE_MODULE_PATH})
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED OFF)

if(YQUAKE2LIBS)
	if(CMAKE_CROSSCOMPILING)
		set(CMAKE_FIND_ROOT_PATH ${YQUAKE2LIBS})
	else()
		set(ENV{CMAKE_PREFIX_PATH} ${YQUAKE2LIBS})
	endif()

	set(ENV{OPENALDIR} ${YQUAKE2LIBS})
	set(ENV{SDL2DIR} ${YQUAKE2LIBS})
endif()

# Add extended path for FreeBSD and Homebrew on OS X.
list(APPEND CMAKE_PREFIX_PATH /usr/local)

if (MSVC)
	add_compile_options(/MP) # parallel build (use all cores, or as many as configured in VS)

	# ignore some compiler warnings
	add_compile_options(/wd4244 /wd4305) # possible loss of data/truncation (double to float etc; ignore)
	add_compile_options(/wd4018) # signed/unsigned mismatch
	add_compile_options(/wd4996) # 'function': was declared deprecated (like all that secure CRT stuff)
	# don't show me warnings for system headers, why the fuck isn't this default
	add_compile_options(/experimental:external /external:W0)
else() # GCC/clang/mingw
# Enforce compiler flags:
#  -Wall                -> More warnings
#  -fno-strict-aliasing -> Quake 2 is far away from strict aliasing
#  -fwrapv              -> Make signed integer overflows defined
#  -fvisibility=hidden	-> Force defaultsymbol visibility to hidden
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -fno-strict-aliasing -fwrapv -fvisibility=hidden")

# Use -O2 as maximum optimization level. -O3 has it's problems with yquake2.
string(REPLACE "-O3" "-O2" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
endif() # MSVC'S else-case

# Switch off some annoying warnings
if (${CMAKE_C_COMPILER_ID} STREQUAL "Clang")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-missing-braces")
elseif (${CMAKE_C_COMPILER_ID} STREQUAL "GNU")
	if (CMAKE_C_COMPILER_VERSION GREATER 7.99)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format-truncation -Wno-format-overflow")
	endif()
endif()

# Compilation time options.
option(CURL_SUPPORT "cURL support" ON)
option(OPENAL_SUPPORT "OpenAL support" ON)
option(SYSTEMWIDE_SUPPORT "Enable systemwide installation of game assets" OFF)
option(SDL3_SUPPORT "Build against SDL 3 instead of SDL2" OFF)
option(GL1_RENDERER "Build the GL1 renderer" ON)
option(GL3_RENDERER "Build the GL3 renderer" ON)
option(GLES1_RENDERER "Build the GLES1 renderer" OFF)
option(GLES3_RENDERER "Build the GLES3 renderer" ON)
option(SOFT_RENDERER "Build the software renderer" ON)

set(SYSTEMDIR "" CACHE STRING "Override the system default directory")

# These variables will act as our list of include folders and linker flags.
set(yquake2IncludeDirectories)
set(yquake2LinkerDirectories)
set(yquake2LinkerFlags)
set(yquake2ClientLinkerFlags)
set(yquake2ServerLinkerFlags)
set(yquake2OpenGLLinkerFlags)
set(yquake2VulkanLinkerFlags)
set(yquake2SDLLinkerFlags)
set(yquake2ZLibLinkerFlags)

# Set directory locations (allowing us to move directories easily)
set(SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/src)
set(BACKENDS_SRC_DIR ${SOURCE_DIR}/backends)
set(COMMON_SRC_DIR ${SOURCE_DIR}/common)
set(GAME_SRC_DIR ${SOURCE_DIR}/game)
set(SERVER_SRC_DIR ${SOURCE_DIR}/server)
set(CLIENT_SRC_DIR ${SOURCE_DIR}/client)
set(REF_SRC_DIR ${SOURCE_DIR}/client/refresh)

# Operating system.
set(YQ2OSTYPE "${CMAKE_SYSTEM_NAME}" CACHE STRING "Override operation system type")
add_definitions(-DYQ2OSTYPE="${YQ2OSTYPE}")

# Architecture string
# work around CMake's useless/broken CMAKE_SYSTEM_PROCESSOR (taken from dhewm3)

set(cpu ${CMAKE_SYSTEM_PROCESSOR})

# Originally, ${CMAKE_SYSTEM_PROCESSOR} was supposed to contain the *target* CPU, according to CMake's documentation.
# As far as I can tell this has always been broken (always returns host CPU) at least on Windows
# (see e.g. https://cmake.org/pipermail/cmake-developers/2014-September/011405.html) and wasn't reliable on
# other systems either, for example on Linux with 32bit userland but 64bit kernel it returned the kernel CPU type
# (e.g. x86_64 instead of i686). Instead of fixing this, CMake eventually updated their documentation in 3.20,
# now it's officially the same as CMAKE_HOST_SYSTEM_PROCESSOR except when cross-compiling (where it's explicitly set)
# So we gotta figure out the actual target CPU type ourselves..
if(NOT (CMAKE_SYSTEM_PROCESSOR STREQUAL CMAKE_HOST_SYSTEM_PROCESSOR))
	# special case: cross-compiling, here CMAKE_SYSTEM_PROCESSOR should be correct, hopefully
	# (just leave cpu at ${CMAKE_SYSTEM_PROCESSOR})
elseif(MSVC)
	# because all this wasn't ugly enough, it turned out that, unlike standalone CMake, Visual Studio's
	# integrated CMake doesn't set CMAKE_GENERATOR_PLATFORM, so I gave up on guessing the CPU arch here
	# and moved the CPU detection to MSVC-specific code in neo/sys/platform.h
else() # not MSVC and not cross-compiling, assume GCC or clang (-compatible), seems to work for MinGW as well
	execute_process(COMMAND ${CMAKE_C_COMPILER} "-dumpmachine"
	                RESULT_VARIABLE cc_dumpmachine_res
	                OUTPUT_VARIABLE cc_dumpmachine_out)
	if(cc_dumpmachine_res EQUAL 0)
		string(STRIP ${cc_dumpmachine_out} cc_dumpmachine_out) # get rid of trailing newline
		message(DEBUG "`${CMAKE_C_COMPILER} -dumpmachine` says: \"${cc_dumpmachine_out}\"")
		# gcc -dumpmachine and clang -dumpmachine seem to print something like "x86_64-linux-gnu" (gcc)
		# or "x64_64-pc-linux-gnu" (clang) or "i686-w64-mingw32" (32bit mingw-w64) i.e. starting with the CPU,
		# then "-" and then OS or whatever - so use everything up to first "-"
		string(REGEX MATCH "^[^-]+" cpu ${cc_dumpmachine_out})
		message(DEBUG "  => CPU architecture extracted from that: \"${cpu}\"")
	else()
		message(WARNING "${CMAKE_C_COMPILER} -dumpmachine failed with error (code) ${cc_dumpmachine_res}")
		message(WARNING "will use the (sometimes incorrect) CMAKE_SYSTEM_PROCESSOR (${cpu}) to determine YQ2ARCH")
	endif()
endif()

if(cpu STREQUAL "powerpc")
	set(cpu "ppc")
elseif(cpu STREQUAL "aarch64")
	# "arm64" is more obvious, and some operating systems (like macOS) use it instead of "aarch64"
	set(cpu "arm64")
elseif(cpu MATCHES "[aA][mM][dD]64" OR cpu MATCHES "[xX].*64")
	set(cpu "x86_64")
elseif(cpu MATCHES "i.86" OR cpu MATCHES "[xX]86")
	set(cpu "i386")
elseif(cpu MATCHES "[aA][rR][mM].*") # some kind of arm..
	# On 32bit Raspbian gcc -dumpmachine returns sth starting with "arm-",
	# while clang -dumpmachine says "arm6k-..." - try to unify that to "arm"
	if(CMAKE_SIZEOF_VOID_P EQUAL 8) # sizeof(void*) == 8 => must be arm64
		set(cpu "arm64")
	else() # should be 32bit arm then (probably "armv7l" "armv6k" or sth like that)
		set(cpu "arm")
	endif()
endif()

if(MSVC)
	# for MSVC YQ2ARCH is set in code (in src/common/header/common.h)
	message(STATUS "Setting YQ2OSTYPE to \"${YQ2OSTYPE}\" - NOT setting YQ2ARCH, because we're targeting MSVC (VisualC++)")
else()
	set(ARCH "${cpu}")
	add_definitions(-DYQ2ARCH="${ARCH}")
	message(STATUS "Setting YQ2OSTYPE to \"${YQ2OSTYPE}\" and YQ2ARCH to \"${ARCH}\".")
endif()
# make sure that ${cpu} isn't used below - if at all use ${ARCH}, but not when compiling with MSVC!
unset(cpu)

# END OF workarounds for CMake's poor choices regarding CPU architecture detection


# Systemwide installation of game assets.
if(${SYSTEMWIDE_SUPPORT})
    add_definitions(-DSYSTEMWIDE)
    if(NOT ${SYSTEMDIR} STREQUAL "")
      add_definitions(-DSYSTEMDIR="${SYSTEMDIR}")
    endif()
endif()

# We need to pass some options to minizip / unzip.
add_definitions(-DNOUNCRYPT)

if(NOT (CMAKE_SYSTEM_NAME MATCHES "Linux") AND NOT (CMAKE_SYSTEM_NAME MATCHES "Windows"))
	add_definitions(-DIOAPI_NO_64)
endif()

# Required libraries to build the different components of the binaries. Find
# them and add the include/linker directories and flags (in case the package
# manager find it in a weird place).
if (SDL3_SUPPORT)
	find_package(SDL3 REQUIRED)
	add_definitions(-DUSE_SDL3)
else()
	find_package(SDL2 REQUIRED)
	list(APPEND yquake2IncludeDirectories "${SDL2_INCLUDE_DIR}/..")
	list(APPEND yquake2SDLLinkerFlags ${SDL2_LIBRARY})
endif()

if(GL1_RENDERER)
     # We need an OpenGL implementation.
     set(OpenGL_GL_PREFERENCE GLVND)
     find_package(OpenGL REQUIRED)
     list(APPEND yquake2IncludeDirectories ${OPENGL_INCLUDE_DIR})
     list(APPEND yquake2OpenGLLinkerFlags ${OPENGL_LIBRARIES})
endif()

# backtrace lookup
# Some systems like Linux has it within the libc some like the BSD, Haiku ...
# into an external libexecinfo library
include(CheckFunctionExists)
include(CheckLibraryExists)
check_function_exists(backtrace HAVE_EXECINFO_SYS)
IF (NOT HAVE_EXECINFO_SYS)
	check_library_exists(execinfo backtrace "" HAVE_EXECINFO_LIB)
	if (HAVE_EXECINFO_LIB)
		list(APPEND yquake2ClientLinkerFlags execinfo)
		list(APPEND yquake2ServerLinkerFlags execinfo)
		add_definitions(-DHAVE_EXECINFO)
	endif()
else()
	add_definitions(-DHAVE_EXECINFO)
endif()

# cURL support.
if (${CURL_SUPPORT})
	find_package(CURL REQUIRED)
	add_definitions(-DUSE_CURL)
endif()

# OpenAL support.
if(${OPENAL_SUPPORT})
	find_package(OpenAL)

	if(${OPENAL_FOUND})
		list(APPEND yquake2IncludeDirectories "${OPENAL_INCLUDE_DIR}")
		list(APPEND yquake2ClientLinkerFlags ${OPENAL_LIBRARY})

		if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="openal32.dll")
		elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.dylib")
		elseif((${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD") OR (${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD"))
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.so")
		else()
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.so.1")
		endif()
	endif()
endif()

# General linker flags.
if(NOT MSVC)
	list(APPEND yquake2LinkerFlags m)
endif()
list(APPEND yquake2LinkerFlags ${CMAKE_DL_LIBS})

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	if(!MSVC)
		list(APPEND yquake2LinkerFlags "-static-libgcc")
	endif()
else()
	if (NOT ${CMAKE_SYSTEM_NAME} MATCHES "Haiku")
		list(APPEND yquake2LinkerFlags "-rdynamic")
	else()
		list(APPEND yquake2LinkerFlags "-lnetwork")
                set(CMAKE_POSITION_INDEPENDENT_CODE ON)
	endif()
	if (${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
		list(APPEND yquake2LinkerFlags "-lsocket -lnsl")
	endif()
endif()

if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin" AND NOT ${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD" AND NOT WIN32)
	list(APPEND yquake2LinkerFlags "-Wl,--no-undefined")
endif()

# With all of those libraries and user defined paths
# added, lets give them to the compiler and linker.
include_directories(${yquake2IncludeDirectories})
link_directories(${yquake2LinkerDirectories})

# these settings only work for GCC and clang
if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
	# If we're building with gcc for i386 let's define -ffloat-store.
	# This helps the old and crappy x87 FPU to produce correct values.
	# Would be nice if Clang had something comparable.
	if ("${ARCH}" STREQUAL "i386" AND ${CMAKE_C_COMPILER_ID} STREQUAL "GNU")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ffloat-store")
	endif()

	# Force SSE math on x86_64. All sane compilers should do this
	# anyway, just to protect us from broken Linux distros.
	if ("${ARCH}" STREQUAL "x86_64")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpmath=sse")
	endif()

	if ("${ARCH}" STREQUAL "arm")
		if (CMAKE_SIZEOF_VOID_P EQUAL 4)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=armv6k")
		endif()
	endif()
endif()

set(Backends-Generic-Source
	${BACKENDS_SRC_DIR}/generic/misc.c
	)

set(Backends-Unix-Source
	${BACKENDS_SRC_DIR}/unix/main.c
	${BACKENDS_SRC_DIR}/unix/network.c
	${BACKENDS_SRC_DIR}/unix/signalhandler.c
	${BACKENDS_SRC_DIR}/unix/system.c
	${BACKENDS_SRC_DIR}/unix/shared/hunk.c
	)

set(Backends-Windows-Source
	${BACKENDS_SRC_DIR}/windows/icon.rc
	${BACKENDS_SRC_DIR}/windows/main.c
	${BACKENDS_SRC_DIR}/windows/network.c
	${BACKENDS_SRC_DIR}/windows/system.c
	${BACKENDS_SRC_DIR}/windows/shared/hunk.c
	)

set(Backends-Windows-Header
	${BACKENDS_SRC_DIR}/windows/header/resource.h
	)

set(REF-Windows-Source
	${BACKENDS_SRC_DIR}/windows/shared/hunk.c
	)

set(REF-Unix-Source
	${BACKENDS_SRC_DIR}/unix/shared/hunk.c
	)

# Set the nessesary platform specific source
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	set(Platform-Specific-Source ${Backends-Windows-Source} ${Backends-Windows-Header})
	set(REF-Platform-Specific-Source ${REF-Windows-Source})
else()
	set(Platform-Specific-Source ${Backends-Unix-Source})
	set(REF-Platform-Specific-Source ${REF-Unix-Source})
endif()

set(Game-Source
	${COMMON_SRC_DIR}/shared/flash.c
	${COMMON_SRC_DIR}/shared/rand.c
	${COMMON_SRC_DIR}/shared/shared.c
	${GAME_SRC_DIR}/g_ai.c
	${GAME_SRC_DIR}/g_chase.c
	${GAME_SRC_DIR}/g_cmds.c
	${GAME_SRC_DIR}/g_combat.c
	${GAME_SRC_DIR}/g_func.c
	${GAME_SRC_DIR}/g_items.c
	${GAME_SRC_DIR}/g_main.c
	${GAME_SRC_DIR}/g_misc.c
	${GAME_SRC_DIR}/g_monster.c
	${GAME_SRC_DIR}/g_phys.c
	${GAME_SRC_DIR}/g_spawn.c
	${GAME_SRC_DIR}/g_svcmds.c
	${GAME_SRC_DIR}/g_target.c
	${GAME_SRC_DIR}/g_trigger.c
	${GAME_SRC_DIR}/g_turret.c
	${GAME_SRC_DIR}/g_utils.c
	${GAME_SRC_DIR}/g_weapon.c
	${GAME_SRC_DIR}/monster/berserker/berserker.c
	${GAME_SRC_DIR}/monster/boss2/boss2.c
	${GAME_SRC_DIR}/monster/boss3/boss3.c
	${GAME_SRC_DIR}/monster/boss3/boss31.c
	${GAME_SRC_DIR}/monster/boss3/boss32.c
	${GAME_SRC_DIR}/monster/brain/brain.c
	${GAME_SRC_DIR}/monster/chick/chick.c
	${GAME_SRC_DIR}/monster/flipper/flipper.c
	${GAME_SRC_DIR}/monster/float/float.c
	${GAME_SRC_DIR}/monster/flyer/flyer.c
	${GAME_SRC_DIR}/monster/gladiator/gladiator.c
	${GAME_SRC_DIR}/monster/gunner/gunner.c
	${GAME_SRC_DIR}/monster/hover/hover.c
	${GAME_SRC_DIR}/monster/infantry/infantry.c
	${GAME_SRC_DIR}/monster/insane/insane.c
	${GAME_SRC_DIR}/monster/medic/medic.c
	${GAME_SRC_DIR}/monster/misc/move.c
	${GAME_SRC_DIR}/monster/mutant/mutant.c
	${GAME_SRC_DIR}/monster/parasite/parasite.c
	${GAME_SRC_DIR}/monster/soldier/soldier.c
	${GAME_SRC_DIR}/monster/supertank/supertank.c
	${GAME_SRC_DIR}/monster/tank/tank.c
	${GAME_SRC_DIR}/player/client.c
	${GAME_SRC_DIR}/player/hud.c
	${GAME_SRC_DIR}/player/trail.c
	${GAME_SRC_DIR}/player/view.c
	${GAME_SRC_DIR}/player/weapon.c
	${GAME_SRC_DIR}/savegame/savegame.c
	)

set(Game-Header
	${GAME_SRC_DIR}/header/game.h
	${GAME_SRC_DIR}/header/local.h
	${GAME_SRC_DIR}/monster/berserker/berserker.h
	${GAME_SRC_DIR}/monster/boss2/boss2.h
	${GAME_SRC_DIR}/monster/boss3/boss31.h
	${GAME_SRC_DIR}/monster/boss3/boss32.h
	${GAME_SRC_DIR}/monster/brain/brain.h
	${GAME_SRC_DIR}/monster/chick/chick.h
	${GAME_SRC_DIR}/monster/flipper/flipper.h
	${GAME_SRC_DIR}/monster/float/float.h
	${GAME_SRC_DIR}/monster/flyer/flyer.h
	${GAME_SRC_DIR}/monster/gladiator/gladiator.h
	${GAME_SRC_DIR}/monster/gunner/gunner.h
	${GAME_SRC_DIR}/monster/hover/hover.h
	${GAME_SRC_DIR}/monster/infantry/infantry.h
	${GAME_SRC_DIR}/monster/insane/insane.h
	${GAME_SRC_DIR}/monster/medic/medic.h
	${GAME_SRC_DIR}/monster/misc/player.h
	${GAME_SRC_DIR}/monster/mutant/mutant.h
	${GAME_SRC_DIR}/monster/parasite/parasite.h
	${GAME_SRC_DIR}/monster/soldier/soldier.h
	${GAME_SRC_DIR}/monster/supertank/supertank.h
	${GAME_SRC_DIR}/monster/tank/tank.h
	${GAME_SRC_DIR}/savegame/tables/clientfields.h
	${GAME_SRC_DIR}/savegame/tables/fields.h
	${GAME_SRC_DIR}/savegame/tables/gamefunc_decs.h
	${GAME_SRC_DIR}/savegame/tables/gamefunc_list.h
	${GAME_SRC_DIR}/savegame/tables/gamemmove_decs.h
	${GAME_SRC_DIR}/savegame/tables/gamemmove_list.h
	${GAME_SRC_DIR}/savegame/tables/levelfields.h
	)

set(Client-Source
	${CLIENT_SRC_DIR}/cl_cin.c
	${CLIENT_SRC_DIR}/cl_console.c
	${CLIENT_SRC_DIR}/cl_download.c
	${CLIENT_SRC_DIR}/cl_effects.c
	${CLIENT_SRC_DIR}/cl_entities.c
	${CLIENT_SRC_DIR}/cl_input.c
	${CLIENT_SRC_DIR}/cl_image.c
	${CLIENT_SRC_DIR}/cl_inventory.c
	${CLIENT_SRC_DIR}/cl_keyboard.c
	${CLIENT_SRC_DIR}/cl_lights.c
	${CLIENT_SRC_DIR}/cl_main.c
	${CLIENT_SRC_DIR}/cl_network.c
	${CLIENT_SRC_DIR}/cl_parse.c
	${CLIENT_SRC_DIR}/cl_particles.c
	${CLIENT_SRC_DIR}/cl_prediction.c
	${CLIENT_SRC_DIR}/cl_screen.c
	${CLIENT_SRC_DIR}/cl_tempentities.c
	${CLIENT_SRC_DIR}/cl_view.c
	${CLIENT_SRC_DIR}/curl/download.c
	${CLIENT_SRC_DIR}/curl/qcurl.c
	${CLIENT_SRC_DIR}/input/gyro.c
	${CLIENT_SRC_DIR}/menu/menu.c
	${CLIENT_SRC_DIR}/menu/qmenu.c
	${CLIENT_SRC_DIR}/menu/videomenu.c
	${CLIENT_SRC_DIR}/sound/ogg.c
	${CLIENT_SRC_DIR}/sound/openal.c
	${CLIENT_SRC_DIR}/sound/qal.c
	${CLIENT_SRC_DIR}/sound/sdl.c
	${CLIENT_SRC_DIR}/sound/sound.c
	${CLIENT_SRC_DIR}/sound/wave.c
	${CLIENT_SRC_DIR}/vid/vid.c
	${COMMON_SRC_DIR}/argproc.c
	${COMMON_SRC_DIR}/clientserver.c
	${COMMON_SRC_DIR}/collision.c
	${COMMON_SRC_DIR}/crc.c
	${COMMON_SRC_DIR}/cmdparser.c
	${COMMON_SRC_DIR}/cvar.c
	${COMMON_SRC_DIR}/filesystem.c
	${COMMON_SRC_DIR}/glob.c
	${COMMON_SRC_DIR}/md4.c
	${COMMON_SRC_DIR}/movemsg.c
	${COMMON_SRC_DIR}/frame.c
	${COMMON_SRC_DIR}/netchan.c
	${COMMON_SRC_DIR}/pmove.c
	${COMMON_SRC_DIR}/szone.c
	${COMMON_SRC_DIR}/zone.c
	${COMMON_SRC_DIR}/shared/flash.c
	${COMMON_SRC_DIR}/shared/rand.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/unzip/ioapi.c
	${COMMON_SRC_DIR}/unzip/unzip.c
	${COMMON_SRC_DIR}/unzip/miniz/miniz.c
	${COMMON_SRC_DIR}/unzip/miniz/miniz_tdef.c
	${COMMON_SRC_DIR}/unzip/miniz/miniz_tinfl.c
	${SERVER_SRC_DIR}/sv_cmd.c
	${SERVER_SRC_DIR}/sv_conless.c
	${SERVER_SRC_DIR}/sv_entities.c
	${SERVER_SRC_DIR}/sv_game.c
	${SERVER_SRC_DIR}/sv_init.c
	${SERVER_SRC_DIR}/sv_main.c
	${SERVER_SRC_DIR}/sv_save.c
	${SERVER_SRC_DIR}/sv_send.c
	${SERVER_SRC_DIR}/sv_user.c
	${SERVER_SRC_DIR}/sv_world.c
	)

if(SDL3_SUPPORT)
	set(Client-SDL-Source
		${CLIENT_SRC_DIR}/input/sdl3.c
		${CLIENT_SRC_DIR}/vid/glimp_sdl3.c
	)
else()
	set(Client-SDL-Source
		${CLIENT_SRC_DIR}/input/sdl2.c
		${CLIENT_SRC_DIR}/vid/glimp_sdl2.c
	)
endif()

set(Client-Header
	${CLIENT_SRC_DIR}/header/client.h
	${CLIENT_SRC_DIR}/header/console.h
	${CLIENT_SRC_DIR}/header/keyboard.h
	${CLIENT_SRC_DIR}/header/screen.h
	${CLIENT_SRC_DIR}/curl/header/download.h
	${CLIENT_SRC_DIR}/curl/header/qcurl.h
	${CLIENT_SRC_DIR}/input/header/gyro.h
	${CLIENT_SRC_DIR}/input/header/input.h
	${CLIENT_SRC_DIR}/menu/header/qmenu.h
	${CLIENT_SRC_DIR}/sound/header/local.h
	${CLIENT_SRC_DIR}/sound/header/qal.h
	${CLIENT_SRC_DIR}/sound/header/sound.h
	${CLIENT_SRC_DIR}/sound/header/stb_vorbis.h
	${CLIENT_SRC_DIR}/sound/header/vorbis.h
	${CLIENT_SRC_DIR}/vid/header/ref.h
	${CLIENT_SRC_DIR}/vid/header/stb_image_write.h
	${CLIENT_SRC_DIR}/vid/header/vid.h
	${COMMON_SRC_DIR}/header/common.h
	${COMMON_SRC_DIR}/header/crc.h
	${COMMON_SRC_DIR}/header/files.h
	${COMMON_SRC_DIR}/header/glob.h
	${COMMON_SRC_DIR}/header/shared.h
	${COMMON_SRC_DIR}/header/zone.h
	${COMMON_SRC_DIR}/unzip/ioapi.h
	${COMMON_SRC_DIR}/unzip/unzip.h
	${COMMON_SRC_DIR}/unzip/miniz/miniz.h
	${COMMON_SRC_DIR}/unzip/miniz/miniz_tdef.h
	${COMMON_SRC_DIR}/unzip/miniz/miniz_tinfl.h
	${COMMON_SRC_DIR}/unzip/miniz/minizconf.h
	${SERVER_SRC_DIR}/header/server.h
	)

set(Server-Source
	${COMMON_SRC_DIR}/argproc.c
	${COMMON_SRC_DIR}/clientserver.c
	${COMMON_SRC_DIR}/collision.c
	${COMMON_SRC_DIR}/crc.c
	${COMMON_SRC_DIR}/cmdparser.c
	${COMMON_SRC_DIR}/cvar.c
	${COMMON_SRC_DIR}/filesystem.c
	${COMMON_SRC_DIR}/glob.c
	${COMMON_SRC_DIR}/md4.c
	${COMMON_SRC_DIR}/frame.c
	${COMMON_SRC_DIR}/movemsg.c
	${COMMON_SRC_DIR}/netchan.c
	${COMMON_SRC_DIR}/pmove.c
	${COMMON_SRC_DIR}/szone.c
	${COMMON_SRC_DIR}/zone.c
	${COMMON_SRC_DIR}/shared/rand.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/unzip/ioapi.c
	${COMMON_SRC_DIR}/unzip/unzip.c
	${COMMON_SRC_DIR}/unzip/miniz/miniz.c
	${COMMON_SRC_DIR}/unzip/miniz/miniz_tdef.c
	${COMMON_SRC_DIR}/unzip/miniz/miniz_tinfl.c
	${SERVER_SRC_DIR}/sv_cmd.c
	${SERVER_SRC_DIR}/sv_conless.c
	${SERVER_SRC_DIR}/sv_entities.c
	${SERVER_SRC_DIR}/sv_game.c
	${SERVER_SRC_DIR}/sv_init.c
	${SERVER_SRC_DIR}/sv_main.c
	${SERVER_SRC_DIR}/sv_save.c
	${SERVER_SRC_DIR}/sv_send.c
	${SERVER_SRC_DIR}/sv_user.c
	${SERVER_SRC_DIR}/sv_world.c
	)

set(Server-Header
	${COMMON_SRC_DIR}/header/common.h
	${COMMON_SRC_DIR}/header/crc.h
	${COMMON_SRC_DIR}/header/files.h
	${COMMON_SRC_DIR}/header/glob.h
	${COMMON_SRC_DIR}/header/shared.h
	${COMMON_SRC_DIR}/header/zone.h
	${COMMON_SRC_DIR}/unzip/ioapi.h
	${COMMON_SRC_DIR}/unzip/unzip.h
	${COMMON_SRC_DIR}/unzip/miniz/miniz.h
	${COMMON_SRC_DIR}/unzip/miniz/miniz_tdef.h
	${COMMON_SRC_DIR}/unzip/miniz/miniz_tinfl.h
	${COMMON_SRC_DIR}/unzip/miniz/minizconf.h
	${SERVER_SRC_DIR}/header/server.h
	)

set(GL1-Source
	${REF_SRC_DIR}/gl1/qgl.c
	${REF_SRC_DIR}/gl1/gl1_draw.c
	${REF_SRC_DIR}/gl1/gl1_image.c
	${REF_SRC_DIR}/gl1/gl1_light.c
	${REF_SRC_DIR}/gl1/gl1_lightmap.c
	${REF_SRC_DIR}/gl1/gl1_main.c
	${REF_SRC_DIR}/gl1/gl1_mesh.c
	${REF_SRC_DIR}/gl1/gl1_misc.c
	${REF_SRC_DIR}/gl1/gl1_model.c
	${REF_SRC_DIR}/gl1/gl1_scrap.c
	${REF_SRC_DIR}/gl1/gl1_surf.c
	${REF_SRC_DIR}/gl1/gl1_warp.c
	${REF_SRC_DIR}/gl1/gl1_sdl.c
	${REF_SRC_DIR}/gl1/gl1_buffer.c
	${REF_SRC_DIR}/files/models.c
	${REF_SRC_DIR}/files/pcx.c
	${REF_SRC_DIR}/files/stb.c
	${REF_SRC_DIR}/files/surf.c
	${REF_SRC_DIR}/files/wal.c
	${REF_SRC_DIR}/files/pvs.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/md4.c
	)

set(Glad-GLES1-Source ${REF_SRC_DIR}/gl1/glad-gles1/src/glad.c)

set(GL1-Header
	${REF_SRC_DIR}/ref_shared.h
	${REF_SRC_DIR}/constants/anorms.h
	${REF_SRC_DIR}/constants/anormtab.h
	${REF_SRC_DIR}/constants/warpsin.h
	${REF_SRC_DIR}/files/stb_image.h
	${REF_SRC_DIR}/files/surf.c
	${REF_SRC_DIR}/gl1/header/local.h
	${REF_SRC_DIR}/gl1/header/model.h
	${REF_SRC_DIR}/gl1/header/qgl.h
	${COMMON_SRC_DIR}/header/shared.h
	)

set(Glad-GLES1-Header
	${REF_SRC_DIR}/gl1/glad-gles1/include/glad/glad.h
	${REF_SRC_DIR}/gl1/glad-gles1/include/KHR/khrplatform.h
	)

set(GL3-Source
	${REF_SRC_DIR}/gl3/gl3_draw.c
	${REF_SRC_DIR}/gl3/gl3_image.c
	${REF_SRC_DIR}/gl3/gl3_light.c
	${REF_SRC_DIR}/gl3/gl3_lightmap.c
	${REF_SRC_DIR}/gl3/gl3_main.c
	${REF_SRC_DIR}/gl3/gl3_mesh.c
	${REF_SRC_DIR}/gl3/gl3_misc.c
	${REF_SRC_DIR}/gl3/gl3_model.c
	${REF_SRC_DIR}/gl3/gl3_sdl.c
	${REF_SRC_DIR}/gl3/gl3_surf.c
	${REF_SRC_DIR}/gl3/gl3_warp.c
	${REF_SRC_DIR}/gl3/gl3_shaders.c
	${REF_SRC_DIR}/files/models.c
	${REF_SRC_DIR}/files/pcx.c
	${REF_SRC_DIR}/files/stb.c
	${REF_SRC_DIR}/files/surf.c
	${REF_SRC_DIR}/files/wal.c
	${REF_SRC_DIR}/files/pvs.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/md4.c
	)

set(Glad-GL3-Source ${REF_SRC_DIR}/gl3/glad/src/glad.c)
set(Glad-GLES3-Source ${REF_SRC_DIR}/gl3/glad-gles3/src/glad.c)

set(GL3-Header
	${REF_SRC_DIR}/ref_shared.h
	${REF_SRC_DIR}/constants/anorms.h
	${REF_SRC_DIR}/constants/anormtab.h
	${REF_SRC_DIR}/constants/warpsin.h
	${REF_SRC_DIR}/files/stb_image.h
	${REF_SRC_DIR}/gl3/header/DG_dynarr.h
	${REF_SRC_DIR}/gl3/header/HandmadeMath.h
	${REF_SRC_DIR}/gl3/header/local.h
	${REF_SRC_DIR}/gl3/header/model.h
	${COMMON_SRC_DIR}/header/shared.h
	)

set(Glad-GL3-Header
	${REF_SRC_DIR}/gl3/glad/include/glad/glad.h
	${REF_SRC_DIR}/gl3/glad/include/KHR/khrplatform.h
	)

set(Glad-GLES3-Header
	${REF_SRC_DIR}/gl3/glad-gles3/include/glad/glad.h
	${REF_SRC_DIR}/gl3/glad-gles3/include/KHR/khrplatform.h
	)

set(SOFT-Source
	${REF_SRC_DIR}/soft/sw_aclip.c
	${REF_SRC_DIR}/soft/sw_alias.c
	${REF_SRC_DIR}/soft/sw_bsp.c
	${REF_SRC_DIR}/soft/sw_draw.c
	${REF_SRC_DIR}/soft/sw_edge.c
	${REF_SRC_DIR}/soft/sw_image.c
	${REF_SRC_DIR}/soft/sw_light.c
	${REF_SRC_DIR}/soft/sw_main.c
	${REF_SRC_DIR}/soft/sw_misc.c
	${REF_SRC_DIR}/soft/sw_model.c
	${REF_SRC_DIR}/soft/sw_part.c
	${REF_SRC_DIR}/soft/sw_poly.c
	${REF_SRC_DIR}/soft/sw_polyset.c
	${REF_SRC_DIR}/soft/sw_rast.c
	${REF_SRC_DIR}/soft/sw_scan.c
	${REF_SRC_DIR}/soft/sw_sprite.c
	${REF_SRC_DIR}/soft/sw_surf.c
	${REF_SRC_DIR}/files/models.c
	${REF_SRC_DIR}/files/pcx.c
	${REF_SRC_DIR}/files/stb.c
	${REF_SRC_DIR}/files/surf.c
	${REF_SRC_DIR}/files/wal.c
	${REF_SRC_DIR}/files/pvs.c
	${COMMON_SRC_DIR}/shared/shared.c
	${COMMON_SRC_DIR}/md4.c
	)

set(SOFT-Header
	${REF_SRC_DIR}/ref_shared.h
	${REF_SRC_DIR}/files/stb_image.h
	${REF_SRC_DIR}/files/stb_image_resize.h
	${REF_SRC_DIR}/soft/header/local.h
	${REF_SRC_DIR}/soft/header/model.h
	${COMMON_SRC_DIR}/header/shared.h
	)

# Main Quake 2 executable

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	add_executable(yquake2 WIN32 ${Client-Source} ${Client-SDL-Source} ${Client-Header}
			${Platform-Specific-Source} ${Backends-Generic-Source})
	set_target_properties(yquake2 PROPERTIES
			RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
			)
	target_link_libraries(yquake2 ${yquake2LinkerFlags} ${yquake2ClientLinkerFlags}
			${yquake2SDLLinkerFlags} ${yquake2ZLibLinkerFlags} ws2_32 winmm)
	if(SDL3_SUPPORT)
		target_link_libraries(yquake2 SDL3::SDL3)
	endif()

	if(MSVC AND CMAKE_MAJOR_VERSION GREATER 3 OR ( CMAKE_MAJOR_VERSION EQUAL 3 AND CMAKE_MINOR_VERSION GREATER_EQUAL 6 ))
		# CMake >= 3.6 supports setting the default project started for debugging (instead of trying to launch ALL_BUILD ...)
		set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT yquake2)
		set_target_properties(yquake2 PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/release)
	endif()

	# Wrapper for the Windows binary
	set(Wrapper-Source
		src/win-wrapper/wrapper.c
		${BACKENDS_SRC_DIR}/windows/icon.rc
		)
	add_executable(quake2 WIN32 ${Wrapper-Source})
	set_target_properties(quake2 PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release)
else()
	add_executable(quake2 ${Client-Source} ${Client-SDL-Source} ${Client-Header}
			${Platform-Specific-Source} ${Backends-Generic-Source})
	set_target_properties(quake2 PROPERTIES
			RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
			)
	target_link_libraries(quake2 ${yquake2LinkerFlags} ${yquake2ClientLinkerFlags}
		${yquake2SDLLinkerFlags} ${yquake2ZLibLinkerFlags})
	if(SDL3_SUPPORT)
		target_link_libraries(quake2 SDL3::SDL3)
	endif()
endif()

# Quake 2 Dedicated Server
add_executable(q2ded ${Server-Source} ${Server-Header} ${Platform-Specific-Source}
		${Backends-Generic-Source})
set_target_properties(q2ded PROPERTIES
	COMPILE_DEFINITIONS "DEDICATED_ONLY"
	RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
	)
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	target_link_libraries(q2ded ${yquake2LinkerFlags} ${yquake2ZLibLinkerFlags} ws2_32 winmm)
else()
	target_link_libraries(q2ded ${yquake2LinkerFlags} ${yquake2ServerLinkerFlags} ${yquake2ZLibLinkerFlags})
endif()

# Build the game dynamic library
add_library(game MODULE ${Game-Source} ${Game-Header})
set_target_properties(game PROPERTIES
		PREFIX ""
		SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX}
		)
get_property(isMultiConfig GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if(isMultiConfig) # multi-config, like Visual Studio solution
	set_target_properties(game PROPERTIES
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release/$<CONFIG>/baseq2
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release/$<CONFIG>/baseq2
		)
else() # single-config, like normal Makefiles
	set_target_properties(game PROPERTIES
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release/baseq2
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release/baseq2
		)
endif()
target_link_libraries(game ${yquake2LinkerFlags})

if(${GL1_RENDERER})

# Build the GL1 dynamic library
add_library(ref_gl1 MODULE ${GL1-Source} ${GL1-Header} ${REF-Platform-Specific-Source})
set_target_properties(ref_gl1 PROPERTIES
		PREFIX ""
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX}
		)
target_link_libraries(ref_gl1 ${yquake2LinkerFlags} ${yquake2OpenGLLinkerFlags}
		${yquake2SDLLinkerFlags})
if(SDL3_SUPPORT)
	target_link_libraries(ref_gl1 SDL3::SDL3)
endif()

endif()

if(${GL3_RENDERER})

# Build the GL3 dynamic library
add_library(ref_gl3 MODULE ${GL3-Source} ${Glad-GL3-Source} ${GL3-Header} ${Glad-GL3-Header} ${REF-Platform-Specific-Source})
set_target_properties(ref_gl3 PROPERTIES
		PREFIX ""
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX}
		)
target_include_directories(ref_gl3 PRIVATE ${CMAKE_SOURCE_DIR}/src/client/refresh/gl3/glad/include)
target_link_libraries(ref_gl3 ${yquake2LinkerFlags} ${yquake2SDLLinkerFlags})
if(SDL3_SUPPORT)
	target_link_libraries(ref_gl3 SDL3::SDL3)
endif()

endif()

if(${GLES3_RENDERER})

# Build the GLES3 dynamic library
add_library(ref_gles3 MODULE ${GL3-Source} ${Glad-GLES3-Source} ${GL3-Header} ${Glad-GLES3-Header} ${REF-Platform-Specific-Source})
set_target_properties(ref_gles3 PROPERTIES
		PREFIX ""
		#COMPILE_DEFINITIONS "YQ2_GL3_GLES3=1;YQ2_GL3_GLES=1"
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX}
		)
target_include_directories(ref_gles3 PRIVATE ${CMAKE_SOURCE_DIR}/src/client/refresh/gl3/glad-gles3/include)
target_compile_definitions(ref_gles3 PRIVATE YQ2_GL3_GLES3=1 YQ2_GL3_GLES=1)
target_link_libraries(ref_gles3 ${yquake2LinkerFlags} ${yquake2SDLLinkerFlags})
if(SDL3_SUPPORT)
	target_link_libraries(ref_gles3 SDL3::SDL3)
endif()

endif()

if(${SOFT_RENDERER})

# Build the soft renderer dynamic library
add_library(ref_soft MODULE ${SOFT-Source} ${SOFT-Header} ${REF-Platform-Specific-Source})
set_target_properties(ref_soft PROPERTIES
		PREFIX ""
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX}
		)
target_link_libraries(ref_soft ${yquake2LinkerFlags} ${yquake2SDLLinkerFlags})
if(SDL3_SUPPORT)
	target_link_libraries(ref_soft SDL3::SDL3)
endif()

endif()

if(${GLES1_RENDERER})

# Build the GLES1 dynamic library
add_library(ref_gles1 MODULE ${GL1-Source} ${Glad-GLES1-Source} ${GL1-Header} ${Glad-GLES1-Header} ${REF-Platform-Specific-Source})
set_target_properties(ref_gles1 PROPERTIES
		PREFIX ""
		#COMPILE_DEFINITIONS "YQ2_GL1_GLES=1"
		LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/release
		SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX}
		)
target_include_directories(ref_gles1 PRIVATE ${CMAKE_SOURCE_DIR}/src/client/refresh/gl1/glad-gles1/include)
target_compile_definitions(ref_gles1 PRIVATE YQ2_GL1_GLES=1)
target_link_libraries(ref_gles1 ${yquake2LinkerFlags} ${yquake2SDLLinkerFlags})
if(SDL3_SUPPORT)
	target_link_libraries(ref_gles1 SDL3::SDL3)
endif()

endif()

