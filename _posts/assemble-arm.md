1.通用数据传送指令.
MOV---- move
MOVSX----extended move with sign data
MOVZX----extended move with zero data
PUSH----push
POP----pop
PUSHA----push all
POPA----pop all
PUSHAD----push all data
POPAD----pop all data
BSWAP----byte swap
XCHG----exchange
CMPXCHG----compare and change
XADD----exchange and add
XLAT----translate
2.输入输出端口传送指令.
IN----input
OUT----output
3.目的地址传送指令.
LEA----load effective address
LDS----load DS
LES----load ES
LFS----load FS
LGS----load GS
LSS----load SS
4.标志传送指令.
LAHF----load AH from flag
SAHF----save AH to flag
PUSHF----push flag
POPF----pop flag
PUSHD----push dflag
POPD----pop dflag

二、算术运算指令
ADD----add
ADC----add with carry
INC----increase 1
AAA----ascii add with adjust
DAA----decimal add with adjust
SUB----substract
SBB----substract with borrow
DEC----decrease 1
NEC----negative
CMP----compare
AAS----ascii adjust on substract
DAS----decimal adjust on substract
MUL----multiplication
IMUL----integer multiplication
AAM----ascii adjust on multiplication
DIV----divide
IDIV----integer divide
AAD----ascii adjust on divide
CBW----change byte to word
CWD----change word to double word
CWDE----change word to double word with sign to EAX
CDQ----change double word to quadrate word

三、逻辑运算指令
───────────────────────────────────────
AND----and
OR----or
XOR----xor
NOT----not
TEST----test
SHL----shift left
SAL----arithmatic shift left
SHR----shift right
SAR----arithmatic shift right
ROL----rotate left
ROR----rotate right
RCL----rotate left with carry
RCR----rotate right with carry

四、串指令
───────────────────────────────────────
MOVS----move string
CMPS----compare string
SCAS----scan string
LODS----load string
STOS----store string
REP----repeat
REPE----repeat when equal
REPZ----repeat when zero flag
REPNE----repeat when not equal
REPNZ----repeat when zero flag
REPC----repeat when carry flag
REPNC----repeat when not carry flag

五、程序转移指令
───────────────────────────────────────
1无条件转移指令(长转移)
JMP----jump
CALL----call
RET----return
RETF----return far
2条件转移指令(短转移,-128到+127的距离内)
JAE----jump when above or equal
JNB----jump when not below
JB----jump when below
JNAE----jump when not above or equal
JBE----jump when below or equal
JNA----jump when not above
JG----jump when greater
JNLE----jump when not less or equal
JGE----jump when greater or equal
JNL----jump when not less
JL----jump when less
JNGE----jump when not greater or equal
JLE----jump when less or equal
JNG----jump when not greater
JE----jump when equal
JZ----jump when has zero flag
JNE----jump when not equal
JNZ----jump when not has zero flag
JC----jump when has carry flag
JNC----jump when not has carry flag
JNO----jump when not has overflow flag
JNP----jump when not has parity flag
JPO----jump when parity flag is odd
JNS----jump when not has sign flag
JO----jump when has overflow flag
JP----jump when has parity flag
JPE----jump when parity flag is even
JS----jump when has sign flag
3循环控制指令(短转移)
LOOP----loop
LOOPE----loop equal
LOOPZ----loop zero
LOOPNE----loop not equal
LOOPNZ----loop not zero
JCXZ----jump when CX is zero
JECXZ----jump when ECX is zero
4中断指令
INT----interrupt
INTO----overflow interrupt
IRET----interrupt return
5处理器控制指令
HLT----halt
WAIT----wait
ESC----escape
LOCK----lock
NOP----no operation
STC----set carry
CLC----clear carry
CMC----carry make change
STD----set direction
CLD----clear direction
STI----set interrupt
CLI----clear interrupt

六、伪指令
─────────────────────────────────────
DW----definw word
PROC----procedure
ENDP----end of procedure
SEGMENT----segment
ASSUME----assume
ENDS----end segment
END----end

str r1, [r0, #4]　　;将 r1 中的数据存储到地址为 r0 + 4 的内存空间中
str r1, [r0, #4]!　　;将 r1 中的数据存储到 r0 + 4 的内存空间中， 然后 r0 = r0 + 4
str r1, [r0, r2, lsl #1]　　;将 r1 中的数据存储到地址为 r0 + (r2 << 1)的内存空间中
str r1, [r0], #4　　;将 r1 中的数据存储到地址为 r0 的内存空间中，然后 r0 = r0 + 4

LDR R1， [R0, #0x12]　　;将R0 + 12地址处的数据读出，保存到R1中（R0保持不变）
LDR Rd，[Rn], #0x04    ;立即数后索引寻址，Rn的值用作传输数据的存储地址。在数据传送后，将偏移量 0x04 与Rn相加，记过写回到 Rd中

stm r0, {r1-r3}　　;将 r1 - r3 寄存器（连续寄存器）中的数据存储到 以 r0 位起始地址的内存空间中
stm r0, {r3, r1}　  ;将 r1 、 r3 寄存器（不连续寄存器）中的数据存储到 以 r0 位起始地址的内存空间中
　　　　　　　　　　　　;批量操作时，低编号的寄存器数据对应内存中的低地址
stm r0!, {r1-r3}　　;寄存器批量存储，也适用自动索引寻址， 操作完成后 　　r0 = r0 + (寄存器个数)*4

swp r0, r1, [r2] @将r0中的数据放入内存地址是r2的地址空间，同时将r2中的值放入r1寄存器;处理器与内存之间进行数据交换  交换过程不会被打断

arm-none-eabi-as -o add.o add.s
arm-none-eabi-ld -Ttext=0x20000000 -o add.elf add.o
arm-none-eabi-objcopy -O binary add.elf add.bin