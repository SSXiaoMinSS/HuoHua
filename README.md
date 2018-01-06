#include<reg52.h>            //包含头文件 
#include<intrins.h>
#define LCD_Data P0
#define Busy 0x80

sbit IR = P3^2;

sbit RS = P1^0;              //寄存器选择位，将RS位定义为P2.0引脚 
sbit RW = P1^1;              //读写选择位，将RW位定义为P2.1引脚 
sbit E  = P2^5;              //使能信号位，将E位定义为P2.2引脚 
sbit BF = P0^7;              //忙碌标志位，，将BF位定义为P0.7引脚

sbit LED1 = P2^2;
sbit LED2 = P2^3;
sbit LED3 = P2^4;

sbit BEEP   = P2^6;

sbit SIGNAL = P1^2;

sbit KEY1 = P3^7;
sbit KEY2 = P3^6;
sbit KEY3 = P3^5;

unsigned char IRcord[4];
unsigned char IRdata[33];
unsigned char code dofly_DuanMa[10]={0x3f,0x06,0x5b,0x4f,0x66,0x6d,0x7d,0x07,0x7f,0x6f};
unsigned char irtime;

unsigned char code welcome[] = {"Welcome"};      //液晶显示字符定义
unsigned char code name[]    = {"Home Alarm"};	//液晶显示字符定义
unsigned char code startup[] = {"Start Up"};     //液晶显示字符定义
unsigned char code standby[] = {"Standby"};	   //液晶显示字符定义
unsigned char code working[] = {"Working"};		//液晶显示字符定义

bit startup_flag = 0;
bit standby_flag = 0;
bit working_flag = 0;

bit IRpro_ok;
bit IR_ok;

void delay1ms();
void delay(unsigned char n);
unsigned char BusyTest(void);
void WriteCommandLCD(unsigned char WCLCD,BuysC);
void LCD_Clear(void); 
void WriteAddress(unsigned char x); 
void WriteData(unsigned char y);   
void DisplayOneChar(unsigned char X, unsigned char Y, unsigned char  DData);
void DisplayListChar(unsigned char X, unsigned char Y, unsigned char code *DData);
void LCDInit(void);

void sys_init(void);
void key_scan(void);
void process(void);

void TIM0init(void);                      //定时器0初始化函数 
void EX0init(void);							//外部中断初始化函数

void Ir_work(void);
void Ircordpro(void);

/************************************************
函数功能：延时1ms  (3j+2)*i=(3×33+2)×10=1010(微秒)，
可以认为是1毫秒	 
************************************************/
void delay1ms() 
{     
 unsigned char i,j;    
   for(i=0;i<10;i++)    
     for(j=0;j<33;j++)     
       ;     
}

/************************************************ 
函数功能：延时若干毫秒 
入口参数：n  
************************************************/  
void delay(unsigned char n)  
{     
 unsigned char i;  
   for(i=0;i<n;i++)     
     delay1ms();  
}

/************************************************
函数功能：判断液晶模块的忙碌状态  
返回值：result。result=1，忙碌;result=0，不忙  
************************************************/  
unsigned char BusyTest(void)   
{     
 bit result;  
 RS = 0;             //根据规定，RS为低电平，RW为高电平时，可以读状态     
 RW = 1;      
 E  = 1;            //E=1，才允许读写     
 _nop_();           //空操作     
 _nop_();     
 _nop_();       
 _nop_();           //空操作四个机器周期，给硬件反应时间      
 result = BF;       //将忙碌标志电平赋给result  
 E = 0;      
 return result;  
}

/******************写指令函数*******************/
void WriteCommandLCD(unsigned char WCLCD,BuysC)  
//BuysC为0时忽略忙检测	    
{
 if (BuysC) 			           //根据需要检测忙
 {
  while(BusyTest()==1);         //如果忙就等待     
 }                 	 
 RS = 0;            //根据规定，RS和R/W同时为低电平时，可以写入指令

 RW = 0;      
 E  = 0;            //E置低电平(根据表8-6，写指令时，E为高脉冲，                             
                    //就是让E从0到1发生正跳变，所以应先置"0"  
 _nop_();   
 _nop_();           //空操作两个机器周期，给硬件反应时间   
 
 P0 = WCLCD;        //将数据送入P0口，即写入指令或地址 
 _nop_();   
 _nop_();   
 _nop_();   
 _nop_();           //空操作四个机器周期，给硬件反应时间   
 
 E = 1;             //E置高电平
 _nop_();   
 _nop_();   
 _nop_();   
 _nop_();           //空操作四个机器周期，给硬件反应时间    
 
 E = 0;             //当E由高电平跳变成低电平时，液晶模块开始执行命令  
}

/*************************************************
 函数功能：指定字符显示的实际地址 
入口参数：x  
*************************************************/  
void WriteAddress(unsigned char x)  
{       
 WriteCommandLCD(x|0x80,0);	 //显示位置的确定方法规定为"80H+地址码x"  
} 

/*************************************************
清屏函数			   
*************************************************/
void LCD_Clear(void) 
{ 
 WriteCommandLCD(0x01,1); 
 delay(5);
}

/************************************************ 
函数功能：将数据(字符的标准ASCII码)写入液晶模块 
入口参数：y(为字符常量)  
************************************************/
void WriteData(unsigned char y)  
{      
 while(BusyTest()==1);      
 RS = 1;                 //RS为高电平，RW为低电平时，可以写入数据    
 RW = 0;    
 E  = 0;                 //E置低电平(根据表8-6，写指令时，E为高脉冲，                        
                         //就是让E从0到1发生正跳变，所以应先置"0"    
 P0 = y;                 //将数据送入P0口，即将数据写入液晶模块
 
 _nop_();    
 _nop_();     
 _nop_();       
 _nop_();                //空操作四个机器周期，给硬件反应时间    
 
 E = 1;                  //E置高电平
 _nop_();    
 _nop_();
 _nop_();   
 _nop_();          //空操作四个机器周期，给硬件反应时间   
 
 E = 0;             //当E由高电平跳变成低电平时，液晶模块开始执行命令 
} 

/**********************按指定位置显示一个字符*********************/
void DisplayOneChar(unsigned char X, unsigned char Y, unsigned char DData)
{
 Y &= 0x1;
 X &= 0xF;                          //X不能大于15，Y不能大于1	    
 if (Y) X |= 0x40;                  //当要显示第二行时地址码+0x40;

 WriteAddress(X|0x80);  
 WriteData(DData);
}

/***********************按指定位置显示一串字符********************/
void DisplayListChar(unsigned char X, unsigned char Y, unsigned char code *DData)
{
 unsigned char ListLength;

 ListLength = 0;
 Y &= 0x1;
 X &= 0xF;                           //X不能大于15，Y不能大于1	 
 while (DData[ListLength]>=0x20)     //若到达字串尾则退出	  
 {
  if (X <= 0xF)                               //X坐标应小于0xF  
  {
   DisplayOneChar(X, Y, DData[ListLength]);   //显示单个字符	 
   ListLength++;
   X++;
  }
 }
}

/************************************************** 
函数功能：对LCD的显示模式进行初始化设置  
/*************************************************/
void LCDInit(void)                  //LCD初始化 
{
 delay(15); 
 WriteCommandLCD(0x38,0);           //三次显示模式设置，不检测忙信号	   
 delay(5); 
 WriteCommandLCD(0x38,0);
 delay(5); 
 WriteCommandLCD(0x38,0);
 delay(5); 

 WriteCommandLCD(0x38,1);     //显示模式设置, 开始要求每次检测忙信号	 
 delay(5);    
 WriteCommandLCD(0x08,1);           //关闭显示 
 delay(5); 
 WriteCommandLCD(0x01,1);           //显示清屏 
 delay(5); 
 WriteCommandLCD(0x06,1);           //显示光标移动设置  
 delay(5);  
 WriteCommandLCD(0x0C,1);           //显示开及光标设置   
 delay(5); 
}

void sys_init()	 
{
 LED1 = 1;						       //LED1熄灭
 LED2 = 1;							   //LED2熄灭
 LED3 = 1;						       //LED3熄灭

 BEEP = 1;							   //蜂鸣器不响

 LCDInit();  						   //液晶初始化
 DisplayListChar(5, 0, welcome);	//液晶1602显示欢迎字符

 startup_flag = 0;			       //启动状态标志位 
 standby_flag = 0;					//待机状态标志位
 working_flag = 0;			       //工作状态标志位

 TIM0init();			               //定时器0初始化 
 EX0init();							 //外部中断初始化 
}

void key_scan(void)
{
 if(KEY1 == 0)                  //判断按键1是否按下
 {
  delay(10);
  if(KEY1 == 0)
  {
   startup_flag = 1;            //系统启动标志位设置为1 		    
   standby_flag = 0;				//系统待机标志位设置为0 
   working_flag = 0;				 //系统工作标志位设置为0

   LCD_Clear();						//液晶1602清屏
   DisplayListChar(3, 0, name);		//液晶1602显示启动状态信息
   DisplayListChar(4, 1, startup);
  }
 }

 if(KEY2 == 0)					  //判断按键2是否按下
 {
  delay(10);
  if(KEY2 == 0)				            
  {
   startup_flag = 0;				   //系统启动标志位设置为0 
   standby_flag = 1;				   //系统待机标志位设置为1
   working_flag = 0;	 			   //系统工作标志位设置为0

   LCD_Clear();						//液晶1602清屏
   DisplayListChar(3, 0, name);		//液晶1602显示待机状态信息
   DisplayListChar(5, 1, standby);
  }
 }

 if(KEY3 == 0)			           //判断按键3是否按下
 {
  delay(10);
  if(KEY3 == 0)
  {
   startup_flag = 0;	             //系统启动标志位设置为0
   standby_flag = 0;				   //系统待机标志位设置为0
   working_flag = 1;				   //系统工作标志位设置为1
	 
   LCD_Clear();						   //液晶1602清屏
   DisplayListChar(3, 0, name);		   //液晶1602显示工作状态信息
   DisplayListChar(5, 1, working);	 
  }
 }
}

void Ir_work(void)                //红外键值散转程序	  
{
 switch(IRcord[2])                //判断第三个数码值	   
 {
  case 0x0C:				          //当接收到的码值为0x0C时
  {
   startup_flag = 1;				   //系统启动标志位设置为1
   standby_flag = 0;				   //系统待机标志位设置为0
   working_flag = 0;				   //系统工作标志位设置为0

   LCD_Clear();						   //液晶1602清屏
   DisplayListChar(3, 0, name);		   //液晶1602显示工作状态信息
   DisplayListChar(4, 1, startup);
  }break; 
  			 
  case 0x18:			               //当接收到的码值为0x18时                             	    
  {
   startup_flag = 0;				   //系统启动标志位设置为0
   standby_flag = 1;				   //系统待机标志位设置为1
   working_flag = 0;	 			   //系统工作标志位设置为0

   LCD_Clear();						   //液晶1602清屏
   DisplayListChar(3, 0, name);		   //液晶1602显示工作状态信息
   DisplayListChar(5, 1, standby);                                  	
  }break; 
    
  case 0x5e:					       //当接收到的码值为0x5e时
  {
   startup_flag = 0;				   //系统启动标志位设置为0
   standby_flag = 0;				   //系统待机标志位设置为0
   working_flag = 1;				   //系统工作标志位设置为1
	 
   LCD_Clear();						   //液晶1602清屏
   DisplayListChar(3, 0, name);		   //液晶1602显示工作状态信息
   DisplayListChar(5, 1, working);     
  }break; 
                      
  default:break;
 }
}

void Ircordpro(void)                       //红外码值处理函数	    
{ 
 unsigned char i, j, k;
 unsigned char cord,value;

 k=1;
 for(i=0;i<4;i++)                          //处理4个字节	 
 {
  for(j=1;j<=8;j++)                        //处理1个字节8位
  {
   cord=IRdata[k];
   if(cord>7)                              //大于某值为1，这个和晶振
   有绝对关系，这里使用12M计算，此值可以有一定误差			    
   value|=0x80;
   if(j<8)
   {
	value>>=1;
   }
   k++;
  }
  IRcord[i]=value;
  value=0;     
 } 
 IRpro_ok=1;                               //处理完毕标志位置1		    
}

void process(void)				//系统处理函数定义 
{															   
 if(SIGNAL == 1)	              //判断信号输入是否为高电平 	 
 {
  delay(10);	                  //延时10ms	  
  if(SIGNAL == 1)               //第二次判断信号输入是否为高电平 	 	 
  {
   LED1  = 0; 					 //LED1二极管发光 
   LED2  = 1;
   LED3  = 1;	      		      //LED3二极管不发光
   BEEP  = 0;		             //蜂鸣器响
  }
  else
  {
   LED1 = 1;					  //LED1二极管熄灭 
   LED2 = 1;					  //LED1二极管熄灭 
   LED3 = 1;                  //LED3熄灭
   BEEP = 1;					  //蜂鸣器不响 
  }
 }
 else
 {
  LED1 = 1;					  //LED1二极管熄灭 
  LED2 = 1;					  //LED1二极管熄灭 
  LED3 = 1;                   //LED3熄灭
  BEEP = 1;					  //蜂鸣器不响 
 }
}

void TIM0init(void)      //定时器0初始化	   
{
 TMOD = 0x02;            //定时器0工作方式2，TH0是重装值，TL0是初值	   
 TH0  = 0x00;                   //重载值	  
 TL0  = 0x00;                   //初始化值    
 ET0  = 1;                      //开中断	  
 TR0  = 1;    
}

void EX0init(void)
{
 IT0 = 1;                         //指定外部中断0下降沿触发，INT0 (P3.2)			 
 EX0 = 1;                        //使能外部中断	   
 EA = 1;                         //开总中断	  
}

void main(void) 
{ 
 sys_init();                     //初始化

 while (1) 
 {
  key_scan();                    //按键扫描

  if(IR_ok)                      //如果接收好了进行红外处理		 
  {   
   Ircordpro();
   IR_ok=0;
  }
  if(IRpro_ok)                   //如果处理好后进行工作处理，如按对应
  的按键后显示对应的数字等			    
  {
   Ir_work();
   IRpro_ok = 0;
  }

  if((startup_flag == 1)&&(standby_flag == 0)&&(working_flag == 0))
  {
   LED1 = 1;
   LED2 = 1;
   LED3 = 0;

   BEEP = 1;
  }
  if((startup_flag == 0)&&(standby_flag == 1)&&(working_flag == 0))
  {
   LED1 = 1;
   LED2 = 0;
   LED3 = 1;

   BEEP = 1;
  }

  if((startup_flag == 0)&&(standby_flag == 0)&&(working_flag == 1))
  {      
   process();
  }
 }
}

void tim0_isr (void) interrupt 1 using 1
{
 irtime++;                             //用于计数2个下降沿之间的时间	    
}

void EX0_ISR (void) interrupt 0        //外部中断0服务函数		   
{
 static unsigned char  i;              //接收红外信号处理		   
 static bit startflag;                 //是否开始处理标志位

 if(startflag)                         
 {
  if(irtime<63&&irtime>=33)        //引导码 TC9012的头码，9ms+4.5ms		  
  i = 0;
  IRdata[i] = irtime;               //存储每个电平的持续时间，用于以后
  判断是0还是1			  
  irtime = 0;
  i++;
  if(i == 33)
  {
   IR_ok=1;
   i=0;
  }
 }
 else
 {
  irtime=0;
  startflag=1;
 }
}
