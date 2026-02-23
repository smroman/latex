cmake_minimum_required(VERSION 3.10)
project(MarkdownDocuments LANGUAGES NONE)

# -------------------------------
# Find pandoc-crossref (required)
# -------------------------------
find_program(PANDOC_CROSSREF_EXECUTABLE NAMES pandoc-crossref pandoc-crossref.exe)

if(NOT PANDOC_CROSSREF_EXECUTABLE)
    message(FATAL_ERROR "pandoc-crossref not found! Install it or add it to PATH.")
endif()

message(STATUS "Found pandoc-crossref: ${PANDOC_CROSSREF_EXECUTABLE}")
get_filename_component(_PANDOC_HINT_DIR "${PANDOC_CROSSREF_EXECUTABLE}" DIRECTORY)

# -------------------------------
# Find pandoc (required)
# -------------------------------
# Allow manual override:
set(PANDOC_EXECUTABLE "" CACHE FILEPATH "Path to pandoc executable (pandoc.exe)")

# Common Windows install locations (NO ProgramFiles(x86) because of parentheses parsing)
set(_PANDOC_HINTS
    "${_PANDOC_HINT_DIR}"
    "$ENV{LOCALAPPDATA}/Pandoc"
    "$ENV{ProgramFiles}/Pandoc"
)

if(NOT PANDOC_EXECUTABLE)
    find_program(PANDOC_EXECUTABLE NAMES pandoc pandoc.exe HINTS ${_PANDOC_HINTS})
endif()

if(NOT PANDOC_EXECUTABLE)
    message(FATAL_ERROR "Pandoc not found! Install it or configure with -DPANDOC_EXECUTABLE=\"C:/path/to/pandoc.exe\"")
endif()

message(STATUS "Found pandoc: ${PANDOC_EXECUTABLE}")

# -------------------------------
# Collect markdown files
# -------------------------------
file(GLOB MD_FILES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/*.md")

if(NOT MD_FILES)
    message(WARNING "No markdown files found in directory!")
    return()
endif()

message(STATUS "Found markdown files: ${MD_FILES}")

set(PDF_OUTPUTS "")
set(DOCX_OUTPUTS "")
set(ALL_OUTPUTS "")

# -------------------------------
# Targets per markdown file
# -------------------------------
foreach(MD_FILE ${MD_FILES})
    get_filename_component(BASE_NAME "${MD_FILE}" NAME_WE)
    message(STATUS "Creating targets for: ${BASE_NAME}")

    add_custom_command(
        OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/${BASE_NAME}.pdf"
        COMMAND "${PANDOC_EXECUTABLE}"
                "${MD_FILE}"
                --pdf-engine=lualatex
                --filter "${PANDOC_CROSSREF_EXECUTABLE}"
                --citeproc
                -o "${BASE_NAME}.pdf"
        DEPENDS "${MD_FILE}" "${CMAKE_CURRENT_SOURCE_DIR}/cite.bib"
        COMMENT "Generating PDF: ${BASE_NAME}.pdf"
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        VERBATIM
    )

    add_custom_command(
        OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/${BASE_NAME}.docx"
        COMMAND "${PANDOC_EXECUTABLE}"
                "${MD_FILE}"
                --filter "${PANDOC_CROSSREF_EXECUTABLE}"
                --citeproc
                -o "${BASE_NAME}.docx"
        DEPENDS "${MD_FILE}" "${CMAKE_CURRENT_SOURCE_DIR}/cite.bib"
        COMMENT "Generating DOCX: ${BASE_NAME}.docx"
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        VERBATIM
    )

    list(APPEND PDF_OUTPUTS  "${CMAKE_CURRENT_SOURCE_DIR}/${BASE_NAME}.pdf")
    list(APPEND DOCX_OUTPUTS "${CMAKE_CURRENT_SOURCE_DIR}/${BASE_NAME}.docx")
    list(APPEND ALL_OUTPUTS  "${CMAKE_CURRENT_SOURCE_DIR}/${BASE_NAME}.pdf" "${CMAKE_CURRENT_SOURCE_DIR}/${BASE_NAME}.docx")
endforeach()

# -------------------------------
# Aggregate targets
# -------------------------------
add_custom_target(build_all ALL
    DEPENDS ${ALL_OUTPUTS}
    COMMENT "Building all markdown documents"
)

add_custom_target(build_pdf
    DEPENDS ${PDF_OUTPUTS}
    COMMENT "Building all PDF documents"
)

add_custom_target(build_docx
    DEPENDS ${DOCX_OUTPUTS}
    COMMENT "Building all DOCX documents"
)

foreach(MD_FILE ${MD_FILES})
    get_filename_component(BASE_NAME "${MD_FILE}" NAME_WE)

    add_custom_target("${BASE_NAME}_pdf"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/${BASE_NAME}.pdf"
        COMMENT "Building ${BASE_NAME}.pdf"
    )

    add_custom_target("${BASE_NAME}_docx"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/${BASE_NAME}.docx"
        COMMENT "Building ${BASE_NAME}.docx"
    )
endforeach()

# -------------------------------
# Clean target
# -------------------------------
add_custom_target(clean_all
    COMMAND "${CMAKE_COMMAND}" -E rm -f ${ALL_OUTPUTS}
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    COMMENT "Cleaning all generated documents"
)

message(STATUS "Available targets:")
message(STATUS "  cmake --build build --target build_all")
message(STATUS "  cmake --build build --target build_pdf")
message(STATUS "  cmake --build build --target build_docx")
foreach(MD_FILE ${MD_FILES})
    get_filename_component(BASE_NAME "${MD_FILE}" NAME_WE)
    message(STATUS "  cmake --build build --target ${BASE_NAME}_pdf")
    message(STATUS "  cmake --build build --target ${BASE_NAME}_docx")
endforeach()
message(STATUS "  cmake --build build --target clean_all")
