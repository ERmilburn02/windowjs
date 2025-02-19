set(CMAKE_CXX_STANDARD 17)

if(MSVC)
  # Enable all warnings.
  add_compile_options(-W3)
  # Warnings are errors.
  add_compile_options(-WX)

  # Disable RTTI.
  string(REPLACE "/GR" "" CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
  add_compile_options(/GR-)

  if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    # Disable numeric conversion warnings.
    add_compile_options(-wd4267)
    add_compile_options(-wd4244)
    add_compile_options(-wd4305)
    # Disable 4068: unknown pragma (for clang pragmas)
    add_compile_options(-wd4068)
    # Disable warnings in external headers.
    add_compile_options(-external:W0)
    add_compile_options(-external:anglebrackets)
  endif()

  add_compile_options(-D_CRT_SECURE_NO_WARNINGS)

  if(CMAKE_BUILD_TYPE STREQUAL Release)
    add_compile_options(-O2)
  else()
    add_compile_options(-Zi)
    if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
      # Uncomment to detect leaks:
      # add_compile_options(-fsanitize=address)
    endif()
  endif()
else()
  add_compile_options(-Wall)
  add_compile_options(-Werror)
  add_compile_options(-fno-exceptions)
  add_compile_options(-fno-rtti)

  if(CMAKE_BUILD_TYPE STREQUAL Release)
    add_compile_options(-O3)
  else()
    add_compile_options(-g)

    # Uncomment to detect leaks:
    # add_compile_options(-fsanitize=address -fno-omit-frame-pointer -O1)
    # add_link_options(-fsanitize=address)
  endif()
endif()

if(CMAKE_BUILD_TYPE STREQUAL Release)
  add_compile_options(-DWINDOWJS_RELEASE_BUILD)
endif()

add_subdirectory(p5)
add_subdirectory(tools)

# Generate the embedded console source.
add_custom_command(
    OUTPUT generated_console.cc
    COMMAND embed ${CMAKE_CURRENT_BINARY_DIR}/generated_console.cc kEmbeddedConsoleSource ./src/console.js
    DEPENDS embed console.js
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/..
)

# Generate the embedded default (welcome) source.
add_custom_command(
    OUTPUT generated_default.cc
    COMMAND embed ${CMAKE_CURRENT_BINARY_DIR}/generated_default.cc kEmbeddedDefaultSource ./src/default.js
    DEPENDS embed default.js
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/..
)

# Generate git revision info.
include(GetGitRevisionDescription)
get_git_head_revision(GIT_REFSPEC GIT_HASH)
git_get_exact_tag(GIT_TAG)
if("${GIT_TAG}" MATCHES "NOTFOUND")
  set(GIT_TAG "")
endif()
configure_file(version.cc.in generated_version.cc)

add_library(windowjs-library STATIC
    args.cc
    args.h
    canvas.cc
    canvas.h
    config.h
    console.cc
    console.h
    css.cc
    css.h
    fail.cc
    fail.h
    file.cc
    file.h
    generated_console.cc
    generated_version.cc
    js.cc
    js.h
    js_api.cc
    js_api.h
    js_api_canvas.cc
    js_api_canvas.h
    js_api_codec.cc
    js_api_codec.h
    js_api_file.cc
    js_api_file.h
    js_api_process.cc
    js_api_process.h
    js_events.cc
    js_events.h
    js_scope.h
    js_strings.cc
    js_strings.h
    json.cc
    json.h
    main.cc
    main.h
    platform.h
    signal.h
    stats.cc
    stats.h
    subprocess.cc
    subprocess.h
    task_queue.cc
    task_queue.h
    thread.cc
    thread.h
    util.h
    version.h
    weak.cc
    weak.h
    window.cc
    window.h
    zip.cc
    zip.h
)

add_executable(windowjs
    config.cc
    generated_default.cc
)

target_include_directories(windowjs-library PRIVATE ../libraries/v8/third_party/zlib)
target_link_libraries(windowjs-library PRIVATE glfw skia uv_a v8 angle)
target_link_libraries(windowjs PRIVATE windowjs-library)

if(MSVC)
  if(CMAKE_BUILD_TYPE STREQUAL Release)
    # Configure the target to be a windows instead of a console application.
    set_target_properties(windowjs PROPERTIES LINK_FLAGS "/entry:mainCRTStartup /subsystem:windows")
  endif()
endif()

set_target_properties(windowjs
  PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/"
)
