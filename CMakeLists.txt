cmake_minimum_required(VERSION 2.8.9 FATAL_ERROR)

foreach(p
    CMP0025 # CMake 3.0
    CMP0042 # CMake 3.0
    CMP0056 # CMake 3.2
    CMP0058 # CMake 3.3
    )
  if(POLICY ${p})
    cmake_policy(SET ${p} NEW)
  endif()
endforeach()

project(ITKPythonInstaller)

find_package(ITK REQUIRED)

if (NOT ${ITK_WRAP_PYTHON})
    message(FATAL_ERROR
        "Python wrappings are disabled in ITK build '${ITK_DIR}'.\n"
        "You should enable 'ITK_WRAP_PYTHON' option and rebuild ITK."
    )
endif()

set(ITK_PACKAGE_VERSION "${ITK_VERSION_MAJOR}.${ITK_VERSION_MINOR}.${ITK_VERSION_PATCH}")

string(REGEX MATCH "^[2-3]\\.[0-9]+"
    python_version_major_minor ${ITK_WRAP_PYTHON_VERSION})

find_package(PythonInterp ${python_version_major_minor} EXACT REQUIRED)

set(TARGET_DIR ${CMAKE_BINARY_DIR}/ITKPython)

set(DST_DIR "${TARGET_DIR}/src")
set(DST_ITK_DIR "${DST_DIR}/itk")

set(COPY_FILES_COMMAND "${TARGET_DIR}/copy_files.py")

# Copying the necessary files
add_custom_target(copy_files ALL
    COMMAND ${PYTHON_EXECUTABLE} ${COPY_FILES_COMMAND} "${ITK_DIR}/lib" "${DST_ITK_DIR}"
    COMMAND ${PYTHON_EXECUTABLE} ${COPY_FILES_COMMAND} "${ITK_DIR}/lib/$(Configuration)" "${DST_ITK_DIR}"
    COMMAND ${PYTHON_EXECUTABLE} ${COPY_FILES_COMMAND} "${ITK_DIR}/bin/$(Configuration)" "${DST_ITK_DIR}"
    COMMAND ${PYTHON_EXECUTABLE} ${COPY_FILES_COMMAND} "${ITK_DIR}/Wrapping/Generators/Python/$(Configuration)" "${DST_ITK_DIR}"
    COMMAND ${PYTHON_EXECUTABLE} ${COPY_FILES_COMMAND} "${ITK_DIR}/Wrapping/Generators/Python/Configuration" "${DST_ITK_DIR}/Configuration"
    COMMAND ${PYTHON_EXECUTABLE} ${COPY_FILES_COMMAND} "${ITK_DIR}/Wrapping/Generators/Python/InstallOnly" "${DST_ITK_DIR}"
)

# Building wheel package
add_custom_target(pip_wheel ALL
    COMMAND ${PYTHON_EXECUTABLE} "-m" "pip" "wheel" "--wheel-dir" "${CMAKE_BINARY_DIR}/bin" "."
    WORKING_DIRECTORY "${DST_DIR}"
    DEPENDS copy_files
)

# Install package to python site-packages
add_custom_target(pip_install ALL
    COMMAND ${PYTHON_EXECUTABLE} "-m" "pip" "install" "-U" "--force-reinstall" "--no-index" "--find-links" "${CMAKE_BINARY_DIR}/bin" "itk"
    DEPENDS pip_wheel
)

# Uninstall package from python site-packages
add_custom_target(pip_uninstall
    COMMAND ${PYTHON_EXECUTABLE} "-m" "pip" "uninstall" "--yes" "itk"
)

set(GLOB_COPY_FILES
    "*.py"
    "*.pyd"
    "*.dll"
    "*.so"
)

configure_file(
    ${CMAKE_SOURCE_DIR}/copy_files.py.in
    ${TARGET_DIR}/copy_files.py
    @ONLY
)

configure_file(
    ${CMAKE_SOURCE_DIR}/setup.py.in
    ${DST_DIR}/setup.py
    @ONLY
)

configure_file(
    ${CMAKE_SOURCE_DIR}/itk.pth.in
    ${DST_DIR}/itk.pth
    @ONLY
)
