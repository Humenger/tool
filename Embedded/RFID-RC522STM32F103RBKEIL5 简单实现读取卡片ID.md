# RFID-RC522/STM32F103RB/KEIL5 简单实现读取卡片ID

## 序章
在这篇文章【 https://blog.csdn.net/qq_28877125/article/details/80437095 】的基础上修改完成！
## 简单使用
1).环境配置
 - MCU芯片型号：STM32F103RB
 - 开发板：理论任何STM32开发板
 - KIDE：KEIL5
 - 下载方式：J-LINK

2).程序编译

 - 使用KEIL5打开工程后直接加载编译即可

 3).程序烧录下载
 - 编译无错后直接烧录即可

 4).硬件配置
 - RC522与开发板连线:  
 ![在这里插入图片描述](./assets/6039dc1a036f1dfb29ea72ec9d415423.png)
 -  串口配置：  
    ![波特率:115200](./assets/9746f4f3067b1d541c9c607fd7b7fb38.png)

  5).读取卡片ID

 - 1.重启开发板
 - 2.将卡片放在感应区
 - 3.串口显示卡片信息
## 核心源码
main.c

```c
#include "delay.h"
#include "sys.h"
#include "usart.h"
#include "rc522.h"

/*
版本:1.0.0:
   1.简单实现RFID-RC522的ID读取并打印到串口；
	 
使用教程:
     1.将RFID-RC522与STM32F103RB开发板按下面连线方式连线；
	 2.然后将开发板通过USB转TLL连接到PC串口工具；
	 3.然后重启开发板；
	 4.将测试卡片放到感应区；
	 5.即可在串口工具看到输出卡片相关信息。
	 
RC522相关配置文件：rc522_config.h
*/

/**
*   连线说明：
*   1--SDA  <----->PA4
*   2--SCK  <----->PA5
*   3--MOSI <----->PA7
*   4--MISO <----->PA6
*   5--悬空
*   6--GND <----->GND
*   7--RST <----->PB0
*   8--VCC <----->VCC
**/

int main(void)
{
    delay_init();	    	 //延时函数初始化
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);//设置中断优先级分组为组2：2位抢占优先级，2位响应优先级
    uart_init(115200);	 	//串口初始化为115200
    RC522_Init();       //初始化射频卡模块
    while(1)
    {
        RC522_Handel();
    }
}



```
rc522_config.h

```c
#ifndef __RC522_CONFIG_H
#define __RC522_CONFIG_H

//##################RC522配置##########

#define RC522_SPI_GPIO                       GPIOA
#define RC522_SPI_GPIO_ENABLED               RCC_APB2Periph_GPIOA
#define RC522_SPI_GPIO_PIN_CS                GPIO_Pin_4
#define RC522_SPI_GPIO_PIN_SCK               GPIO_Pin_5
#define RC522_SPI_GPIO_PIN_MISO              GPIO_Pin_6
#define RC522_SPI_GPIO_PIN_MOSI              GPIO_Pin_7

#define RC522_SPI_GPIO_RST                   GPIOB
#define RC522_SPI_GPIO_RST_ENABLED           RCC_APB2Periph_GPIOB
#define RC522_SPI_GPIO_PIN_RST               GPIO_Pin_0

#endif

```

rc522.c
```c
#include "sys.h"
#include "rc522.h"
#include "delay.h"
#include "usart.h"
#include "string.h"
#include "rc522_config.h"

//////////////////////////////////////////////////////////
// M1卡分为16个扇区，每个扇区由四个块（块0、块1、块2、块3）组成
// 将16个扇区的64个块按绝对地址编号为：0~63
// 第0个扇区的块0（即绝对地址0块），用于存放厂商代码，已经固化不可更改
// 每个扇区的块0、块1、块2为数据块，可用于存放数据
// 每个扇区的块3为控制块（绝对地址为:块3、块7、块11.....）包括密码A，存取控制、密码B等

/*******************************
*连线说明：
*1--SDA  <----->PA4
*2--SCK  <----->PA5
*3--MOSI <----->PA7
*4--MISO <----->PA6
*5--悬空
*6--GND <----->GND
*7--RST <----->PB0
*8--VCC <----->VCC
************************************/

/*全局变量*/
unsigned char CT[2];//卡类型
unsigned char SN[4]; //卡号
unsigned char RFID[16];			//存放RFID
unsigned char lxl_bit=0;
unsigned char card1_bit=0;
unsigned char card2_bit=0;
unsigned char card3_bit=0;
unsigned char card4_bit=0;
unsigned char total=0;
unsigned char lxl[4]= {137,240,42,131};
unsigned char card_1[4]= {83,106,11,1};
unsigned char card_2[4]= {208,121,31,57};
unsigned char card_3[4]= {176,177,143,165};
unsigned char card_4[4]= {5,158,10,136};
u8 KEY[6]= {0xff,0xff,0xff,0xff,0xff,0xff};
u8 AUDIO_OPEN[6] = {0xAA, 0x07, 0x02, 0x00, 0x09, 0xBC};
unsigned char RFID1[16]= {0x00,0x00,0x00,0x00,0x00,0x00,0xff,0x07,0x80,0x29,0xff,0xff,0xff,0xff,0xff,0xff};
/*函数声明*/
unsigned char status;
unsigned char s=0x08;


#define   RC522_DELAY()  delay_us( 20 )


void RC522_Handel(void)
{

    status = PcdRequest(PICC_REQALL,CT);//寻卡

    //printf("\r\nstatus>>>>>>%d\r\n", status);

    if(status==MI_OK)// 寻卡成功
    {
        status=MI_ERR;
        status = PcdAnticoll(SN);// 防冲撞
    }

    if (status==MI_OK)// 防冲撞成功
    {
        status=MI_ERR;
        ShowID(SN); // 串口打印卡的ID号

        if((SN[0]==lxl[0])&&(SN[1]==lxl[1])&&(SN[2]==lxl[2])&&(SN[3]==lxl[3]))
        {
            lxl_bit=1;
            printf("\r\nThe User is:card_0\r\n");

        }
        if((SN[0]==card_1[0])&&(SN[1]==card_1[1])&&(SN[2]==card_1[2])&&(SN[3]==card_1[3]))
        {
            card1_bit=1;
            printf("\r\nThe User is:card_1\r\n");

        }
        if((SN[0]==card_2[0])&&(SN[1]==card_2[1])&&(SN[2]==card_2[2])&&(SN[3]==card_2[3]))
        {
            card2_bit=1;
            printf("\r\nThe User is:card_2\r\n");

        }

        if((SN[0]==card_3[0])&&(SN[1]==card_3[1])&&(SN[2]==card_3[2])&&(SN[3]==card_3[3]))
        {
            card3_bit=1;
            printf("\r\nThe User is:card_3\r\n");

        }
        if((SN[0]==card_4[0])&&(SN[1]==card_4[1])&&(SN[2]==card_4[2])&&(SN[3]==card_4[3]))
        {
            card4_bit=1;
            printf("\r\nThe User is:card_4\r\n");

        }
        //total=card1_bit+card2_bit+card3_bit+card4_bit+lxl_bit;
        status =PcdSelect(SN);

    }

    if(status==MI_OK)//选卡成功
    {

        status=MI_ERR;
        status =PcdAuthState(0x60,0x09,KEY,SN);
    }
    if(status==MI_OK)//验证成功
    {
        status=MI_ERR;
        status=PcdRead(s,RFID);
    }

    if(status==MI_OK)//读卡成功
    {
        status=MI_ERR;
        delay_ms(100);
    }

}

void RC522_Init ( void )
{
    SPI1_Init();

    RC522_Reset_Disable();

    RC522_CS_Disable();

    PcdReset ();

    M500PcdConfigISOType ( 'A' );//设置工作方式

}

void SPI1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    RCC_APB2PeriphClockCmd(	RC522_SPI_GPIO_ENABLED | RC522_SPI_GPIO_RST_ENABLED, ENABLE );//PORTA、B时钟使能

    // CS
    GPIO_InitStructure.GPIO_Pin = RC522_SPI_GPIO_PIN_CS;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP; 		 //推挽输出
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;		 //IO口速度为50MHz
    GPIO_Init(RC522_SPI_GPIO, &GPIO_InitStructure);					 //根据设定参数初始化PF0、PF1

    // SCK
    GPIO_InitStructure.GPIO_Pin = RC522_SPI_GPIO_PIN_SCK;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP; 		 //推挽输出
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;		 //IO口速度为50MHz
    GPIO_Init(RC522_SPI_GPIO , &GPIO_InitStructure);

    // MISO
    GPIO_InitStructure.GPIO_Pin = RC522_SPI_GPIO_PIN_MISO;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING; 		 //推挽输出
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;		 //IO口速度为50MHz
    GPIO_Init(RC522_SPI_GPIO , &GPIO_InitStructure);

    // MOSI
    GPIO_InitStructure.GPIO_Pin = RC522_SPI_GPIO_PIN_MOSI;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP; 		 //推挽输出
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;		 //IO口速度为50MHz
    GPIO_Init(RC522_SPI_GPIO , &GPIO_InitStructure);

    // RST
    GPIO_InitStructure.GPIO_Pin = RC522_SPI_GPIO_PIN_RST;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP; 		 //推挽输出
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;		 //IO口速度为50MHz
    GPIO_Init(RC522_SPI_GPIO_RST, &GPIO_InitStructure);

}


/*
 * 函数名：SPI_RC522_SendByte
 * 描述  ：向RC522发送1 Byte 数据
 * 输入  ：byte，要发送的数据
 * 返回  : RC522返回的数据
 * 调用  ：内部调用
 */
void SPI_RC522_SendByte ( u8 byte )
{
    u8 counter;

    for(counter=0; counter<8; counter++)
    {
        if ( byte & 0x80 )
            RC522_MOSI_1 ();
        else
            RC522_MOSI_0 ();

        RC522_DELAY();

        RC522_SCK_0 ();

        RC522_DELAY();

        RC522_SCK_1();

        RC522_DELAY();

        byte <<= 1;

    }

}


/*
 * 函数名：SPI_RC522_ReadByte
 * 描述  ：从RC522发送1 Byte 数据
 * 输入  ：无
 * 返回  : RC522返回的数据
 * 调用  ：内部调用
 */
u8 SPI_RC522_ReadByte ( void )
{
    u8 counter;
    u8 SPI_Data;


    for(counter=0; counter<8; counter++)
    {
        SPI_Data <<= 1;

        RC522_SCK_0 ();

        RC522_DELAY();

        if ( RC522_MISO_GET() == 1)
            SPI_Data |= 0x01;

        RC522_DELAY();

        RC522_SCK_1 ();

        RC522_DELAY();

    }


//	printf("****%c****",SPI_Data);
    return SPI_Data;
}


/*
 * 函数名：ReadRawRC
 * 描述  ：读RC522寄存器
 * 输入  ：ucAddress，寄存器地址
 * 返回  : 寄存器的当前值
 * 调用  ：内部调用
 */
u8 ReadRawRC ( u8 ucAddress )
{
    u8 ucAddr, ucReturn;


    ucAddr = ( ( ucAddress << 1 ) & 0x7E ) | 0x80;

    RC522_CS_Enable();

    SPI_RC522_SendByte ( ucAddr );

    ucReturn = SPI_RC522_ReadByte ();

    RC522_CS_Disable();

    return ucReturn;
}


/*
 * 函数名：WriteRawRC
 * 描述  ：写RC522寄存器
 * 输入  ：ucAddress，寄存器地址
 *         ucValue，写入寄存器的值
 * 返回  : 无
 * 调用  ：内部调用
 */
void WriteRawRC ( u8 ucAddress, u8 ucValue )
{
    u8 ucAddr;

    ucAddr = ( ucAddress << 1 ) & 0x7E;

    RC522_CS_Enable();

    SPI_RC522_SendByte ( ucAddr );

    SPI_RC522_SendByte ( ucValue );

    RC522_CS_Disable();
}


/*
 * 函数名：SetBitMask
 * 描述  ：对RC522寄存器置位
 * 输入  ：ucReg，寄存器地址
 *         ucMask，置位值
 * 返回  : 无
 * 调用  ：内部调用
 */
void SetBitMask ( u8 ucReg, u8 ucMask )
{
    u8 ucTemp;

    ucTemp = ReadRawRC ( ucReg );

    WriteRawRC ( ucReg, ucTemp | ucMask );         // set bit mask

}


/*
 * 函数名：ClearBitMask
 * 描述  ：对RC522寄存器清位
 * 输入  ：ucReg，寄存器地址
 *         ucMask，清位值
 * 返回  : 无
 * 调用  ：内部调用
 */
void ClearBitMask ( u8 ucReg, u8 ucMask )
{
    u8 ucTemp;

    ucTemp = ReadRawRC ( ucReg );

    WriteRawRC ( ucReg, ucTemp & ( ~ ucMask) );  // clear bit mask

}


/*
 * 函数名：PcdAntennaOn
 * 描述  ：开启天线
 * 输入  ：无
 * 返回  : 无
 * 调用  ：内部调用
 */
void PcdAntennaOn ( void )
{
    u8 uc;

    uc = ReadRawRC ( TxControlReg );

    if ( ! ( uc & 0x03 ) )
        SetBitMask(TxControlReg, 0x03);

}


/*
 * 函数名：PcdAntennaOff
 * 描述  ：开启天线
 * 输入  ：无
 * 返回  : 无
 * 调用  ：内部调用
 */
void PcdAntennaOff ( void )
{

    ClearBitMask ( TxControlReg, 0x03 );

}


/*
 * 函数名：PcdRese
 * 描述  ：复位RC522
 * 输入  ：无
 * 返回  : 无
 * 调用  ：外部调用
 */
void PcdReset ( void )
{
    RC522_Reset_Disable();

    delay_us ( 1 );

    RC522_Reset_Enable();

    delay_us ( 1 );

    RC522_Reset_Disable();

    delay_us ( 1 );

    WriteRawRC ( CommandReg, 0x0f );

    while ( ReadRawRC ( CommandReg ) & 0x10 );

    delay_us ( 1 );

    WriteRawRC ( ModeReg, 0x3D );            //定义发送和接收常用模式 和Mifare卡通讯，CRC初始值0x6363

    WriteRawRC ( TReloadRegL, 30 );          //16位定时器低位
    WriteRawRC ( TReloadRegH, 0 );			 //16位定时器高位

    WriteRawRC ( TModeReg, 0x8D );		      //定义内部定时器的设置

    WriteRawRC ( TPrescalerReg, 0x3E );			 //设置定时器分频系数

    WriteRawRC ( TxAutoReg, 0x40 );				   //调制发送信号为100%ASK


}


/*
 * 函数名：M500PcdConfigISOType
 * 描述  ：设置RC522的工作方式
 * 输入  ：ucType，工作方式
 * 返回  : 无
 * 调用  ：外部调用
 */
void M500PcdConfigISOType ( u8 ucType )
{
    if ( ucType == 'A')                     //ISO14443_A
    {
        ClearBitMask ( Status2Reg, 0x08 );

        WriteRawRC ( ModeReg, 0x3D );//3F

        WriteRawRC ( RxSelReg, 0x86 );//84

        WriteRawRC( RFCfgReg, 0x7F );   //4F

        WriteRawRC( TReloadRegL, 30 );//tmoLength);// TReloadVal = 'h6a =tmoLength(dec)

        WriteRawRC ( TReloadRegH, 0 );

        WriteRawRC ( TModeReg, 0x8D );

        WriteRawRC ( TPrescalerReg, 0x3E );

        delay_us ( 2 );

        PcdAntennaOn ();//开天线

    }

}


/*
 * 函数名：PcdComMF522
 * 描述  ：通过RC522和ISO14443卡通讯
 * 输入  ：ucCommand，RC522命令字
 *         pInData，通过RC522发送到卡片的数据
 *         ucInLenByte，发送数据的字节长度
 *         pOutData，接收到的卡片返回数据
 *         pOutLenBit，返回数据的位长度
 * 返回  : 状态值
 *         = MI_OK，成功
 * 调用  ：内部调用
 */
char PcdComMF522 ( u8 ucCommand, u8 * pInData, u8 ucInLenByte, u8 * pOutData, u32 * pOutLenBit )
{
    char cStatus = MI_ERR;
    u8 ucIrqEn   = 0x00;
    u8 ucWaitFor = 0x00;
    u8 ucLastBits;
    u8 ucN;
    u32 ul;

    switch ( ucCommand )
    {
    case PCD_AUTHENT:		//Mifare认证
        ucIrqEn   = 0x12;		//允许错误中断请求ErrIEn  允许空闲中断IdleIEn
        ucWaitFor = 0x10;		//认证寻卡等待时候 查询空闲中断标志位
        break;

    case PCD_TRANSCEIVE:		//接收发送 发送接收
        ucIrqEn   = 0x77;		//允许TxIEn RxIEn IdleIEn LoAlertIEn ErrIEn TimerIEn
        ucWaitFor = 0x30;		//寻卡等待时候 查询接收中断标志位与 空闲中断标志位
        break;

    default:
        break;

    }

    WriteRawRC ( ComIEnReg, ucIrqEn | 0x80 );		//IRqInv置位管脚IRQ与Status1Reg的IRq位的值相反
    ClearBitMask ( ComIrqReg, 0x80 );			//Set1该位清零时，CommIRqReg的屏蔽位清零
    WriteRawRC ( CommandReg, PCD_IDLE );		//写空闲命令
    SetBitMask ( FIFOLevelReg, 0x80 );			//置位FlushBuffer清除内部FIFO的读和写指针以及ErrReg的BufferOvfl标志位被清除

    for ( ul = 0; ul < ucInLenByte; ul ++ )
        WriteRawRC ( FIFODataReg, pInData [ ul ] );    		//写数据进FIFOdata

    WriteRawRC ( CommandReg, ucCommand );					//写命令


    if ( ucCommand == PCD_TRANSCEIVE )
        SetBitMask(BitFramingReg,0x80);  				//StartSend置位启动数据发送 该位与收发命令使用时才有效

    ul = 1000;//根据时钟频率调整，操作M1卡最大等待时间25ms

    do 														//认证 与寻卡等待时间
    {
        ucN = ReadRawRC ( ComIrqReg );							//查询事件中断
        ul --;
    } while ( ( ul != 0 ) && ( ! ( ucN & 0x01 ) ) && ( ! ( ucN & ucWaitFor ) ) );		//退出条件i=0,定时器中断，与写空闲命令

    ClearBitMask ( BitFramingReg, 0x80 );					//清理允许StartSend位

    if ( ul != 0 )
    {
        if ( ! (( ReadRawRC ( ErrorReg ) & 0x1B )) )			//读错误标志寄存器BufferOfI CollErr ParityErr ProtocolErr
        {
            cStatus = MI_OK;

            if ( ucN & ucIrqEn & 0x01 )					//是否发生定时器中断
                cStatus = MI_NOTAGERR;

            if ( ucCommand == PCD_TRANSCEIVE )
            {
                ucN = ReadRawRC ( FIFOLevelReg );			//读FIFO中保存的字节数

                ucLastBits = ReadRawRC ( ControlReg ) & 0x07;	//最后接收到得字节的有效位数

                if ( ucLastBits )
                    * pOutLenBit = ( ucN - 1 ) * 8 + ucLastBits;   	//N个字节数减去1（最后一个字节）+最后一位的位数 读取到的数据总位数
                else
                    * pOutLenBit = ucN * 8;   					//最后接收到的字节整个字节有效

                if ( ucN == 0 )
                    ucN = 1;

                if ( ucN > MAXRLEN )
                    ucN = MAXRLEN;

                for ( ul = 0; ul < ucN; ul ++ )
                    pOutData [ ul ] = ReadRawRC ( FIFODataReg );
            }
        }
        else
            cStatus = MI_ERR;
//			printf(ErrorReg);
    }

    SetBitMask ( ControlReg, 0x80 );           // stop timer now
    WriteRawRC ( CommandReg, PCD_IDLE );

    return cStatus;

}


/*
 * 函数名：PcdRequest
 * 描述  ：寻卡
 * 输入  ：ucReq_code，寻卡方式
 *                     = 0x52，寻感应区内所有符合14443A标准的卡
 *                     = 0x26，寻未进入休眠状态的卡
 *         pTagType，卡片类型代码
 *                   = 0x4400，Mifare_UltraLight
 *                   = 0x0400，Mifare_One(S50)
 *                   = 0x0200，Mifare_One(S70)
 *                   = 0x0800，Mifare_Pro(X))
 *                   = 0x4403，Mifare_DESFire
 * 返回  : 状态值
 *         = MI_OK，成功
 * 调用  ：外部调用
 */
char PcdRequest ( u8 ucReq_code, u8 * pTagType )
{
    char cStatus;
    u8 ucComMF522Buf [ MAXRLEN ];
    u32 ulLen;

    ClearBitMask ( Status2Reg, 0x08 );	//清理指示MIFARECyptol单元接通以及所有卡的数据通信被加密的情况
    WriteRawRC ( BitFramingReg, 0x07 );	//	发送的最后一个字节的 七位
    SetBitMask ( TxControlReg, 0x03 );	//TX1,TX2管脚的输出信号传递经发送调制的13.56的能量载波信号

    ucComMF522Buf [ 0 ] = ucReq_code;		//存入 卡片命令字

    cStatus = PcdComMF522 ( PCD_TRANSCEIVE,	ucComMF522Buf, 1, ucComMF522Buf, & ulLen );	//寻卡

    if ( ( cStatus == MI_OK ) && ( ulLen == 0x10 ) )	//寻卡成功返回卡类型
    {
        * pTagType = ucComMF522Buf [ 0 ];
        * ( pTagType + 1 ) = ucComMF522Buf [ 1 ];
    }

    else
        cStatus = MI_ERR;

    return cStatus;

}


/*
 * 函数名：PcdAnticoll
 * 描述  ：防冲撞
 * 输入  ：pSnr，卡片序列号，4字节
 * 返回  : 状态值
 *         = MI_OK，成功
 * 调用  ：外部调用
 */
char PcdAnticoll ( u8 * pSnr )
{
    char cStatus;
    u8 uc, ucSnr_check = 0;
    u8 ucComMF522Buf [ MAXRLEN ];
    u32 ulLen;

    ClearBitMask ( Status2Reg, 0x08 );		//清MFCryptol On位 只有成功执行MFAuthent命令后，该位才能置位
    WriteRawRC ( BitFramingReg, 0x00);		//清理寄存器 停止收发
    ClearBitMask ( CollReg, 0x80 );			//清ValuesAfterColl所有接收的位在冲突后被清除

    ucComMF522Buf [ 0 ] = 0x93;	//卡片防冲突命令
    ucComMF522Buf [ 1 ] = 0x20;

    cStatus = PcdComMF522 ( PCD_TRANSCEIVE, ucComMF522Buf, 2, ucComMF522Buf, & ulLen);//与卡片通信

    if ( cStatus == MI_OK)		//通信成功
    {
        for ( uc = 0; uc < 4; uc ++ )
        {
            * ( pSnr + uc )  = ucComMF522Buf [ uc ];			//读出UID
            ucSnr_check ^= ucComMF522Buf [ uc ];
        }

        if ( ucSnr_check != ucComMF522Buf [ uc ] )
            cStatus = MI_ERR;

    }

    SetBitMask ( CollReg, 0x80 );

    return cStatus;

}


/*
 * 函数名：CalulateCRC
 * 描述  ：用RC522计算CRC16
 * 输入  ：pIndata，计算CRC16的数组
 *         ucLen，计算CRC16的数组字节长度
 *         pOutData，存放计算结果存放的首地址
 * 返回  : 无
 * 调用  ：内部调用
 */
void CalulateCRC ( u8 * pIndata, u8 ucLen, u8 * pOutData )
{
    u8 uc, ucN;

    ClearBitMask(DivIrqReg,0x04);

    WriteRawRC(CommandReg,PCD_IDLE);

    SetBitMask(FIFOLevelReg,0x80);

    for ( uc = 0; uc < ucLen; uc ++)
        WriteRawRC ( FIFODataReg, * ( pIndata + uc ) );

    WriteRawRC ( CommandReg, PCD_CALCCRC );

    uc = 0xFF;

    do
    {
        ucN = ReadRawRC ( DivIrqReg );
        uc --;
    } while ( ( uc != 0 ) && ! ( ucN & 0x04 ) );

    pOutData [ 0 ] = ReadRawRC ( CRCResultRegL );
    pOutData [ 1 ] = ReadRawRC ( CRCResultRegM );

}


/*
 * 函数名：PcdSelect
 * 描述  ：选定卡片
 * 输入  ：pSnr，卡片序列号，4字节
 * 返回  : 状态值
 *         = MI_OK，成功
 * 调用  ：外部调用
 */
char PcdSelect ( u8 * pSnr )
{
    char ucN;
    u8 uc;
    u8 ucComMF522Buf [ MAXRLEN ];
    u32  ulLen;

    ucComMF522Buf [ 0 ] = PICC_ANTICOLL1;
    ucComMF522Buf [ 1 ] = 0x70;
    ucComMF522Buf [ 6 ] = 0;

    for ( uc = 0; uc < 4; uc ++ )
    {
        ucComMF522Buf [ uc + 2 ] = * ( pSnr + uc );
        ucComMF522Buf [ 6 ] ^= * ( pSnr + uc );
    }

    CalulateCRC ( ucComMF522Buf, 7, & ucComMF522Buf [ 7 ] );

    ClearBitMask ( Status2Reg, 0x08 );

    ucN = PcdComMF522 ( PCD_TRANSCEIVE, ucComMF522Buf, 9, ucComMF522Buf, & ulLen );

    if ( ( ucN == MI_OK ) && ( ulLen == 0x18 ) )
        ucN = MI_OK;
    else
        ucN = MI_ERR;

    return ucN;

}


/*
 * 函数名：PcdAuthState
 * 描述  ：验证卡片密码
 * 输入  ：ucAuth_mode，密码验证模式
 *                     = 0x60，验证A密钥
 *                     = 0x61，验证B密钥
 *         u8 ucAddr，块地址
 *         pKey，密码
 *         pSnr，卡片序列号，4字节
 * 返回  : 状态值
 *         = MI_OK，成功
 * 调用  ：外部调用
 */
char PcdAuthState ( u8 ucAuth_mode, u8 ucAddr, u8 * pKey, u8 * pSnr )
{
    char cStatus;
    u8 uc, ucComMF522Buf [ MAXRLEN ];
    u32 ulLen;

    ucComMF522Buf [ 0 ] = ucAuth_mode;
    ucComMF522Buf [ 1 ] = ucAddr;

    for ( uc = 0; uc < 6; uc ++ )
        ucComMF522Buf [ uc + 2 ] = * ( pKey + uc );

    for ( uc = 0; uc < 6; uc ++ )
        ucComMF522Buf [ uc + 8 ] = * ( pSnr + uc );

    cStatus = PcdComMF522 ( PCD_AUTHENT, ucComMF522Buf, 12, ucComMF522Buf, & ulLen );

    if ( ( cStatus != MI_OK ) || ( ! ( ReadRawRC ( Status2Reg ) & 0x08 ) ) )

    {
//			if(cStatus != MI_OK)
//					printf("666")	;
//			else
//				printf("888");
        cStatus = MI_ERR;
    }

    return cStatus;

}


/*
 * 函数名：PcdWrite
 * 描述  ：写数据到M1卡一块
 * 输入  ：u8 ucAddr，块地址
 *         pData，写入的数据，16字节
 * 返回  : 状态值
 *         = MI_OK，成功
 * 调用  ：外部调用
 */
char PcdWrite ( u8 ucAddr, u8 * pData )
{
    char cStatus;
    u8 uc, ucComMF522Buf [ MAXRLEN ];
    u32 ulLen;

    ucComMF522Buf [ 0 ] = PICC_WRITE;
    ucComMF522Buf [ 1 ] = ucAddr;

    CalulateCRC ( ucComMF522Buf, 2, & ucComMF522Buf [ 2 ] );

    cStatus = PcdComMF522 ( PCD_TRANSCEIVE, ucComMF522Buf, 4, ucComMF522Buf, & ulLen );

    if ( ( cStatus != MI_OK ) || ( ulLen != 4 ) || ( ( ucComMF522Buf [ 0 ] & 0x0F ) != 0x0A ) )
        cStatus = MI_ERR;

    if ( cStatus == MI_OK )
    {
        memcpy(ucComMF522Buf, pData, 16);
        for ( uc = 0; uc < 16; uc ++ )
            ucComMF522Buf [ uc ] = * ( pData + uc );

        CalulateCRC ( ucComMF522Buf, 16, & ucComMF522Buf [ 16 ] );

        cStatus = PcdComMF522 ( PCD_TRANSCEIVE, ucComMF522Buf, 18, ucComMF522Buf, & ulLen );

        if ( ( cStatus != MI_OK ) || ( ulLen != 4 ) || ( ( ucComMF522Buf [ 0 ] & 0x0F ) != 0x0A ) )
            cStatus = MI_ERR;

    }
    return cStatus;
}


/*
 * 函数名：PcdRead
 * 描述  ：读取M1卡一块数据
 * 输入  ：u8 ucAddr，块地址
 *         pData，读出的数据，16字节
 * 返回  : 状态值
 *         = MI_OK，成功
 * 调用  ：外部调用
 */
char PcdRead ( u8 ucAddr, u8 * pData )
{
    char cStatus;
    u8 uc, ucComMF522Buf [ MAXRLEN ];
    u32 ulLen;

    ucComMF522Buf [ 0 ] = PICC_READ;
    ucComMF522Buf [ 1 ] = ucAddr;

    CalulateCRC ( ucComMF522Buf, 2, & ucComMF522Buf [ 2 ] );

    cStatus = PcdComMF522 ( PCD_TRANSCEIVE, ucComMF522Buf, 4, ucComMF522Buf, & ulLen );

    if ( ( cStatus == MI_OK ) && ( ulLen == 0x90 ) )
    {
        for ( uc = 0; uc < 16; uc ++ )
            * ( pData + uc ) = ucComMF522Buf [ uc ];
    }

    else
        cStatus = MI_ERR;

    return cStatus;

}


/*
 * 函数名：PcdHalt
 * 描述  ：命令卡片进入休眠状态
 * 输入  ：无
 * 返回  : 状态值
 *         = MI_OK，成功
 * 调用  ：外部调用
 */
char PcdHalt( void )
{
    u8 ucComMF522Buf [ MAXRLEN ];
    u32  ulLen;

    ucComMF522Buf [ 0 ] = PICC_HALT;
    ucComMF522Buf [ 1 ] = 0;

    CalulateCRC ( ucComMF522Buf, 2, & ucComMF522Buf [ 2 ] );
    PcdComMF522 ( PCD_TRANSCEIVE, ucComMF522Buf, 4, ucComMF522Buf, & ulLen );

    return MI_OK;

}


void IC_CMT ( u8 * UID, u8 * KEY, u8 RW, u8 * Dat )
{
    u8 ucArray_ID [ 4 ] = { 0 };//先后存放IC卡的类型和UID(IC卡序列号)

    PcdRequest ( 0x52, ucArray_ID );//寻卡

    PcdAnticoll ( ucArray_ID );//防冲撞

    PcdSelect ( UID );//选定卡

    PcdAuthState ( 0x60, 0x10, KEY, UID );//校验

    if ( RW )//读写选择，1是读，0是写
        PcdRead ( 0x10, Dat );

    else
        PcdWrite ( 0x10, Dat );

    PcdHalt ();

}

void ShowID(u8 *p)	 //显示卡的卡号，以十六进制显示
{
    u8 num[9];
    u8 i;

    for(i=0; i<4; i++)
    {
        num[i*2]=p[i]/16;
        num[i*2]>9?(num[i*2]+='7'):(num[i*2]+='0');
        num[i*2+1]=p[i]%16;
        num[i*2+1]>9?(num[i*2+1]+='7'):(num[i*2+1]+='0');
    }
    num[8]=0;
    printf("ID>>>%s\r\n", num);
}


```
## 工程下载
完整工程下载:https://download.csdn.net/download/qq_26914291/10705432