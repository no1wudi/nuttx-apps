# ##############################################################################
# apps/mlearning/iree/CMakeLists.txt
#
# SPDX-License-Identifier: Apache-2.0
#
# Licensed to the Apache Software Foundation (ASF) under one or more contributor
# license agreements.  See the NOTICE file distributed with this work for
# additional information regarding copyright ownership.  The ASF licenses this
# file to you under the Apache License, Version 2.0 (the "License"); you may not
# use this file except in compliance with the License.  You may obtain a copy of
# the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
# WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  See the
# License for the specific language governing permissions and limitations under
# the License.
#
# ##############################################################################

if(CONFIG_IREE)

  # ############################################################################
  # Build IREE as an external project
  # ############################################################################

  include(ExternalProject)

  set(IREE_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/iree)
  set(IREE_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/iree-build)
  set(IREE_INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/iree-install)

  # Inherit compiler flags from parent project and add our specific flags
  set(INHERITED_C_FLAGS "${CMAKE_C_FLAGS}")
  set(INHERITED_CXX_FLAGS "${CMAKE_CXX_FLAGS}")

  # Add required platform definitions
  set(PLATFORM_DEFS "-DIREE_FILE_IO_ENABLE=0 -DIREE_PLATFORM_GENERIC=1 -DIREE_TARGET_PLATFORM_GENERIC=1 -DIREE_SYNCHRONIZATION_DISABLE_UNSAFE=1")

  # Add format warning suppressions
  set(FORMAT_SUPPRESSIONS "-Wno-error=format -Wno-format -Wno-format-security -Wno-char-subscripts -Wno-implicit-function-declaration")

  # Combine all flags
  set(COMBINED_C_FLAGS "${INHERITED_C_FLAGS} ${PLATFORM_DEFS} ${FORMAT_SUPPRESSIONS}")
  set(COMBINED_CXX_FLAGS "${INHERITED_CXX_FLAGS} ${PLATFORM_DEFS} ${FORMAT_SUPPRESSIONS}")

  # Define IREE build options
  set(IREE_CMAKE_ARGS
    # Mark that C and C++ compilers work for the external project
    -DCMAKE_C_COMPILER_WORKS=1
    -DCMAKE_CXX_COMPILER_WORKS=1
    # Build the IREE runtime only
    -DIREE_BUILD_COMPILER=OFF
    -DIREE_ERROR_ON_MISSING_SUBMODULES=OFF
    # Tell CMake to skip targeting a specific operating system
    -DCMAKE_SYSTEM_NAME=Generic
    # Inherit sysroot from parent project
    -DCMAKE_SYSROOT=${CMAKE_SYSROOT}
    # Disable multi-thread library support
    -DIREE_ENABLE_THREADING=OFF
    # Only enable the local synchronous HAL driver
    -DIREE_HAL_DRIVER_DEFAULTS=OFF
    -DIREE_HAL_DRIVER_LOCAL_SYNC=ON
    # Only enable some executable loaders
    -DIREE_HAL_EXECUTABLE_LOADER_DEFAULTS=OFF
    -DIREE_HAL_EXECUTABLE_LOADER_EMBEDDED_ELF=ON
    -DIREE_HAL_EXECUTABLE_LOADER_VMVX_MODULE=ON
    # Only enable the embedded ELF executable plugin
    -DIREE_HAL_EXECUTABLE_PLUGIN_DEFAULTS=OFF
    -DIREE_HAL_EXECUTABLE_PLUGIN_EMBEDDED_ELF=ON
    # Disable tests until IREE supports running them on bare-metal platforms
    -DIREE_BUILD_TESTS=OFF
    # Build samples
    -DIREE_BUILD_SAMPLES=OFF
    # Installation directory
    -DCMAKE_INSTALL_PREFIX=${IREE_INSTALL_DIR}
    # Use same build type as main project
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    # Pass compiler flags
    -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    # Pass the combined flags with include directories
    "-DCMAKE_C_FLAGS=${COMBINED_C_FLAGS}"
    "-DCMAKE_CXX_FLAGS=${COMBINED_CXX_FLAGS}"
  )

  ExternalProject_Add(
    iree_external
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/iree
    SOURCE_DIR ${IREE_SOURCE_DIR}
    BINARY_DIR ${IREE_BINARY_DIR}
    INSTALL_DIR ${IREE_INSTALL_DIR}
    CMAKE_ARGS ${IREE_CMAKE_ARGS}
    BUILD_COMMAND ${CMAKE_COMMAND} --build <BINARY_DIR> --config Release -t libiree_runtime_unified.a
    INSTALL_COMMAND ${CMAKE_COMMAND} --build <BINARY_DIR> --config Release -t libiree_runtime_unified.a
  )

  # Add include directories for the built IREE
  include_directories(${IREE_INSTALL_DIR}/include)
  link_directories(${IREE_INSTALL_DIR}/lib)

  # Add the IREE library to the target
  add_dependencies(apps iree_external)

endif()
