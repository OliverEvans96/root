# TInterpreter implementation for cling. Only in libCling; needs to resolve
# symbols from libCore.

ROOT_GLOB_SOURCES(sources ${CMAKE_CURRENT_SOURCE_DIR}/*.cxx)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../res
        ${CMAKE_CURRENT_SOURCE_DIR}/../../clingutils/res
        ${CMAKE_CURRENT_SOURCE_DIR}/../../foundation/res)
include_directories(${CLING_INCLUDE_DIRS})
# Register the llvm include directories after clangs. This instructs the compiler to resolve
# headers from our builtin clang. That's an issue when we are building with bultin_llvm=Off
# and we have installed clang headers, too.
include_directories(SYSTEM ${CLANG_INCLUDE_DIRS} ${LLVM_INCLUDE_DIRS})
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CLING_CXXFLAGS}")

if(MSVC)
  set_source_files_properties(${CMAKE_CURRENT_SOURCE_DIR}/TClingCallbacks.cxx COMPILE_FLAGS -GR-)
else()
  set_source_files_properties(${CMAKE_CURRENT_SOURCE_DIR}/TClingCallbacks.cxx COMPILE_FLAGS -fno-rtti)
endif()

# This to avoid warning coming from llvm/src/tools/clang/include/clang/Sema/Lookup.h:441
# in clang + llvm version r227800.
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
  ROOT_ADD_CXX_FLAG(CMAKE_CXX_FLAGS -Wno-conditional-uninitialized)
endif()

# This is to avoid warnings coming from GCC 7 in llvm/src/include/llvm/ADT/DenseMap.h:1010
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU")
  ROOT_ADD_CXX_FLAG(CMAKE_CXX_FLAGS -Wno-maybe-uninitialized)
endif()

ROOT_OBJECT_LIBRARY(MetaCling ${sources})
add_dependencies(MetaCling CLING)


##### libCling #############################################################

if(NOT builtin_clang)
  set(prefixed_link_libraries)
  foreach(dep ${CLING_DEPEND_LIBS})
    if("${dep}" MATCHES "^clang")
      set(dep "${LLVM_LIBRARY_DIR}/lib${dep}.a")
     endif()
     list(APPEND prefixed_link_libraries "${dep}")
   endforeach()
  set(LINK_LIBS "${prefixed_link_libraries}")
  link_directories("${LLVM_LIBRARY_DIR}")
endif()

ROOT_LINKER_LIBRARY(Cling
        $<TARGET_OBJECTS:ClingUtils>
        $<TARGET_OBJECTS:Dictgen>
        $<TARGET_OBJECTS:MetaCling>
        LIBRARIES ${CLING_LIBRARIES} ${LINK_LIBS})

if(MSVC)
  set_target_properties(Cling PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
  set(cling_exports
      __std_terminate
      ??_7type_info@@6B@
      ?__type_info_root_node@@3U__type_info_node@@A
      ??2@YAPAXI@Z
      ??3@YAXPAX@Z
      ??3@YAXPAXI@Z
      ??_U@YAPAXI@Z
      ??_V@YAXPAX@Z
      ??_V@YAXPAXI@Z
      ??_7type_info@@6B@
      ?__type_info_root_node@@3U__type_info_node@@A
      ?nothrow@std@@3Unothrow_t@1@B
      ??6?$basic_ostream@DU?$char_traits@D@std@@@std@@QAEAAV01@H@Z
      ??6?$basic_ostream@DU?$char_traits@D@std@@@std@@QAEAAV01@M@Z
      ??6?$basic_ostream@DU?$char_traits@D@std@@@std@@QAEAAV01@N@Z
      ??$?6U?$char_traits@D@std@@@std@@YAAAV?$basic_ostream@DU?$char_traits@D@std@@@0@AAV10@D@Z
      cling_runtime_internal_throwIfInvalidPointer
      ?_Facet_Register@std@@YAXPAV_Facet_base@1@@Z
      ??1Value@cling@@QAE@XZ
      ?isValid@Value@cling@@QBE_NXZ
      ?AssertOnUnsupportedTypeCast@Value@cling@@IBEXXZ
      ?setValueNoAlloc@internal@runtime@cling@@YAXPAX00D@Z
      ?setValueNoAlloc@internal@runtime@cling@@YAXPAX00DM@Z
      ?setValueNoAlloc@internal@runtime@cling@@YAXPAX00DN@Z
      ?setValueNoAlloc@internal@runtime@cling@@YAXPAX00DO@Z
      ?setValueNoAlloc@internal@runtime@cling@@YAXPAX00D_K@Z
      ?setValueNoAlloc@internal@runtime@cling@@YAXPAX00DPBX@Z
      ?setValueWithAlloc@internal@runtime@cling@@YAPAXPAX00D@Z
      ?printValue@cling@@YA?AV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@PBX@Z
      ?printValue@cling@@YA?AV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@PB_N@Z
      ?printValue@cling@@YA?AV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@PAPBX@Z
      ?printValue@cling@@YA?AV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@PAPBD@Z
      ??$toUTF8@D@cling@@YA?AV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@QBDID@Z
      ?EvaluateDynamicExpression@internal@runtime@cling@@YA?AVValue@3@PAVInterpreter@3@PAVDynamicExprInfo@123@PAVDeclContext@clang@@@Z
  )
  if(MSVC_VERSION LESS 1914)
    set(cling_exports ${cling_exports} ??3@YAXPAX0@Z ??_V@YAXPAX0@Z)
  endif()
  foreach(sym ${cling_exports})
    set(cling_link_str "${cling_link_str} /EXPORT:${sym}")
  endforeach(sym ${cling_exports})
  set_property(TARGET Cling APPEND_STRING PROPERTY LINK_FLAGS ${cling_link_str})
  add_dependencies(Cling Core RIO)
  target_link_libraries(Cling Core RIO)
endif()

if(APPLE)
  target_link_libraries(Cling -Wl,-bind_at_load -Wl,-undefined -Wl,dynamic_lookup)
elseif(NOT MSVC)
  target_link_libraries(Cling -Wl,--unresolved-symbols=ignore-in-object-files)
endif()
