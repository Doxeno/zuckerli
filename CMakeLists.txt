cmake_minimum_required(VERSION 3.10)

# Set compilers to default to Clang is not set.
if ("$ENV{CC}" STREQUAL "")
  set(ENV{CC} clang)
endif()
if ("$ENV{CXX}" STREQUAL "")
  set(ENV{CXX} clang++)
endif()


project(zuckerli)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED True)

add_compile_options(-fcolor-diagnostics -Wall)

find_package(Threads REQUIRED)

add_subdirectory(abseil-cpp)

include(CheckIPOSupported)
check_ipo_supported(RESULT supported OUTPUT error)
if( supported )
    message(STATUS "IPO / LTO enabled")
    set_property(GLOBAL PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
else()
    message(STATUS "IPO / LTO not supported: <${error}>")
endif()

add_library(
  common
  src/common.cc
  src/flags.cc
  src/common.h
)

target_link_libraries(common absl::flags absl::flags_parse)


add_library(
  bit_reader
  src/bit_reader.cc
  src/bit_reader.h
)
target_link_libraries(bit_reader common)

add_library(
  bit_writer
  src/bit_writer.cc
  src/bit_writer.h
)
target_link_libraries(bit_writer common)


# A library cannot contain just a headerfile.
#add_library(
#  entropy_coder_common
#  src/integer_coder.h
#)
add_library(entropy_coder_common INTERFACE)
target_include_directories(entropy_coder_common INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCDIR}>/src)

target_link_libraries(entropy_coder_common INTERFACE bit_reader bit_writer)


add_library(
  huffman
  src/huffman.cc
  src/huffman.h
)
target_link_libraries(huffman entropy_coder_common)


add_library(
  ans
  src/ans.cc
  src/ans.h
)
target_link_libraries(ans entropy_coder_common)


add_library(
  uncompressed_graph
  src/uncompressed_graph.cc
  src/uncompressed_graph.h
)
target_link_libraries(uncompressed_graph entropy_coder_common)



add_executable(traversal_main_uncompressed src/traversal_main_uncompressed.cc)
target_link_libraries(traversal_main_uncompressed uncompressed_graph Threads::Threads)

add_library(encode src/encode.h src/encode.cc src/context_model.h src/checksum.h)
target_link_libraries(encode ans huffman uncompressed_graph)


# A library cannot contain just headerfiles.
#add_library(decode src/decode.h src/context_model.h src/checksum.h)

add_library(decode INTERFACE)
target_include_directories(decode INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCDIR}>/src)

target_link_libraries(decode INTERFACE ans huffman)


add_library(
  compressed_graph
  src/compressed_graph.cc
  src/compressed_graph.h
)
target_link_libraries(compressed_graph decode)

add_executable(traversal_main_compressed src/traversal_main_compressed.cc)
target_link_libraries(traversal_main_compressed compressed_graph Threads::Threads)




add_executable(encoder src/encode_main.cc)
target_link_libraries(encoder encode)

add_executable(decoder src/decode_main.cc)
target_link_libraries(decoder decode)
