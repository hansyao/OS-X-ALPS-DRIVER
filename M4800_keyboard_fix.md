# Dell Precision M4800 VoodoPS2Controller 键盘/触控驱动修复过程

## 前言
对于**Dell Precision M4800**来说，触控板是**ALPS**，如果直接使用2017年2月发布的 [DrHurt](https://github.com/DrHurt) 版本的[VoodooPS2Controller](https://github.com/DrHurt/OS-X-ALPS-DRIVER)有以下bug:
1. 数字锁定键<kbd>Num Lock</kbd>和LED指示灯，及功能切换不工作
2. 亮度调节键被映射到<kbd>Fn</kbd>+<kbd>F3</kbd>和<kbd>Fn</kbd>+<kbd>Insert</kbd>，Dell原生亮度键调节键<kbd>Fn</kbd>+<kbd>↓</kbd>和<kbd>Fn</kbd>+<kbd>↑</kbd>不工作。
3. 触控板开关键<kbd>Fn</kbd>+<kbd>F5</kbd>不工作
4. <kbd>SysRq</kbd>/<kbd>PrntScrn</kbd> 不工作
5. <kbd>Pause</kbd>(<kbd>Fn</kbd>+<kbd>Insert</kbd>)不工作
6. 计算器键<kbd>Calc</kbd>不工作
7. <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Delete</kbd>不工作

&nbsp;
因此，为了完美使用Big Sur，必须对其进行修复。

&nbsp;

## **步骤一** 准备工作 
工欲善其事，必先利其器。为了修复工作，我们先做准备。
1. 抓取[DrHurt](https://github.com/DrHurt)的远程仓库[VoodooPS2Controller](https://github.com/DrHurt/OS-X-ALPS-DRIVER)
    ```
    git clone git@github.com:DrHurt/OS-X-ALPS-DRIVER.git
    cd OS-X-ALPS-DRIVER
    ```
2. 用**Xcode**打开项目`VoodooPS2Controller.xcodeproj`，并根据你的MacOS版本配置好环境，确保**Xcode**下`Product/Build`测试编译成功
 
3. 下载[ioio](https://bitbucket.org/RehabMan/os-x-ioio/downloads/RehabMan-ioio-2014-0122.zip)调试工具, 解压后将**ioio**二进制文件复制到目录`/usr/local/bin/`
    ```
    wget https://bitbucket.org/RehabMan/os-x-ioio/downloads/RehabMan-ioio-2014-0122.zip
    ```
4. 为了动态抓取键位的`PS code/ADB code`映射我们先写个简单的脚本`vim ioio_debug.sh`， 复制以下几行代码然后在**vim**中`:wq`保存退出 
    ```
    #!/bin/sh
    ioio -s ApplePS2Keyboard LogScanCodes 1
    watch "sudo dmesg | grep ApplePS2Keyboard | tail -20"
    ```
    并赋予可执行权限 ` chmod 755 ./ioio_debug.sh `。 （注：如没有watch命令需要安装 `brew install watch`)

5. 终端下运行`./ioio_debug.sh`测试键位调试是否成功,成功后终端会有以下输出  
    ```   
    argv[0] = { ioio }
    argv[1] = { -s }
    argv[2] = { ApplePS2Keyboard }
    argv[3] = { LogScanCodes }
    argv[4] = { 1 }
    ioio: setting property 'ApplePS2Keyboard:LogScanCodes' as number to 1 (0x1)
    Password:
    ```
    键入本机管理员密码后, 然后按任意键屏幕上会按照你的键位返回对应的**PS Code**和**ADB Code**，就会显示如下终端输出信息.

    ```
    [ 2585.028727]: ApplePS2Keyboard: sending key 31=2d down
    [ 2585.102607]: ApplePS2Keyboard: sending key 1f=1 down
    [ 2585.683917]: ApplePS2Keyboard: sending key 7=16 down
    [ 2585.846499]: ApplePS2Keyboard: sending key 5=15 down
    [ 2586.028108]: ApplePS2Keyboard: sending key 3=13 down
    ```
至此，准备工作完成，现在我们可以愉快的进行修复工作了。

&nbsp;

## **步骤二** 修复数字锁定键和小键盘映射
首先分析源码，得到键盘驱动源码主文件为**VoodooPS2Keyboard.cpp**，

先在终端下运行`./ioio_debug.sh`，点按小键盘上的每一个按键，先得到数字锁定键的PS2键位码为 **[0x45]**, 然后再得到数字小键盘区域每个按键得到数字小键盘**PS2 Code**映射表，并结合**ApplePS2ToADBMap.h**整理出一份Apple的**ADB Code**映射关系如下:

| PS2 Code | ADB Code(数字键) | ADB Code(功能键) | 描述(数字键/功能键) |
| :------: | :------: | :------: | :------ |
| [0x48] | 0x5b | 0x7e | 8 up arrow |
| [0x50] | 0x54 | 0x7d | 2 down arrow |
| [0x4B] | 0x56 | 0x7b | 4 left arrow |
| [0x4D] | 0x58 | 0x7c | 6 right arrow |    
| [0x52] | 0x52 | 0x92 | 0 insert / CDROM inject |
| [0x53] | 0x41 | 0x75 | . delete |
| [0x49] | 0x5c | 0x74 | 9 page up |
| [0x51] | 0x55 | 0x79 | 3 page down |
| [0x47] | 0x59 | 0x73 | 7 home |
| [0x4F] | 0x53 | 0x77 | 1 end |

&nbsp;

然后在 `bool ApplePS2Keyboard::init(OSDictionary * dict)` 函数加一行 "`_numKeypadLocked = true;`" 初始化`_numKeypadLocked`变量.

下一步再在 `dispatchKeyboardEventWithPacket`函数的`switch (keyCode)`方法中加入数字锁定键 **[0x45]** 的执行逻辑如下:
```
bool ApplePS2Keyboard::dispatchKeyboardEventWithPacket(const UInt8* packet)
{

    // handle special cases
    switch (keyCode)
    {

        // ......    

        case 0x45:  //num lock remapping
            keyCode = 0;

            //NUM LOCK fix For DELL Precision M4800
            if(goingDown)
            {
                setNumLockFeedback(_numKeypadLocked);
                _numKeypadLocked = !_numKeypadLocked;
            }

            // remap NUM PAD by NUMLOCK LED status
            if(!_numKeypadLocked)
            {
                _PS2ToADBMap[0x48] = 0x5b;     // 8 up arrow
                _PS2ToADBMap[0x50] = 0x54;     // 2 down arrow
                _PS2ToADBMap[0x4B] = 0x56;     // 4 left arrow
                _PS2ToADBMap[0x4D] = 0x58;     // 6 right arrow
                _PS2ToADBMap[0x52] = 0x52;     // 0 insert / CDROM inject
                _PS2ToADBMap[0x53] = 0x41;     // . delete
                _PS2ToADBMap[0x49] = 0x5c;     // 9 page up
                _PS2ToADBMap[0x51] = 0x55;     // 3 page down
                _PS2ToADBMap[0x47] = 0x59;     // 7 home
                _PS2ToADBMap[0x4F] = 0x53;     // 1 end
                
            }
            else
            {
                _PS2ToADBMap[0x48] = 0x7e;      // 8 up arrow
                _PS2ToADBMap[0x50] = 0x7d;      // 2 down arrow
                _PS2ToADBMap[0x4B] = 0x7b;      // 4 left arrow
                _PS2ToADBMap[0x4D] = 0x7c;      // 6 right arrow
                _PS2ToADBMap[0x52] = 0x92;      // 0 insert / CDROM inject
                _PS2ToADBMap[0x53] = 0x75;      // . delete
                _PS2ToADBMap[0x49] = 0x74;      // 9 page up
                _PS2ToADBMap[0x51] = 0x79;      // 3 page down
                _PS2ToADBMap[0x47] = 0x73;      // 7 home
                _PS2ToADBMap[0x4F] = 0x77;      // 1 end

            }
            break;

        // .......    
    }
}
```

最后在键盘初始化函数initkeyboard里加入一行代码 "`setNumLockFeedback(_numKeypadLocked)；`" 启用开机小键盘数字键锁定Num Lock。
```
void ApplePS2Keyboard::initKeyboard()
{
    //......


    setNumLockFeedback(_numKeypadLocked);       //开机启用小键盘数字键锁定Num Lock，点亮LED指示灯


    //......
}
```

至此，数字小键盘按键修复完成。编译打包，将生成的**ApplePS2Controller.kext**复制到`/EFI/OC/Kexts/`替换掉原来的文件，重启测试成功。

&nbsp;

## **步骤三** 修复Dell原生亮度调节键<kbd>Fn</kbd>+<kbd>↑</kbd>/<kbd>↓</kbd>
第一步: 修改SSDT，启用 `e005`和`e006` Dell原生PS2键位码(此处为不再详述，可参详[SSDT源码](https://github.com/badfellow/Hackintosh_M4800/blob/master/OpenCore/SSDT-Dell-M4800.dsl))。

第二步: 在`VoodooPS2Keyboard-Info.plist`的Custom ADB Map中加入以下映射将亮度调节键映射到<kbd>F14</kbd>和<kbd>F15</kbd>。

```
e005=6b;FN+down arrow to brightness down
e006=71;FN+up arrow to brightness up
```

原生亮度调节键修复完成。

&nbsp;

## **步骤四** 修复Dell原生触控板开关键<kbd>Fn</kbd>+<kbd>F5</kbd>

先在终端下运行`./ioio_debug.sh`，点按<kbd>Fn</kbd>+<kbd>F5</kbd>得到PS2键位码为 **[e01e]** 。
在 `dispatchKeyboardEventWithPacket`函数的`switch (keyCode)`方法中加入 **[0x011e]** (*注:e0为扩展码标志，程序执行为0x01*)的执行逻辑如下:

```
bool ApplePS2Keyboard::dispatchKeyboardEventWithPacket(const UInt8* packet)
{
    // handle special cases
    switch (keyCode)
    {

        case 0x011e:    // fn+f5 (Dell precision M4800)
        {
            unsigned origKeyCode = keyCode;
            keyCode = 0;
            if (!goingDown)
                break;
            if (!checkModifierState(kMaskLeftControl))
            {
                // get current enabled status, and toggle it
                bool enabled;
                _device->dispatchMouseMessage(kPS2M_getDisableTouchpad, &enabled);
                enabled = !enabled;
                _device->dispatchMouseMessage(kPS2M_setDisableTouchpad, &enabled);
                break;
            }
            if (origKeyCode != 0x011e)
                break; // do not fall through for 0x0128
            // fall through
        }

        // .......    
    }
}
```
编译打包，原生触控键修复完成。

&nbsp;

## **步骤五** 修复截屏键<kbd>SysRq</kbd>/<kbd>PrntScrn</kbd>

第一步: 先在终端下运行`./ioio_debug.sh`，点按<kbd>Fn</kbd>+<kbd>End</kbd>/<kbd>Home</kbd>得到PS2键位码为 **[e037]** 

第二步: 分析源码，我们可以看到源码中有`case 0x0137:`的执行逻辑是将 **[e037]** 映射到了触控板控制开关。所以，我们将原这段代码删除或者注释掉。
```
//        case 0x0128:    // alternate that cannot fnkeys toggle (discrete trackpad toggle)
//        case 0x0137:    // prt sc/sys rq
//        {
//            unsigned origKeyCode = keyCode;
//            keyCode = 0;
//            if (!goingDown)
//                break;
//            if (!checkModifierState(kMaskLeftControl))
//            {
//                // get current enabled status, and toggle it
//                bool enabled;
//                _device->dispatchMouseMessage(kPS2M_getDisableTouchpad, &enabled);
//                enabled = !enabled;
//                _device->dispatchMouseMessage(kPS2M_setDisableTouchpad, &enabled);
//                break;
//            }
//            if (origKeyCode != 0x0137)
//                break; // do not fall through for 0x0128
//            // fall through
//        }
```
第三步: 在`VoodooPS2Keyboard-Info.plist`的Custom ADB Map中加入以下映射将<kbd>SysRq</kbd>和<kbd>PrntScrn</kbd>映射到<kbd>F13</kbd>

```
e037=69;fn+Home/End to F13
```
第四步: 编译打包，将生成的**ApplePS2Controller.kext**复制到`/EFI/OC/Kexts/`替换掉原来的文件，重启。打开系统偏好设置>键盘>快捷键, 将截屏映射到<kbd>SysRq</kbd>/<kbd>PrntScrn</kbd>（<kbd>F13</kbd>）。

&nbsp;

## **步骤六** 修复<kbd>Pause</kbd>(<kbd>Fn</kbd>+<kbd>Insert</kbd>)

第一步: 先在终端下运行`./ioio_debug.sh`，点按<kbd>Fn</kbd>+<kbd>Insert</kbd>得到PS2键位码为 **[e045]**

第二步: 在`VoodooPS2Keyboard-Info.plist`的Custom ADB Map中加入以下映射将<kbd>Pause</kbd>(<kbd>Fn</kbd>+<kbd>Insert</kbd>)映射到<kbd>F18</kbd>。

第三步: 重复 **步骤五** >第四步 将<kbd>Pause</kbd>(<kbd>Fn</kbd>+<kbd>Insert</kbd>)映射到你需要的功能键。


&nbsp;

## **步骤七** 修复 计算器键<kbd>Calc</kbd>

第一步: 先在终端下运行`./ioio_debug.sh`，点按<kbd>Fn</kbd>+<kbd>Insert</kbd>得到PS2键位码为 **[e021]**

第二步: 在`VoodooPS2Keyboard-Info.plist`的Custom ADB Map中加入以下映射将<kbd>Calc</kbd>映射到<kbd>F19</kbd>。

第三步: 编译打包，将生成的**ApplePS2Controller.kext**复制到`/EFI/OC/Kexts/`替换掉原来的文件，重启

第四步：用`MacOS`字带的工具`自动操作`将<kbd>Calc</kbd>(<kbd>F19</kbd>)映射到计算器app。

&nbsp;

## **步骤八** 修复<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Delete</kbd>

第一步: 先在终端下运行`./ioio_debug.sh`，点按<kbd>Delete</kbd>得到PS2键位码为 **[e053]**

第二步: 分析源码**VoodooPS2Keyboard.cpp**, 得到` case 0x0153 ` 的运作逻辑是屏蔽了<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Delete</kbd>以避免触发电源开关造成异常关机. 
        为了实现与Windows一样的锁屏效果，修改` case 0x0153 `源码如下将<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Delete</kbd>映射到<kbd>Ctrl</kbd>+<kbd>Command</kbd>+<kbd>Q</kbd>。

```
bool ApplePS2Keyboard::dispatchKeyboardEventWithPacket(const UInt8* packet)
{
    // handle special cases
    switch (keyCode)
    {

        case 0x0153:    // delete
            
            // check for Ctrl+Alt+Delete? (three finger salute)
            if (checkModifierState(kMaskLeftControl|kMaskLeftAlt))
            {
                keyCode = 0;
                if (goingDown)
                {
                    // Note: If OS X thinks the Command and Control keys are down at the time of
                    //  receiving an ADB 0x7f (power button), it will unconditionaly and unsafely
                    //  reboot the computer, much like the old PC/AT Ctrl+Alt+Delete!
                    // That's why we make sure Control (0x3b) and Alt (0x37) are up!!
                    // then map to Ctrl + Command + Q (screen lock)
                    dispatchKeyboardEventX(0x37, true, now_abs);
                    dispatchKeyboardEventX(0x3b, true, now_abs);
                    dispatchKeyboardEventX(0xc, true, now_abs);
                    dispatchKeyboardEventX(0x7f, false, now_abs);
                }
                
                dispatchKeyboardEventX(0x37, false, now_abs);
                dispatchKeyboardEventX(0x3b, false, now_abs);
                dispatchKeyboardEventX(0xc, false, now_abs);
                dispatchKeyboardEventX(0x7f, false, now_abs);
            }

            break;
        // .......    
    }
}
```

第三步: 编译打包，将生成的**ApplePS2Controller.kext**复制到`/EFI/OC/Kexts/`替换掉原来的文件，重启


&nbsp;
&nbsp;
&nbsp;



自此，历经<font color=#0099ff size=5 face="黑体">步骤一</font>到<font color=#0099ff size=5 face="黑体">步骤八</font>，我们所有按键均修复完成，可以愉快地玩耍了。。。。。。