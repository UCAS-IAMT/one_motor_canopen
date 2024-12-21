# one_motor_canopen——2024.3.22
通过canopen报文通讯一代驱动板，并驱动关节电机

作者：胡翰泽

**系统环境** ：Ubuntu20.04 + Ros1-noetic

**效果展示：**

**PPM:**
<div align="center">
<table>
<tr>
<td>

![](https://github.com/UCAS-IAMT/one_motor_canopen/blob/main/PPM.gif)  

</td>
</tr>
</table>
</div>

**PVM:**
<div align="center">
<table>
<tr>
<td>

![](https://github.com/UCAS-IAMT/one_motor_canopen/blob/main/PVM.gif)  

</td>
</tr>
</table>
</div>

# CANopen报文记录&使用心得

25kg协作机械臂，一体化电机，假设电机**id = 6**；

切换模式时，电机要处于**失能状态**;

在ros终端输入can命令之前，记得**打开roscore**

## 与正常的nimotion电机对象字典的不同
2002h不用管；

参考对象字典：Cnitec_Rob_V1.2.eds（已上传）

## canopen device
1.**can0**（can_pro收发器）
```
//最好把can_pro接到USB3.0的接口上，感觉对通讯有帮助
sudo ip link set can0 type can bitrate 500000
sudo ip link set can0 txqueuelen 1000
sudo ip link set can0 up
```
```
//一步到位，启动
sudo ip link set can0 up type can bitrate 500000
```

```
candump can0  //监听can0上的报文
ros2 service list -t
```


## SDO —— PP(轮廓位置模式)
### 测试通过
```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f60600001000000  //pp
cansend can0 606#237a600000003200  //目标位置，3276800（320000 h）,第一个位置
cansend can0 606#23816000aa2a0400  //目标速度，273066（42aaa h）
cansend can0 606#2383600055150200  //减速度，136533（21555 h）
cansend can0 606#2384600055150200  //加速度，136533（21555 h）
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060000f000000  //使能
cansend can0 606#2b4060001f000000  //触发电机运行

//其他位置
//8192*4*100 = 3276800
//8192*4*160 = 
cansend can0 606#2b40600007000000  //失能
cansend can0 606#237a600000006400  //目标位置，3276800*2=6553600（640000 h）
cansend can0 606#2b4060000f000000  //使能
cansend can0 606#2b4060001f000000  //触发电机运行

cansend can0 606#237a600000005000  //目标位置，

//其他速度

```
```
//6040h的变化:电机以该控制字bit4的上升沿，接收新的位置信号，所以每次执行完一次之后要把该位清零
0x0f——>0x1f   //绝对位置，非立刻更新
0x2f——>0x3f   //绝对位置，立刻更新
0x4f——>0x5f   //相对位置，非立刻更新
0x6f——>0x7f   //相对位置，立刻更新
```

## ！！！SDO —— VM(速度模式)！！！找不到6042h、6048h和6049h

```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f60600002000000  //vm
//cansend can0 606#2b4260002c010000  //目标速度为300rpm，电机以该速度运行
//cansend can0 606#23486001f4010000  //写加速度：01h = 500rmp
//cansend can0 606#2b48600201000000  //写加速度：02h = 1s
//cansend can0 606#23496001f4010000  //写减速度：01h = 500rmp
//cansend can0 606#2b49600201000000  //写减速度：02h = 1s
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060007f000000  //使能
```

## SDO —— PV(轮廓速度模式)，先当成VM！！！用
### 测试通过
```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f60600003000000  //pv
cansend can0 606#23ff6000400d0d00  //设置目标速度为855360（d0d40 h）
cansend can0 606#23836000b80b0000  //写轮廓加速度为3000（bb8 h），用户单位/s^2
cansend can0 606#23846000b80b0000  //写轮廓减速度为3000（bb8 h），用户单位/s^2
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060007f000000  //使能
```

## SDO —— PT(轮廓转矩模式)！！！CST？

```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f60600004000000  //pt
cansend can0 606#2b716000f4010000  //设置目标转矩为500, 500 * 0.1% = 电机以50%的额定转矩运动
cansend can0 606#2b88600002000000  //写转矩斜坡类型为无斜坡；0-斜坡，2-无斜坡；
cansend can0 606#238760000a000000  //写转矩斜坡为10
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060000f000000  //使能
```
```
cansend can0 606#2387600088130000  //写转矩斜坡为5000（1388 h）
cansend can0 606#237260004cff0000  //写最大转矩为65356（ff4c h）
cansend can0 606#23806000d0070000  //写最大速度为2000（7d0 h）

//转矩2500（9c4 h）
cansend can0 606#2b716000c4090000
```

## ！！！SDO —— HM(原点回归模式)！！！找不到2003h

```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f60600006000000  //hm
//cansend can0 606#2b0320030f000000  //03h=15,设置DI1功能为负限位开关
//cansend can0 606#2b03200400000000  //04h=0,设置DI1逻辑为低电平有效
cansend can0 606#2f98600011000000  //设置回零方式为17
cansend can0 606#2399600127100000  //设置01h寻找限位开关速度为10000
cansend can0 606#2399600227100000  //设置02h寻找原点信号速度为10000
cansend can0 606#239a6000400d0300  //设置回零加速度为200000
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060000f000000  //使能
cansend can0 606#2b4060001f000000  //触发电机运行 0fh——>1fh
```

## SDO —— IP(插补模式)

```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f60600007000000  //ip
cansend can0 606#2fc2600114000000  //设置01h时间常数为20
cansend can0 606#2fc26002fd000000  //设置02h时间指数为-3；因此时间为20ms
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060000f000000  //使能
cansend can0 606#2b4060001f000000  //触发电机运行 0fh——>1fh
//上位机按照插补周期，写插补位置60C1h:01h(只支持绝对位置指令，用户单位)
```

## SDO —— CSP(循环同步位置模式)

```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f60600008000000  //csp
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060000f000000  //使能
//上位机按照同步周期发送目标位置607A(只支持绝对位置指令，用户单位)
```

## SDO —— CSV(循环同步速度模式)

```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f60600009000000  //csv
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060000f000000  //使能
//上位机需要按照同步周期写速度对象60FFh
```

## SDO —— CST(循环同步转矩模式)

```
//cansend can0 606#2b02200100000000  //设置为CiA402模式
cansend can0 606#2f6060000a000000  //cst
cansend can0 606#2b40600006000000  //写控制字，电机准备
cansend can0 606#2b40600007000000  //失能
cansend can0 606#2b4060000f000000  //使能
//上位机按照同步周期发送目标转矩6071h(单位0.1%)
```


