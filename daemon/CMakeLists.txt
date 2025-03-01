add_executable(belnet belnet.cpp)
add_executable(belnet-vpn belnet-vpn.cpp)
add_executable(belnet-bootstrap belnet-bootstrap.cpp)
enable_lto(belnet belnet-vpn belnet-bootstrap)

if(TRACY_ROOT)
  target_sources(belnet PRIVATE ${TRACY_ROOT}/TracyClient.cpp)
endif()

set(should_install ON)
set(SETCAP)

if(CMAKE_SYSTEM_NAME MATCHES "Linux")
  option(WITH_SETCAP "use setcap when installing" ON)
  if(WITH_SETCAP)
    find_program(SETCAP NAMES setcap HINTS /sbin /usr/sbin)
    if(SETCAP)
      message(STATUS "Found setcap binary: ${SETCAP}")
    else()
      message(WARNING "cannot find setcap binary you will not be able use the install targets unless you use -DWITH_SETCAP=OFF")
      set(should_install OFF)
    endif()
  endif()
endif()

target_link_libraries(belnet-bootstrap PUBLIC cpr::cpr)
if(NOT WIN32)
  find_package(OpenSSL REQUIRED)
  # because debian sid's curl doesn't link against openssl for some godawful cursed reason
  target_link_libraries(belnet-bootstrap PUBLIC OpenSSL::SSL OpenSSL::Crypto)
endif()

foreach(exe belnet belnet-vpn belnet-bootstrap)
  if(WIN32 AND NOT MSVC_VERSION)
    target_sources(${exe} PRIVATE ../llarp/win32/version.rc)
    target_link_libraries(${exe} PRIVATE -static-libstdc++ -static-libgcc --static -Wl,--pic-executable,-e,mainCRTStartup,--subsystem,console:5.00)
    target_link_libraries(${exe} PRIVATE ws2_32 iphlpapi)
  elseif(CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
    target_link_directories(${exe} PRIVATE /usr/local/lib)
  endif()
  target_link_libraries(${exe} PRIVATE libbelnet)
  if(WITH_JEMALLOC)
    target_link_libraries(${exe} PUBLIC jemalloc)
  endif()
  target_include_directories(${exe} PRIVATE ${CMAKE_SOURCE_DIR})
  target_compile_definitions(${exe} PRIVATE -DVERSIONTAG=${GIT_VERSION_REAL})
  add_log_tag(${exe})
  if(should_install)
    install(TARGETS ${exe} RUNTIME DESTINATION bin COMPONENT belnet)
  endif()
endforeach()

if(SETCAP)
  install(CODE "execute_process(COMMAND ${SETCAP} cap_net_admin,cap_net_bind_service=+eip ${CMAKE_INSTALL_PREFIX}/bin/belnet)")
endif()
