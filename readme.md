# Create and debug TouchGFX application with Secure and Non-Secure project setup

Summary:

This tutorial guides you through the process of creating a **TouchGFX** application starting from **STM32CubeMX** with a **STM32H5** device with **TrustZone®** (TZ) activated and **Secure** and **Non-Secure** project structure.

Prerequisites:
- [STM32CubeMX (1)](https://www.st.com/en/development-tools/stm32cubemx.html)
- [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html)
- [STM32H573I-DK](https://www.st.com/en/evaluation-tools/stm32h573i-dk.html) + USB-C cable
- [TouchGFX Framework](https://support.touchgfx.com/docs/introduction/installation) (X-CUBE-TOUCHGFX middleware, TouchGFX Designer installed)
- [STM32CubeProgrammer GUI](https://www.st.com/en/development-tools/stm32cubeprog.html)

Content:  
- Configure the **Option Bytes**
- Create a new project using **STM32CubeMX**
- Open project in the **STM32CubeIDE** and setup **Debug**
- Add TouchGFX SW packgage **X-CUBE-TouchGFX**
- Generate TouchGFX project in the **TouchGFX Designer**
- **GPIOs** settings
- Adjust avaliable **heap** and **stack** size
- **MPCWM** settings in GTZC (Global TZ Controller)
- **GPDMA** settings
- Add needed **source code**

> Note: In this repository you will NOT find a complete working project tested on the **STM32H573I-DK** board, because sharing here complete TouchGFX project is against ST rules. 

The **secure** application is located in **Flash Bank 1** (1MB) and basically performs only an initialization of GTZC or other initialization, then **jumps** to the **non secure** application. The **Non secure** application performs the remaining functionality and is located in **Flash Bank 2** (1MB).
```
     Flash Bank 1               Flash Bank 2
   +---------------+          +---------------+
   |  Secure part  |          |  NonSec part  |
   |               |          |               |
   |               |          | TouchGFX      |
   |               |          | CRC           |
   | init sec attr |          | FMC           |
   | jump to NS    |          | GPIOs         |
   +---------------+          +---------------+
```

***Note:***
>The touch screen handling or external flash memory storage is not implemented in this tutorial to keep the tutorial simple.

>The tutorial does not focus on application memory layout optimization.

## Configure the Option bytes

At first, it is needed to prepare STM32H5 device and configure its ***Option Bytes*** to be able to run secure and non-secure application.

1) Connect the board **to the PC** using **USB-C** cable.
2) Open **STM32CubeProgrammer** GUI.
3) Connect to the STM32H573 by clicking green **Connect** button.
4) Go to **OB** icon (left vertical bar) and activate TZ: ***TZEN*** == B4 and ***Flash Water Mark*** for Flash ***Bank 1*** (Secure) and ***Bank2*** (Non-Secure)*
5) Click blue button **Apply**

(*) *keeping the _STRT value greater than _END value means sector disabled. See the picture bellow and Bank2 configuration.*

![](imgs/programmer.png)

Double-check that the secure watermark area is **enabled** for **Bank1 (0x0 to 0x7f range)** and **disabled** for the **Bank2 (0x1 to 0x0)**. If the secure watermak was enabled also for Bank2 (0x0 to 0x7f) then set back (0x1 to 0x0) and click again blue **Apply** button. The result must match the picture above.

6) Disconnect from the board by clicking green **Disconnect** button.

## Create a new project using STM32CubeMX

1) Open **STM32CubeMX**
2) Start **New project** form MCU and type "STM32H573i" into the searching field and select the parnumber on **STM32H573I-DK** board. Then click blue **Start Project** button.
![](imgs/MX_stm32h573i.png)

3) Click **Yes** for MPU default settings.
![](imgs/ICacheMPUYes.png)

4) Select "**With TrustZone activated**" and click **OK**.  
![](imgs/TZYes.png)

5) In the **Pinout & Configuration** view, configure **PI1** pin as GPIO output push/pull and add **User Label** "GREEN_LED". Assign the pin to the **non-secure** world.
![](imgs/PinMX.png)
![](imgs/PI1.png)

6) Set **HCLK** to maximum frequency **250** MHz. Just put value 250 in HCLK box, confirm by pressing enter and let the **STM32CubeMX** do the job to find the solution.
![](imgs/CLK250MHz.png)

7) Give the project some name and don't forgot to **uncheck** "**Generate Under Root**" (this is required by TouchGFX project which will be added later. TouchGFX generator needs to touch .cproject file and if generated under root, TouchGFX will not find that project file resulting in an error after code generation in TouchGFX Designer). As a toolchain select **STM32CubeIDE**. Click the button **Generate Code**.
![](imgs/UnderRoot.png)

## Open project in the STM32CubeIDE and setup Debug

STM32CubeMX created a hierarchical project with a root project with the given name and two subprojects with the given name and the _Secure and _NonSecure suffixes.

![](imgs/SecNonSec.png)

1) Add the lines of code making LED blinking in **Non-Secure** **main.c**.
```cpp
int main(void)
{
...
  /* USER CODE BEGIN WHILE */
  while (1)
  {
	  HAL_GPIO_TogglePin(GREEN_LED_GPIO_Port, GREEN_LED_Pin);
	  HAL_Delay(250);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```
2) Press **Ctrl + B** to build all, no errors should appear.

### Setup Debug

1) In the **Project Explorer**, click with **right mouse button** on **Secure** project and select **Debug As** > **STM32 C/C++ Application**

2) In the **debug Configuration** click on **Startup** tab and add a **NonSecure image** in **Load Image and Symbols** list. Change **Build Configuration** to **Debug** for NonSecure project. Keep the rest untouched.
![](imgs/DebugSetup.png)

The application boots in a secure state when ***TrustZone®*** is enabled. The debugger sets the ***Program Counter*** using information from the last image in the ***Load image and Symbols*** table. Make sure the **Secure** image is **at the bottom** of the load list.
![](imgs/SecureBottom.png)

3) Click **OK** button to launch the debug session.

You should see the PC at **main.c** in **Secure** project. If you run the application (using **F8** key or **Play** button), the **Secure** code will execute and jump to **NonSecure** application and you should see the **LD6** blinking. This is to verify we have working **Secure** and **NonSecure** application setup and we are able to debug it.

When you run debug next time, be sure to select Secure project when launching debug.

![](imgs/DebugLaunch.png)

## Add TouchGFX SW packgage X-CUBE-TouchGFX

> Note: You can find this step described also at ST Community article [here](https://community.st.com/t5/stm32-mcus/assigning-touchgfx-to-a-nonsecure-trustzone-application-on-an/ta-p/851163)

1) Open again **STM32CubeMX**.

2) Go to **Software Packs** drop down menu and click **Select Components** option.  
![](imgs/OpenSWPacks.png)

3) In the **Software Packs Component Selector** be sure to select **Cortex-M33 non secure** context because the **TouchGFX** can run only in **non secure** context.  
![](imgs/SWPacksTGFX.png)

4) Then we need to activate **TouchGFX** middleware.  
![](imgs/EnableTGFX.png)

5) Solve the **Dependencies error**. The **TouchGFX** needs to have available **CRC** peripheral to proper function. You don't need to configure the **CRC** peripheral, just activate it using check box. **CRC** belongs to **non secure** context.  
![](imgs/ActivateCRC.png)

6) Activate the **FMC** for **non secure** context. The display on **STM32H573I-DK** board is connected with 16-bit data bus. To interface this display (Sitronix ST7789H2 controller) we can use **FMC** peripheral with **LCD mode**. Disable the **Write FIFO**. Configure the **FMC** according to the picture bellow.   
![](imgs/MX_FMC_1.png)

7) Check the **FMC** pin assignment sorted by **Signal name** (move pins to alternate position if needed to be inline with the picture bellow). All the pins, as **FMC** peripheral itself, belongs to the **non secure** context.  
![](imgs/MX_FMC_pins.png)

8) Adjust **TouchGFX** configuration in **STM32CubeMX**. Once we have activated the **FMC** controller, we can adjust configuration in the **STM32CubeMX** for **TouchGFX** middleware to use **FMC**. Set proper display size (**240 x 240 px**) and select **double frame buffer** - don't worry, it will fit inside the internal SRAM (double FB size = 240 x 240 x 2 x 2 = 225 KB)  
![](imgs/CubeMX_TGFX_FMC.png)

If you generate the project by **STM32CubeMX** now and then try to build the project, you will receive **expected errors**, because the project is **not complete**.

![](imgs/Errors.png)

We must generate project in the **TouchGFX Designer** to have complete project.

## Generate TouchGFX project in the TouchGFX Designer

1) Open (double click) the **ApplicationTemplate.touchgfx.part** partial project file located in /NonSecure/TouchGFX/:
![](imgs/part.png)  

***Note:***
> If you open the partial project file using Open project dialog in TouchGFX Designer, the next step (Import GUI) is skipped by TouchGFX Designer and Blank UI is automatically used.

2) This will open the project in the ***TouchGFX Designer***. There you can select ***Blank UI*** for initial project and click on ***Import*** button.    
![](imgs/TouchGFXDesigner-BlankUI.png)

3) Add some basic shapes on the screen (canvas) to see at least something on the display later. You can put a white rectangle (a ***Box*** widget) and spread it across all the screen to create a white background and then place somewhere a **Circle** widget of any color.  
![](imgs/TouchGFXDesigner_canvas.png)

4) Now just click **Generate button** (or press F4) to generate **TouchGFX** project.
![](imgs/TouchGFXDesignergenerate.png)

5) You should see green label at the bottom status bar informing about successfull generation.
![](imgs/TouchGFXDesigner-success.png)


***Note:***
> The **TouchGFX Designer** has also generated final **.touchgfx** project file in the same folder next to the partial **ApplicationTemplate.touchgfx.part** file. Use the **.touchgfx** final project file for opening your TouchGFX project next time.    
![](imgs/TGFXprjfile.png)

At this point we are **able to build** the application **without any error**. If you start debugging, you will see running **TouchGFX** application. But the **TouchGFX** application will not be showing anything on the display and requires some more adjustments. 

Before we proceed further, **remove** or comment out **testing code** in the main.c (**non secure**) which blinks the green LED.

```cpp
...
  // HAL_GPIO_TogglePin(GREEN_LED_GPIO_Port, GREEN_LED_Pin);
  // HAL_Delay(250);
...
```

## GPIOs settings

We need to add some more **GPIOs** to handle display correctly. 

Configure the GPIO pins with the given **User Labels**.

All the GPIO pins belong to **non secure** context. Don't leave these GPIO pins context **Free**.

1) Open again **STM32CubeMX**

2) The pin enabling power to display **PC6** (with label "**LCD_DISP**") - GPIO **output** push/pull. Default **GPIO output level** **Low** means the display will have a power just after initialization of this pin.

3) **Input** pin catching tearing effect (TE) signal from the display **PD3** (put there label "**LCD_TE**"). We need to enable **GPIO_EXTI3** on this pin to catch interrupts from the display TE pin. (**TouchGFX** rendering is triggered by this signal)  
![](imgs/EXTI3.png)   
Then go to **NVIC_NS** settings and enable interrupts on **EXTI line 3** by clicking on the checkbox.
![](imgs/NVIC_EXTI3.png)

4) The pin controlling the display reset line **PH13** (with label "**LCD_RESET**"). GPIO **output** push/pull with **GPIO output level** **High**.

5) The pin controlling the backlight **PI3** (with label "**LCD_BL_CTRL**"). GPIO **output** push/pull with **GPIO output level** - **High** which means that after the pin initialization the display backlight will be on.

See the summary bellow:
![](imgs/GPIOs.png)

## Adjust avaliable heap and stack size

TouchGFX application would require more RAM memory than the default values for the heap and stack size. **Enlarge stack and heap** size for the **non secure** application in STM32CubeMX **Project Manager**.

```
M33NS
     Minimum Heap Size  = 0x800
     Minimum Stack Size = 0x1000
```

![](imgs/MX_heapStack.png)

***Note:***
> Be aware that if you modify the values manually in the linker file then they will be re-generated (reverted) to default values (or better to say, to the actual values set in STM32CubeMX) when you re-generate the project next time.

## MPCWM settings in GTZC (Global TZ Controller)

By default all external memories address ranges are allocated for **secure** world. The **FMC Bank1** address space will not be working until we enable the access for **non secure** application. To enable it we need to configure **MPCWM** (Memory Protection Controller - Watermark Based) in STM32CubeMX under **GTZC_S** section.

We need to set just the very beginning of the **FMC bank 1** address space to **not secure**. The minimum (not zero) size is **0x20000** (corresponds to 128 KB), let's use this value for **MPCWM2 (FMC_NOR)** and **Area 1**. Check the picture bellow.

![](imgs/MPCWM.png)

## GPDMA settings

**GPDMA2** is used to offload the CPU when the FB is transfered to the GRAM of the display through the **FMC** interface. Just select the **GPDMA2 channel 6**. It is enough to activate it. All configuration is completed in the code that is added later. And one more thing: You need to change the **GPDMA channel 6** security attribute to "**Priviledged**"...

![](imgs/GPDMA_priviledged.png)

... and enable **NVIC** interrupt for **GPDMA2 CH6**.

![](imgs/NVIC_GPDMA2CH6.png)

**Generate the project** again in STM32CubeMX.

## Add needed source code

Finally we must add a source code in the **TouchGFXHAL.cpp**
```
 c:\....\[your prj name]\NonSecure\TouchGFX\target\TouchGFXHAL.cpp
 ```
 ![](imgs/TouchGFXHALcpp.png)

 To make things easier here, replace all the content of your **TouchGFXHAL.cpp** with the content from this repository:

 [TouchGFXHAL.cpp](NonSecure/TouchGFX/target/TouchGFXHAL.cpp)

Build the application (Ctrl + B) in CubeIDE and launch debug or flash the application.

**Reminder**: *Don't forget to select firstly **Secure project** before clicking bug icon to launch debug session.* [see here](README.md#setup-debug)

---
### Now you should see finally your screen layout on the display!

---

***Note:***
> By default, STM32CubeMX allocates a large portion of SRAM for the **secure** application, which is unnecessary. The remaining available SRAM is allocated to the **non-secure** application. You can reduce the allocated SRAM for the **secure** application, allowing more SRAM to be allocated to the **non-secure** application. Adjust the linker file and the **Block-based memory protection controller** tab in STM32CubeMX under the **GTZC_S** section.

---

Related [ST Community article](https://community.st.com/stm32-mcus-60/create-and-debug-a-touchgfx-application-with-a-secure-and-nonsecure-project-setup-167084) which directs here.