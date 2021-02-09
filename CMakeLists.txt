cmake_minimum_required (VERSION 3.5)
project ("bare_metal_cpp" NONE)

find_program (ASCIIDOCTOR_EXE asciidoctor)
if (NOT ASCIIDOCTOR_EXE)
    message(FATAL_ERROR "asciidoctor is not found")
endif ()

find_program (ASCIIDOCTOR_PDF_EXE asciidoctor-pdf)
if (NOT ASCIIDOCTOR_PDF_EXE)
    message(FATAL_ERROR "asciidoctor-pdf is not found")
endif ()

find_program (DIA_EXE dia)
if (NOT DIA_EXE)
    message(FATAL_ERROR "dia is not found")
endif ()

set (VERSION "v1.0")
set (MAIN_FILE "${PROJECT_SOURCE_DIR}/bare_metal_cpp.adoc")

set(IMAGES_TARGET "images")
set(PDF_TARGET "pdf")
set(SITE_TARGET "site")
set(BUILD_IMAGES_DIR "${PROJECT_BINARY_DIR}/images")

###################################################################

function (images)
    file (GLOB dia_files ${PROJECT_SOURCE_DIR}/dia/*.dia)
    file (GLOB images_files ${PROJECT_SOURCE_DIR}/images/*)
    set (images_targets)
    foreach (f ${dia_files})
        string (REPLACE ".dia" ".png" temp ${f})
        string (REPLACE "${PROJECT_SOURCE_DIR}/dia" "${BUILD_IMAGES_DIR}" out_file ${temp})
        string (REPLACE "${BUILD_IMAGES_DIR}/" "" target ${out_file})
        
        add_custom_command(
            OUTPUT "${out_file}"
            COMMAND ${CMAKE_COMMAND} -E make_directory "${BUILD_IMAGES_DIR}"
            COMMAND dia -e ${out_file} -t png-libart ${f}
            COMMAND ${CMAKE_COMMAND} -E copy_directory ${PROJECT_SOURCE_DIR}/images ${BUILD_IMAGES_DIR}
            DEPENDS ${f} ${images_files}
        )

        add_custom_target(
            "${target}" DEPENDS "${out_file}"
        )

        list(APPEND images_targets ${target})
    endforeach ()

    add_custom_target(
        ${IMAGES_TARGET} ALL DEPENDS ${images_targets}
    )

endfunction ()

###################################################################

function (pdf)
    add_custom_target(${PDF_TARGET} ALL
        COMMAND ${ASCIIDOCTOR_PDF_EXE} -a imagesdir=${PROJECT_BINARY_DIR} ${MAIN_FILE}
        COMMAND ${CMAKE_COMMAND} -E rename "${PROJECT_SOURCE_DIR}/bare_metal_cpp.pdf" "${PROJECT_BINARY_DIR}/bare_metal_cpp_${VERSION}.pdf"
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
        DEPENDS ${IMAGES_TARGET}
    )

endfunction ()

###################################################################

function (site)
    add_custom_target(${SITE_TARGET} ALL
        COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/site/images
        COMMAND ${ASCIIDOCTOR_EXE} -D ${PROJECT_BINARY_DIR}/site -o index.html ${MAIN_FILE}
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${BUILD_IMAGES_DIR} ${PROJECT_BINARY_DIR}/site/images/
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
        DEPENDS ${IMAGES_TARGET}
    )
endfunction ()

###################################################################

images()
pdf()
site()
