# USB 键盘开发记录
最近接触到 `stm32` 平台实现 `usb keyboard` 的内容，遍搜网络后最终选定了两个开源方案做参考。



[一号方案： STM32完整开发一台双模机械键盘](http://blog.csdn.net/BG2CRW/article/details/79475752)

[二号方案：STM32硬核DIY机械键盘|蓝牙USB双模|灯控](https://blog.csdn.net/yougeng123/article/details/103803593?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)



> 两个方案各有特点，一号方案用C++实现，用面向对象的理念把普通按键跟复合按键拆分开；二号方案实现了蓝牙键盘的功能；考虑到减少工作量需求，最终选择了**一号方案**。



**基于 `CoopBoard` 工程进行验证及二次开发**

- 构建 Keil 工程

从 github 上克隆下来的工程没有 keil proj , 因此首先要做的就是手动新建一个 keil 工程，把程序文件增加到工程中。详细操作步骤在此不再表述，**注意点**： `startup_stm32f10x_hd.s` 中 stack 和 heap 的容量要增大，否则程序起来后会因为堆栈越限而死机。

- GPIO 管脚配置

根据自己板子的情况修改 GPIO 口的定义和功能。**注意点：** 原工程把 SWI 等管脚功能关闭了，注意屏蔽这部分，避免在调试过程中不能顺利用 ulink 工具烧录程序。

- USB 设备识别功能验证

这部分刚开始因为手头的 USB 线有问题，一直不能被 PC 正确识别，折腾了半天**换了根线**就好了，可见开源方案的可靠性还是有保障的！

- 键盘选择

[矩阵键盘超链接](https://detail.tmall.com/item.htm?spm=a230r.1.14.126.5a694ce8W6mdM3&id=613587201610&ns=1&abbucket=4)
从淘宝买了一个 4x4 的矩阵键盘，满足常规按键、组合按键的功能的验证。这里稍微描述一下矩阵键盘的识别原理。简单说就是行做输出列做输入。进行逐行扫描，当行为低电平的时候读到列也是低电平，说明按键被按下。但是存在同一列组合键不能被识别的情况，比如 ctrl + shift ，这个还没想好改怎么处理。

- 代码初识

工程中最核心的就是 key 模块，该方案把 key 分成了 FunctionKey 、ContentKey、CompositeKey 三类，这里一定要贴一张图，原作者因为键盘精细化，所以只用了5行，通过 FunctionKey 来切换如第一行按键的实际功能，做标准键盘的就不需要看 FunctionKey 模块了，只需要参考 Content Key 和 CompositeKey 两类即可。

![CoopBoard](D:\01Sync_File\Project\CoopBoard\DOC\CoopBoard_Key.jpg)

- 进阶调试

经过几天的调试，普通按键和例如 shift + a 识别成 A 等功能均已确认正常，不过发现 ctrl + shift （已经调整两个按键不在同一列）依然识别存在问题。反复确认代码之后，调整下述地方能解决。

```c
void CompositeKey::afterAllProcess(KeyState &ks) {
		if(
			ks.isDown(rc_place.getRow(), rc_place.getCol())
			&&
			ks.code_to_send.size() == 0
		) {
        ks.code_to_send.push_back(KeyCode(0, ks.composite_mark)); //this->compositeByte 更换为 ks.composite_mark
    }
}
```



