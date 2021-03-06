# analysis and tensorrt must be added before creating static library,
# otherwise, there would be undefined reference to them in static library.
add_subdirectory(analysis)
if (TENSORRT_FOUND)
  add_subdirectory(tensorrt)
endif()

set(FLUID_CORE_MODULES proto_desc memory lod_tensor executor)

# TODO(panyx0718): Should this be called paddle_fluid_inference_api_internal?
cc_library(paddle_fluid_api
    SRCS io.cc
    DEPS ${FLUID_CORE_MODULES} ${GLOB_OP_LIB}) 

get_property(fluid_modules GLOBAL PROPERTY FLUID_MODULES)

# paddle_fluid_origin exclude inference api interface
cc_library(paddle_fluid_origin DEPS ${fluid_modules} paddle_fluid_api)

add_subdirectory(api)

set(STATIC_INFERENCE_APIS paddle_fluid_api paddle_inference_api analysis_predictor)
set(SHARED_INFERENCE_SRCS
    io.cc ${CMAKE_CURRENT_SOURCE_DIR}/api/api.cc ${CMAKE_CURRENT_SOURCE_DIR}/api/api_impl.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/api/analysis_predictor.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/api/details/zero_copy_tensor.cc)
if (WITH_GPU AND TENSORRT_FOUND)
  set(STATIC_INFERENCE_APIS ${STATIC_INFERENCE_APIS} paddle_inference_tensorrt_subgraph_engine)
  set(SHARED_INFERENCE_SRCS ${SHARED_INFERENCE_SRCS} ${CMAKE_CURRENT_SOURCE_DIR}/api/api_tensorrt_subgraph_engine.cc)
endif()

# Create static library
cc_library(paddle_fluid DEPS ${fluid_modules} ${STATIC_INFERENCE_APIS} zero_copy_tensor)

if(NOT APPLE)
  # TODO(liuyiqu: Temporarily disable the link flag because it is not support on Mac.
  set(LINK_FLAGS "-Wl,--retain-symbols-file ${CMAKE_CURRENT_SOURCE_DIR}/paddle_fluid.sym")
  set_target_properties(paddle_fluid PROPERTIES LINK_FLAGS "${LINK_FLAGS}")
endif()

# Create shared library
cc_library(paddle_fluid_shared SHARED SRCS ${SHARED_INFERENCE_SRCS}
    DEPS ${fluid_modules} paddle_fluid_api)

set_target_properties(paddle_fluid_shared PROPERTIES OUTPUT_NAME paddle_fluid)
if(NOT APPLE)
  # TODO(liuyiqun): Temporarily disable the link flag because it is not support on Mac.
  set(LINK_FLAGS "-Wl,--version-script ${CMAKE_CURRENT_SOURCE_DIR}/paddle_fluid.map")
  set_target_properties(paddle_fluid_shared PROPERTIES LINK_FLAGS "${LINK_FLAGS}")
  # check symbol hidden
  FILE(WRITE ${CMAKE_CURRENT_BINARY_DIR}/check_symbol.cmake
    "execute_process(COMMAND bash -c \"${CMAKE_CURRENT_SOURCE_DIR}/check_symbol.sh"
    " ${CMAKE_CURRENT_BINARY_DIR}/libpaddle_fluid.so\" RESULT_VARIABLE symbol_res)\n"
    "if(NOT \"\${symbol_res}\" STREQUAL \"0\")\n"
    "  message(FATAL_ERROR \"Check symbol failed.\")\n"
    "endif()\n")
  add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/.check_symbol"
    COMMAND ${CMAKE_COMMAND} -P "${CMAKE_CURRENT_BINARY_DIR}/check_symbol.cmake"
    DEPENDS paddle_fluid_shared)
  add_custom_target(check_symbol ALL DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/.check_symbol")
endif()

if(WITH_TESTING)
    # tests/book depends the models that generated by python/paddle/fluid/tests/book
  add_subdirectory(tests/book)
  if(WITH_INFERENCE_API_TEST)
    add_subdirectory(tests/api)
  endif()  
endif()
