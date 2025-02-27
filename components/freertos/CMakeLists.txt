if(BOOTLOADER_BUILD)
    # bootloader only needs FreeRTOS for config, not for anything else
    idf_component_register()
    return()
endif()

idf_build_get_property(target IDF_TARGET)

if(CONFIG_IDF_TARGET_ARCH_XTENSA)
    set(srcs
        "FreeRTOS-Kernel/portable/xtensa/port.c"
        "FreeRTOS-Kernel/portable/xtensa/portasm.S"
        "FreeRTOS-Kernel/portable/xtensa/xtensa_context.S"
        "FreeRTOS-Kernel/portable/xtensa/xtensa_init.c"
        "FreeRTOS-Kernel/portable/xtensa/xtensa_overlay_os_hook.c"
        "FreeRTOS-Kernel/portable/xtensa/xtensa_vector_defaults.S"
        "FreeRTOS-Kernel/portable/xtensa/xtensa_vectors.S")

    set(include_dirs
        FreeRTOS-Kernel/include
        esp_additions/include/freertos              # For files with #include "FreeRTOSConfig.h"
        FreeRTOS-Kernel/portable/xtensa/include     # For arch-specific FreeRTOSConfig_arch.h in portable/<arch>/include
        esp_additions/include)                      # For files with #include "freertos/FreeRTOSConfig.h"

    set(private_include_dirs
        FreeRTOS-Kernel/portable/xtensa/include/freertos
        FreeRTOS-Kernel/portable/xtensa
        FreeRTOS-Kernel/portable/priv_include
        .)

elseif(CONFIG_IDF_TARGET_ARCH_RISCV)
    set(srcs
        "FreeRTOS-Kernel/portable/riscv/port.c"
        "FreeRTOS-Kernel/portable/riscv/portasm.S")

    set(include_dirs
        FreeRTOS-Kernel/include
        esp_additions/include/freertos              # For files with #include "FreeRTOSConfig.h"
        FreeRTOS-Kernel/portable/riscv/include      # For arch-specific FreeRTOSConfig_arch.h in portable/<arch>/include
        esp_additions/include)                      # For files with #include "freertos/FreeRTOSConfig.h"

    set(private_include_dirs
        FreeRTOS-Kernel/portable/riscv/include/freertos
        FreeRTOS-Kernel/portable/riscv
        FreeRTOS-Kernel/portable/priv_include
        .)

endif()

list(APPEND srcs
    "esp_additions/task_snapshot.c"
    "FreeRTOS-Kernel/portable/port_common.c"
    "FreeRTOS-Kernel/portable/port_systick.c"
    "FreeRTOS-Kernel/croutine.c"
    "FreeRTOS-Kernel/event_groups.c"
    "FreeRTOS-Kernel/list.c"
    "FreeRTOS-Kernel/queue.c"
    "FreeRTOS-Kernel/tasks.c"
    "FreeRTOS-Kernel/timers.c"
    "FreeRTOS-Kernel/stream_buffer.c"
    "FreeRTOS-openocd.c"
    "esp_additions/freertos_v8_compat.c")

list(APPEND private_include_dirs
    "FreeRTOS-Kernel/include/freertos")

if(CONFIG_ESP32_IRAM_AS_8BIT_ACCESSIBLE_MEMORY)
    list(APPEND srcs "FreeRTOS-Kernel/portable/xtensa/xtensa_loadstore_handler.S")
endif()

# esp_timer is required by FreeRTOS when we use esp_timer_get_time() to do profiling
# [refactor-todo]: make this an optional requirement depending on CONFIG_FREERTOS_RUN_TIME_STATS_USING_ESP_TIMER
# Note, many other components implicitly include esp_timer.h via freertos portmacro.h and some
# components implicitly depend on esp_timer via freertos.
set(required_components esp_timer)

idf_component_register(SRCS "${srcs}"
                    INCLUDE_DIRS ${include_dirs}
                    PRIV_INCLUDE_DIRS  ${private_include_dirs}
                    LDFRAGMENTS linker.lf
                    REQUIRES ${required_components}
                    PRIV_REQUIRES soc esp_pm)

idf_component_get_property(COMPONENT_DIR freertos COMPONENT_DIR)
idf_component_set_property(freertos ORIG_INCLUDE_PATH "${COMPONENT_DIR}/include/freertos/")

if(CONFIG_FREERTOS_DEBUG_OCDAWARE)
    target_link_libraries(${COMPONENT_LIB} INTERFACE "-Wl,--undefined=uxTopUsedPriority")
endif()

set_source_files_properties(
    tasks.c
    event_groups.c
    timers.c
    queue.c
    stream_buffer.c
    PROPERTIES COMPILE_DEFINITIONS
    _ESP_FREERTOS_INTERNAL
    )

# The freertos component provides the `start_app` and `start_app_other_cores`
# if it is included in the build. It then calls `app_main`
# from the main task created, which must be provided by the user.
# Like for `start_app` and `start_app_other_cores`,
# we can't establish dependency on what we don't yet know, so we force the
# linker to not drop this symbol.
target_link_libraries(${COMPONENT_LIB} INTERFACE "-u app_main")

if(CONFIG_APPTRACE_SV_ENABLE)
    # FreeRTOS headers have a dependency on app_trace when SystemView tracing is enabled
    idf_component_optional_requires(PUBLIC app_trace)
elseif(CONFIG_APPTRACE_ENABLE)
    # [refactor-todo]: port.c has a dependency on esp_apptrace_init, for running it on APP CPU.
    # this should be resolved when link-time registration of startup functions is added.
    idf_component_optional_requires(PRIVATE app_trace)
endif()
