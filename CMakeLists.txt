cmake_minimum_required(VERSION 2.6)

PROJECT(procd C)
INCLUDE(GNUInstallDirs)
ADD_DEFINITIONS(-Os -ggdb -Wall -Werror --std=gnu99 -Wmissing-declarations)

SET(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "")

include_directories(..)
link_directories(../libubox)

SET(LIBS ubox ubus json-c blobmsg_json json_script)

IF(DEBUG)
  ADD_DEFINITIONS(-DDEBUG -g3)
ENDIF()

ADD_CUSTOM_COMMAND(
	OUTPUT syscall-names.h
	COMMAND ./make_syscall_h.sh ${CMAKE_C_COMPILER} > ./syscall-names.h
	DEPENDS ./make_syscall_h.sh
)
ADD_CUSTOM_TARGET(syscall-names-h DEPENDS syscall-names.h)

ADD_CUSTOM_COMMAND(
	OUTPUT capabilities-names.h
	COMMAND ./make_capabilities_h.sh ${CMAKE_C_COMPILER} > ./capabilities-names.h
	DEPENDS ./make_capabilities_h.sh
)
ADD_CUSTOM_TARGET(capabilities-names-h DEPENDS capabilities-names.h)

IF(SECCOMP_SUPPORT)
ADD_LIBRARY(preload-seccomp SHARED jail/preload.c jail/seccomp.c)
TARGET_LINK_LIBRARIES(preload-seccomp dl ubox blobmsg_json)
INSTALL(TARGETS preload-seccomp
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
ADD_DEPENDENCIES(preload-seccomp syscall-names-h)
endif()

IF(JAIL_SUPPORT)
ADD_EXECUTABLE(ujail jail/jail.c jail/elf.c jail/fs.c jail/capabilities.c)
TARGET_LINK_LIBRARIES(ujail ubox blobmsg_json)
INSTALL(TARGETS ujail
	RUNTIME DESTINATION ${CMAKE_INSTALL_SBINDIR}
)
ADD_DEPENDENCIES(ujail capabilities-names-h)
endif()

IF(UTRACE_SUPPORT)
ADD_EXECUTABLE(utrace trace/trace.c)
TARGET_LINK_LIBRARIES(utrace ubox ${json} blobmsg_json)
INSTALL(TARGETS utrace
	RUNTIME DESTINATION ${CMAKE_INSTALL_SBINDIR}
)
ADD_DEPENDENCIES(utrace syscall-names-h)

ADD_LIBRARY(preload-trace SHARED trace/preload.c)
TARGET_LINK_LIBRARIES(preload-trace dl)
INSTALL(TARGETS preload-trace
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
endif()
