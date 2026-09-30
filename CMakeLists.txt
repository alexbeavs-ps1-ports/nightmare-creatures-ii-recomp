cmake_minimum_required(VERSION 3.20)
set(PSXRECOMP_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/psxrecomp" CACHE PATH
    "Path to psxrecomp")
if(NOT EXISTS "${PSXRECOMP_ROOT}/runtime/runtime.cmake")
    message(FATAL_ERROR
        "PSXRecomp runtime is missing: ${PSXRECOMP_ROOT}/runtime/runtime.cmake\n"
        "This project expects the psxrecomp framework checkout at ${PSXRECOMP_ROOT}.\n"
        "Run: git submodule update --init --recursive\n"
        "If psxrecomp/ is not a submodule, set -DPSXRECOMP_ROOT=<path-to-psxrecomp>.")
endif()

project(Nightmare_Creatures_II LANGUAGES C CXX)

# --- Opt-in product surfaces (OFF by default in the framework) -------------
# Scaffold prompts / flags fill these. recomp-ui IS present in this repo, so the
# launcher is ON and its assets/common/{fonts,img} stage into the build tree,
# which package_setup_host.sh requires.
# Setup-host CI requires WIZARD_CMAKE_BLOCK / ENABLE_SETUP_WIZARD (FORCE_SETUP_HOST
# alone does not open first-run / Generate & rebuild).
set(PSX_RECOMP_UI ON CACHE BOOL
    "Shared recomp-ui launcher; recomp-ui submodule is present" FORCE)
# Full netplay UI -- enable after adding recomp-ui + testing:
# if(EXISTS "${PSXRECOMP_ROOT}/lib/recomp-net/CMakeLists.txt")
#     set(PSX_NETPLAY ON CACHE BOOL "..." FORCE)
# endif()
# First-run setup wizard + Generate & rebuild: the owned-input kit IS this
# wizard. Without it the setup exe links no codegen host at all and a player
# can never generate the game (the c2 pilot ZIP shipped exactly that).
set(PSX_SETUP_WIZARD ON CACHE BOOL
    "Advertise first-run setup wizard + Generate & rebuild in recomp-ui" FORCE)

# --- Retail-BIOS-only BIOS stems -------------------------------------------
# Narrow the recompiled-BIOS stems to the retail image this recipe ships a
# profile for (psxrecomp/bios/SCPH1001.toml). Two reasons OpenBIOS cannot be a
# stem here:
#   1. runtime.cmake FATAL_ERRORs with "Bundled OpenBIOS image is missing"
#      whenever "OpenBIOS" is in PSXRECOMP_BIOS_STEMS, so keeping the default
#      forces psxrecomp/bios/openbios.bin into every build.
#   2. The release policy requires the OpenBIOS payload to be absent
#      (.release/policy.json "openbios_required": false, game.toml [runtime]
#      openbios = false), and scripts/package_setup_release.sh removes it.
# The setup host links no BIOS at all; the player's first-run Generate emits
# SCPH1001 from their own dump. Must be set BEFORE include(runtime.cmake),
# which declares this as a CACHE STRING.
set(PSXRECOMP_BIOS_STEMS "SCPH1001" CACHE STRING
    "Recompiled BIOS stems to link; retail-only kit ships no OpenBIOS payload")
set(PSXRECOMP_BIOS_STEM "SCPH1001" CACHE STRING "Accepted BIOS" FORCE)
set(PSXRECOMP_BIOS_PROFILE "${PSXRECOMP_ROOT}/bios/SCPH1001.toml" CACHE FILEPATH "Accepted BIOS" FORCE)

include("${PSXRECOMP_ROOT}/runtime/runtime.cmake")

# This project's own mod catalog. The FRAMEWORK stages it -- together with its
# builtin psx.* packages -- into <exe-dir>/mods/bundled, and refuses to
# configure if a catalog exists here but is not declared. Never copy it by
# hand: five titles did, and framework 4cc04be3's rename of the staged
# directory broke all five silently (bead beads-eio.3.101).
set(PSX_PRELOADED_MODS "${CMAKE_CURRENT_SOURCE_DIR}/mods/preloaded")
if(NOT IS_DIRECTORY "${PSX_PRELOADED_MODS}")
    set(PSX_PRELOADED_MODS "NONE")
endif()

psxrecomp_add_game_runtime(psx-runtime
    PRELOADED_MODS_DIR "${PSX_PRELOADED_MODS}"
    # ENABLE_NETPLAY_IF_PRESENT
    # NETPLAY_LOBBY_URL "ws://netplay.retcomm.net:8765"
    ENABLE_SETUP_WIZARD
    WINDOW_TITLE "Nightmare Creatures II"
    EXE_NAME "Nightmare_Creatures_II"
    DEFAULT_GAME_CONFIG_PATH "game.toml"
    MAX_PLAYERS 1
    APP_ICON "${CMAKE_CURRENT_SOURCE_DIR}/assets/psxrecomp.ico"
    GEN_MARKER "generated/SLUS_011.12_dispatch.c"
    GEN_FULL_GLOB "generated/SLUS_011.12_full_*.c"
    CODEGEN_SETUP_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/codegen_setup.c"
)

# Stage runtime configs next to the exe for direct build/ launches. Only copy
# files that exist — never fail POST_BUILD when game_options.toml is absent.
foreach(_PSX_RUNTIME_CONFIG game.toml game_options.toml)
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${_PSX_RUNTIME_CONFIG}")
        add_custom_command(TARGET psx-runtime POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                "${CMAKE_CURRENT_SOURCE_DIR}/${_PSX_RUNTIME_CONFIG}"
                "$<TARGET_FILE_DIR:psx-runtime>/${_PSX_RUNTIME_CONFIG}"
            COMMENT "Staging ${_PSX_RUNTIME_CONFIG}"
            VERBATIM)
    endif()
endforeach()
