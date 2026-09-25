#
# This file is part of the CMaNGOS Project. See AUTHORS file for Copyright information
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
#

if(BUILD_GAME_SERVER OR BUILD_LOGIN_SERVER OR BUILD_EXTRACTORS)
  add_subdirectory(framework)
  add_subdirectory(shared)
endif()

# Playerbots module
if(BUILD_PLAYERBOTS)
  include(FetchContent)

  FetchContent_Declare(
    PlayerBots
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/modules/PlayerBots"
    GIT_REPOSITORY "https://github.com/cmangos/playerbots.git"
    GIT_TAG "master"
  )

  FetchContent_GetProperties(PlayerBots)
  if (NOT playerbots_POPULATED)
    FetchContent_Populate(PlayerBots)
    message(STATUS "Playerbots module source dir: ${playerbots_SOURCE_DIR}")
  else()
    message(STATUS "Playerbots module already populated: ${playerbots_POPULATED}")
  endif()

  # NOTE: local patch — support FETCHCONTENT_SOURCE_DIR_PLAYERBOTS pointing outside the core tree
  # (out-of-tree source requires an explicit binary dir, matching the default in-tree location)
  add_subdirectory(${playerbots_SOURCE_DIR} ${CMAKE_CURRENT_BINARY_DIR}/modules/PlayerBots)

else()
  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/modules/PlayerBots)
    message(STATUS "PlayerBots module exists, but not building. You can remove it manually if you don't need it. (rm -rf ${CMAKE_CURRENT_SOURCE_DIR}/modules/PlayerBots)")

    #file(REMOVE_RECURSE ${CMAKE_CURRENT_SOURCE_DIR}/modules/PlayerBots)
  endif()
endif()

# Same layout as PlayerBots: src/modules/<name> must already exist (symlink,
# junction, or submodule). This function does not create that link. Fetch only
# when the module is enabled and that directory has no checkout. Existing
# directories are never deleted.
function(cmangos_resolve_module folder repo)
  set(_dest "${CMAKE_SOURCE_DIR}/src/modules/${folder}")

  if(EXISTS "${_dest}/CMakeLists.txt")
    message(STATUS "Using local ${folder} module: ${_dest}")
  else()
    message(STATUS "Local ${folder} module not found, fetching ${repo}")
    include(FetchContent)
    FetchContent_Declare(
      ${folder}
      GIT_REPOSITORY "${repo}"
      GIT_TAG "main"
      SOURCE_DIR "${_dest}"
    )
    FetchContent_GetProperties(${folder})
    if(NOT ${folder}_POPULATED)
      FetchContent_Populate(${folder})
    endif()
  endif()

  if(NOT EXISTS "${_dest}/CMakeLists.txt")
    message(FATAL_ERROR "Module ${folder} is not available at ${_dest}")
  endif()
endfunction()

# Feature modules before the framework, and both before game, so their targets
# exist when game links them. The framework is required whenever BUILD_MODULES is on.
if(BUILD_MODULES)
  set(_feature_modules transmog dualspec achievements barber)
  set(_enabled_features "")
  foreach(_folder ${_feature_modules})
    string(TOUPPER ${_folder} _upper)
    if(BUILD_MODULE_${_upper})
      list(APPEND _enabled_features ${_folder})
    endif()
  endforeach()

  foreach(_folder ${_enabled_features})
    if(_folder STREQUAL "barber")
      cmangos_resolve_module(${_folder} "https://github.com/celguar/cmangos-${_folder}.git")
    else()
      cmangos_resolve_module(${_folder} "https://github.com/flekz-games/cmangos-${_folder}.git")
    endif()
  endforeach()
  cmangos_resolve_module(modules "https://github.com/flekz-games/cmangos-modules.git")

  foreach(_folder ${_enabled_features})
    add_subdirectory("${CMAKE_SOURCE_DIR}/src/modules/${_folder}" "${CMAKE_CURRENT_BINARY_DIR}/modules/${_folder}")
  endforeach()
  add_subdirectory("${CMAKE_SOURCE_DIR}/src/modules/modules" "${CMAKE_CURRENT_BINARY_DIR}/modules/modules")
endif()

if(BUILD_GAME_SERVER)
  add_subdirectory(game)
  add_subdirectory(mangosd)
endif()

if(BUILD_LOGIN_SERVER)
  add_subdirectory(realmd)
endif()