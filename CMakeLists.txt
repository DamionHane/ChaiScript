cmake_minimum_required(VERSION 3.12)
cmake_policy(SET CMP0054 NEW)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# required since cmake 3.4 at least for libc++
set(CMAKE_ENABLE_EXPORTS ON)

project(chaiscript)

option(MULTITHREAD_SUPPORT_ENABLED "Multithreaded Support Enabled" TRUE)
option(DYNLOAD_ENABLED "Dynamic Loading Support Enabled" TRUE)


option(BUILD_MODULES "Build Extra Modules (stl)" TRUE)
option(BUILD_SAMPLES "Build Samples Folder" FALSE)
option(RUN_FUZZY_TESTS "Run tests generated by AFL" FALSE)
option(USE_STD_MAKE_SHARED "Use std::make_shared instead of chaiscript::make_shared" FALSE)
option(RUN_PERFORMANCE_TESTS "Run Performance Tests" FALSE)

mark_as_advanced(USE_STD_MAKE_SHARED)

if(USE_STD_MAKE_SHARED)
  add_definitions(-DCHAISCRIPT_USE_STD_MAKE_SHARED)
endif()

if(CMAKE_COMPILER_IS_GNUCC)
  option(ENABLE_COVERAGE "Enable Coverage Reporting in GCC" FALSE)

  if(ENABLE_COVERAGE)
    add_definitions(--coverage -O0)
    set(LINKER_FLAGS "${LINKER_FLAGS} --coverage")
  endif()
endif()

if(CMAKE_COMPILER_IS_GNUCC OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  option(ENABLE_THREAD_SANITIZER "Enable thread sanitizer testing in gcc/clang" FALSE)
  if(ENABLE_THREAD_SANITIZER)
    add_definitions(-fsanitize=thread -g)
    set(LINKER_FLAGS "${LINKER_FLAGS} -fsanitize=thread")
  endif()

  option(ENABLE_ADDRESS_SANITIZER "Enable address sanitizer testing in gcc/clang" FALSE)
  if(ENABLE_ADDRESS_SANITIZER)
    add_definitions(-fsanitize=address -g)
    set(LINKER_FLAGS "${LINKER_FLAGS} -fsanitize=address")

    option(BUILD_LIBFUZZ_TESTER "Build libfuzzer tool" FALSE)
  endif()

  option(ENABLE_MEMORY_SANITIZER "Enable memory sanitizer testing in gcc/clang" FALSE)
  if(ENABLE_MEMORY_SANITIZER)
    add_definitions(-fsanitize=memory -fsanitize-memory-track-origins -g)
    set(LINKER_FLAGS "${LINKER_FLAGS} -fsanitize=memory -fsanitize-memory-track-origins ")
  endif()

  option(ENABLE_UNDEFINED_SANITIZER "Enable undefined behavior sanitizer testing in gcc/clang" FALSE)
  if(ENABLE_UNDEFINED_SANITIZER)
    add_definitions(-fsanitize=undefined -g)
    set(LINKER_FLAGS "${LINKER_FLAGS} -fsanitize=undefined")
  endif()

  option(ENABLE_LTO "Enable Link Time Optimization" FALSE)
  if(ENABLE_LTO)
    check_ipo_supported()
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
  endif()

  option(GPROF_OUTPUT "Generate profile data" FALSE)
  if(GPROF_OUTPUT)
    add_definitions(-pg)
    set(LINKER_FLAGS "${LINKER_FLAGS} -pg")
  endif()


  option(PROFILE_GENERATE "Generate profile data" FALSE)
  if(PROFILE_GENERATE)
    add_definitions(-fprofile-generate)
    set(LINKER_FLAGS "${LINKER_FLAGS} -fprofile-generate")
  endif()

  option(PROFILE_USE "Use profile data" FALSE)
  if(PROFILE_USE)
    add_definitions(-fprofile-use)
    set(LINKER_FLAGS "${LINKER_FLAGS} -fprofile-use")
  endif()


endif()

list(APPEND CPACK_SOURCE_IGNORE_FILES "${CMAKE_CURRENT_BINARY_DIR}")
list(APPEND CPACK_SOURCE_IGNORE_FILES "\\\\.svn")
list(APPEND CPACK_SOURCE_IGNORE_FILES "\\\\.git")
list(APPEND CPACK_SOURCE_IGNORE_FILES ".swp")
list(APPEND CPACK_SOURCE_IGNORE_FILES ".*~")

set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/license.txt")
set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/readme.md")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/description.txt")

set(CPACK_PACKAGE_VERSION_MAJOR 7)
set(CPACK_PACKAGE_VERSION_MINOR 0)
set(CPACK_PACKAGE_VERSION_PATCH 0)

set(CPACK_PACKAGE_EXECUTABLES "chai;ChaiScript Eval")
set(CPACK_PACKAGE_VENDOR "ChaiScript.com")
set(CPACK_PACKAGE_CONTACT "contact@chaiscript.com")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "An embedded scripting language for C++")

set(CPACK_DEBIAN_PACKAGE_SECTION "devel")
set(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")

set(CPACK_RPM_PACKAGE_LICENSE "BSD")
set(CPACK_RPM_PACKAGE_GROUP "Programming")

set(CHAI_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})

configure_file(Doxyfile.in ${CMAKE_BINARY_DIR}/Doxyfile)

include(CTest)
include(CPack)
include(cmake/Catch.cmake)

if(NOT MINGW)
  find_library(READLINE_LIBRARY NAMES readline PATH /usr/lib /usr/local/lib /opt/local/lib)
endif()

if(UNIX AND NOT APPLE)
  find_program(VALGRIND NAMES valgrind PATH /usr/bin /usr/local/bin)
endif()

enable_testing()


message(STATUS "Detecting readline support")
if(READLINE_LIBRARY)
  message(STATUS "Found: ${READLINE_LIBRARY}")
  set(READLINE_LIB readline)
  add_definitions(/DREADLINE_AVAILABLE)
else()
  message(STATUS "Not Found")
  set(READLINE_LIB)
  set(READLINE_FLAG)
endif()

if(MSVC)
  add_definitions(/W4 /w14545 /w34242 /w34254 /w34287 /w44263 /w44265 /w44296 /w44311 /w44826 /we4289 /w14546 /w14547 /w14549 /w14555 /w14619 /w14905 /w14906 /w14928)

  if(MSVC_VERSION STREQUAL "1800")
    # VS2013 doesn't have magic statics
    add_definitions(/w44640)
  else()
    # enum warnings are too noisy on MSVC2013
    add_definitions(/w34062)
  endif()

  add_definitions(/bigobj /permissive- /utf-8)
  # Note on MSVC compiler flags.
  # The code base selective disables warnings as necessary when the compiler is complaining too much
  # about something that is perfectly valid, or there is simply no technical way around it
  # This particular warning, C4503 is in regards to the decorated names that MSVC generates internally.
  # The error did not come up until the move to C++11, but the compiler doesn't give enough information
  # to determine where the error is coming from, and the internet provides no real information for
  # how to workaround or fix the error. So I'm disabling it globally.
  add_definitions(/wd4503)
else()
  add_definitions(-Wall -Wextra -Wconversion -Wshadow -Wnon-virtual-dtor -Wold-style-cast -Wcast-align -Wcast-qual -Wunused -Woverloaded-virtual -Wno-noexcept-type -Wpedantic -Werror=return-type)

  if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "AppleClang")
    add_definitions(-Weverything -Wno-c++98-compat-pedantic  -Wno-c++98-compat -Wno-documentation -Wno-switch-enum -Wno-weak-vtables -Wno-missing-prototypes -Wno-padded -Wno-missing-noreturn -Wno-exit-time-destructors -Wno-documentation-unknown-command -Wno-unused-template -Wno-undef -Wno-double-promotion)
  else()
    add_definitions(-Wnoexcept)
  endif()

  if(APPLE)
    add_definitions(-Wno-sign-compare)
  endif()
endif()

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  option(USE_LIBCXX "Use clang's libcxx" FALSE)

  if(USE_LIBCXX)
    add_definitions(-stdlib=libc++)
    set(LINKER_FLAGS "${LINKER_FLAGS} -stdlib=libc++")
  endif()
endif()

# limitations in MinGW require us to make an optimized build
# for the sake of object sizes or something
if(MINGW OR CYGWIN)
  add_definitions(-O3)
endif()

include_directories(include)


set(Chai_INCLUDES include/chaiscript/chaiscript.hpp include/chaiscript/chaiscript_threading.hpp include/chaiscript/dispatchkit/bad_boxed_cast.hpp include/chaiscript/dispatchkit/bind_first.hpp include/chaiscript/dispatchkit/bootstrap.hpp include/chaiscript/dispatchkit/bootstrap_stl.hpp include/chaiscript/dispatchkit/boxed_cast.hpp include/chaiscript/dispatchkit/boxed_cast_helper.hpp include/chaiscript/dispatchkit/boxed_number.hpp include/chaiscript/dispatchkit/boxed_value.hpp include/chaiscript/dispatchkit/dispatchkit.hpp include/chaiscript/dispatchkit/type_conversions.hpp include/chaiscript/dispatchkit/dynamic_object.hpp include/chaiscript/dispatchkit/exception_specification.hpp include/chaiscript/dispatchkit/function_call.hpp include/chaiscript/dispatchkit/function_call_detail.hpp include/chaiscript/dispatchkit/handle_return.hpp include/chaiscript/dispatchkit/operators.hpp include/chaiscript/dispatchkit/proxy_constructors.hpp include/chaiscript/dispatchkit/proxy_functions.hpp include/chaiscript/dispatchkit/proxy_functions_detail.hpp include/chaiscript/dispatchkit/register_function.hpp include/chaiscript/dispatchkit/type_info.hpp include/chaiscript/language/chaiscript_algebraic.hpp include/chaiscript/language/chaiscript_common.hpp include/chaiscript/language/chaiscript_engine.hpp include/chaiscript/language/chaiscript_eval.hpp include/chaiscript/language/chaiscript_parser.hpp include/chaiscript/language/chaiscript_prelude.hpp include/chaiscript/language/chaiscript_prelude_docs.hpp include/chaiscript/utility/utility.hpp include/chaiscript/utility/json.hpp include/chaiscript/utility/json_wrap.hpp)

set_source_files_properties(${Chai_INCLUDES} PROPERTIES HEADER_FILE_ONLY TRUE)

if(NOT MULTITHREAD_SUPPORT_ENABLED)
  add_definitions(-DCHAISCRIPT_NO_THREADS)
endif()

if(NOT DYNLOAD_ENABLED)
  add_definitions(-DCHAISCRIPT_NO_DYNLOAD)
endif()

if(CMAKE_HOST_UNIX)
  if(DYNLOAD_ENABLED)
    if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD" AND NOT ${CMAKE_SYSTEM_NAME} MATCHES "Haiku")
      list(APPEND LIBS "dl")
    endif()
  endif()

  if(MULTITHREAD_SUPPORT_ENABLED)
    if(CMAKE_COMPILER_IS_GNUCC)
      execute_process(COMMAND ${CMAKE_C_COMPILER} --version OUTPUT_VARIABLE GCC_FULL_VERSION)
      if(GCC_FULL_VERSION MATCHES "4.8.1.*ubuntu")
        set(LINKER_FLAGS "${LINKER_FLAGS} -Wl,--no-as-needed -pthread")
      else()
        set(LINKER_FLAGS "${LINKER_FLAGS} -pthread")
      endif()
    else()
      set(LINKER_FLAGS "${LINKER_FLAGS} -pthread")
    endif()

    add_definitions(-pthread)
  endif()

endif()

list(APPEND LIBS ${READLINE_LIB})

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${LINKER_FLAGS}")

add_library(stdlib STATIC static_libs/chaiscript_stdlib.cpp)
add_library(parser STATIC static_libs/chaiscript_parser.cpp)

add_library(chaiscript_stdlib-${CHAI_VERSION} MODULE src/chaiscript_stdlib_module.cpp)
target_link_libraries(chaiscript_stdlib-${CHAI_VERSION} ${LIBS} ${CMAKE_THREAD_LIBS_INIT})

set(CHAISCRIPT_LIBS stdlib parser)

add_executable(chai src/main.cpp ${Chai_INCLUDES})
target_link_libraries(chai ${LIBS} ${CHAISCRIPT_LIBS})

add_library(chaiscript INTERFACE)
target_include_directories(chaiscript INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)

if(BUILD_SAMPLES)
  add_executable(sanity_checks src/sanity_checks.cpp)
  target_link_libraries(sanity_checks ${LIBS})
  add_executable(test_num_exceptions samples/test_num_exceptions.cpp)
  target_link_libraries(test_num_exceptions ${LIBS} ${CHAISCRIPT_LIBS})
  add_executable(memory_leak_test samples/memory_leak_test.cpp)
  target_link_libraries(memory_leak_test ${LIBS} ${CHAISCRIPT_LIBS})
  add_executable(inheritance samples/inheritance.cpp)
  target_link_libraries(inheritance ${LIBS} ${CHAISCRIPT_LIBS})
  add_executable(factory samples/factory.cpp)
  target_link_libraries(factory ${LIBS} ${CHAISCRIPT_LIBS})
  add_executable(fun_call_performance samples/fun_call_performance.cpp)
  target_link_libraries(fun_call_performance ${LIBS} ${CHAISCRIPT_LIBS})
endif()


if(BUILD_MODULES)
  add_library(test_module MODULE src/test_module.cpp)
  target_link_libraries(test_module ${LIBS})

  add_library(stl_extra MODULE src/stl_extra.cpp)
  target_link_libraries(stl_extra ${LIBS})

  set(MODULES stl_extra)
endif()


file(GLOB UNIT_TESTS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/unittests/ ${CMAKE_CURRENT_SOURCE_DIR}/unittests/*.chai ${CMAKE_CURRENT_SOURCE_DIR}/unittests/3.x/*.chai)
list(SORT UNIT_TESTS)

file(GLOB PERFORMANCE_TESTS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/performance_tests/ ${CMAKE_CURRENT_SOURCE_DIR}/performance_tests/*.chai)
list(SORT PERFORMANCE_TESTS)


if(RUN_FUZZY_TESTS)

  file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/unittests")

  execute_process(
    COMMAND ${CMAKE_COMMAND} -E tar xjf ${CMAKE_CURRENT_SOURCE_DIR}/unittests/fuzzy_tests-2017-07-20.tar.bz2
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/unittests
    )


  file(GLOB FUZZY_TESTS RELATIVE ${CMAKE_BINARY_DIR}/unittests/ ${CMAKE_BINARY_DIR}/unittests/MINIMIZED/*)
  list(SORT FUZZY_TESTS)

  foreach(filename ${FUZZY_TESTS})
    message(STATUS "Adding test ${filename}")
    add_test(fuzz.${filename} chai "-e" "--exception" "--any-exception" ${CMAKE_CURRENT_SOURCE_DIR}/unittests/fuzz_unit_test.inc ${CMAKE_BINARY_DIR}/unittests/${filename})
  endforeach()

  set_property(TEST ${FUZZY_EXCEPTION_TESTS}
    PROPERTY ENVIRONMENT
    "CHAI_USE_PATH=${CMAKE_CURRENT_SOURCE_DIR}/unittests/"
    "CHAI_MODULE_PATH=${CMAKE_CURRENT_BINARY_DIR}/"
    )

endif()


if(BUILD_TESTING)

  option(UNIT_TEST_LIGHT "Unit tests light (expect module loading failures)" FALSE)

  add_test(version_check chai -c  "if(\"\\\${ version() };\\\${version_major()};\\\${version_minor()};\\\${version_patch()}\" != \"${CHAI_VERSION};${CPACK_PACKAGE_VERSION_MAJOR};${CPACK_PACKAGE_VERSION_MINOR};${CPACK_PACKAGE_VERSION_PATCH}\") { exit(-1) }")
  set_property(TEST version_check
    PROPERTY ENVIRONMENT
    "CHAI_USE_PATH=${CMAKE_CURRENT_SOURCE_DIR}/unittests/"
    "CHAI_MODULE_PATH=${CMAKE_CURRENT_BINARY_DIR}/"
  )

  add_test(version_check_2 chai --version )
  set_property(TEST version_check_2
    PROPERTY ENVIRONMENT
    "CHAI_MODULE_PATH=${CMAKE_CURRENT_BINARY_DIR}/"
    PROPERTY PASS_REGULAR_EXPRESSION "${CHAI_VERSION}"
  )

  add_test(help chai --help )
  set_property(TEST help
    PROPERTY ENVIRONMENT
    "CHAI_MODULE_PATH=${CMAKE_CURRENT_BINARY_DIR}/"
  )


  set(TESTS "")

  foreach(filename ${UNIT_TESTS})
    message(STATUS "Adding unit test ${filename}")
    add_test(unit.${filename} chai ${CMAKE_CURRENT_SOURCE_DIR}/unittests/unit_test.inc ${CMAKE_CURRENT_SOURCE_DIR}/unittests/${filename})
    list(APPEND TESTS unit.${filename})
  endforeach()

  if(RUN_PERFORMANCE_TESTS)
    foreach(filename ${PERFORMANCE_TESTS})
      message(STATUS "Adding performance test ${filename}")

      add_test(NAME performance.${filename} COMMAND ${VALGRIND} --tool=callgrind --callgrind-out-file=callgrind.performance.${filename} $<TARGET_FILE:chai> ${CMAKE_CURRENT_SOURCE_DIR}/performance_tests/${filename})
      list(APPEND TESTS performance.${filename})
    endforeach()

    add_executable(profile_cpp_calls_2 performance_tests/profile_cpp_calls_2.cpp)
    target_link_libraries(profile_cpp_calls_2 ${LIBS})
    add_test(NAME performance.profile_cpp_calls_2 COMMAND ${VALGRIND} --tool=callgrind --callgrind-out-file=callgrind.performance.profile_cpp_calls_2 $<TARGET_FILE:profile_cpp_calls_2>)

    add_executable(profile_fun_wrappers performance_tests/profile_fun_wrappers.cpp)
    target_link_libraries(profile_fun_wrappers ${LIBS})
    add_test(NAME performance.profile_fun_wrappers COMMAND ${VALGRIND} --tool=callgrind --callgrind-out-file=callgrind.performance.profile_fun_wrappers $<TARGET_FILE:profile_fun_wrappers>)
  endif()

  set_property(TEST ${TESTS}
    PROPERTY ENVIRONMENT
    "CHAI_USE_PATH=${CMAKE_CURRENT_SOURCE_DIR}/unittests/"
    "CHAI_MODULE_PATH=${CMAKE_CURRENT_BINARY_DIR}/"
  )

  if(NOT UNIT_TEST_LIGHT)
    add_executable(compiled_tests unittests/compiled_tests.cpp)
    target_link_libraries(compiled_tests ${LIBS} ${CHAISCRIPT_LIBS})
    catch_discover_tests(compiled_tests TEST_PREFIX "compiled.")

    add_executable(static_chaiscript_test unittests/static_chaiscript.cpp)
    target_link_libraries(static_chaiscript_test ${LIBS})
    add_test(NAME Static_ChaiScript_Test COMMAND static_chaiscript_test)

    add_executable(boxed_cast_test unittests/boxed_cast_test.cpp)
    target_link_libraries(boxed_cast_test ${LIBS})
    add_test(NAME Boxed_Cast_Test COMMAND boxed_cast_test)

    add_executable(type_info_test unittests/type_info_test.cpp)
    target_link_libraries(type_info_test ${LIBS})
    add_test(NAME Type_Info_Test COMMAND type_info_test)

    add_executable(c_linkage_test unittests/c_linkage_test.cpp)
    target_link_libraries(c_linkage_test ${LIBS} ${CHAISCRIPT_LIBS})
    add_test(NAME C_Linkage_Test COMMAND c_linkage_test)

    add_executable(integer_literal_test unittests/integer_literal_test.cpp)
    target_link_libraries(integer_literal_test ${LIBS} ${CHAISCRIPT_LIBS})
    add_test(NAME Integer_Literal_Test COMMAND integer_literal_test)

    if(MULTITHREAD_SUPPORT_ENABLED)
      add_executable(multithreaded_test unittests/multithreaded_test.cpp)
      target_link_libraries(multithreaded_test ${LIBS})
      add_test(NAME Multithreaded_Test COMMAND multithreaded_test)
      set_property(TEST Multithreaded_Test
        PROPERTY ENVIRONMENT
        "CHAI_USE_PATH=${CMAKE_CURRENT_SOURCE_DIR}/unittests/"
        "CHAI_MODULE_PATH=${CMAKE_CURRENT_BINARY_DIR}/"
      )
    endif()

    add_executable(multifile_test
      unittests/multifile_test_main.cpp
      unittests/multifile_test_chai.cpp
      unittests/multifile_test_module.cpp
    )
    target_link_libraries(multifile_test ${LIBS})
    add_test(NAME MultiFile_Test COMMAND multifile_test)

    install(TARGETS test_module RUNTIME DESTINATION bin LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}/chaiscript")
  endif()
endif()


if(BUILD_LIBFUZZ_TESTER)
  add_executable(fuzzer src/libfuzzer_client.cpp src/sha3.cpp)
  target_compile_options(fuzzer PRIVATE "-fsanitize=fuzzer,address")
  target_link_libraries(fuzzer PRIVATE ${LIBS} "-fsanitize=fuzzer,address")
endif()


install(TARGETS chai chaiscript_stdlib-${CHAI_VERSION} ${MODULES} RUNTIME DESTINATION bin LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}/chaiscript")

install(DIRECTORY include/chaiscript DESTINATION include
  PATTERN "*.hpp"
  PATTERN "*/.svn*" EXCLUDE
  PATTERN "*/.git*" EXCLUDE
  PATTERN "*~" EXCLUDE)
install(DIRECTORY unittests DESTINATION share/chaiscript
  PATTERN "*.chai"
  PATTERN "*.inc"
  PATTERN "*/.svn*" EXCLUDE
  PATTERN "*/.git*" EXCLUDE
  PATTERN "*~" EXCLUDE)
install(DIRECTORY samples DESTINATION share/chaiscript
  PATTERN "*.chai"
  PATTERN "*/.svn*" EXCLUDE
  PATTERN "*/.git*" EXCLUDE
  PATTERN "*~" EXCLUDE)

configure_file(contrib/pkgconfig/chaiscript.pc.in lib/pkgconfig/chaiscript.pc @ONLY)
install(FILES "${chaiscript_BINARY_DIR}/lib/pkgconfig/chaiscript.pc"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")

    
