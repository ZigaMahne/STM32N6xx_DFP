
# FSBL_XIP Template for STM32N6 series

Project provides a reference FSBL XIP template that can be used to build any firmware application to execute in external Flash (sub-project Appli).
Subproject ExtMemLoader is a used to generate a binary library capable of downloading an application to external memory.

## Introduction

After bootROM execution, the FSBL project executes in internal RAM, ensures caches and clock setting then configures the external Flash in execution mode. When done, the program counter is set to the application entry point and the application in turn executes.
Once the application runs, the green LED (GPIO PO.01) toggles in an infinite loop with a 0.5 second period.

## Steps to Configure, Build, Load, and Debug using the Basic Template csolution project

> **Note:** 
> **Installed packs and extensions:**
> - Keil.STM32N6xx_DFP.1.0.1-dev3
> - Keil.STM32N6570-DK_BSP.1.0.0-dev
> - Arm CMSIS Debugger 1.3.0
> - Arm CMSIS Solution 1.62.2
> - Cbridge27.exe
> - STM32CubeMX.6.16.1
> - STM32Cube_FW_U0_V1.3.0
> - STM32_SigningTool_CLI_V2.20.0
> - STM32CubeProgrammer_V2.18.0


## In VSCode

### In Activity bar under CMSIS click **Create Solution**

- Select **STM32N6570-DK** Target Board (from BSP) and **CubeMX FSBL_XIP solution** (from DFP) and click **Create**
- Select **AC6** compiler and click **OK**
- Launch STM32CubeMX (CMSIS → Components → Device:CubeMX) and click **Run Configuration Generator**

## In STM32CubeMX

### Select **Secure domain only** TrustZone feature

### Navigate to Project Manager:

- Project tab: Ensure the **FSBL**, **Appli** and **ExtMemLoader** checkbox are selected
- Core Generator tab: Check that **Copy only necessary library files** are selected

### Navigate to Pinout & Configuration:

#### System core:

- CORTEX_M55M_FSBL: Enable CPU ICache and DCache
- GPIO: Select PO1 pin as **GPIO_output**:
  - Pin Context Assignement: **application**
  - GPIO output level: High
  - Add user label: `LED1_green`

#### Connectivity:

- SDMMC2: Unselect **First Stage Boot Loader** (disable this peripheral, because have configuration issue)
- XSPI1: Unselect **First Stage Boot Loader** (disable this peripheral, because have partly conflict with PO1 `LED1_green`)
- XSPI2: Check that **First Stage Boot Loader** and **External Memory Loader** is selected under Runtime contexts and modify following parameter settings:
  - Fifo Treshold: **4**
  - Memory Type: **Macronix**
  - Memory size: **1GBits**

#### Middleware:

- EXTMEM_MANAGER: Select **First Stage Boot Loader** under Runtime contexts and select **Activate External Memory Manager** checkbox
  - Boot usecase:
    - Boot:
	  - Select boot code generation: **Checked**
	  - Selection of the boot system: **Execute In Place**
	- XIP:
	  - select the source memory: **Memory 1**
	  - Application offset: **0x00100000**
  - Memory 1:
    - Memory Instance: XSPI2
    - Number of memory data lines: EXTMEM_LINK_CONFIG_8LINES

- EXTMEM_LOADER: Select **Activate External Memory Loader** and configure following External Memory Loader parameters:
  - Number of sectors: 0x8000 (32768)
  - Sector size: 0x1000 Bytes (4096)

### Navigate to Clock Configuration:

- PLL1 select **/24** divider for "50" To IC3 (MHz)
- XSPI2 Source Mux: IC3; **50** To XSPI2 (MHz)

### Click **GENERATE CODE** and confirm warnings

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
    run: STM32_SigningTool_CLI.exe -bin $input$ -s -nk -of 0x80000000 -t fsbl -o $output$ -hv 2.3
    input:
      - $bin()$
    output:
      - $OutDir()$/FSBL-trusted.bin
```
> **Note:** Note: The OTP configuration for flash source selection is configurable via BOOTROM_CONFIG2 - OTP_WORD11 using **STM32CubeProgrammer**. For more information, please check [UM3234](https://www.st.com/resource/en/user_manual/um3234-how-to-proceed-with-boot-rom-on-stm32n6-mcus-stmicroelectronics.pdf)
---

### STM32CubeMX/STM32N657X0HxQ/FSBL.cgen.yml

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

### Appli/Appli.cproject.yml

- #### Add linker script

```yaml
# Linker script definition
linker:
  - script: ../STM32CubeMX/STM32N657X0HxQ/STM32CubeMX/MDK-ARM/Appli/stm32n657xx_ROMxspi2.sct
    for-compiler: AC6
```

- #### Add post-build command to add header to the generated binary

```yaml
# Post-build commands to add header
executes:
  - execute: Generate_trusted_bin
    run: STM32_SigningTool_CLI.exe -bin $input$ -s -nk -of 0x80000000 -t fsbl -o $output$ -hv 2.3
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
HAL_GPIO_TogglePin(LED1_green_GPIO_Port, LED1_green_Pin);
HAL_Delay(500);
/* USER CODE END 3 */
```

---

### ExtMemLoader/ExtMemLoader.cproject.yml

- #### Modify TrustZone mode from **secure** to **off**

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

- Select **build_ExtMemLoader** Target Set and click **Build solution**
  - **ExtMemLoader** project should successfully build to have configured flash algorithm (check in root folder if ExtMemLoader.axf file appears)

- Continue with select **build_load_FSBL_Appli** Target Set and click **Save** then click **Build solution**
  - FSBL and Appli projects should successfully build to out folder
  - Open the CubeMX.csolution.yml file, **uncomment** the following entries, and **save** the file

```yaml
# - image: $OutDir(FSBL)$/FSBL-trusted.bin
#   load-offset: 0x70000000
#   load: image
# - image: $OutDir(Appli)$/Appli-trusted.bin
#   load-offset: 0x70100000
#   load: image
```

  - In STM32N6570-DK board: Set the boot mode in **development mode** (BOOT1 switch position is 1-3) and reset board
  - To flash device click **Views and More Actions** and click **Load application to target**
  - In STM32N6570-DK board: Set the boot mode in **flash mode** (BOOT1 switch position is 1-2) and reset board
  - Green led should blink as we configured previous (in FSBL/Src/main.c)

- To debug application, select **debug_Appli** Target Set and click **Save**
  - **FLASH MODE:**
    - In STM32N6570-DK board: Set the boot mode in **flash mode** (BOOT1 switch position is 1-2) and reset board
    - Click **Load & Debug application** button and now program  run (todo: should wait in main function to start debug
    - With Continue (F5) button green led should blink in flash mode

todo:
  - **DEVELOPMENT MODE:**
    - In STM32N6570-DK board: Set the boot mode in **development mode** (BOOT1 switch position is 1-3) and reset board
    - Open .vscode\launch.json and add commands in initCommands after **monitor reset halt**:

      ```json
      "load",
      "set $pc = Reset_Handler",
      "set $sp = (int) &Image$$ARM_LIB_STACK$$ZI$$Limit",
      ```

    - Save launch.json
    - Click **Load & Debug application** button and now program run (should wait in main function to start debug)
    - With Continue (F5) button green led should blink in development mode
