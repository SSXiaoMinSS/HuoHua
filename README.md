#include<reg52.h>            //����ͷ�ļ� 
#include<intrins.h>
#define LCD_Data P0
#define Busy 0x80

sbit IR = P3^2;

sbit RS = P1^0;              //�Ĵ���ѡ��λ����RSλ����ΪP2.0���� 
sbit RW = P1^1;              //��дѡ��λ����RWλ����ΪP2.1���� 
sbit E  = P2^5;              //ʹ���ź�λ����Eλ����ΪP2.2���� 
sbit BF = P0^7;              //æµ��־λ������BFλ����ΪP0.7����

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

unsigned char code welcome[] = {"Welcome"};      //Һ����ʾ�ַ�����
unsigned char code name[]    = {"Home Alarm"};	//Һ����ʾ�ַ�����
unsigned char code startup[] = {"Start Up"};     //Һ����ʾ�ַ�����
unsigned char code standby[] = {"Standby"};	   //Һ����ʾ�ַ�����
unsigned char code working[] = {"Working"};		//Һ����ʾ�ַ�����

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

void TIM0init(void);                      //��ʱ��0��ʼ������ 
void EX0init(void);							//�ⲿ�жϳ�ʼ������

void Ir_work(void);
void Ircordpro(void);

/************************************************
�������ܣ���ʱ1ms  (3j+2)*i=(3��33+2)��10=1010(΢��)��
������Ϊ��1����	 
************************************************/
void delay1ms() 
{     
 unsigned char i,j;    
   for(i=0;i<10;i++)    
     for(j=0;j<33;j++)     
       ;     
}

/************************************************ 
�������ܣ���ʱ���ɺ��� 
��ڲ�����n  
************************************************/  
void delay(unsigned char n)  
{     
 unsigned char i;  
   for(i=0;i<n;i++)     
     delay1ms();  
}

/************************************************
�������ܣ��ж�Һ��ģ���æµ״̬  
����ֵ��result��result=1��æµ;result=0����æ  
************************************************/  
unsigned char BusyTest(void)   
{     
 bit result;  
 RS = 0;             //���ݹ涨��RSΪ�͵�ƽ��RWΪ�ߵ�ƽʱ�����Զ�״̬     
 RW = 1;      
 E  = 1;            //E=1���������д     
 _nop_();           //�ղ���     
 _nop_();     
 _nop_();       
 _nop_();           //�ղ����ĸ��������ڣ���Ӳ����Ӧʱ��      
 result = BF;       //��æµ��־��ƽ����result  
 E = 0;      
 return result;  
}

/******************дָ���*******************/
void WriteCommandLCD(unsigned char WCLCD,BuysC)  
//BuysCΪ0ʱ����æ���	    
{
 if (BuysC) 			           //������Ҫ���æ
 {
  while(BusyTest()==1);         //���æ�͵ȴ�     
 }                 	 
 RS = 0;            //���ݹ涨��RS��R/WͬʱΪ�͵�ƽʱ������д��ָ��

 RW = 0;      
 E  = 0;            //E�õ͵�ƽ(���ݱ�8-6��дָ��ʱ��EΪ�����壬                             
                    //������E��0��1���������䣬����Ӧ����"0"  
 _nop_();   
 _nop_();           //�ղ��������������ڣ���Ӳ����Ӧʱ��   
 
 P0 = WCLCD;        //����������P0�ڣ���д��ָ����ַ 
 _nop_();   
 _nop_();   
 _nop_();   
 _nop_();           //�ղ����ĸ��������ڣ���Ӳ����Ӧʱ��   
 
 E = 1;             //E�øߵ�ƽ
 _nop_();   
 _nop_();   
 _nop_();   
 _nop_();           //�ղ����ĸ��������ڣ���Ӳ����Ӧʱ��    
 
 E = 0;             //��E�ɸߵ�ƽ����ɵ͵�ƽʱ��Һ��ģ�鿪ʼִ������  
}

/*************************************************
 �������ܣ�ָ���ַ���ʾ��ʵ�ʵ�ַ 
��ڲ�����x  
*************************************************/  
void WriteAddress(unsigned char x)  
{       
 WriteCommandLCD(x|0x80,0);	 //��ʾλ�õ�ȷ�������涨Ϊ"80H+��ַ��x"  
} 

/*************************************************
��������			   
*************************************************/
void LCD_Clear(void) 
{ 
 WriteCommandLCD(0x01,1); 
 delay(5);
}

/************************************************ 
�������ܣ�������(�ַ��ı�׼ASCII��)д��Һ��ģ�� 
��ڲ�����y(Ϊ�ַ�����)  
************************************************/
void WriteData(unsigned char y)  
{      
 while(BusyTest()==1);      
 RS = 1;                 //RSΪ�ߵ�ƽ��RWΪ�͵�ƽʱ������д������    
 RW = 0;    
 E  = 0;                 //E�õ͵�ƽ(���ݱ�8-6��дָ��ʱ��EΪ�����壬                        
                         //������E��0��1���������䣬����Ӧ����"0"    
 P0 = y;                 //����������P0�ڣ���������д��Һ��ģ��
 
 _nop_();    
 _nop_();     
 _nop_();       
 _nop_();                //�ղ����ĸ��������ڣ���Ӳ����Ӧʱ��    
 
 E = 1;                  //E�øߵ�ƽ
 _nop_();    
 _nop_();
 _nop_();   
 _nop_();          //�ղ����ĸ��������ڣ���Ӳ����Ӧʱ��   
 
 E = 0;             //��E�ɸߵ�ƽ����ɵ͵�ƽʱ��Һ��ģ�鿪ʼִ������ 
} 

/**********************��ָ��λ����ʾһ���ַ�*********************/
void DisplayOneChar(unsigned char X, unsigned char Y, unsigned char DData)
{
 Y &= 0x1;
 X &= 0xF;                          //X���ܴ���15��Y���ܴ���1	    
 if (Y) X |= 0x40;                  //��Ҫ��ʾ�ڶ���ʱ��ַ��+0x40;

 WriteAddress(X|0x80);  
 WriteData(DData);
}

/***********************��ָ��λ����ʾһ���ַ�********************/
void DisplayListChar(unsigned char X, unsigned char Y, unsigned char code *DData)
{
 unsigned char ListLength;

 ListLength = 0;
 Y &= 0x1;
 X &= 0xF;                           //X���ܴ���15��Y���ܴ���1	 
 while (DData[ListLength]>=0x20)     //�������ִ�β���˳�	  
 {
  if (X <= 0xF)                               //X����ӦС��0xF  
  {
   DisplayOneChar(X, Y, DData[ListLength]);   //��ʾ�����ַ�	 
   ListLength++;
   X++;
  }
 }
}

/************************************************** 
�������ܣ���LCD����ʾģʽ���г�ʼ������  
/*************************************************/
void LCDInit(void)                  //LCD��ʼ�� 
{
 delay(15); 
 WriteCommandLCD(0x38,0);           //������ʾģʽ���ã������æ�ź�	   
 delay(5); 
 WriteCommandLCD(0x38,0);
 delay(5); 
 WriteCommandLCD(0x38,0);
 delay(5); 

 WriteCommandLCD(0x38,1);     //��ʾģʽ����, ��ʼҪ��ÿ�μ��æ�ź�	 
 delay(5);    
 WriteCommandLCD(0x08,1);           //�ر���ʾ 
 delay(5); 
 WriteCommandLCD(0x01,1);           //��ʾ���� 
 delay(5); 
 WriteCommandLCD(0x06,1);           //��ʾ����ƶ�����  
 delay(5);  
 WriteCommandLCD(0x0C,1);           //��ʾ�����������   
 delay(5); 
}

void sys_init()	 
{
 LED1 = 1;						       //LED1Ϩ��
 LED2 = 1;							   //LED2Ϩ��
 LED3 = 1;						       //LED3Ϩ��

 BEEP = 1;							   //����������

 LCDInit();  						   //Һ����ʼ��
 DisplayListChar(5, 0, welcome);	//Һ��1602��ʾ��ӭ�ַ�

 startup_flag = 0;			       //����״̬��־λ 
 standby_flag = 0;					//����״̬��־λ
 working_flag = 0;			       //����״̬��־λ

 TIM0init();			               //��ʱ��0��ʼ�� 
 EX0init();							 //�ⲿ�жϳ�ʼ�� 
}

void key_scan(void)
{
 if(KEY1 == 0)                  //�жϰ���1�Ƿ���
 {
  delay(10);
  if(KEY1 == 0)
  {
   startup_flag = 1;            //ϵͳ������־λ����Ϊ1 		    
   standby_flag = 0;				//ϵͳ������־λ����Ϊ0 
   working_flag = 0;				 //ϵͳ������־λ����Ϊ0

   LCD_Clear();						//Һ��1602����
   DisplayListChar(3, 0, name);		//Һ��1602��ʾ����״̬��Ϣ
   DisplayListChar(4, 1, startup);
  }
 }

 if(KEY2 == 0)					  //�жϰ���2�Ƿ���
 {
  delay(10);
  if(KEY2 == 0)				            
  {
   startup_flag = 0;				   //ϵͳ������־λ����Ϊ0 
   standby_flag = 1;				   //ϵͳ������־λ����Ϊ1
   working_flag = 0;	 			   //ϵͳ������־λ����Ϊ0

   LCD_Clear();						//Һ��1602����
   DisplayListChar(3, 0, name);		//Һ��1602��ʾ����״̬��Ϣ
   DisplayListChar(5, 1, standby);
  }
 }

 if(KEY3 == 0)			           //�жϰ���3�Ƿ���
 {
  delay(10);
  if(KEY3 == 0)
  {
   startup_flag = 0;	             //ϵͳ������־λ����Ϊ0
   standby_flag = 0;				   //ϵͳ������־λ����Ϊ0
   working_flag = 1;				   //ϵͳ������־λ����Ϊ1
	 
   LCD_Clear();						   //Һ��1602����
   DisplayListChar(3, 0, name);		   //Һ��1602��ʾ����״̬��Ϣ
   DisplayListChar(5, 1, working);	 
  }
 }
}

void Ir_work(void)                //�����ֵɢת����	  
{
 switch(IRcord[2])                //�жϵ���������ֵ	   
 {
  case 0x0C:				          //�����յ�����ֵΪ0x0Cʱ
  {
   startup_flag = 1;				   //ϵͳ������־λ����Ϊ1
   standby_flag = 0;				   //ϵͳ������־λ����Ϊ0
   working_flag = 0;				   //ϵͳ������־λ����Ϊ0

   LCD_Clear();						   //Һ��1602����
   DisplayListChar(3, 0, name);		   //Һ��1602��ʾ����״̬��Ϣ
   DisplayListChar(4, 1, startup);
  }break; 
  			 
  case 0x18:			               //�����յ�����ֵΪ0x18ʱ                             	    
  {
   startup_flag = 0;				   //ϵͳ������־λ����Ϊ0
   standby_flag = 1;				   //ϵͳ������־λ����Ϊ1
   working_flag = 0;	 			   //ϵͳ������־λ����Ϊ0

   LCD_Clear();						   //Һ��1602����
   DisplayListChar(3, 0, name);		   //Һ��1602��ʾ����״̬��Ϣ
   DisplayListChar(5, 1, standby);                                  	
  }break; 
    
  case 0x5e:					       //�����յ�����ֵΪ0x5eʱ
  {
   startup_flag = 0;				   //ϵͳ������־λ����Ϊ0
   standby_flag = 0;				   //ϵͳ������־λ����Ϊ0
   working_flag = 1;				   //ϵͳ������־λ����Ϊ1
	 
   LCD_Clear();						   //Һ��1602����
   DisplayListChar(3, 0, name);		   //Һ��1602��ʾ����״̬��Ϣ
   DisplayListChar(5, 1, working);     
  }break; 
                      
  default:break;
 }
}

void Ircordpro(void)                       //������ֵ������	    
{ 
 unsigned char i, j, k;
 unsigned char cord,value;

 k=1;
 for(i=0;i<4;i++)                          //����4���ֽ�	 
 {
  for(j=1;j<=8;j++)                        //����1���ֽ�8λ
  {
   cord=IRdata[k];
   if(cord>7)                              //����ĳֵΪ1������;���
   �о��Թ�ϵ������ʹ��12M���㣬��ֵ������һ�����			    
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
 IRpro_ok=1;                               //������ϱ�־λ��1		    
}

void process(void)				//ϵͳ���������� 
{															   
 if(SIGNAL == 1)	              //�ж��ź������Ƿ�Ϊ�ߵ�ƽ 	 
 {
  delay(10);	                  //��ʱ10ms	  
  if(SIGNAL == 1)               //�ڶ����ж��ź������Ƿ�Ϊ�ߵ�ƽ 	 	 
  {
   LED1  = 0; 					 //LED1�����ܷ��� 
   LED2  = 1;
   LED3  = 1;	      		      //LED3�����ܲ�����
   BEEP  = 0;		             //��������
  }
  else
  {
   LED1 = 1;					  //LED1������Ϩ�� 
   LED2 = 1;					  //LED1������Ϩ�� 
   LED3 = 1;                  //LED3Ϩ��
   BEEP = 1;					  //���������� 
  }
 }
 else
 {
  LED1 = 1;					  //LED1������Ϩ�� 
  LED2 = 1;					  //LED1������Ϩ�� 
  LED3 = 1;                   //LED3Ϩ��
  BEEP = 1;					  //���������� 
 }
}

void TIM0init(void)      //��ʱ��0��ʼ��	   
{
 TMOD = 0x02;            //��ʱ��0������ʽ2��TH0����װֵ��TL0�ǳ�ֵ	   
 TH0  = 0x00;                   //����ֵ	  
 TL0  = 0x00;                   //��ʼ��ֵ    
 ET0  = 1;                      //���ж�	  
 TR0  = 1;    
}

void EX0init(void)
{
 IT0 = 1;                         //ָ���ⲿ�ж�0�½��ش�����INT0 (P3.2)			 
 EX0 = 1;                        //ʹ���ⲿ�ж�	   
 EA = 1;                         //�����ж�	  
}

void main(void) 
{ 
 sys_init();                     //��ʼ��

 while (1) 
 {
  key_scan();                    //����ɨ��

  if(IR_ok)                      //������պ��˽��к��⴦��		 
  {   
   Ircordpro();
   IR_ok=0;
  }
  if(IRpro_ok)                   //�������ú���й��������簴��Ӧ
  �İ�������ʾ��Ӧ�����ֵ�			    
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
 irtime++;                             //���ڼ���2���½���֮���ʱ��	    
}

void EX0_ISR (void) interrupt 0        //�ⲿ�ж�0������		   
{
 static unsigned char  i;              //���պ����źŴ���		   
 static bit startflag;                 //�Ƿ�ʼ�����־λ

 if(startflag)                         
 {
  if(irtime<63&&irtime>=33)        //������ TC9012��ͷ�룬9ms+4.5ms		  
  i = 0;
  IRdata[i] = irtime;               //�洢ÿ����ƽ�ĳ���ʱ�䣬�����Ժ�
  �ж���0����1			  
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
