cmake_minimum_required(VERSION 3.19)
project(hyprwire_tests LANGUAGES CXX)

enable_testing()

set(CMAKE_CXX_STANDARD 23)
add_compile_options(
  -Wall
  -Wextra
  -Wpedantic
  -Wno-unused-parameter
  -Wno-unused-value
  -Wno-missing-field-initializers
  -Wno-narrowing
  -Wno-pointer-arith)

find_package(PkgConfig REQUIRED)
pkg_check_modules(hyprwire REQUIRED IMPORTED_TARGET hyprwire)

find_package(hyprwire-scanner REQUIRED)
include_directories(${CMAKE_BINARY_DIR})
link_libraries(PkgConfig::hyprwire)

add_executable(server tests/Server.cpp)
add_executable(client tests/Client.cpp)

set(generatedDir generated)

function(hyprwire_protocol targets isClient outputPath protoName protoPath protoFile)
    if(isClient)
        set(mode "--client")
        set(suffix "client")
    else()
        set(mode "")
        set(suffix "server")
    endif()

    set(outdir "${CMAKE_SOURCE_DIR}/tests/generated")
    set(xml_in "${protoPath}/${protoFile}")

    set(out_cpp "${outdir}/${protoName}-${suffix}.cpp")
    set(out_hpp "${outdir}/${protoName}-${suffix}.hpp")

    add_custom_command(
        OUTPUT "${out_cpp}" "${out_hpp}"
        COMMAND "${CMAKE_COMMAND}" -E make_directory "${outdir}"
        COMMAND hyprwire-scanner ${mode} "${xml_in}" "${outdir}"
        DEPENDS "${xml_in}"
        VERBATIM
        COMMENT
        "Generating ${suffix} protocol from ${protoFile} -> ${protoName}-${suffix}.{cpp,hpp}"
    )

    foreach(target ${targets})
        target_sources(${target} PRIVATE "${out_cpp}")
        target_sources(${target} PRIVATE "${out_hpp}")
    endforeach()
endfunction()

hyprwire_protocol(
  server
  FALSE
  "${generatedDir}"
  "test_protocol_v1"
  ${CMAKE_SOURCE_DIR}/tests
  "protocol-v1.xml"
)
hyprwire_protocol(
  client
  TRUE
  "${generatedDir}"
  "test_protocol_v1"
  ${CMAKE_SOURCE_DIR}/tests
  "protocol-v1.xml"
)

enable_testing()
add_test(
    NAME test
    COMMAND "${CMAKE_SOURCE_DIR}/test-runner"
)
