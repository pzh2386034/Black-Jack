#include <stdio.h>
#include <stdlib.h>
#include <sys/io.h>
#include <string.h>
#include "linux/capability.h"

//kcs registers
#define KCS_CMD_RED 0XCA3
#define KCS_DATA_REG 0XCA2

#define GET_STATUS  0x60 //send 0x60 to cmd/status reg to get ksc status
#define WAITE_START 0x61 //to cmd/status reg to begin write data
#define WAITE_END   0x62 //to cmd/status reg to end write data
#define READ        0x68 //to data_in/out reg to read data

#define IDLE_STATE   0 //kcs idle status
#define READ_STATE  0x40 //read data status, cmd/status reg value
#define WRITE_STATE 0x80 //write data status, cmd/status reg value
#define ERROR_STATE 0xc0 //err status, cmd/status reg value

void timedelay(int ms)
{
    int x,y,k = 0;
    for(x = 100; x> 0; x--)
        for(y = ms; y >0; y--)
            k++;
}
void waitIBFClear()
{
    int IBFStatus;
    do{
        // wait for command reg IBF ->1
        timedelay(100);
        IBFStatus = inb(KCS_CMD_REG); 
    }while((IBFStatus & 0x02) == 1);
}

void clearOBF()
{
    // system read data reg, obf will be clear
    inb(KCS_DATA_REG);
}
void waitOBFSet()
{
    int OBFStatus;
    do
    {
        timedelay(100);
        OBFStatus = inb(KCS_CMD_REG);
    } while ((OBFStatus & 0x01) == 0);
    
}
char getKCSState()
{
    int KCSState;
    KCSState = inb(KCS_CMD_REG);
    return (KCSState & 0xc0);
}
void readIPMI()
{

}

/*
关键内核patch:http://lkml.iu.edu/hypermail/linux/kernel/1801.2/00302.html
BT:ASPEED_BT_IPMI_BMC
obj-$(CONFIG_IPMI_WATCHDOG) += ipmi_watchdog.o
obj-$(CONFIG_IPMI_POWEROFF) += ipmi_poweroff.o
obj-$(CONFIG_ASPEED_BT_IPMI_BMC) += bt-bmc.o
+obj-$(CONFIG_ASPEED_KCS_IPMI_BMC) += kcs-bmc.o

ipmi_bmc.h===>define MAGIC NUM and command
kcs_bmc.c====>init kcs driver, code to operate ksc
kcs_bmc_aspeed.c===>aspeed kcs driver probe
kcs_bmc_ioctl
aspeed_kcs_set_address:注册寄存器地址

aspeed_kcs_probe:
    kcs_bmc->ioreg = ast_kcs_bmc_ioregs[chan - 1];
    kcs_bmc->io_inputb = aspeed_kcs_inb;
    kcs_bmc->io_outputb = aspeed_kcs_outb;
    aspeed_kcs_config_irq :注册软中断

aspeed-lpc-ctrl.h ===>提供host访问bmc资源的通道(bmc flash/raw)
*/


lpc-->AHB/APB-->UART1/UART2
    直连APB bus
        master: 升级bios， TPM，LPC controller
        slave: I/O read,write cycle
    virtual UART
    支持IPMI 2.0 kcs, BT (3 sets of KCS mode registers and 1+1 BT mode registers)


spi flash controller:
    1. spi master
        连接AHB总线， access by arm, coprocessor or from lpc interface by host system
        支持系统启动BIOS flash Memory
    2. spi pass-through
        1 spi input interface + 1 spi output interface ; switch controller
        通过switch controller, update bios by bmc

memory integrity check(MIC) engine
    直连AHB bus
    每次检测4kb内存 checksum unit
pwm controller
    支持 8 pwm outputs
fan tachometer controller
peci controller
JTAG Master controller
MCTP controller
MSI controller
X-DMA controller
Software Specifications




BIOS将VT-d设置成enable