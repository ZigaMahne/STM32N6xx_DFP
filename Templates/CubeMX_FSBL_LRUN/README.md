
# FSBL_LRUN Template for STM32N6 series

Project provides a reference FSBL LRUN template that can be used to build any firmware application to execute in internal RAM (sub-project Appli).
The ExtMemLoader subproject is a flash algorithm that generates a binary library capable of programming an application into external memory.

## Introduction

The bootROM copies FSBL image from external Flash (Octo SPI Flash Memory) into internal RAM (AXI SRAM2) and begins execution to initialize the caches and configure the clocks. Once this is complete, the application binary is copied from external flash into internal RAM (AXI SRAM2). After the copy operation finishes, the application begins execution.

## Steps to Configure, Build, Load and Debug using the Basic Template csolution project

> **Note:**
>
>- **Required Packs:**
>   - Keil.STM32N6xx_DFP.1.1.0
>   - Keil.STM32N6570-DK_BSP.1.0.0
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

- Select **STM32N6570-DK** Target Board (from BSP) and **CubeMX FSBL_LRUN solution** (from DFP) and click **Create**
- Select **AC6** compiler and click **OK**
- Launch STM32CubeMX (CMSIS view → expand Project Target → expand and locate Device:CubeMX component) and click **Run Configuration Generator**

## In STM32CubeMX

### Select **Secure domain only** TrustZone feature

### Navigate to Project Manager

- Project tab: Ensure the **FSBL**, **Appli** and **ExtMemLoader** checkboxes are selected
- Code Generator tab: Check that **Copy only necessary library files** are selected

### Navigate to Pinout & Configuration

#### System core

- CORTEX_M55M_FSBL: Enable **CPU ICache** and **CPU DCache**
- GPIO: Select PO1 pin as **GPIO_output** and configure PO1 configuration:
  - Pin Context Assignement: **Application**
  - GPIO output level: Low
  - Add user label: `LD1_green`

#### Connectivity

- SDMMC2: Unselect **First Stage Boot Loader** checkbox to disable this peripheral (To avoid configuration issues)
- XSPI1: Unselect **First Stage Boot Loader** checkbox to disable this peripheral (To avoid configuration issues with PO1 `LD1_green`)
- XSPI2: Check that **First Stage Boot Loader** and **External Memory Loader** are selected under Runtime contexts and modify following parameter settings:
  - Fifo Treshold: **4**
  - Memory Type: **Macronix**
  - Memory size: **1GBits**

#### Middleware

- EXTMEM_MANAGER: Select **First Stage Boot Loader** under Runtime contexts and select **Activate External Memory Manager** checkbox and configure following
  - Boot usecase:
    - Boot:
      - Select boot code generation: **Checked**
      - Selection of the boot system: **Load and Run**
    - LRUN source:
      - select the source memory: **Memory 1**
      - source address offset: **0x00100000**
      - source code size: **0x00010000**
    - LRUN destination:
      - selection of the memory: **Internal Memory**
      - destination address: **0x34000000**	  
  - Memory 1:
    - Memory Instance: **XSPI2**
    - Number of memory data lines: **EXTMEM_LINK_CONFIG_8LINES**

- EXTMEM_LOADER: Select **Activate External Memory Loader** checkbox and configure following **External Memory Loader** parameters:
  - Number of sectors: **0x8000** (32768)
  - Sector size: **0x1000** Bytes (4096)

### Navigate to Clock Configuration

- Under **IC3 Clock Source** PLL section configure **IC3 Div** to **/24** to have **IC3 Clock** **50 MHz**

   ![STM32CubeMX IC3 CLOCK CONFIGURATION](ic3_clock_conf.png)
- Under **XSPI2 Source Mux** select IC3 to have  **XSPI2 Clock** **50 MHz**

   ![STM32CubeMX XSPI2 SOURCE CONFIGURATION](xspi2_source_mux_conf.png)

### Click **GENERATE CODE** if you get warnings click Yes to generate code

## In VSCode: FSBL and ExtMemLoader modifications

### FSBL/FSBL.cproject.yml

- #### Add linker script

```yaml
  # Linker script definition
  linker:
    - script: ../STM32CubeMX/STM32N657X0HxQ/STM32CubeMX/MDK-ARM/FSBL/stm32n657xx_axisram2_fsbl.sct
      for-compiler: AC6
```

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

> **Note:** The OTP configuration for flash source selection is configurable via fuses in BOOTROM_CONFIG_2[8:5], OTP_WORD11 using **STM32CubeProgrammer**. Requires the **default** boot configuration to have sNOR device (connected to XSPIM_P2) attached boot. For more information, please check [UM3234](https://www.st.com/resource/en/user_manual/um3234-how-to-proceed-with-boot-rom-on-stm32n6-mcus-stmicroelectronics.pdf)
---

### STM32CubeMX/STM32N657X0HxQ/FSBL.cgen.yml

- #### Comment following redundant files

```yaml
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/core/memory_wrapper.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/core/systick_management.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/MDK-ARM/FlashDev.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/MDK-ARM/FlashPrg.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/STM32Cube/stm32_device_info.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/STM32Cube/stm32_loader_api.c
```

---

### Appli/Appli.cproject.yml

- #### Add linker script

```yaml
  # Linker script definition
  linker:
    - script: ../STM32CubeMX/STM32N657X0HxQ/STM32CubeMX/MDK-ARM/Appli/stm32n657xx_LRUN.sct
      for-compiler: AC6
```

- #### Add post-build command to add header to the generated binary

```yaml
  # Post-build commands to add header
  executes:
    - execute: Generate_trusted_bin
      run: $ENV{STM32_PRG_PATH}/STM32_SigningTool_CLI.exe -bin $input$ -s -nk -of 0x80000000 -align -t fsbl -o $output$ -hv 2.3

      input:
        - $bin()$
      output:
        - $OutDir()$/Appli-trusted.bin
```

---

### STM32CubeMX/STM32N657X0HxQ/Appli.cgen.yml

- #### Comment following redundant files (temporarily issue with cmsis toolbox extension)

```yaml
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/core/memory_wrapper.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/core/systick_management.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/MDK-ARM/FlashDev.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/MDK-ARM/FlashPrg.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/STM32Cube/stm32_device_info.c
  # - file: ./STM32CubeMX/Middlewares/ST/STM32_ExtMem_Loader/STM32Cube/stm32_loader_api.c
```

---

### STM32CubeMX/STM32N657X0HxQ/STM32CubeMX/Appli/Src/main.c

- #### Add following line to toggle green led

```c
    /* USER CODE BEGIN 3 */
    HAL_GPIO_TogglePin(LD1_green_GPIO_Port, LD1_green_Pin);
    HAL_Delay(500);
   }
    /* USER CODE END 3 */
```

---

### ExtMemLoader/ExtMemLoader.cproject.yml

- #### Modify TrustZone mode from secure to "off"

- #### Add linker script

```yaml
  # Linker script definition
  linker:
    - script: ../STM32CubeMX/STM32N657X0HxQ/STM32CubeMX/MDK-ARM/ExtMemLoader/stm32n6xx_extmemloader_mdkarm.sct
      for-compiler: AC6
```

- #### Add post build commands to move .axf to root

```yaml
  # Post-build commands
  executes:
    - execute: Copy_ExtMemLoader_to_root
      run: ${CMAKE_COMMAND} -E copy $input$ $output$
      input:
        - ../out/ExtMemLoader/STM32N657X0HxQ/Debug/ExtMemLoader.axf
      output:
        - ../ExtMemLoader.axf
```

---

### STM32CubeMX/STM32N657X0HxQ/ExtMemLoader.cgen.yml

- #### Comment following redundant file

```yaml
  # - file: ./STM32CubeMX/MDK-ARM/startup_stm32n657xx_fsbl.c
```

---

### In Activity bar under CMSIS click **Manage Solution Settings**

- Select **ExtMemLoader** Target Set and click **Build solution**
  - **ExtMemLoader** project should successfully build to have configured flash algorithm (check in root folder if ExtMemLoader.axf file appears)

- Continue with select **FSBL_Appli** Target Set
- Ensure **ST-Link@pyOCD** Debug Adapter is selected and **Update launch.json and tasks.json** checkbox is selected and click **Save** then click **Build solution**
  - FSBL and Appli projects should successfully build into out folder

  - On STM32N6570-DK board: Set the boot mode in **development mode** (BOOT1 switch position is 1-3) and reset board
  - To flash device click **Views and More Actions** and click **Load application to target**
  - On STM32N6570-DK board: Set the boot mode in **flash mode** (BOOT1 switch position is 1-2) and reset board
  - Green led should blink as we configured previous (in Appli/Src/main.c)

- To debug application in:
  - **FLASH MODE:**
    - On STM32N6570-DK board: Set the boot mode in **flash mode** (BOOT1 switch position is 1-2) and reset board

      > **Note:** To flash an unprogrammed (virgin) device, ensure that the board is in development mode.

    - Open .vscode\launch.json file and add following modification to the configuration named "STLink@pyOCD (launch)" under **initCommands** and **customResetCommands** commands:
      - Modify the command name from **tbreak main** to **thbreak main**
    - Click **Load & Debug application** button and now program should wait in main function to start debug
    - With Continue (F5) button green led should blink in flash mode

  - **DEVELOPMENT MODE:**
    - On STM32N6570-DK board: Set the boot mode in **development mode** (BOOT1 switch position is 1-3) and reset board
    - Open .vscode\launch.json file and add following modification to the configuration named "STLink@pyOCD (launch)"
      - Comment line

      ```json
      // "preLaunchTask": "CMSIS Load",
      ```

      - add commands into initCommands

      ```json
      "initCommands": [
          "monitor reset halt",
          "load out/Appli/STM32N657X0HxQ/Debug/Appli.hex",
          "set $pc = Reset_Handler",
          "set $sp = (int) &Image$$ARM_LIB_STACK$$ZI$$Limit",
          "thbreak main"
      ```

      - add commands into customResetCommands

      ```json
      "customResetCommands": [
          "monitor reset halt",
          "maintenance flush register-cache",
          "maintenance flush dcache",
          "load out/Appli/STM32N657X0HxQ/Debug/Appli.hex",
          "set $pc = Reset_Handler",
          "set $sp = (int) &Image$$ARM_LIB_STACK$$ZI$$Limit",
          "thbreak main",
          "continue"
      ```

    - Save launch.json
    - Click **Load & Debug application** button and now program should wait in main function to start debug
    - With Continue (F5) button green led should blink in development mode
