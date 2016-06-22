# Compile internal irrXML only if system is not requested
if( NOT SYSTEM_IRRXML )
    if(HUNTER_ENABLED)
      hunter_add_package(irrXML)
      find_package(irrXML CONFIG REQUIRED)
    else()
      add_subdirectory(irrXML)
    endif()
endif( NOT SYSTEM_IRRXML )
