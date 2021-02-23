#[[
   Copyright (c) 2021 Sibi Siddharthan

   Distributed under the MIT license.
   Refer to the LICENSE file at the root directory of the parent project
   for details.
]]

cmake_minimum_required(VERSION 3.12)
project(npth
		VERSION 1.6.0
		LANGUAGES C)

add_compile_definitions(HAVE_CONFIG_H)

set(PACKAGE \"${PROJECT_NAME}\")
set(PACKAGE_BUGREPORT \"https://bugs.gnupg.org\")
set(PACKAGE_NAME \"${PROJECT_NAME}\")
set(PACKAGE_STRING \"${PROJECT_NAME}\ ${PROJECT_VERSION}\")
set(PACKAGE_TARNAME \"${PROJECT_NAME}\")
set(PACKAGE_URL \"\")
set(PACKAGE_VERSION \"${PROJECT_VERSION}\")
set(VERSION \"${PROJECT_VERSION}\")
math(EXPR VERSION_HEX "(${PROJECT_VERSION_MAJOR} << 16) + (${PROJECT_VERSION_MINOR} << 8) + ${PROJECT_VERSION_PATCH}" OUTPUT_FORMAT HEXADECIMAL)

include(UtilityFunctions)

check_c_headers(sys/time.h time.h sys/select.h signal.h)
if(HAVE_SYS_TIME_H)
	set(INSERT_SYS_TIME_H "include\ <sys/time.h>")
endif()
if(HAVE_TIME_H)
	set(INSERT_TIME_H "include\ <time.h>")
endif()
if(HAVE_SYS_SELECT_H)
	set(INSERT_SYS_SELECT_H "include\ <sys/select.h>")
endif()
if(HAVE_SIGNAL_H)
	set(INSERT_SIGNAL_H "include\ <signal.h>")
endif()

# Linux has socklen_t
set(INSERT_SOCKLEN_T "define\ _npth_socklen_t\ socklen_t")

if(UNIX)
	set(CMAKE_REQUIRED_LIBRARIES pthread)
endif()
check_functions(pthread_atfork pthread_getname_np pthread_setname_np pthread_tryjoin_np
pthread_mutex_timedlock pthread_rwlock_rdlock pthread_rwlock_wrlock pthread_rwlock_timedrdlock
pthread_rwlock_timedwrlock pthread_rwlock_tryrdlock pthread_rwlock_trywrlock
pselect gettimeofday)
unset(CMAKE_REQUIRED_LIBRARIES)

if(UNIX)
	set(CMAKE_REQUIRED_EXTRA_INCLUDES "pthread.h")
	check_types(pthread_rwlock_t)
	unset(CMAKE_REQUIRED_EXTRA_INCLUDES)
	if(HAVE_PTHREAD_RWLOCK_T)
		set(INSERT_NO_RWLOCK "undef\ _NPTH_NO_RWLOCK")
	else()
		set(INSERT_NO_RWLOCK "define\ _NPTH_NO_RWLOCK\ 1")
	endif()
	set(INSERT_EXPOSE_RWLOCK_API "defined(__USE_UNIX98)\ ||\ defined(__USE_XOPEN2K)")
endif()

if(WIN32)
	set(HAVE_W32_SYSTEM 1)
	set(HAVE_W64_SYSTEM 1)
endif()

configure_file(npth-config.h.in config.h)

if(WIN32)
	file(COPY w32/npth.h DESTINATION ${CMAKE_BINARY_DIR})
	add_library(npth SHARED w32/npth.c w32/npth.def)
	set_target_properties(npth PROPERTIES PUBLIC_HEADER w32/npth.h)
	target_include_directories(npth PUBLIC ${CMAKE_CURRENT_BINARY_DIR} w32)
	set_dll_output_directory(npth ${CMAKE_BINARY_DIR})
	target_link_libraries(npth ws2_32)
else()
	configure_file(src/npth.h.in ${CMAKE_BINARY_DIR}/npth.h @ONLY)
	add_library(npth SHARED src/npth.c src/npth-sigev.c)
	set_target_properties(npth PROPERTIES PUBLIC_HEADER ${CMAKE_BINARY_DIR}/npth.h)
	target_include_directories(npth PUBLIC ${CMAKE_CURRENT_BINARY_DIR} src)
	target_link_libraries(npth pthread)
endif()
set_target_properties(npth PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
add_library(NPTH::NPTH ALIAS npth)

install(TARGETS npth
	RUNTIME DESTINATION bin
	LIBRARY DESTINATION lib
	ARCHIVE DESTINATION lib
	PUBLIC_HEADER DESTINATION include)

include(CTest)
if(BUILD_TESTING)

set(tests t-mutex t-thread)
if(UNIX)
	list(APPEND tests t-fork)
endif()

foreach(t ${tests})
	add_executable(${t} tests/${t}.c)
	target_link_libraries(${t} npth)
	add_test(NAME ${t} COMMAND ${t})
	if(WIN32)
		set_tests_properties(${t} PROPERTIES ENVIRONMENT "PATH=${CMAKE_BINARY_DIR}\;${NEW_PATH}")
	endif()
endforeach()

endif()
