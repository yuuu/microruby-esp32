# Create the include directory in advance
file(MAKE_DIRECTORY ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/lib/mbedtls/include)
file(MAKE_DIRECTORY ${COMPONENT_DIR}/picoruby/build/esp32-microruby/include)

idf_component_register(
  SRCS
    ${COMPONENT_DIR}/microruby-esp32.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-esp32/ports/esp32.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-machine/ports/esp32/machine.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-machine/ports/common/machine.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-io-console/ports/esp32/io-console.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-filesystem-fat/ports/esp32/flash_disk.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-env/ports/esp32/env.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-gpio/ports/esp32/gpio.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-adc/ports/esp32/adc.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-spi/ports/esp32/spi.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-rng/ports/esp32/rng.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-pwm/ports/esp32/pwm.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-rmt/ports/esp32/rmt.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-watchdog/ports/esp32/watchdog.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/ports/esp32/timing_alt.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/ports/common/cipher.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/ports/common/cmac.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/ports/common/digest.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/ports/common/hmac.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/ports/common/pkey.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/ports/common/md.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-socket/ports/esp32/net_helpers.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-socket/ports/esp32/ssl_socket.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-socket/ports/esp32/tcp_server.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-socket/ports/esp32/tcp_socket.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-socket/ports/esp32/udp_socket.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-i2c/ports/esp32/i2c.c
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-uart/ports/esp32/uart.c
  INCLUDE_DIRS
    ${COMPONENT_DIR}
    ${COMPONENT_DIR}/picoruby/include
    ${COMPONENT_DIR}/picoruby/build/esp32-microruby/include
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mruby/lib/mruby/include
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-machine/include
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/include
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-mbedtls/lib/mbedtls/include
    ${COMPONENT_DIR}/picoruby/mrbgems/picoruby-filesystem-fat/ports/esp32
    ${COMPONENT_DIR}/picoruby/mrbgems/mruby-compiler2/include
    ${COMPONENT_DIR}/picoruby/mrbgems/mruby-compiler2/lib/prism/include
  PRIV_REQUIRES
    esp_driver_gpio
    esp_driver_uart
    esp_driver_spi
    esp_driver_ledc
    esp_driver_rmt
    esp_driver_i2c
    esp_adc
    esp_timer
    esp_hw_support
    spi_flash
    nvs_flash
    esp_wifi
)

add_definitions(
  -DMRB_TICK_UNIT=10
  -DMRB_TIMESLICE_TICK_COUNT=1
  -DMRB_INT64=1
  -DMRBC_CONVERT_CRLF=1
  -DUSE_FAT_FLASH_DISK
  -DNDEBUG
  -DESP32_PLATFORM

  # for picoruby.h
  -DPICORB_VM_MRUBY
)

set(PICORUBY_DIR ${COMPONENT_DIR}/picoruby)
set(LIBMRUBY_FILE ${PICORUBY_DIR}/build/esp32-microruby/lib/libmruby.a)

# Build PicoRuby
add_custom_command(
  OUTPUT ${LIBMRUBY_FILE}
  COMMAND ${CMAKE_COMMAND} -E echo "MRUBY_CONFIG=${IDF_TARGET_ARCH}-esp-microruby rake"
  COMMAND ${CMAKE_COMMAND} -E env MRUBY_CONFIG=${IDF_TARGET_ARCH}-esp-microruby rake
  WORKING_DIRECTORY ${PICORUBY_DIR}
  COMMENT "PicoRuby build"
  VERBATIM
)

add_prebuilt_library(
  libmruby ${LIBMRUBY_FILE}
  REQUIRES ${COMPONENT_NAME}
)
target_link_libraries(${COMPONENT_LIB} PRIVATE libmruby)
target_include_directories(
  ${COMPONENT_LIB}
  PRIVATE
    ${COMPONENT_DIR}/picoruby/mrbgems/mruby-compiler2/include
    ${COMPONENT_DIR}/picoruby/mrbgems/mruby-compiler2/lib/prism/include
    ${COMPONENT_DIR}/picoruby/build/esp32/mrbgems
)

add_custom_target(
  microruby DEPENDS ${LIBMRUBY_FILE}
  DEPENDS ${LIBMRUBY_FILE}
)
add_dependencies(${COMPONENT_LIB} microruby)

# Compile Ruby files
set(RUBY_FILES main_task)
set(PICORBC ${COMPONENT_DIR}/picoruby/bin/picorbc)
set(GENERATED_C_FILES "")

foreach(rb ${RUBY_FILES})
  set(C_FILE ${COMPONENT_DIR}/mrb/${rb}.c)
  set(RB_FILE ${COMPONENT_DIR}/mrblib/${rb}.rb)
  add_custom_command(
    OUTPUT ${C_FILE}
    COMMAND ${CMAKE_COMMAND} -E echo "Compiling ${RB_FILE}..."
    COMMAND ${CMAKE_COMMAND} -E make_directory ${COMPONENT_DIR}/mrb
    COMMAND ${PICORBC} -B${rb} -o${C_FILE} ${RB_FILE}
    DEPENDS microruby ${RB_FILE}
    COMMENT "Compiling ${RB_FILE}"
    VERBATIM
  )
  list(APPEND GENERATED_C_FILES ${C_FILE})
endforeach(rb)

target_sources(${COMPONENT_LIB} PRIVATE ${GENERATED_C_FILES})
