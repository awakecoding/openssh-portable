
cmake_minimum_required(VERSION 3.18)

if(WIN32)
    set(WINSDK_VERSION "10.0.17763.0")
    set(CMAKE_SYSTEM_VERSION ${WINSDK_VERSION})
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

    get_filename_component(KITS_ROOT_10
	    "[HKEY_LOCAL_MACHINE\\SOFTWARE\\WOW6432Node\\Microsoft\\Microsoft SDKs\\Windows\\v10.0;InstallationFolder]" ABSOLUTE CACHE)
    file(TO_CMAKE_PATH "${KITS_ROOT_10}" WindowsSdkDir)
    string(APPEND WindowsSdkDir "/")
    set(VC_INCLUDE_PATH "${WindowsSdkDir}include/${WINSDK_VERSION}/ucrt")
endif()

project(OpenSSH C)

include(CheckIncludeFiles)
include(CheckCCompilerFlag)
include(CMakeDependentOption)

cmake_policy(SET CMP0091 NEW)

set(CMAKE_SHARED_LIBRARY_PREFIX "")
set(CMAKE_STATIC_LIBRARY_PREFIX "")

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)

set(OPENSSH_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(OPENSSH_BIN_DIR "${CMAKE_CURRENT_BINARY_DIR}")

set(OPENSSH_WIN32_SRC_DIR "${OPENSSH_SRC_DIR}/contrib/win32")
set(OPENSSH_WIN32_BIN_DIR "${OPENSSH_BIN_DIR}/contrib/win32")

# detect target processor architecture

if(DEFINED CMAKE_OSX_ARCHITECTURES AND CMAKE_OSX_ARCHITECTURES)
    list(LENGTH CMAKE_OSX_ARCHITECTURES CMAKE_OSX_ARCHITECTURES_COUNT)
    if(NOT (CMAKE_OSX_ARCHITECTURES_COUNT EQUAL 1))
        message(WARNING "CMAKE_OSX_ARCHITECTURES can only be used to specify one architecture at a time")
    endif()
    list(GET CMAKE_OSX_ARCHITECTURES 0 CMAKE_SYSTEM_PROCESSOR)
endif()

if(DEFINED CMAKE_GENERATOR_PLATFORM AND CMAKE_GENERATOR_PLATFORM)
    set(CMAKE_SYSTEM_PROCESSOR "${CMAKE_GENERATOR_PLATFORM}" CACHE STRING "" FORCE)
endif()

message(STATUS "CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")

if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm64|ARM64")
    set(TARGET_ARCH "arm64")
    set(FIDO2_ARCH "ARM64")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm|^ARM")
    set(TARGET_ARCH "arm")
    set(FIDO2_ARCH "ARM")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|x64|amd64|AMD64")
    set(TARGET_ARCH "x64")
    set(FIDO2_ARCH "Win64")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "x86|Win32|i386|i486|i586|i686")
    set(TARGET_ARCH "x86")
    set(FIDO2_ARCH "Win32")
else()
    message(FATAL_ERROR "Could not detect processor architecture for \"${CMAKE_SYSTEM_PROCESSOR}\"")
endif()

# compiler-specific flags

if(WIN32)
    set(C_FLAGS "")
    set(C_FLAGS "${C_FLAGS} -DWIN32")
    set(C_FLAGS "${C_FLAGS} -D_LIB")
    set(C_FLAGS "${C_FLAGS} -D_WIN32_WINNT=0x601")
    set(C_FLAGS "${C_FLAGS} -D_CRT_SECURE_NO_WARNINGS")
    set(C_FLAGS "${C_FLAGS} -D_CRT_NONSTDC_NO_WARNINGS")
    set(C_FLAGS "${C_FLAGS} -D_WINSOCK_DEPRECATED_NO_WARNINGS")
    #set(C_FLAGS "${C_FLAGS} -D_UNICODE") # requires patching to enable properly
    set(CMAKE_C_FLAGS "${C_FLAGS} ${CMAKE_C_FLAGS}")

    # required for all executables except ssh-shellhost
    set(C_FLAGS_ENTRY_POINT "/ENTRY:wmainCRTStartup")
else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_XOPEN_SOURCE=600 -D_BSD_SOURCE -D_DEFAULT_SOURCE")

    if(APPLE)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_DARWIN_C_SOURCE")
    endif()

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Werror-implicit-function-declaration")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-incompatible-pointer-types")

    check_c_compiler_flag(-Wno-format-truncation CFLAG_W_NO_FORMAT_TRUNCATION)
    if(CFLAG_W_NO_FORMAT_TRUNCATION)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format-truncation")
    endif()
endif()

# detect prebuilt dependencies

set(USE_OPENSSL TRUE)

if(WIN32)
    set(PREBUILT_DEPS_AVAILABLE ON)
else()
    set(PREBUILT_DEPS_AVAILABLE OFF)
endif()

option(USE_PREBUILT_DEPS "Use prebuilt Microsoft dependencies" ${PREBUILT_DEPS_AVAILABLE})

set(OPENSSH_DEPS_DIR "${OPENSSH_BIN_DIR}/contrib/win32/openssh")

if(USE_PREBUILT_DEPS)
    message(STATUS "Using prebuilt Microsoft dependencies")

    set(ZLIB_VERSION "1.2.11")
    set(ZLIB_ZIP_URL "https://github.com/PowerShell/zlib/releases/download/V${ZLIB_VERSION}/ZLib.zip")

    file(DOWNLOAD "${ZLIB_ZIP_URL}" "${OPENSSH_DEPS_DIR}/ZLib.zip"
        EXPECTED_HASH "SHA256=67AC5CCDBFDC64C275B7AD3D70FCF702F5F8086B10F2C7A91D8F71709F8A7339")
    file(ARCHIVE_EXTRACT INPUT "${OPENSSH_DEPS_DIR}/ZLib.zip" DESTINATION "${OPENSSH_DEPS_DIR}")

    set(LIBRESSL_VERSION "3.4.3.0")
    set(LIBRESSL_ZIP_URL "https://github.com/PowerShell/LibreSSL/releases/download/V${LIBRESSL_VERSION}/LibreSSL.zip")

    file(DOWNLOAD "${LIBRESSL_ZIP_URL}" "${OPENSSH_DEPS_DIR}/LibreSSL.zip"
        EXPECTED_HASH "SHA256=6CA33ACF85377B8DFC67810EF295CB8F28879135E0752834986BC8CE597710B2")
    file(ARCHIVE_EXTRACT INPUT "${OPENSSH_DEPS_DIR}/LibreSSL.zip" DESTINATION "${OPENSSH_DEPS_DIR}")

    set(FIDO2_VERSION "1.10.0")
    set(FIDO2_ZIP_URL "https://github.com/PowerShell/libfido2/releases/download/${FIDO2_VERSION}/libfido2.zip")

    file(DOWNLOAD "${FIDO2_ZIP_URL}" "${OPENSSH_DEPS_DIR}/libfido2.zip"
        EXPECTED_HASH "SHA256=4060F42974B166C66166FBA30D660D518E16A76EB105EEA912C9E2851FE375A4")
    file(ARCHIVE_EXTRACT INPUT "${OPENSSH_DEPS_DIR}/libfido2.zip" DESTINATION "${OPENSSH_DEPS_DIR}/libfido2")
endif()

# detect zlib

find_path(ZLIB_INCLUDE_DIR
    NAMES "zlib.h"
    HINTS ${ZLIB_INCLUDE_DIR}
        "${OPENSSH_DEPS_DIR}/ZLib/sdk")

find_library(ZLIB_LIBRARY
    NAMES zlib z
    HINTS ${ZLIB_LIBRARY_DIR}
        "${OPENSSH_DEPS_DIR}/ZLib/bin/${TARGET_ARCH}")

get_filename_component(ZLIB_LIBRARY_DIR ${ZLIB_LIBRARY} DIRECTORY CACHE)

add_library(zlib STATIC IMPORTED GLOBAL)
set_target_properties(zlib PROPERTIES INTERFACE_INCLUDE_DIRECTORIES "${ZLIB_INCLUDE_DIR}")
set_target_properties(zlib PROPERTIES IMPORTED_LOCATION "${ZLIB_LIBRARY}")

# detect libressl

find_path(LIBRESSL_INCLUDE_DIR
    NAMES "openssl/opensslv.h"
    HINTS ${LIBRESSL_INCLUDE_DIR}
        "${OPENSSH_DEPS_DIR}/LibreSSL/sdk/include")

find_library(LIBRESSL_LIBRARY
    NAMES libcrypto crypto
    HINTS ${LIBRESSL_LIBRARY_DIR}
        "${OPENSSH_DEPS_DIR}/LibreSSL/bin/desktop/${TARGET_ARCH}")

get_filename_component(LIBRESSL_LIBRARY_DIR ${LIBRESSL_LIBRARY} DIRECTORY CACHE)

add_library(libcrypto STATIC IMPORTED GLOBAL)
set_target_properties(libcrypto PROPERTIES INTERFACE_INCLUDE_DIRECTORIES "${LIBRESSL_INCLUDE_DIR}")
set_target_properties(libcrypto PROPERTIES IMPORTED_LOCATION "${LIBRESSL_LIBRARY}")

# detect libfido2

find_path(FIDO2_INCLUDE_DIR
    NAMES "include/fido.h"
    HINTS ${FIDO2_INCLUDE_DIR}
        "${OPENSSH_DEPS_DIR}/libfido2")

find_library(FIDO2_LIBRARY
    NAMES libfido2 fido2
    HINTS ${FIDO2_LIBRARY_DIR}
        "${OPENSSH_DEPS_DIR}/libfido2/${FIDO2_ARCH}/Release/static")

get_filename_component(FIDO2_LIBRARY_DIR ${FIDO2_LIBRARY} DIRECTORY CACHE)

add_library(libfido2 STATIC IMPORTED GLOBAL)
set_target_properties(libfido2 PROPERTIES INTERFACE_INCLUDE_DIRECTORIES "${LIBFIDO2_INCLUDE_DIR}")
set_target_properties(libfido2 PROPERTIES IMPORTED_LOCATION "${FIDO2_LIBRARY}")

if(EXISTS "${LIBRESSL_LIBRARY_DIR}/libcrypto.dll")
    set(LIBRESSL_LIBCRYPTO_DLL "${LIBRESSL_LIBRARY_DIR}/libcrypto.dll")
endif()

# detect libcbor

find_path(CBOR_INCLUDE_DIR
    NAMES "include/cbor.h"
    HINTS ${CBOR_INCLUDE_DIR}
        "${OPENSSH_DEPS_DIR}/libfido2")

find_library(CBOR_LIBRARY
    NAMES "cbor.lib"
    HINTS ${CBOR_LIBRARY_DIR}
        "${OPENSSH_DEPS_DIR}/libfido2/${FIDO2_ARCH}/Release/static")

get_filename_component(CBOR_LIBRARY_DIR ${CBOR_LIBRARY} DIRECTORY CACHE)

add_library(libcbor STATIC IMPORTED GLOBAL)
set_target_properties(libcbor PROPERTIES IMPORTED_LOCATION "${FIDO2_LIBRARY}")

include_directories("${ZLIB_INCLUDE_DIR}")
include_directories("${LIBRESSL_INCLUDE_DIR}")
include_directories("${FIDO2_INCLUDE_DIR}")

set(OPENSSH_DEPENDENCIES zlib libcrypto libfido2 libcbor)

# system libraries

if(WIN32)
    set(OPENSSH_SYSTEM_LIBRARIES
        bcrypt.lib
        Userenv.lib
        Crypt32.lib
        Ws2_32.lib
        Secur32.lib
        Shlwapi.lib
        kernel32.lib
        user32.lib
        gdi32.lib
        winspool.lib
        comdlg32.lib
        advapi32.lib
        shell32.lib
        ole32.lib
        oleaut32.lib
        uuid.lib
        odbc32.lib
        odbccp32.lib
        Netapi32.lib
        Rpcrt4.lib
        ntdll.lib)
else()
    set(OPENSSH_SYSTEM_LIBRARIES
        resolv util crypt)
endif()

# config

if(WIN32)
    file(MAKE_DIRECTORY "${OPENSSH_WIN32_BIN_DIR}/openssh")
    file(MAKE_DIRECTORY "${OPENSSH_WIN32_BIN_DIR}/win32compat/inc")

    execute_process(COMMAND "powershell.exe"
        "-Executionpolicy" "Bypass"
        "-File" "${OPENSSH_WIN32_SRC_DIR}/openssh/config.ps1"
        "-Config_h_vs" "${OPENSSH_WIN32_SRC_DIR}/openssh/config.h.vs"
        "-Config_h" "${OPENSSH_SRC_DIR}/config.h"
        "-VCIncludePath" "${VC_INCLUDE_PATH}"
        "-OutCRTHeader" "${OPENSSH_WIN32_SRC_DIR}/win32compat/inc/crtheaders.h"
        WORKING_DIRECTORY "${OPENSSH_WIN32_SRC_DIR}/openssh")
else()
    set(BROKEN_CLOSEFROM 1)
    set(BROKEN_STRNVIS 1)
    set(DISABLE_WTMPX 1)
    set(ENABLE_PKCS11 "/**/")
    set(ENABLE_SK "/**/")
    set(GETPGRP_VOID 1)
    set(GLOB_HAS_ALTDIRFUNC 1)
    set(HAS_SHADOW_EXPIRE 1)
    set(HAVE_ADDR_IN_UTMP 1)
    set(HAVE_ADDR_IN_UTMPX 1)
    set(HAVE_ADDR_V6_IN_UTMP 1)
    set(HAVE_ADDR_V6_IN_UTMPX 1)
    set(HAVE_ASPRINTF 1)
    set(HAVE_ATTRIBUTE__NONNULL__ 1)
    set(HAVE_BASENAME 1)
    set(HAVE_BCOPY 1)
    set(HAVE_BN_IS_PRIME_EX 1)
    set(HAVE_BZERO 1)
    set(HAVE_CALLOC 1)
    set(HAVE_CLOCK 1)
    set(HAVE_CLOCK_GETTIME 1)
    set(HAVE_CLOCK_T 1)
    set(HAVE_CONST_GAI_STRERROR_PROTO 1)
    set(HAVE_CONTROL_IN_MSGHDR 1)
    set(HAVE_CRYPT 1)
    set(HAVE_DAEMON 1)
    set(HAVE_DECL_AI_NUMERICSERV 1)
    set(HAVE_DECL_BZERO 1)
    set(HAVE_DECL_GETPEEREID 0)
    set(HAVE_DECL_GLOB_NOMATCH 1)
    set(HAVE_DECL_HOWMANY 1)
    set(HAVE_DECL_H_ERRNO 1)
    set(HAVE_DECL_MAXSYMLINKS 1)
    set(HAVE_DECL_MEMMEM 0)
    set(HAVE_DECL_NFDBITS 1)
    set(HAVE_DECL_OFFSETOF 1)
    set(HAVE_DECL_O_NONBLOCK 1)
    set(HAVE_DECL_READV 1)
    set(HAVE_DECL_SHUT_RD 1)
    set(HAVE_DECL_UINT32_MAX 1)
    set(HAVE_DECL_WRITEV 1)
    set(HAVE_DECL__GETLONG 0)
    set(HAVE_DECL__GETSHORT 0)
    set(HAVE_DES_CRYPT 1)
    set(HAVE_DH_GET0_KEY 1)
    set(HAVE_DH_GET0_PQG 1)
    set(HAVE_DH_SET0_KEY 1)
    set(HAVE_DH_SET0_PQG 1)
    set(HAVE_DH_SET_LENGTH 1)
    set(HAVE_DIRFD 1)
    set(HAVE_DIRNAME 1)
    set(HAVE_DLOPEN 1)
    set(HAVE_DSA_GENERATE_PARAMETERS_EX 1)
    set(HAVE_DSA_GET0_KEY 1)
    set(HAVE_DSA_GET0_PQG 1)
    set(HAVE_DSA_SET0_KEY 1)
    set(HAVE_DSA_SET0_PQG 1)
    set(HAVE_DSA_SIG_GET0 1)
    set(HAVE_DSA_SIG_SET0 1)
    set(HAVE_ECDSA_SIG_GET0 1)
    set(HAVE_ECDSA_SIG_SET0 1)
    set(HAVE_EC_KEY_METHOD_NEW 1)
    set(HAVE_ENDGRENT 1)
    set(HAVE_ENDUTENT 1)
    set(HAVE_ENDUTXENT 1)
    set(HAVE_ERR 1)
    set(HAVE_ERRX 1)
    set(HAVE_EVP_CHACHA20 1)
    set(HAVE_EVP_CIPHER_CTX_CTRL 1)
    set(HAVE_EVP_CIPHER_CTX_IV 1)
    set(HAVE_EVP_CIPHER_CTX_IV_NOCONST 1)
    set(HAVE_EVP_DIGESTFINAL_EX 1)
    set(HAVE_EVP_DIGESTINIT_EX 1)
    set(HAVE_EVP_MD_CTX_COPY_EX 1)
    set(HAVE_EVP_MD_CTX_FREE 1)
    set(HAVE_EVP_MD_CTX_NEW 1)
    set(HAVE_EVP_PKEY_GET0_RSA 1)
    set(HAVE_EVP_SHA256 1)
    set(HAVE_EVP_SHA384 1)
    set(HAVE_EVP_SHA512 1)
    set(HAVE_EXIT_IN_UTMP 1)
    set(HAVE_EXPLICIT_BZERO 1)
    set(HAVE_FCHMOD 1)
    set(HAVE_FCHMODAT 1)
    set(HAVE_FCHOWN 1)
    set(HAVE_FCHOWNAT 1)
    set(HAVE_FD_MASK 1)
    set(HAVE_FLOCK 1)
    set(HAVE_FNMATCH 0)
    set(HAVE_FNMATCH_H 0)
    set(HAVE_FREEADDRINFO 1)
    set(HAVE_FSBLKCNT_T 1)
    set(HAVE_FSFILCNT_T 1)
    set(HAVE_FSTATFS 1)
    set(HAVE_FSTATVFS 1)
    set(HAVE_FUTIMES 1)
    set(HAVE_GAI_STRERROR 1)
    set(HAVE_GETADDRINFO 1)
    set(HAVE_GETCWD 1)
    set(HAVE_GETGROUPLIST 1)
    set(HAVE_GETLINE 1)
    set(HAVE_GETNAMEINFO 1)
    set(HAVE_GETOPT 1)
    set(HAVE_GETPAGESIZE 1)
    set(HAVE_GETPGID 1)
    set(HAVE_GETPGRP 1)
    set(HAVE_GETRANDOM 1)
    set(HAVE_GETRLIMIT 1)
    set(HAVE_GETSID 1)
    set(HAVE_GETTIMEOFDAY 1)
    set(HAVE_GETTTYENT 1)
    set(HAVE_GETUTENT 1)
    set(HAVE_GETUTID 1)
    set(HAVE_GETUTLINE 1)
    set(HAVE_GETUTXENT 1)
    set(HAVE_GETUTXID 1)
    set(HAVE_GETUTXLINE 1)
    set(HAVE_GLOB 1)
    set(HAVE_HEADER_AD 1)
    set(HAVE_HOST_IN_UTMP 1)
    set(HAVE_HOST_IN_UTMPX 1)
    set(HAVE_ID_IN_UTMP 1)
    set(HAVE_ID_IN_UTMPX 1)
    set(HAVE_INET_ATON 1)
    set(HAVE_INET_NTOA 1)
    set(HAVE_INET_NTOP 1)
    set(HAVE_INNETGR 1)
    set(HAVE_INT64_T 1)
    set(HAVE_INTMAX_T 1)
    set(HAVE_INTXX_T 1)
    set(HAVE_IN_ADDR_T 1)
    set(HAVE_IN_PORT_T 1)
    set(HAVE_ISBLANK 1)
    set(HAVE_LIBZ 1)
    set(HAVE_LLABS 1)
    set(HAVE_LOCALTIME_R 1)
    set(HAVE_LOGIN 1)
    set(HAVE_LOGOUT 1)
    set(HAVE_LOGWTMP 1)
    set(HAVE_LONG_DOUBLE 1)
    set(HAVE_LONG_LONG 1)
    set(HAVE_MALLOC 1)
    set(HAVE_MBLEN 1)
    set(HAVE_MBTOWC 1)
    set(HAVE_MEMMEM 1)
    set(HAVE_MEMMOVE 1)
    set(HAVE_MKDTEMP 1)
    set(HAVE_MODE_T 1)
    set(HAVE_NANOSLEEP 1)
    set(HAVE_NFDS_T 1)
    set(HAVE_NL_LANGINFO 1)
    set(HAVE_OPENPTY 1)
    set(HAVE_OPENSSL_ADD_ALL_ALGORITHMS 1)
    set(HAVE_OPENSSL_INIT_CRYPTO 1)
    set(HAVE_OPENSSL_VERSION 1)
    set(HAVE_OPENSSL_VERSION_NUM 1)
    set(HAVE_PID_IN_UTMP 1)
    set(HAVE_PID_T 1)
    set(HAVE_POLL 1)
    set(HAVE_PPOLL 1)
    set(HAVE_PRCTL 1)
    set(HAVE_PROC_PID 1)
    set(HAVE_PSELECT 1)
    set(HAVE_PUTUTLINE 1)
    set(HAVE_PUTUTXLINE 1)
    set(HAVE_RAISE 1)
    set(HAVE_REALLOC 1)
    set(HAVE_REALLOCARRAY 1)
    set(HAVE_REALPATH 1)
    set(HAVE_RECVMSG 1)
    set(HAVE_RLIMIT_NPROC "/**/")
    set(HAVE_RRESVPORT_AF 1)
    set(HAVE_RSA_GENERATE_KEY_EX 1)
    set(HAVE_RSA_GET0_CRT_PARAMS 1)
    set(HAVE_RSA_GET0_FACTORS 1)
    set(HAVE_RSA_GET0_KEY 1)
    set(HAVE_RSA_GET_DEFAULT_METHOD 1)
    set(HAVE_RSA_METH_DUP 1)
    set(HAVE_RSA_METH_FREE 1)
    set(HAVE_RSA_METH_GET_FINISH 1)
    set(HAVE_RSA_METH_SET1_NAME 1)
    set(HAVE_RSA_METH_SET_FINISH 1)
    set(HAVE_RSA_METH_SET_PRIV_DEC 1)
    set(HAVE_RSA_METH_SET_PRIV_ENC 1)
    set(HAVE_RSA_SET0_CRT_PARAMS 1)
    set(HAVE_RSA_SET0_FACTORS 1)
    set(HAVE_RSA_SET0_KEY 1)
    set(HAVE_SA_FAMILY_T 1)
    set(HAVE_SENDMSG 1)
    set(HAVE_SETEGID 1)
    set(HAVE_SETENV 1)
    set(HAVE_SETEUID 1)
    set(HAVE_SETGROUPS 1)
    set(HAVE_SETLINEBUF 1)
    set(HAVE_SETREGID 1)
    set(HAVE_SETRESGID 1)
    set(HAVE_SETRESUID 1)
    set(HAVE_SETREUID 1)
    set(HAVE_SETRLIMIT 1)
    set(HAVE_SETSID 1)
    set(HAVE_SETUTENT 1)
    set(HAVE_SETUTXENT 1)
    set(HAVE_SETVBUF 1)
    set(HAVE_SIGACTION 1)
    set(HAVE_SIG_ATOMIC_T 1)
    set(HAVE_SIZE_T 1)
    set(HAVE_SNPRINTF 1)
    set(HAVE_SOCKETPAIR 1)
    set(HAVE_SO_PEERCRED 1)
    set(HAVE_SSIZE_T 1)
    set(HAVE_SS_FAMILY_IN_SS 1)
    set(HAVE_STATFS 1)
    set(HAVE_STATVFS 1)
    set(HAVE_STRCASESTR 1)
    set(HAVE_STRDUP 1)
    set(HAVE_STRERROR 1)
    set(HAVE_STRFTIME 1)
    set(HAVE_STRNDUP 1)
    set(HAVE_STRNLEN 1)
    set(HAVE_STRPTIME 1)
    set(HAVE_STRSEP 1)
    set(HAVE_STRSIGNAL 1)
    set(HAVE_STRTOLL 1)
    set(HAVE_STRTOUL 1)
    set(HAVE_STRTOULL 1)
    set(HAVE_STRUCT_ADDRINFO 1)
    set(HAVE_STRUCT_IN6_ADDR 1)
    set(HAVE_STRUCT_PASSWD_PW_GECOS 1)
    set(HAVE_STRUCT_POLLFD_FD 1)
    set(HAVE_STRUCT_SOCKADDR_IN6 1)
    set(HAVE_STRUCT_SOCKADDR_IN6_SIN6_SCOPE_ID 1)
    set(HAVE_STRUCT_SOCKADDR_STORAGE 1)
    set(HAVE_STRUCT_STATFS_F_FILES 1)
    set(HAVE_STRUCT_STATFS_F_FLAGS 1)
    set(HAVE_STRUCT_STAT_ST_BLKSIZE 1)
    set(HAVE_STRUCT_STAT_ST_MTIM 1)
    set(HAVE_STRUCT_STAT_ST_MTIME 1)
    set(HAVE_STRUCT_TIMESPEC 1)
    set(HAVE_STRUCT_TIMEVAL 1)
    set(HAVE_SYSCONF 1)
    set(HAVE_SYS_ERRLIST 1)
    set(HAVE_SYS_NERR 1)
    set(HAVE_TCGETPGRP 1)
    set(HAVE_TCSENDBREAK 1)
    set(HAVE_TIME 1)
    set(HAVE_TRUNCATE 1)
    set(HAVE_TV_IN_UTMP 1)
    set(HAVE_TV_IN_UTMPX 1)
    set(HAVE_TYPE_IN_UTMP 1)
    set(HAVE_TYPE_IN_UTMPX 1)
    set(HAVE_UINTMAX_T 1)
    set(HAVE_UINTXX_T 1)
    set(HAVE_UNSETENV 1)
    set(HAVE_UNSIGNED_LONG_LONG 1)
    set(HAVE_UPDWTMP 1)
    set(HAVE_UPDWTMPX 1)
    set(HAVE_USLEEP 1)
    set(HAVE_UTIMENSAT 1)
    set(HAVE_UTIMES 1)
    set(HAVE_UTMPNAME 1)
    set(HAVE_UTMPXNAME 1)
    set(HAVE_VASPRINTF 1)
    set(HAVE_VA_COPY 1)
    set(HAVE_VSNPRINTF 1)
    set(HAVE_WAITPID 1)
    set(HAVE_WARN 1)
    set(HAVE_WCWIDTH 1)
    set(HAVE__GETLONG 1)
    set(HAVE__GETSHORT 1)
    set(HAVE__RES_EXTERN 1)
    set(HAVE___FUNCTION__ 1)
    set(HAVE___PROGNAME 1)
    set(HAVE___VA_COPY 1)
    set(HAVE___func__ 1)
    set(IPV4_IN_IPV6 1)
    set(LINK_OPNOTSUPP_ERRNO "EPERM")
    set(LINUX_OOM_ADJUST 1)
    set(LOCKED_PASSWD_PREFIX "!")
    set(OPENSSL_HAS_ECC 1)
    set(OPENSSL_HAS_NISTP256 1)
    set(OPENSSL_HAS_NISTP384 1)
    set(OPENSSL_HAS_NISTP521 1)
    set(OPENSSL_HAVE_EVPCTR 1)
    set(OPENSSL_HAVE_EVPGCM 1)
    set(OPENSSL_PRNG_ONLY 1)
    set(PACKAGE_BUGREPORT "openssh-unix-dev@mindrot.org")
    set(PACKAGE_NAME "OpenSSH")
    set(PACKAGE_STRING "OpenSSH Portable")
    set(PACKAGE_TARNAME "openssh")
    set(PACKAGE_URL "")
    set(PACKAGE_VERSION "Portable")
    set(PAM_TTY_KLUDGE 1)
    set(SANDBOX_SECCOMP_FILTER 1)
    set(SECCOMP_AUDIT_ARCH "AUDIT_ARCH_X86_64")
    set(SIZEOF_INT 4)
    set(SIZEOF_LONG_INT 8)
    set(SIZEOF_LONG_LONG_INT 8)
    set(SIZEOF_SHORT_INT 2)
    set(SIZEOF_TIME_T 8)
    set(SNPRINTF_CONST "const")
    set(SPT_TYPE "SPT_REUSEARGV")
    set(SSH_PRIVSEP_USER "sshd")
    set(SSH_TUN_COMPAT_AF 1)
    set(SSH_TUN_LINUX 1)
    set(SSH_TUN_PREPEND_AF 1)
    set(STDC_HEADERS 1)
    set(SYS_RDOMAIN_LINUX 1)
    set(USER_PATH "/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin")
    set(USE_BTMP 1)
    set(VARIABLE_DECLARATION_AFTER_CODE 1)
    set(VARIABLE_LENGTH_ARRAYS 1)
    set(WITH_OPENSSL 1)
    set(WITH_ZLIB 1)
    set(XAUTH_PATH "/usr/bin/xauth")
    set(_PATH_BTMP "/var/log/btmp")
    set(_PATH_PASSWD_PROG "/usr/bin/passwd")
    set(_PATH_SSH_PIDDIR "/var/run")

    check_include_files("stddef.h" HAVE_STDDEF_H)
    check_include_files("stdint.h" HAVE_STDINT_H)
    check_include_files("stdlib.h" HAVE_STDLIB_H)
    check_include_files("unistd.h" HAVE_UNISTD_H)
    check_include_files("strings.h" HAVE_STRINGS_H)
    check_include_files("string.h" HAVE_STRING_H)
    check_include_files("wchar.h" HAVE_WCHAR_H)
    check_include_files("time.h" HAVE_TIME_H)
    check_include_files("utime.h" HAVE_UTIME_H)

    check_include_files("elf.h" HAVE_ELF_H)
    check_include_files("err.h" HAVE_ERR_H)
    check_include_files("endian.h" HAVE_ENDIAN_H)
    check_include_files("dirent.h" HAVE_DIRENT_H)
    check_include_files("lastlog.h" HAVE_LASTLOG_H)
    check_include_files("features.h" HAVE_FEATURES_H)
    check_include_files("fcntl.h" HAVE_FCNTL_H)
    check_include_files("getopt.h" HAVE_GETOPT_H)
    check_include_files("glob.h" HAVE_GLOB_H)
    check_include_files("ifaddrs.h" HAVE_IFADDRS_H)
    check_include_files("inttypes.h" HAVE_INTTYPES_H)
    check_include_files("langinfo.h" HAVE_LANGINFO_H)
    check_include_files("limits.h" HAVE_LIMITS_H)
    check_include_files("locale.h" HAVE_LOCALE_H)
    check_include_files("memory.h" HAVE_MEMORY_H)
    check_include_files("netdb.h" HAVE_NETDB_H)
    check_include_files("net/route.h" HAVE_NET_ROUTE_H)
    check_include_files("paths.h" HAVE_PATHS_H)
    check_include_files("poll.h" HAVE_POLL_H)
    check_include_files("pty.h" HAVE_PTY_H)
    check_include_files("ttyent.h" HAVE_TTYENT_H)
    check_include_files("utmpx.h" HAVE_UTMPX_H)
    check_include_files("utmp.h" HAVE_UTMP_H)
    check_include_files("rpc/types.h" HAVE_RPC_TYPES_H)
    check_include_files("pam/pam_appl.h" HAVE_PAM_PAM_APPL_H)
    check_include_files("security/pam_appl.h" HAVE_SECURITY_PAM_APPL_H)

    check_include_files("sys/cdefs.h" HAVE_SYS_CDEFS_H)
    check_include_files("sys/dir.h" HAVE_SYS_DIR_H)
    check_include_files("sys/file.h" HAVE_SYS_FILE_H)
    check_include_files("sys/mman.h" HAVE_SYS_MMAN_H)
    check_include_files("sys/mount.h" HAVE_SYS_MOUNT_H)
    check_include_files("sys/param.h" HAVE_SYS_PARAM_H)
    check_include_files("sys/poll.h" HAVE_SYS_POLL_H)
    check_include_files("sys/prctl.h" HAVE_SYS_PRCTL_H)
    check_include_files("sys/ptrace.h" HAVE_SYS_PTRACE_H)
    check_include_files("sys/random.h" HAVE_SYS_RANDOM_H)
    check_include_files("sys/select.h" HAVE_SYS_SELECT_H)
    check_include_files("sys/statvfs.h" HAVE_SYS_STATVFS_H)
    check_include_files("sys/stat.h" HAVE_SYS_STAT_H)
    check_include_files("sys/sysctl.h" HAVE_SYS_SYSCTL_H)
    check_include_files("sys/time.h" HAVE_SYS_TIME_H)
    check_include_files("sys/types.h" HAVE_SYS_TYPES_H)
    check_include_files("sys/un.h" HAVE_SYS_UN_H)
    check_include_files("sys/vfs.h" HAVE_SYS_VFS_H)
    check_include_files("sys/bitypes.h" HAVE_SYS_BITYPES_H)
    check_include_files("sys/sysmacros.h" HAVE_SYS_SYSMACROS_H)

    check_include_files("crypt.h" HAVE_CRYPT_H)
    check_include_files("shadow.h" HAVE_SHADOW_H)
    check_include_files("libgen.h" HAVE_LIBGEN_H)
    check_include_files("libproc.h" HAVE_LIBPROC_H)
    check_include_files("libutil.h" HAVE_LIBUTIL_H)

    set(HAVE_U_CHAR 1)
    set(HAVE_U_INT 1)
    set(HAVE_U_INT64_T 1)
    set(HAVE_U_INTXX_T 1)

    if(UNIX AND NOT APPLE)
        set(HAVE_LINUX_AUDIT_H 1)
        set(HAVE_LINUX_FILTER_H 1)
        set(HAVE_LINUX_IF_TUN_H 1)
        set(HAVE_LINUX_SECCOMP_H 1)
    endif()

    if(APPLE)
        set(HAVE_STRLCPY 1)
        set(HAVE_STRLCAT 1)
        set(HAVE_EXPLICIT_BZERO 0)
    endif()

    if(ANDROID)
        set(DISABLE_SHADOW 1)
        set(HAVE_REALLOCARRAY 0)
        set(HAVE_EXPLICIT_BZERO 1)
        set(HAVE_EVP_CIPHER_CTX_IV 0)
        set(HAVE_EVP_CIPHER_CTX_IV_NOCONST 0)
    endif()

    configure_file("${OPENSSH_SRC_DIR}/cmake-config.h.in"
        "${OPENSSH_SRC_DIR}/config.h")
endif()

# global include directories

set(OPENSSH_GLOBAL_INCLUDE_DIRS
    "${OPENSSH_BIN_DIR}"
    "${OPENSSH_SRC_DIR}"
    "${OPENSSH_SRC_DIR}/include"
    "${OPENSSH_SRC_DIR}/openbsd-compat")

if(WIN32)
    list(APPEND OPENSSH_GLOBAL_INCLUDE_DIRS
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat")
endif()

include_directories(${OPENSSH_GLOBAL_INCLUDE_DIRS})

if(WIN32)
    set(WIN32_COMPAT_INCLUDE_DIRS
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc"
        "${OPENSSH_BIN_DIR}/contrib/win32/win32compat/inc")

    include_directories(BEFORE SYSTEM ${WIN32_COMPAT_INCLUDE_DIRS})
endif()

# posix_compat

if(WIN32)
    set(OPENSSH_POSIX_COMPAT_HEADERS
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/w32fd.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/fcntl.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/signal.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/select.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/socket.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/stat.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/time.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/wait.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/unistd.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/poll.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/statvfs.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/dlfcn.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/syslog.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/signal_internal.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/utf.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/arpa/inet.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/arpa/nameser.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/grp.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/libgen.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/netdb.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/netinet/in.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/netinet/in_systm.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/netinet/ip.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/netinet/tcp.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/resolv.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/ioctl.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/resource.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/uio.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/un.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/termios.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/dirent.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/pwd.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/misc_internal.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/stdio.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/string.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/strings.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/sys/types.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/ctype.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/stdlib.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/debug.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/console.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/tnnet.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/spawn.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/net/if.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/time.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/inc/gssapi.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh/MicrosoftTelemetry.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh/sshTelemetry.h"
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh/sshTelemetryInternal.h")

    set(OPENSSH_POSIX_COMPAT_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/w32fd.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/signal.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/socketio.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/fileio.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/termio.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/misc.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/signal_sigalrm.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/signal_sigchld.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/w32api_proxies.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32_usertoken_utils.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/w32log.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/pwd.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32_dirent.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/no-ops.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/ansiprsr.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/console.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/tncon.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/tnnet.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/utf.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/spawn.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/signal_wait.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32_pty.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/gss-sspi.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh/sshTelemetry.c")

    add_library(posix_compat STATIC
        ${OPENSSH_POSIX_COMPAT_HEADERS}
        ${OPENSSH_POSIX_COMPAT_SOURCES})

    set_target_properties(posix_compat PROPERTIES
        INCLUDE_DIRECTORIES "")
endif()

# openbsd_compat

set(OPENSSH_OPENBSD_COMPAT_HEADERS
    "${OPENSSH_SRC_DIR}/openbsd-compat/base64.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/blf.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-cygwin_util.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-misc.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-nextstep.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-poll.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-setres_id.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-statvfs.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-waitpid.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/chacha_private.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/charclass.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/fake-rfc2553.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/getopt.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/getrrsetbyname.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/glob.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/md5.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/openbsd-compat.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/openssl-compat.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-aix.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-irix.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-linux.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-solaris.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-net.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-uw.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/readpassphrase.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/sha1.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/sha2.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/sys-queue.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/sys-tree.h"
    "${OPENSSH_SRC_DIR}/openbsd-compat/vis.h")

set(OPENSSH_OPENBSD_COMPAT_SOURCES
    "${OPENSSH_SRC_DIR}/openbsd-compat/arc4random.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/base64.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/basename.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bcrypt_pbkdf.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bindresvport.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/blowfish.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-asprintf.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-closefrom.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-cygwin_util.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-getline.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-getpagesize.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-getpeereid.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-misc.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-nextstep.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-openpty.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-poll.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-pselect.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-setres_id.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-snprintf.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-statvfs.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/bsd-waitpid.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/daemon.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/dirname.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/explicit_bzero.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/fake-rfc2553.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/fmt_scaled.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/getcwd.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/getgrouplist.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/getopt_long.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/getrrsetbyname-ldns.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/inet_aton.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/inet_ntoa.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/inet_ntop.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/kludge-fd_set.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/libressl-api-compat.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/md5.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/memmem.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/mktemp.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/openssl-compat.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-irix.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-linux.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-solaris.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-net.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/port-uw.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/readpassphrase.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/reallocarray.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/recallocarray.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/rresvport.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/setenv.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/setproctitle.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/sha1.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/sha2.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strlcat.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strlcpy.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strmode.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strptime.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strsep.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strtoll.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strtonum.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strtoul.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strtoull.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/timingsafe_bcmp.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/vis.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/xcrypt.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/glob.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strcasestr.c"
    "${OPENSSH_SRC_DIR}/openbsd-compat/strndup.c")

if(NOT WIN32)
    list(APPEND OPENSSH_OPENBSD_COMPAT_SOURCES
        "${OPENSSH_SRC_DIR}/openbsd-compat/getrrsetbyname.c"
        "${OPENSSH_SRC_DIR}/openbsd-compat/pwcache.c"
        "${OPENSSH_SRC_DIR}/openbsd-compat/freezero.c")
endif()

add_library(openbsd_compat STATIC
    ${OPENSSH_OPENBSD_COMPAT_HEADERS}
    ${OPENSSH_OPENBSD_COMPAT_SOURCES})

target_include_directories(openbsd_compat PRIVATE ${OPENSSH_GLOBAL_INCLUDE_DIRS})

if(WIN32)
    target_include_directories(openbsd_compat SYSTEM BEFORE PRIVATE ${WIN32_COMPAT_INCLUDE_DIRS})
    set_target_properties(openbsd_compat PROPERTIES LINK_FLAGS "/ignore:4221")
else()
    target_link_libraries(openbsd_compat dl)
endif()

# libssh

set(OPENSSH_LIBSSH_SOURCES
    "${OPENSSH_SRC_DIR}/addr.c"
    "${OPENSSH_SRC_DIR}/addrmatch.c"
    "${OPENSSH_SRC_DIR}/atomicio.c"
    "${OPENSSH_SRC_DIR}/authfd.c"
    "${OPENSSH_SRC_DIR}/authfile.c"
    "${OPENSSH_SRC_DIR}/bitmap.c"
    "${OPENSSH_SRC_DIR}/canohost.c"
    "${OPENSSH_SRC_DIR}/chacha.c"
    "${OPENSSH_SRC_DIR}/channels.c"
    "${OPENSSH_SRC_DIR}/cipher-aes.c"
    "${OPENSSH_SRC_DIR}/cipher-aesctr.c"
    "${OPENSSH_SRC_DIR}/cipher-chachapoly.c"
    "${OPENSSH_SRC_DIR}/cipher-ctr.c"
    "${OPENSSH_SRC_DIR}/cipher.c"
    "${OPENSSH_SRC_DIR}/cleanup.c"
    "${OPENSSH_SRC_DIR}/compat.c")

if(USE_OPENSSL)
    list(APPEND OPENSSH_LIBSSH_SOURCES "${OPENSSH_SRC_DIR}/dh.c")
endif()

if(NOT USE_OPENSSL)
    list(APPEND OPENSSH_LIBSSH_SOURCES "${OPENSSH_SRC_DIR}/digest-libc.c")
endif()

list(APPEND OPENSSH_LIBSSH_SOURCES
    "${OPENSSH_SRC_DIR}/dispatch.c"
    "${OPENSSH_SRC_DIR}/dns.c"
    "${OPENSSH_SRC_DIR}/ed25519.c"
    "${OPENSSH_SRC_DIR}/entropy.c"
    "${OPENSSH_SRC_DIR}/fatal.c"
    "${OPENSSH_SRC_DIR}/fe25519.c"
    "${OPENSSH_SRC_DIR}/ge25519.c"
    "${OPENSSH_SRC_DIR}/gss-genr.c"
    "${OPENSSH_SRC_DIR}/hash.c"
    "${OPENSSH_SRC_DIR}/hmac.c"
    "${OPENSSH_SRC_DIR}/hostfile.c"
    "${OPENSSH_SRC_DIR}/kex.c"
    "${OPENSSH_SRC_DIR}/kexc25519.c")

if(USE_OPENSSL)
    list(APPEND OPENSSH_LIBSSH_SOURCES
        "${OPENSSH_SRC_DIR}/kexdh.c"
        "${OPENSSH_SRC_DIR}/kexecdh.c"
        "${OPENSSH_SRC_DIR}/kexgex.c"
        "${OPENSSH_SRC_DIR}/kexgexc.c")
endif()

list(APPEND OPENSSH_LIBSSH_SOURCES
    "${OPENSSH_SRC_DIR}/kexgen.c"
    "${OPENSSH_SRC_DIR}/kexsntrup761x25519.c"
    "${OPENSSH_SRC_DIR}/krl.c"
    "${OPENSSH_SRC_DIR}/log.c"
    "${OPENSSH_SRC_DIR}/mac.c"
    "${OPENSSH_SRC_DIR}/match.c"
    "${OPENSSH_SRC_DIR}/misc.c"
    "${OPENSSH_SRC_DIR}/moduli.c"
    "${OPENSSH_SRC_DIR}/monitor_fdpass.c"
    "${OPENSSH_SRC_DIR}/msg.c"
    "${OPENSSH_SRC_DIR}/nchan.c"
    "${OPENSSH_SRC_DIR}/packet.c"
    "${OPENSSH_SRC_DIR}/poly1305.c"
    "${OPENSSH_SRC_DIR}/progressmeter.c"
    "${OPENSSH_SRC_DIR}/readpass.c"
    "${OPENSSH_SRC_DIR}/rijndael.c"
    "${OPENSSH_SRC_DIR}/sc25519.c"
    "${OPENSSH_SRC_DIR}/smult_curve25519_ref.c"
    "${OPENSSH_SRC_DIR}/sntrup761.c")

if(USE_OPENSSL)
    list(APPEND OPENSSH_LIBSSH_SOURCES
        "${OPENSSH_SRC_DIR}/ssh-dss.c"
        "${OPENSSH_SRC_DIR}/ssh-ecdsa.c")
endif()

list(APPEND OPENSSH_LIBSSH_SOURCES
    "${OPENSSH_SRC_DIR}/ssh-ed25519.c"
    "${OPENSSH_SRC_DIR}/ssh-pkcs11.c")

if(USE_OPENSSL)
    list(APPEND OPENSSH_LIBSSH_SOURCES
        "${OPENSSH_SRC_DIR}/ssh-rsa.c")
endif()

list(APPEND OPENSSH_LIBSSH_SOURCES
    "${OPENSSH_SRC_DIR}/sshbuf-getput-basic.c"
    "${OPENSSH_SRC_DIR}/sshbuf-getput-crypto.c"
    "${OPENSSH_SRC_DIR}/sshbuf-misc.c"
    "${OPENSSH_SRC_DIR}/sshbuf.c"
    "${OPENSSH_SRC_DIR}/ssherr.c"
    "${OPENSSH_SRC_DIR}/sshkey.c"
    "${OPENSSH_SRC_DIR}/ssh_api.c"
    "${OPENSSH_SRC_DIR}/umac.c"
    "${OPENSSH_SRC_DIR}/verify.c"
    "${OPENSSH_SRC_DIR}/xmalloc.c"
    "${OPENSSH_SRC_DIR}/platform-misc.c"
    "${OPENSSH_SRC_DIR}/platform-pledge.c"
    "${OPENSSH_SRC_DIR}/platform-tracing.c"
    "${OPENSSH_SRC_DIR}/platform.c"
    "${OPENSSH_SRC_DIR}/sandbox-pledge.c")

if(WIN32)
    list(APPEND OPENSSH_LIBSSH_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/ttymodes_windows.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/w32-sshfileperm.c")
else()
    list(APPEND OPENSSH_LIBSSH_SOURCES
        "${OPENSSH_SRC_DIR}/ttymodes.c"
        "${OPENSSH_SRC_DIR}/uidswap.c"
        "${OPENSSH_SRC_DIR}/umac128.c"
        "${OPENSSH_SRC_DIR}/mux.c")
endif()

if(USE_OPENSSL)
    list(APPEND OPENSSH_LIBSSH_SOURCES
        "${OPENSSH_SRC_DIR}/digest-openssl.c")
endif()

list(APPEND OPENSSH_LIBSSH_SOURCES
    "${OPENSSH_SRC_DIR}/kexgexs.c"
    "${OPENSSH_SRC_DIR}/ssh-ecdsa-sk.c"
    "${OPENSSH_SRC_DIR}/ssh-ed25519-sk.c"
    "${OPENSSH_SRC_DIR}/ssh-sk.c"
    "${OPENSSH_SRC_DIR}/sshbuf-io.c"
    "${OPENSSH_SRC_DIR}/cipher-chachapoly-libcrypto.c")

if(WIN32)
    list(APPEND OPENSSH_LIBSSH_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/spawn-ext.c")
endif()

set(OPENSSH_LIBSSH_HEADERS "")

if(EXISTS "${OPENSSH_SRC_DIR}/sshfileperm.h")
    list(APPEND OPENSSH_LIBSSH_HEADERS
        "${OPENSSH_SRC_DIR}/sshfileperm.h")
endif()

add_library(libssh STATIC
    ${OPENSSH_LIBSSH_HEADERS}
    ${OPENSSH_LIBSSH_SOURCES})

if(WIN32)
    set_target_properties(libssh PROPERTIES LINK_FLAGS "/ignore:4221")
endif()

if(WIN32)
    target_link_libraries(libssh posix_compat)
endif()

target_link_libraries(libssh openbsd_compat)

if(WIN32)
    if(DEFINED LIBRESSL_LIBCRYPTO_DLL)
        add_custom_command(TARGET libssh POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy
            "${LIBRESSL_LIBCRYPTO_DLL}" "$<TARGET_FILE_DIR:libssh>/libcrypto.dll")
    endif()
endif()

# common executable definitions

set(OPENSSH_COMMON_LIBRARIES libssh ${OPENSSH_DEPENDENCIES} ${OPENSSH_SYSTEM_LIBRARIES})

if(WIN32)
    set(OPENSSH_TARGETOS_MANIFEST
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh/targetos.manifest")

    set(OPENSSH_VERSION_RC
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh/version.rc")
endif()

# ssh

set(OPENSSH_SSH_SOURCES
    "${OPENSSH_SRC_DIR}/clientloop.c"
    "${OPENSSH_SRC_DIR}/readconf.c"
    "${OPENSSH_SRC_DIR}/ssh.c"
    "${OPENSSH_SRC_DIR}/sshconnect.c"
    "${OPENSSH_SRC_DIR}/sshconnect2.c"
    "${OPENSSH_SRC_DIR}/ssh-sk-client.c")

if(WIN32)
    list(APPEND OPENSSH_SSH_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32_sshtty.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32-utf8.c")
else()
    list(APPEND OPENSSH_SSH_SOURCES
        "${OPENSSH_SRC_DIR}/sshtty.c"
        "${OPENSSH_SRC_DIR}/utf8.c")
endif()

set(OPENSSH_SSH_HEADERS
    "${OPENSSH_SRC_DIR}/atomicio.h"
    "${OPENSSH_SRC_DIR}/audit.h"
    "${OPENSSH_SRC_DIR}/auth-options.h"
    "${OPENSSH_SRC_DIR}/auth-pam.h"
    "${OPENSSH_SRC_DIR}/auth-sia.h"
    "${OPENSSH_SRC_DIR}/auth.h"
    "${OPENSSH_SRC_DIR}/authfd.h"
    "${OPENSSH_SRC_DIR}/authfile.h"
    "${OPENSSH_SRC_DIR}/bitmap.h"
    "${OPENSSH_SRC_DIR}/canohost.h"
    "${OPENSSH_SRC_DIR}/chacha.h"
    "${OPENSSH_SRC_DIR}/channels.h"
    "${OPENSSH_SRC_DIR}/cipher-aesctr.h"
    "${OPENSSH_SRC_DIR}/cipher-chachapoly.h"
    "${OPENSSH_SRC_DIR}/cipher.h"
    "${OPENSSH_SRC_DIR}/clientloop.h"
    "${OPENSSH_SRC_DIR}/compat.h"
    "${OPENSSH_SRC_DIR}/crypto_api.h"
    "${OPENSSH_SRC_DIR}/defines.h"
    "${OPENSSH_SRC_DIR}/dh.h"
    "${OPENSSH_SRC_DIR}/digest.h"
    "${OPENSSH_SRC_DIR}/dispatch.h"
    "${OPENSSH_SRC_DIR}/dns.h"
    "${OPENSSH_SRC_DIR}/entropy.h"
    "${OPENSSH_SRC_DIR}/fe25519.h"
    "${OPENSSH_SRC_DIR}/ge25519.h"
    "${OPENSSH_SRC_DIR}/groupaccess.h"
    "${OPENSSH_SRC_DIR}/hmac.h"
    "${OPENSSH_SRC_DIR}/hostfile.h"
    "${OPENSSH_SRC_DIR}/kex.h"
    "${OPENSSH_SRC_DIR}/krl.h"
    "${OPENSSH_SRC_DIR}/log.h"
    "${OPENSSH_SRC_DIR}/loginrec.h"
    "${OPENSSH_SRC_DIR}/mac.h"
    "${OPENSSH_SRC_DIR}/match.h"
    "${OPENSSH_SRC_DIR}/misc.h"
    "${OPENSSH_SRC_DIR}/monitor.h"
    "${OPENSSH_SRC_DIR}/monitor_fdpass.h"
    "${OPENSSH_SRC_DIR}/monitor_wrap.h"
    "${OPENSSH_SRC_DIR}/msg.h"
    "${OPENSSH_SRC_DIR}/myproposal.h"
    "${OPENSSH_SRC_DIR}/packet.h"
    "${OPENSSH_SRC_DIR}/pathnames.h"
    "${OPENSSH_SRC_DIR}/pkcs11.h"
    "${OPENSSH_SRC_DIR}/platform.h"
    "${OPENSSH_SRC_DIR}/poly1305.h"
    "${OPENSSH_SRC_DIR}/progressmeter.h"
    "${OPENSSH_SRC_DIR}/readconf.h"
    "${OPENSSH_SRC_DIR}/rijndael.h"
    "${OPENSSH_SRC_DIR}/sc25519.h"
    "${OPENSSH_SRC_DIR}/servconf.h"
    "${OPENSSH_SRC_DIR}/serverloop.h"
    "${OPENSSH_SRC_DIR}/session.h"
    "${OPENSSH_SRC_DIR}/sftp-client.h"
    "${OPENSSH_SRC_DIR}/sftp-common.h"
    "${OPENSSH_SRC_DIR}/sftp.h"
    "${OPENSSH_SRC_DIR}/ssh-gss.h"
    "${OPENSSH_SRC_DIR}/ssh-pkcs11.h"
    "${OPENSSH_SRC_DIR}/ssh-sandbox.h"
    "${OPENSSH_SRC_DIR}/ssh.h"
    "${OPENSSH_SRC_DIR}/ssh2.h"
    "${OPENSSH_SRC_DIR}/sshbuf.h"
    "${OPENSSH_SRC_DIR}/sshconnect.h"
    "${OPENSSH_SRC_DIR}/ssherr.h"
    "${OPENSSH_SRC_DIR}/sshkey.h"
    "${OPENSSH_SRC_DIR}/sshlogin.h"
    "${OPENSSH_SRC_DIR}/sshpty.h"
    "${OPENSSH_SRC_DIR}/ssh_api.h"
    "${OPENSSH_SRC_DIR}/ttymodes.h"
    "${OPENSSH_SRC_DIR}/uidswap.h"
    "${OPENSSH_SRC_DIR}/umac.h"
    "${OPENSSH_SRC_DIR}/version.h"
    "${OPENSSH_SRC_DIR}/xmalloc.h")

set(OPENSSH_SSH_RESOURCES
    "${OPENSSH_TARGETOS_MANIFEST}"
    "${OPENSSH_VERSION_RC}")

add_executable(ssh
    ${OPENSSH_SSH_SOURCES}
    ${OPENSSH_SSH_HEADERS}
    ${OPENSSH_SSH_RESOURCES})

if(WIN32)
    set_target_properties(ssh PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT} /ignore:4098 /ignore:4099")
endif()

target_link_libraries(ssh ${OPENSSH_COMMON_LIBRARIES})

# sshd

set(OPENSSH_SSHD_SOURCES
    "${OPENSSH_SRC_DIR}/audit-bsm.c"
    "${OPENSSH_SRC_DIR}/audit-linux.c"
    "${OPENSSH_SRC_DIR}/audit.c"
    "${OPENSSH_SRC_DIR}/auth-bsdauth.c"
    "${OPENSSH_SRC_DIR}/auth-krb5.c"
    "${OPENSSH_SRC_DIR}/auth-options.c"
    "${OPENSSH_SRC_DIR}/auth-pam.c"
    "${OPENSSH_SRC_DIR}/auth-passwd.c"
    "${OPENSSH_SRC_DIR}/auth-rhosts.c"
    "${OPENSSH_SRC_DIR}/auth-shadow.c"
    "${OPENSSH_SRC_DIR}/auth-sia.c"
    "${OPENSSH_SRC_DIR}/auth.c"
    "${OPENSSH_SRC_DIR}/auth2-chall.c"
    "${OPENSSH_SRC_DIR}/auth2-gss.c"
    "${OPENSSH_SRC_DIR}/auth2-hostbased.c"
    "${OPENSSH_SRC_DIR}/auth2-kbdint.c"
    "${OPENSSH_SRC_DIR}/auth2-none.c"
    "${OPENSSH_SRC_DIR}/auth2-passwd.c"
    "${OPENSSH_SRC_DIR}/auth2-pubkey.c"
    "${OPENSSH_SRC_DIR}/auth2.c"
    "${OPENSSH_SRC_DIR}/gss-serv-krb5.c"
    "${OPENSSH_SRC_DIR}/gss-serv.c"
    "${OPENSSH_SRC_DIR}/loginrec.c"
    "${OPENSSH_SRC_DIR}/monitor.c"
    "${OPENSSH_SRC_DIR}/monitor_wrap.c"
    "${OPENSSH_SRC_DIR}/platform.c"
    "${OPENSSH_SRC_DIR}/sandbox-capsicum.c"
    "${OPENSSH_SRC_DIR}/sandbox-darwin.c"
    "${OPENSSH_SRC_DIR}/sandbox-null.c"
    "${OPENSSH_SRC_DIR}/sandbox-rlimit.c"
    "${OPENSSH_SRC_DIR}/sandbox-seccomp-filter.c"
    "${OPENSSH_SRC_DIR}/sandbox-systrace.c"
    "${OPENSSH_SRC_DIR}/servconf.c"
    "${OPENSSH_SRC_DIR}/serverloop.c"
    "${OPENSSH_SRC_DIR}/session.c"
    "${OPENSSH_SRC_DIR}/sftp-common.c"
    "${OPENSSH_SRC_DIR}/sshd.c"
    "${OPENSSH_SRC_DIR}/sshlogin.c"
    "${OPENSSH_SRC_DIR}/srclimit.c"
    "${OPENSSH_SRC_DIR}/ssh-sk-client.c")

if(WIN32)
    list(APPEND OPENSSH_SSHD_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32_groupaccess.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32_sshpty.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_sshd.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/w32-doexec.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32-utf8.c")
else()
    list(APPEND OPENSSH_SSHD_SOURCES
        "${OPENSSH_SRC_DIR}/sftp-realpath.c"
        "${OPENSSH_SRC_DIR}/sftp-server.c"
        "${OPENSSH_SRC_DIR}/groupaccess.c"
        "${OPENSSH_SRC_DIR}/sshpty.c"
        "${OPENSSH_SRC_DIR}/utf8.c")
endif()

set(OPENSSH_SSHD_RESOURCES
    "${OPENSSH_TARGETOS_MANIFEST}"
    "${OPENSSH_VERSION_RC}")

add_executable(sshd
    ${OPENSSH_SSHD_SOURCES}
    ${OPENSSH_SSHD_RESOURCES})

if(WIN32)
    set_target_properties(sshd PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT} /ignore:4099")
endif()

target_link_libraries(sshd ${OPENSSH_COMMON_LIBRARIES})

# scp

set(OPENSSH_SCP_SOURCES
    "${OPENSSH_SRC_DIR}/scp.c"
    "${OPENSSH_SRC_DIR}/sftp-client.c"
    "${OPENSSH_SRC_DIR}/sftp-common.c"
    "${OPENSSH_SRC_DIR}/sftp-glob.c")

if(WIN32)
    list(APPEND OPENSSH_SCP_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32-utf8.c")
else()
    list(APPEND OPENSSH_SCP_SOURCES
        "${OPENSSH_SRC_DIR}/utf8.c")
endif()

set(OPENSSH_SCP_RESOURCES
    "${OPENSSH_TARGETOS_MANIFEST}"
    "${OPENSSH_VERSION_RC}")

add_executable(scp
    ${OPENSSH_SCP_SOURCES}
    ${OPENSSH_SCP_RESOURCES})

if(WIN32)
    set_target_properties(scp PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(scp ${OPENSSH_COMMON_LIBRARIES})

# sftp

set(OPENSSH_SFTP_SOURCES
    "${OPENSSH_SRC_DIR}/progressmeter.c"
    "${OPENSSH_SRC_DIR}/sftp-client.c"
    "${OPENSSH_SRC_DIR}/sftp-common.c"
    "${OPENSSH_SRC_DIR}/sftp-glob.c"
    "${OPENSSH_SRC_DIR}/sftp.c")

if(WIN32)
    list(APPEND OPENSSH_SFTP_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32-utf8.c")
else()
    list(APPEND OPENSSH_SFTP_SOURCES
        "${OPENSSH_SRC_DIR}/utf8.c")
endif()

set(OPENSSH_SFTP_RESOURCES
    "${OPENSSH_TARGETOS_MANIFEST}"
    "${OPENSSH_VERSION_RC}")

add_executable(sftp
    ${OPENSSH_SFTP_SOURCES}
    ${OPENSSH_SFTP_RESOURCES})

if(WIN32)
    set_target_properties(sftp PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(sftp ${OPENSSH_COMMON_LIBRARIES})

# sftp-server

set(OPENSSH_SFTP_SERVER_SOURCES
    "${OPENSSH_SRC_DIR}/sftp-common.c"
    "${OPENSSH_SRC_DIR}/sftp-server-main.c"
    "${OPENSSH_SRC_DIR}/sftp-server.c")

if(WIN32)
    list(APPEND OPENSSH_SFTP_SERVER_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")
else()
    list(APPEND OPENSSH_SFTP_SERVER_SOURCES
        "${OPENSSH_SRC_DIR}/sftp-realpath.c"
        "${OPENSSH_SRC_DIR}/utf8.c")
endif()

set(OPENSSH_SFTP_SERVER_RESOURCES
    "${OPENSSH_TARGETOS_MANIFEST}"
    "${OPENSSH_VERSION_RC}")

add_executable(sftp-server
    ${OPENSSH_SFTP_SERVER_SOURCES}
    ${OPENSSH_SFTP_SERVER_RESOURCES})

if(WIN32)
    set_target_properties(sftp-server PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(sftp-server ${OPENSSH_COMMON_LIBRARIES})

# ssh-add

set(OPENSSH_SSH_ADD_SOURCES
    "${OPENSSH_SRC_DIR}/ssh-add.c"
    "${OPENSSH_SRC_DIR}/ssh-sk-client.c")

if(WIN32)
    list(APPEND OPENSSH_SSH_ADD_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32-utf8.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")
else()
    list(APPEND OPENSSH_SSH_ADD_SOURCES
        "${OPENSSH_SRC_DIR}/utf8.c")
endif()

set(OPENSSH_SSH_ADD_RESOURCES
    "${OPENSSH_TARGETOS_MANIFEST}"
    "${OPENSSH_VERSION_RC}")

add_executable(ssh-add
    ${OPENSSH_SSH_ADD_SOURCES}
    ${OPENSSH_SSH_ADD_RESOURCES})

if(WIN32)
    set_target_properties(ssh-add PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(ssh-add ${OPENSSH_COMMON_LIBRARIES})

# ssh-agent

set(OPENSSH_SSH_AGENT_SOURCES
    "${OPENSSH_SRC_DIR}/ssh-pkcs11-client.c"
    "${OPENSSH_SRC_DIR}/ssh-sk-client.c")

if(WIN32)
    list(APPEND OPENSSH_SSH_AGENT_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/ssh-agent/agent-main.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/ssh-agent/agent.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/ssh-agent/connection.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/ssh-agent/keyagent-request.c")
else()
    list(APPEND OPENSSH_SSH_AGENT_SOURCES
        "${OPENSSH_SRC_DIR}/ssh-agent.c")
endif()

if(WIN32)
    set(OPENSSH_SSH_AGENT_RESOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh/openssh-events.rc"
        "${OPENSSH_SRC_DIR}/contrib/win32/openssh/version.rc")
endif()

add_executable(ssh-agent
    ${OPENSSH_SSH_AGENT_SOURCES}
    ${OPENSSH_SSH_AGENT_RESOURCES})

if(WIN32)
    set_target_properties(ssh-agent PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(ssh-agent ${OPENSSH_COMMON_LIBRARIES})

# ssh-keygen

set(OPENSSH_SSH_KEYGEN_SOURCES
    "${OPENSSH_SRC_DIR}/ssh-keygen.c"
    "${OPENSSH_SRC_DIR}/sshsig.c"
    "${OPENSSH_SRC_DIR}/ssh-sk-client.c")

if(WIN32)
    list(APPEND OPENSSH_SSH_KEYGEN_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32-utf8.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")
else()
    list(APPEND OPENSSH_SSH_KEYGEN_SOURCES
        "${OPENSSH_SRC_DIR}/utf8.c")
endif()

set(OPENSSH_SSH_KEYGEN_RESOURCES
    "${OPENSSH_TARGETOS_MANIFEST}"
    "${OPENSSH_VERSION_RC}")

add_executable(ssh-keygen
    ${OPENSSH_SSH_KEYGEN_SOURCES}
    ${OPENSSH_SSH_KEY_RESOURCES})

if(WIN32)
    set_target_properties(ssh-keygen PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(ssh-keygen ${OPENSSH_COMMON_LIBRARIES})

# ssh-keyscan

set(OPENSSH_SSH_KEYSCAN_SOURCES
    "${OPENSSH_SRC_DIR}/ssh-keyscan.c"
    "${OPENSSH_SRC_DIR}/ssh-sk-client.c")

if(WIN32)
    list(APPEND OPENSSH_SSH_KEYSCAN_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")
endif()

if(WIN32)
    set(OPENSSH_SSH_KEYSCAN_RESOURCES
        "${OPENSSH_TARGETOS_MANIFEST}"
        "${OPENSSH_VERSION_RC}")
endif()

add_executable(ssh-keyscan
    ${OPENSSH_SSH_KEYSCAN_SOURCES}
    ${OPENSSH_SSH_KEYSCAN_RESOURCES})

if(WIN32)
    set_target_properties(ssh-keyscan PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(ssh-keyscan ${OPENSSH_COMMON_LIBRARIES})

# ssh-shellhost

if(WIN32)
    set(OPENSSH_SSH_SHELLHOST_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/utf.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/shell-host.c")

    set(OPENSSH_SSH_SHELLHOST_RESOURCES
        "${OPENSSH_TARGETOS_MANIFEST}"
        "${OPENSSH_VERSION_RC}")

    add_executable(ssh-shellhost
        ${OPENSSH_SSH_SHELLHOST_SOURCES}
        ${OPENSSH_SSH_SHELLHOST_RESOURCES})

    set_target_properties(ssh-shellhost PROPERTIES LINK_FLAGS "${C_FLAGS_FORCE_MULTIPLE}")

    target_link_libraries(ssh-shellhost ${OPENSSH_COMMON_LIBRARIES})
endif()

# ssh-sk-helper

set(OPENSSH_SSH_SK_HELPER_SOURCES
    "${OPENSSH_SRC_DIR}/ssh-sk-helper.c"
    "${OPENSSH_SRC_DIR}/ssh-sk-client.c")

if(WIN32)
    list(APPEND OPENSSH_SSH_SK_HELPER_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")
endif()

if(WIN32)
    set(OPENSSH_SSH_SK_HELPER_RESOURCES
        "${OPENSSH_TARGETOS_MANIFEST}"
        "${OPENSSH_VERSION_RC}")
endif()

add_executable(ssh-sk-helper
    ${OPENSSH_SSH_SK_HELPER_SOURCES}
    ${OPENSSH_SSH_SK_HELPER_RESOURCES})

if(WIN32)
    set_target_properties(ssh-sk-helper PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(ssh-sk-helper ${OPENSSH_COMMON_LIBRARIES})

# ssh-pkcs11-helper

set(OPENSSH_SSH_PKCS11_HELPER_SOURCES
    "${OPENSSH_SRC_DIR}/ssh-pkcs11-helper.c"
    "${OPENSSH_SRC_DIR}/ssh-pkcs11.c"
    "${OPENSSH_SRC_DIR}/ssh-sk-client.c")

if(WIN32)
    list(APPEND OPENSSH_SSH_PKCS11_HELPER_SOURCES
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")
endif()

if(WIN32)
    set(OPENSSH_SSH_PKCS11_HELPER_RESOURCES
        "${OPENSSH_TARGETOS_MANIFEST}"
        "${OPENSSH_VERSION_RC}")
endif()

add_executable(ssh-pkcs11-helper
    ${OPENSSH_SSH_PKCS11_HELPER_SOURCES}
    ${OPENSSH_SSH_PKCS11_HELPER_RESOURCES})

if(WIN32)
    set_target_properties(ssh-pkcs11-helper PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
endif()

target_link_libraries(ssh-pkcs11-helper ${OPENSSH_COMMON_LIBRARIES})

# unit tests

if(WITH_UNIT_TESTS)
    enable_testing()
    set(OPENSSH_TEST_TARGETS "")

    # unit-test-bitmap
    
    set(OPENSSH_UNIT_TEST_BITMAP_SOURCES
        "${OPENSSH_SRC_DIR}/regress/unittests/bitmap/tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/test_helper.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")

    add_executable(unit-test-bitmap ${OPENSSH_UNIT_TEST_BITMAP_SOURCES})
    set_target_properties(unit-test-bitmap PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
    target_link_libraries(unit-test-bitmap ${OPENSSH_COMMON_LIBRARIES})
    list(APPEND OPENSSH_TEST_TARGETS "unit-test-bitmap")

    # unit-test-hostkeys

    set(OPENSSH_UNIT_TEST_HOSTKEYS_SOURCES
        "${OPENSSH_SRC_DIR}/ssh-sk-client.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/hostkeys/tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/hostkeys/test_iterate.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/test_helper.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32-utf8.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")

    add_executable(unit-test-hostkeys ${OPENSSH_UNIT_TEST_HOSTKEYS_SOURCES})
    set_target_properties(unit-test-hostkeys PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
    target_link_libraries(unit-test-hostkeys ${OPENSSH_COMMON_LIBRARIES})
    list(APPEND OPENSSH_TEST_TARGETS "unit-test-hostkeys")

    # unit-test-kex

    set(OPENSSH_UNIT_TEST_KEX_SOURCES
        "${OPENSSH_SRC_DIR}/ssh-sk-client.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/kex/tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/kex/test_kex.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/test_helper.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")

    add_executable(unit-test-kex ${OPENSSH_UNIT_TEST_KEX_SOURCES})
    set_target_properties(unit-test-kex PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
    target_link_libraries(unit-test-kex ${OPENSSH_COMMON_LIBRARIES})
    list(APPEND OPENSSH_TEST_TARGETS "unit-test-kex")

    # unit-test-match

    set(OPENSSH_UNIT_TEST_MATCH_SOURCES
        "${OPENSSH_SRC_DIR}/regress/unittests/match/tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/test_helper.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")

    add_executable(unit-test-match ${OPENSSH_UNIT_TEST_MATCH_SOURCES})
    set_target_properties(unit-test-match PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
    target_link_libraries(unit-test-match ${OPENSSH_COMMON_LIBRARIES})
    list(APPEND OPENSSH_TEST_TARGETS "unit-test-match")

    # unit-test-misc

    set(OPENSSH_UNIT_TEST_MISC_SOURCES
        "${OPENSSH_SRC_DIR}/regress/unittests/misc/tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/misc/test_parse.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/misc/test_expand.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/misc/test_convtime.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/misc/test_argv.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/misc/test_strdelim.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/misc/test_hpdelim.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/test_helper.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")

    add_executable(unit-test-misc ${OPENSSH_UNIT_TEST_MISC_SOURCES})
    set_target_properties(unit-test-misc PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
    target_link_libraries(unit-test-misc ${OPENSSH_COMMON_LIBRARIES})
    list(APPEND OPENSSH_TEST_TARGETS "unit-test-misc")

    # unit-test-sshbuf

    set(OPENSSH_UNIT_TEST_SSHBUF_SOURCES
        "${OPENSSH_SRC_DIR}/regress/unittests/sshbuf/tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshbuf/test_sshbuf.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshbuf/test_sshbuf_fixed.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshbuf/test_sshbuf_fuzz.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshbuf/test_sshbuf_getput_basic.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshbuf/test_sshbuf_getput_crypto.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshbuf/test_sshbuf_getput_fuzz.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshbuf/test_sshbuf_misc.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/fuzz.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/test_helper.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")

    add_executable(unit-test-sshbuf ${OPENSSH_UNIT_TEST_SSHBUF_SOURCES})
    set_target_properties(unit-test-sshbuf PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
    target_link_libraries(unit-test-sshbuf ${OPENSSH_COMMON_LIBRARIES})
    list(APPEND OPENSSH_TEST_TARGETS "unit-test-sshbuf")

    # unit-test-sshkey

    set(OPENSSH_UNIT_TEST_SSHKEY_SOURCES
        "${OPENSSH_SRC_DIR}/ssh-sk-client.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshkey/tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshkey/common.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshkey/test_file.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshkey/test_fuzz.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/sshkey/test_sshkey.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/fuzz.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/test_helper.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/win32-utf8.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")

    add_executable(unit-test-sshkey ${OPENSSH_UNIT_TEST_SSHKEY_SOURCES})
    set_target_properties(unit-test-sshkey PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
    target_link_libraries(unit-test-sshkey ${OPENSSH_COMMON_LIBRARIES})
    list(APPEND OPENSSH_TEST_TARGETS "unit-test-sshkey")

    # unit-test-win32compat

    set(OPENSSH_UNIT_TEST_WIN32COMPAT_SOURCES
        "${OPENSSH_SRC_DIR}/regress/unittests/win32compat/file_tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/win32compat/socket_tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/win32compat/tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/win32compat/dir_tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/win32compat/miscellaneous_tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/win32compat/string_tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/win32compat/signal_tests.c"
        "${OPENSSH_SRC_DIR}/regress/unittests/test_helper/test_helper.c"
        "${OPENSSH_SRC_DIR}/contrib/win32/win32compat/wmain_common.c")

    add_executable(unit-test-win32compat ${OPENSSH_UNIT_TEST_WIN32COMPAT_SOURCES})
    set_target_properties(unit-test-win32compat PROPERTIES LINK_FLAGS "${C_FLAGS_ENTRY_POINT}")
    target_link_libraries(unit-test-win32compat ${OPENSSH_COMMON_LIBRARIES})
    list(APPEND OPENSSH_TEST_TARGETS "unit-test-win32compat")

    # ctest targets

    foreach(OPENSSH_TEST_TARGET ${OPENSSH_TEST_TARGETS})
        add_test(NAME ${OPENSSH_TEST_TARGET} COMMAND ${OPENSSH_TEST_TARGET})
    endforeach()
endif()

# installation

set(OPENSSH_INSTALL_TARGETS
    "ssh"
    "sshd"
    "scp"
    "sftp"
    "sftp-server"
    "ssh-add"
    "ssh-agent"
    "ssh-keygen"
    "ssh-keyscan"
    "ssh-sk-helper"
    "ssh-pkcs11-helper")

if(WIN32)
    list(APPEND OPENSSH_INSTALL_TARGETS "ssh-shellhost")
endif()

install(TARGETS ${OPENSSH_INSTALL_TARGETS}
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin)
