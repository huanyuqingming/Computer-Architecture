# Lab 2
涂宇清 522030910152

## 计算器搭建
将教材中的样例代码整合到同一个文件后编译，出现报错。修改如下：
* 将无法用9位offset表示的``BR``指令改为``JSR``指令；
* 将``OpAdd``、``OpMult``、``OpNeg``中的``RET``改为``JSR NewCommand``。

执行编译完成的.obj文件，观察到将第一个数字压入栈后，输入D仍显示栈内元素不足，故修改代码如下：
* 将``POP``中的``BRz Underflow``改为``BRp Underflow``。

运行搭建完成的计算器，输入输出结果如下：
* 入栈及上溢出
![](https://notes.sjtu.edu.cn/uploads/upload_e64d98c6406d672e2a0b77bcbf3eba5d.png)
* 清空栈及下溢出
![](https://notes.sjtu.edu.cn/uploads/upload_35872da48145c4068a13581e0ebd8011.png)
* 范围检查
![](https://notes.sjtu.edu.cn/uploads/upload_6c6a7fbc86ca8eef5781f43969d95320.png)
* 加法
![](https://notes.sjtu.edu.cn/uploads/upload_fa67f712d672c9caa36828ea9c261443.png)
* 乘法
![](https://notes.sjtu.edu.cn/uploads/upload_bb805241cbed550726ff6e6b2b2c9815.png)
* 取反
![](https://notes.sjtu.edu.cn/uploads/upload_aca9330b9ad8397e7150327c900d92fa.png)

## 运算符扩展
### 取模
取模运算实现代码如下：
1. 取操作数
```
OpMod           JSR         POP         ; Get first source operand.
                ADD         R5,R5,#0    ; Test if POP was successful.
                BRp         modExit     ; Branch if not successful.
                ADD         R1,R0,#0    ; Make room for second operand
                JSR         POP         ; Get second source operand.
                ADD         R5,R5,#0    ; Test if POP was successful.
                BRp         modRestore1 ; Not successful, put back first.
```
2. 判断除数的符号，若除数为0则输出报错
```
                ADD         R1,R1,#0
                BRp         bIsP           
                BRn         bIsN
                LEA         R0,Mod0Msg  ; there is an error if b = 0
                TRAP        x22         ; Output character string
                JSR         modRestore2       
```
3. 若除数为正，则循环执行**被除数减除数**，以得到模值
```
bIsP            NOT         R1,R1
                ADD         R1,R1,#1    ; R1 <-- -b
modLoop1        ADD         R0,R0,R1    ; R0 <-- mR0-b
                BRzp        modLoop1
                ADD         R1,R1,#-1
                NOT         R1,R1       ; R1 <-- b
modLoop2        ADD         R0,R0,R1    ; R0 <-- mR0+b
                BRn         modLoop2
                JSR         RangeCheck  ; Check size of     result.
                BRp         modRestore2 ; Out of range,  restore both.
                JSR         PUSH        ; Push sum on the stack.
                JSR         NewCommand  ; On to the next task...
```
4. 若除数为负，则先将**被除数和除数取相反数**后，执行**3.**中的操作，将**结果取相反数**即为模值
```
bIsN            NOT         R0,R0
                ADD         R0,R0,#1    ; R2 <-- -a
modLoop3        ADD         R0,R0,R1    ; R0 <-- mR0-b
                BRzp        modLoop3
                ADD         R1,R1,#-1
                NOT         R1,R1       ; R1 <-- b
modLoop4        ADD         R0,R0,R1    ; R0 <-- mR0+b
                BRn         modLoop4
                NOT         R0,R0
                ADD         R0,R0,#1
                JSR         RangeCheck  ; Check size of result.
                BRp         modRestore2 ; Out of range, restore both.
                JSR         PUSH        ; Push sum on the stack.
                JSR         NewCommand  ; On to the next task...
```
5. 存储标志
```
modRestore2     ADD         R6,R6,#-1   ; Decrement stack pointer.
modRestore1     ADD         R6,R6,#-1   ; Decrement stack pointer.
modExit         JSR         NewCommand 
Mod0Msg         .FILL       x000A
                .STRINGZ    "Error: The divisor cannot be 0."
```
将取模运算添加至计算器，输入输出结果如图：
![](https://notes.sjtu.edu.cn/uploads/upload_8d00e0e1f8fb9ccc2435161cdcf87f08.png)
![](https://notes.sjtu.edu.cn/uploads/upload_7c74a4f9b065357a572f50d10a9df3c6.png)
![](https://notes.sjtu.edu.cn/uploads/upload_8a67e0ccd1534f9d11af642d3738781d.png)

### 异或
1. 取操作数
```
OpXOR           JSR         POP         ; Get first source operand.
                ADD         R5,R5,#0    ; Test if POP was successful.
                BRp         XORExit     ; Branch if not successful.
                ADD         R2,R0,#0    ; Make room for second operand
                JSR         POP         ; Get second source operand.
                ADD         R5,R5,#0    ; Test if POP was successful.
                BRp         XORRestore1 ; Not successful, put back first.
                ADD         R3,R0,#0    ; R2 <-- a, R3 <-- b
```
2. 使用``Mask``从低到高依次取操作数的每一位，将两个操作数的**相同位相加**，所得结果的当前位即为当前位的异或
```
                LD          R1,Mask     ; R1 <--Mask
                AND         R0,R0,#0
XORLoop         AND         R4,R2,R1
                AND         R5,R3,R1    ; get one digit of a and b
                ADD         R4,R4,R5    ;
                AND         R4,R4,R1    ;
                ADD         R0,R0,R4    ; XOR
                ADD         R1,R1,R1    ; get mask into next digit
                BRp         XORLoop
                AND         R4,R2,R1
                AND         R5,R3,R1    ; get the highest digit of a and b
                ADD         R4,R4,R5    ;
                AND         R4,R4,R1    ;
                ADD         R0,R0,R4    ; XOR
                JSR         RangeCheck  ; Check size of result.
                BRp         XORRestore2 ; Out of range, restore both.
                JSR         PUSH        ; Push sum on the stack.
                JSR         NewCommand  ; On to the next task...
```
3. 存储标志
```
XORRestore2     ADD         R6,R6,#-1   ; Decrement stack pointer.
XORRestore1     ADD         R6,R6,#-1   ; Decrement stack pointer.
XORExit         JSR         NewCommand 
Mask            .FILL       x0001
```
将异或运算添加至计算器，输入输出结果如图：
![](https://notes.sjtu.edu.cn/uploads/upload_a289567abd4de006048eee4455af81eb.png)