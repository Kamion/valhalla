## Lists tests
set(tests aabb2 access_restriction actor admin attributes_controller datetime directededge
  distanceapproximator double_bucket_queue edgecollapser edge_elevation edgestatus ellipse encode
  enhancedtrippath factory graphid graphreader graphtile graphtileheader gridded_data grid_range_query grid_traversal
  json laneconnectivity linesegment2 location logging maneuversbuilder map_matcher_factory
  narrativebuilder narrative_dictionary nodeinfo obb2 openlr optimizer  point2 pointll
  polyline2 queue routing sample sequence sign signs streetname streetnames streetnames_factory
  streetnames_us streetname_us tilehierarchy tiles traffic_matcher transitdeparture transitroute transitschedule
  transitstop turn util_midgard util_odin util_skadi vector2 verbal_text_formatter verbal_text_formatter_us
  verbal_text_formatter_us_co verbal_text_formatter_us_tx viterbi_search)

if(ENABLE_DATA_TOOLS)
  list(APPEND tests astar countryaccess edgeinfobuilder graphbuilder graphparser graphtilebuilder
    idtable mapmatch matrix names node_search refs search signinfo trivial_paths uniquenames utrecht)
endif()

if(ENABLE_SERVICES)
  list(APPEND tests loki_service skadi_service thor_service)
endif()

## Add executable targets
foreach(test ${tests})
  add_executable(${test} EXCLUDE_FROM_ALL	${test}.cc test.cc)
  set_target_properties(${test} PROPERTIES
    COMPILE_DEFINITIONS VALHALLA_SOURCE_DIR="${VALHALLA_SOURCE_DIR}/")
  target_link_libraries(${test} valhalla)
endforeach()

set(cost_tests autocost bicyclecost motorscootercost pedestriancost transitcost truckcost)
foreach(test ${cost_tests})
  add_executable(${test} EXCLUDE_FROM_ALL	${VALHALLA_SOURCE_DIR}/src/sif/${test}.cc test.cc)
  set_target_properties(${test} PROPERTIES
    COMPILE_DEFINITIONS INLINE_TEST)
  target_link_libraries(${test} valhalla)
endforeach()

## Test-specific data, properties and dependencies
set_target_properties(logging PROPERTIES COMPILE_DEFINITIONS LOGGING_LEVEL_ALL)

add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/test/data/utrecht_tiles/0/003/196.gph
  COMMAND ${CMAKE_BINARY_DIR}/valhalla_build_tiles
      --inline-config '{"mjolnir":{"tile_dir":"test/data/utrecht_tiles","hierarchy":true,"shortcuts":true,"concurrency":4,"logging":{"type":""}}}'
      ${VALHALLA_SOURCE_DIR}/test/data/utrecht_netherlands.osm.pbf
  COMMENT "Building Utrecht Tiles..."
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
  DEPENDS valhalla_build_tiles ${VALHALLA_SOURCE_DIR}/test/data/utrecht_netherlands.osm.pbf)
add_custom_target(utrecht_tiles DEPENDS ${CMAKE_BINARY_DIR}/test/data/utrecht_tiles/0/003/196.gph)

add_custom_command(OUTPUT .locales.timestamp
  COMMAND /bin/bash -c "for loc in $(jq --raw-output .posix_locale *.json); do \
       $<$<NOT:$<BOOL:${APPLE}>>:  localedef -i $\{loc\%.\*\} -f $\{loc\#\#\*.\} ./$\{loc\}   ; > \
       $<$<BOOL:${APPLE}>:         localedef -i /usr/share/locale/$\{loc\} ./$\{loc\} || true ; > \
     done \
     && touch ${CMAKE_CURRENT_BINARY_DIR}/.locales.timestamp"
  WORKING_DIRECTORY ${VALHALLA_SOURCE_DIR}/locales
  DEPENDS ${VALHALLA_SOURCE_DIR}/locales/*.json
  COMMENT "Compile locale definition files..."
  VERBATIM)
add_custom_target(localedef DEPENDS .locales.timestamp)

file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/test/data)
add_custom_command(OUTPUT test/data/sample/N00
  COMMAND ${CMAKE_COMMAND} -E make_directory test/data/sample/N00
  COMMAND ${CMAKE_COMMAND} -E make_directory test/data/sample/N40
  COMMAND ${CMAKE_COMMAND} -E make_directory test/data/samplegz/N40
  COMMAND ${CMAKE_COMMAND} -E make_directory test/data/samplelz/N40
  COMMAND ${CMAKE_COMMAND} -E make_directory test/data/service
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
  COMMENT "Creating test directories")
add_custom_target(test_directories DEPENDS test/data/sample/N00)


## Test run targets
foreach(test ${tests} ${cost_tests})
  add_custom_command(OUTPUT ${test}.log
    COMMAND
      LOCPATH=${VALHALLA_SOURCE_DIR}/locales
      /bin/bash -c "${CMAKE_CURRENT_BINARY_DIR}/${test} >& ${CMAKE_CURRENT_BINARY_DIR}/${test}.log \
      && echo $(tput setaf 2)$(tput bold)[PASS]$(tput sgr0) ${test} \
      || ( exit=$? ; \
           echo $(tput setaf 1)$(tput bold)[FAIL]$(tput sgr0) ${test} ; \
           cat ${CMAKE_CURRENT_BINARY_DIR}/${test}.log ; \
           exit $exit )"
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS ${test}
    VERBATIM)
  add_custom_target(run-${test} DEPENDS ${test}.log)
endforeach()

## Run test dependencies
add_dependencies(run-util_odin localedef)
add_dependencies(run-narrativebuilder localedef)
add_dependencies(run-sample test_directories)
if(ENABLE_DATA_TOOLS)
  add_dependencies(run-mapmatch utrecht_tiles)
  add_dependencies(run-matrix utrecht_tiles)
  add_dependencies(run-trivial_paths utrecht_tiles)
endif()

if(ENABLE_SERVICES)
  add_dependencies(run-skadi_service test_directories)
endif()

if(ENABLE_PYTHON_BINDINGS)
  find_package(PythonInterp)
  add_custom_command(OUTPUT python_valhalla.log
    COMMAND
      LOCPATH=${VALHALLA_SOURCE_DIR}/locales
      PYTHONPATH=src/bindings/python
      /bin/bash -c "${PYTHON_EXECUTABLE} ${VALHALLA_SOURCE_DIR}/test/bindings/python/utrecht-test.py > test/python_valhalla.log"
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS
      ${VALHALLA_SOURCE_DIR}/test/bindings/python/utrecht-test.py
      ${VALHALLA_SOURCE_DIR}/test/bindings/python/valhalla.json
      utrecht_tiles
      python_valhalla
    VERBATIM)
  add_custom_target(run-python_valhalla DEPENDS python_valhalla.log)
endif()

## High-level targets
string(REGEX REPLACE "([^;]+)" "run-\\1" test_targets "${tests};${cost_tests};python_valhalla")
add_custom_target(check DEPENDS ${test_targets})
add_custom_target(tests DEPENDS ${tests} ${cost_tests})
