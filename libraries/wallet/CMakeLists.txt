file(GLOB HEADERS "include/graphene/wallet/*.hpp")

find_package( Perl )
find_package( Doxygen )

if( PERL_FOUND AND DOXYGEN_FOUND AND NOT "${CMAKE_GENERATOR}" STREQUAL "Ninja" )
  configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile )
  add_custom_command( OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/doxygen/perlmod/DoxyDocs.pm
                      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                      COMMAND ${DOXYGEN_EXECUTABLE}
                      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile include/graphene/wallet/wallet.hpp )
  if(MSVC)
    add_custom_command( OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp
                        COMMAND ${PERL_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/generate_api_documentation.pl ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp.new
                        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp.new ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp
                        COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp.new
                        DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/generate_api_documentation.pl ${CMAKE_CURRENT_BINARY_DIR}/doxygen/perlmod/DoxyDocs.pm )
  else(MSVC)
    add_custom_command( OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp
                        COMMAND ${PERL_EXECUTABLE} -I "${CMAKE_CURRENT_BINARY_DIR}" ${CMAKE_CURRENT_SOURCE_DIR}/generate_api_documentation.pl ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp.new
                        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp.new ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp
                        COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp.new
                        DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/generate_api_documentation.pl ${CMAKE_CURRENT_BINARY_DIR}/doxygen/perlmod/DoxyDocs.pm )
  endif(MSVC)
else()
  # no perl and doxygen, generate the best docs we can at runtime from reflection
  add_custom_command( OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp
                      COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_SOURCE_DIR}/api_documentation_standin.cpp ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp
                      DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/api_documentation_standin.cpp )
endif()

set( SOURCES
     operation_printer.cpp
     reflect_util.cpp
     wallet.cpp
     wallet_account.cpp
     wallet_api_impl.cpp
     wallet_asset.cpp
     wallet_builder.cpp
     wallet_debug.cpp
     wallet_network.cpp
     wallet_results.cpp
     wallet_sign.cpp
     wallet_transfer.cpp
     wallet_voting.cpp
   )

add_library( graphene_wallet ${SOURCES} ${CMAKE_CURRENT_BINARY_DIR}/api_documentation.cpp ${HEADERS} )
target_link_libraries( graphene_wallet PRIVATE graphene_app graphene_chain graphene_utilities fc
                       ${CMAKE_DL_LIBS} ${PLATFORM_SPECIFIC_LIBS} )
target_include_directories( graphene_db PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/include" )

set( GRAPHENE_WALLET_BIG_FILES
     wallet.cpp
     wallet_api_impl.cpp
     wallet_voting.cpp
   )

if(MSVC)
  set_source_files_properties( ${GRAPHENE_WALLET_BIG_FILES} PROPERTIES COMPILE_FLAGS "/bigobj" )
else( MSVC )
   if( MINGW )
      set_source_files_properties( ${GRAPHENE_WALLET_BIG_FILES} PROPERTIES COMPILE_FLAGS -Wa,-mbig-obj )
   endif( MINGW )
endif(MSVC)

install( TARGETS
   graphene_wallet

   RUNTIME DESTINATION bin
   LIBRARY DESTINATION lib
   ARCHIVE DESTINATION lib
)
install( FILES ${HEADERS} DESTINATION "include/graphene/wallet" )
