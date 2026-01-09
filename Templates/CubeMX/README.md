
# Basic Template for STM32N6 series

Project provides a reference basic template that can be used to build 512KB firmware application to execute in internal RAM (Application as part of the FSBL).
The ExtMemLoader subproject is a flash algorithm that generates a binary library capable of programming an application into external memory.

## Introduction

The bootROM copies FSBL image (512KB) from external Flash (Octo SPI Flash Memory) to the internal RAM (AXI SRAM2) and starts executing it.
> For board STM32N6570-DK: In the application, once the clock is configured, the `LD1_green` LED (GPIO PO.01) blinks in an infinite loop with a 0.5 second period.

## Steps to Configure, Build, Load and Debug using the Basic Template csolution project

> **Note:**
>
>- **Required Packs:**
>   - [Keil.STM32N6xx_DFP](https://github.com/Open-CMSIS-Pack/STM32N6xx_DFP)
>   - Board specific pack
>     > For board STM32N6570-DK: [Keil.STM32N6570-DK_BSP](https://github.com/Open-CMSIS-Pack/STM32N6570-DK_BSP)
>- **Required CMSIS Tools and Extensions:**
>   - Arm CMSIS Solution 1.64.1 (todo:check latest cbridge.exe)
>   - Arm CMSIS Debugger 1.3.0
>- **Required ST tools and Firmware Package:**
>   - [STM32CubeMX 6.16.1](https://www.st.com/en/development-tools/stm32cubemx.html)
>     - [STM32Cube_FW_N6 1.3.0](https://www.st.com/en/embedded-software/stm32cuben6.html)
>   - [STM32CubeProgrammer 2.21.0](https://www.st.com/en/development-tools/stm32cubeprog.html)
>     - STM32_SigningTool_CLI: Ensure the environment variable `STM32_PRG_PATH` points to the folder that contains `STM32_SigningTool_CLI.exe`

## In VSCode

### In Activity bar under CMSIS view click **Create Solution**

- Select Target (Device/Board) and **CubeMX Basic solution** (from DFP) and click **Create**
  > For board STM32N6570-DK: Select STM32N6570-DK Target Board
- Select **AC6** compiler and click **OK**
- Launch STM32CubeMX (CMSIS view → expand Project Target → expand and locate Device:CubeMX component) and click **Run Configuration Generator**

## In STM32CubeMX

Configure Target (Device/Board) in STM32CubeMX

> STM32CubeMX configuration for [STM32N6570-DK board](https://github.com/ZigaMahne/STM32N6570-DK_BSP/blob/main/Documents/Basic_STM32CubeMX.md)

## In VSCode: FSBL and ExtMemLoader modifications

### FSBL/FSBL.cproject.yml for Target

- #### Add linker script
```yaml
  # Linker script definition
  linker:
    - script: ../STM32CubeMX/Target/STM32CubeMX/MDK-ARM/FSBL/stm32n6xxxx_axisram2_fsbl.sct
      for-compiler: AC6
```

  > For board STM32N6570-DK:
  > ```yaml
  >   # Linker script definition
  >   linker:
  >     - script: ../STM32CubeMX/STM32N657X0HxQ/STM32CubeMX/MDK-ARM/FSBL/stm32n657xx_axisram2_fsbl.sct
  >       for-compiler: AC6
  > ```


- #### Add post-build command to add header to the generated binary

```yaml
  # Post-build commands to add header
  executes:
    - execute: Generate_trusted_bin
      run: $ENV{STM32_PRG_PATH}/STM32_SigningTool_CLI.exe -bin $input$ -s -nk -of 0x80000000 -align -t fsbl -o $output$ -hv 2.3
      input:
        - $bin()$
      output:
        - $OutDir()$/FSBL-trusted.bin
```

> **Note:** The OTP configuration for flash source selection is configurable via fuses in BOOTROM_CONFIG_2[8:5], OTP_WORD11 using **STM32CubeProgrammer**. Requires the **default** boot configuration to have sNOR device attached boot. For more information, please check [UM3234](https://www.st.com/resource/en/user_manual/um3234-how-to-proceed-with-boot-rom-on-stm32n6-mcus-stmicroelectronics.pdf)

---

### STM32CubeMX/Target/FSBL.cgen.yml for Target

  > For board STM32N6570-DK:
  > ### STM32CubeMX/STM32N657X0HxQ/FSBL.cgen.yml

- #### Comment redundant files

```yaml
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Manager/stm32_extmem.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Manager/sal/stm32_sal_xspi.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Manager/sal/stm32_sal_sd.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Manager/nor_sfdp/stm32_sfdp_data.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Manager/nor_sfdp/stm32_sfdp_driver.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Manager/psram/stm32_psram_driver.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Manager/sdcard/stm32_sdcard_driver.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Manager/user/stm32_user_driver.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/core/memory_wrapper.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/core/systick_management.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/MDK-ARM/FlashDev.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/MDK-ARM/FlashPrg.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/STM32Cube/stm32_device_info.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/STM32Cube/stm32_loader_api.c
```

---

### STM32CubeMX/Target/STM32CubeMX/FSBL/Src/main.c for Target

- #### Add line to toggle green led

  > For board STM32N6570-DK:
  > ### STM32CubeMX/STM32N657X0HxQ/STM32CubeMX/FSBL/Src/main.c
  > ```c
  >     /* USER CODE BEGIN 3 */
  >     HAL_GPIO_TogglePin(LD1_green_GPIO_Port, LD1_green_Pin);
  >     HAL_Delay(500);
  >    }
  >     /* USER CODE END 3 */
  > ```

---

### ExtMemLoader/ExtMemLoader.cproject.yml for Target

- #### Modify TrustZone mode from secure to "off"

- #### Add linker script
```yaml
  # Linker script definition
  linker:
    - script: ../STM32CubeMX/Target/STM32CubeMX/MDK-ARM/ExtMemLoader/stm32n6xx_extmemloader_mdkarm.sct
      for-compiler: AC6
```

  > For board STM32N6570-DK:
  > ```yaml
  >   # Linker script definition
  >   linker:
  >     - script: ../STM32CubeMX/STM32N657X0HxQ/STM32CubeMX/MDK-ARM/ExtMemLoader/stm32n6xx_extmemloader_mdkarm.sct
  >       for-compiler: AC6
  > ```

- #### Add post build commands to move .axf to root for Target
```yaml
  # Post-build commands
  executes:
    - execute: Copy_ExtMemLoader_to_root
      run: ${CMAKE_COMMAND} -E copy $input$ $output$
      input:
        - ../out/ExtMemLoader/Target/Debug/ExtMemLoader.axf
      output:
        - ../ExtMemLoader.axf
```

  > For board STM32N6570-DK:
  > ```yaml
  >   # Post-build commands
  >   executes:
  >     - execute: Copy_ExtMemLoader_to_root
  >       run: ${CMAKE_COMMAND} -E copy $input$ $output$
  >       input:
  >         - ../out/ExtMemLoader/STM32N657X0HxQ/Debug/ExtMemLoader.axf
  >       output:
  >         - ../ExtMemLoader.axf
  > ```

---

### STM32CubeMX/Target/ExtMemLoader.cgen.yml for Target

- #### Comment redundant file

  > For board STM32N6570-DK:
  > ### STM32CubeMX/STM32N657X0HxQ/ExtMemLoader.cgen.yml
  > ```yaml
  >   # - file: ./STM32CubeMX/MDK-ARM/startup_stm32n657xx_fsbl.c
  > ```

---

### In Activity bar under CMSIS click **Manage Solution Settings**

- Select **ExtMemLoader** Target Set and click **Build solution**
  - **ExtMemLoader** project should successfully build to have configured flash algorithm (check in root folder if ExtMemLoader.axf file appears)

- Continue with select **FSBL** Target Set
- Ensure **ST-Link@pyOCD** Debug Adapter is selected and **Update launch.json and tasks.json** checkbox is selected and click **Save** then click **Build solution**
  - FSBL project should successfully build into out folder
  - Set the boot mode configuration in **development mode** and reset board
    > For board STM32N6570-DK: BOOT1 switch position to 1-3 to set development mode
  - To flash Target click **Views and More Actions** and click **Load application to target**
  - Set the boot mode configuration in **flash mode** and reset board
    > For board STM32N6570-DK: BOOT1 switch position to 1-2 to set flash mode
  - Configured pin should toggle (in FSBL/Src/main.c)
    > For board STM32N6570-DK: `LD1_green` (GPIO PO.01) should blink

- To debug application in:
  - **FLASH MODE:**
    - Set the boot mode configuration in **flash mode** and reset board
      > For board STM32N6570-DK: BOOT1 switch position to 1-2 to set flash mode

    > **Note:** To flash an unprogrammed (virgin) Target, ensure that the board is in development mode.

    - Open .vscode\launch.json file and add following modification to the configuration named "STLink@pyOCD (launch)" under **initCommands** and **customResetCommands** commands:
      - Modify the command name from **tbreak main** to **thbreak main**
    - Click **Load & Debug application** button and now program should wait in main function to start debug
    - With Continue (F5) button green led should blink in flash mode

  - **DEVELOPMENT MODE:**
    - Set the boot mode configuration in **development mode** and reset board
      > For board STM32N6570-DK: BOOT1 switch position to 1-3 to set development mode
    - Open .vscode\launch.json file and add following modification to the configuration named "STLink@pyOCD (launch)"
      - Comment line

      ```json
      // "preLaunchTask": "CMSIS Load",
      ```

      - add commands into initCommands

      ```json
      "initCommands": [
          "monitor reset halt",
          "load out/FSBL/Target/Debug/FSBL.hex",
          "set $pc = Reset_Handler",
          "set $sp = (int) &Image$$ARM_LIB_STACK$$ZI$$Limit",
          "thbreak main"
      ```

         > For board STM32N6570-DK:
         >   ```json
         >   "initCommands": [
         >       "monitor reset halt",
         >       "load out/FSBL/STM32N657X0HxQ/Debug/FSBL.hex",
         >       "set $pc = Reset_Handler",
         >       "set $sp = (int) &Image$$ARM_LIB_STACK$$ZI$$Limit",
         >       "thbreak main"
         >   ```

      - add commands into customResetCommands

      ```json
      "customResetCommands": [
          "monitor reset halt",
          "maintenance flush register-cache",
          "maintenance flush dcache",
          "load out/FSBL/Target/Debug/FSBL.hex",
          "set $pc = Reset_Handler",
          "set $sp = (int) &Image$$ARM_LIB_STACK$$ZI$$Limit",
          "thbreak main",
          "continue"
      ```

         > For board STM32N6570-DK:
         >   ```json
         >   "customResetCommands": [
         >       "monitor reset halt",
         >       "maintenance flush register-cache",
         >       "maintenance flush dcache",
         >       "load out/FSBL/STM32N657X0HxQ/Debug/FSBL.hex",
         >       "set $pc = Reset_Handler",
         >       "set $sp = (int) &Image$$ARM_LIB_STACK$$ZI$$Limit",
         >       "thbreak main",
         >       "continue"
         >   ```

    - Save launch.json
    - Click **Load & Debug application** button and now program should wait in main function to start debug
    - With Continue (F5) button configured pin should toggle in development mode
      > For board STM32N6570-DK: `LD1_green` (GPIO PO.01) should blink
