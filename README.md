

/* Includes ------------------------------------------------------------------*/

#include "includes.h"

/* Private define ------------------------------------------------------------*/




  /*重新发送服务器注册包标志位*/
  volatile uint8_t ReSend_SVR_LOGIN_Flag=0;
  volatile uint8_t ISReSend_SVR_LOGIN_CNT=0;

//  /*显示TSP或者PM10标志位*/
//  volatile uint8_t TSPPM10Flag=0;

  /*TSP或者PM10的分段值设定*/
  volatile uint8_t TSPsection[20] ={0,100,0,200,1,44,1,144,1,244,2,88,2,188,3,32,3,132,3,232};				 
  volatile uint8_t PM10section[20]={0,50,0,100,0,150,0,200,0,250,1,44,1,94,1,144,1,194,1,244};					 

  /*温度或者湿度的分段值设定*/
  volatile uint8_t Tempsection[6] ={20,40,50,60,80,100};														 
  volatile uint8_t Humisection[5] ={30,50,70,80,90};

  /*TSP或者PM10的标定系数*/
  volatile uint8_t TSPcoeffi[6][24]= {0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100}; 	
  volatile uint8_t PM10coeffi[6][24]={0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100,0,100};	
  /*GPRS传输和LCD显示时的滑动滤波窗口宽度*/
  volatile uint8_t WindowLen_Updata_Display[2]={8,8};
   /*激光管温度补偿系数,分别标*/
  //volatile uint8_t TH_Base_Coeffi[8]={20,60,99,99,20,60,99,99};						  	 
  volatile uint8_t TH_Base_Coeffi[8]={20,99,20,99,60,99,60,99};   	/*0位代表TSP的温度补偿基准*/
														 			/*1位代表TSP的温度补偿系数*/
														 			/*2位代表PM10的温度补偿基准*/
		  										  	 	 			/*3位代表PM10的温度补偿系数*/
														 			/*4位代表TSP的湿度补偿基准*/
												  					/*5位代表TSP的湿度补偿系数*/ 
					  											    /*6位代表PM10的湿度补偿基准*/	
																    /*7位代表PM10的湿度补偿系数*/	
															   									  				  									
    /*系统参数页*/						  	 
  volatile uint8_t SysParameter[9]={10,0,0,0,0,  0,0,0,0}; 			/*0位代表自标零时间*/
														 			/*1位代表TSP曲线选择*/
														 			/*2位代表PM10曲线选择*/
		  										  	 	 			/*3位代表调试模式：0-PM10跟随TSP，1-PM10不跟随TSP*/
														 			/*4位代表曲线选择：0-根据湿度，1-根据温度，2-根据湿度与温度   */
												  					/*5位代表TP分段的切换*/ 
					  											    /*6位代表TP系数的切换*/	
																    /*7位代表温湿/基准与系数/的切换*/	
															    	/*8位代表温湿分段的切换*/
	float    gettemper=0;
	float    getrh=0;	
	uint32_t TSPbackgroval=0;
	uint32_t PM10backgroval=0;										  			
/* --------------------main function ------------------------------------*/														
 

main()
{

	uint32_t CaliZeroTimeFlag=0;
	uint32_t TemperUpDateFlag=0;
	uint8_t  WindSpeedGetFlag=0;
	uint8_t  Dir_Speed_flag=0;

	uint8_t  i=0;
	uint8_t  dontFeed=0;
//		for(i=0;i<10;i++)
//			Delay_ms(1000);
//    //设置中断向量表的位置在 0x3000
//    NVIC_SetVectorTable(NVIC_VectTab_FLASH, 0x3000);
	/*Delay time initialization */
	Delay_Init();
	/*WJB_Y initialization */
	WJBY_SwitchGPIOConfig();							
	WJBY_ON();
	/*TIMx configuration and initial*/
	TIMx_RCCConfig();
	TIMx_GPIOConfig();
	TIMx_Initial();
	WJB_GPIOConfig();
	/*Gas switch configuration*/ 
	Gas_SwitchGPIOConfig();
	Gas_SwitchInitialOff();
	/*KEY configuration*/
	Key_GPIOConfiguration();
	/*AD8400 initialization and configuration*/
	Ad8400_RccInitialize();
	Ad8400_Control(0x57);
	/*IIC initialization and configuration*/
	IIC_Initialize();
	/*UART initialization and configuration*/
	RCC_Configuration();
	NVIC_Configuration();	                                   
	GPIO_Configuration();
	Initial_ITUSART();									   
	Rs485_RxTxControl(1);
	/*TRH initialization and configuration*/
	TRH_Initialize();
	TRH_ConnectionReset();
	/*LCD initialization and configuration*/
	Lcd_GPIOInitialize();                  
	Initial_LCDST75256();
	Clear_DDRAM();

	ReadAllParametersFromEEPROM();
	WrtGet_BackgroundValue();
	Delay_ms(100);
	/*read Back ground Value*/
	Read_BackgroundValue(&TSPbackgroval,&PM10backgroval);
	DBGMCU_Config(DBGMCU_IWDG_STOP, ENABLE);
	IWDG_Init(6,1875);      //预分频数为64，重装载值为1875，溢出时间为12s

	USART_Cmd(USARTy , ENABLE);
	USART_Cmd(USARTz , ENABLE);

	while(1)
	{
		Scan_Key();	

		for(i=0;i<12;i++)
		{
			if(4!=i && All_Results_Float_Buffer[i]<0) dontFeed=1;		
		}
		if(0==dontFeed)
			IWDG_Feed();			/*在结果异常，或者程序卡死跑飞后，就停止喂狗*/
//		NoiseValueGetFlag++; 		/*用于环境噪声值*/
		ISReSend_SVR_LOGIN_CNT++;	/*用于若1min都还没有服务器指令中断产生，则对ReSend_SVR_LOGIN_Flag置0，使设备重新发送注册包*/

		if(1!=SysParameter[5])		//若温湿度传感器坏掉，可以通过TPS关掉温湿度测量功能
			Get_FloatTeperHumi(&gettemper,&getrh);
//		gettemper=0.983*gettemper-0.217;												
		if(0==ReSend_SVR_LOGIN_Flag)
		{
			Rs485_RxTxControl(1);
			Delay_ms(10);
			Send_SVR_LOGIN();
			Delay_ms(10);
			Rs485_RxTxControl(0);
			ReSend_SVR_LOGIN_Flag=1;	
		}
		if(20<=ISReSend_SVR_LOGIN_CNT)
		{
			ReSend_SVR_LOGIN_Flag=0;
			ISReSend_SVR_LOGIN_CNT=0;
		}		

		/*每隔1.5min读取更新一次风向/风速值/环境噪声值,*/
		WindSpeedGetFlag++;  		
		if(WindSpeedGetFlag==10)
		{
			if(0==Dir_Speed_flag)
			{
				AskWindDirSpeedData(0);		//发出获取风向数据的指令
				Dir_Speed_flag=1;
			}
			else if(1==Dir_Speed_flag)
			{
				AskWindDirSpeedData(1);		//发出获取风速数据的指令
				Dir_Speed_flag=2;
			}
			else if(2==Dir_Speed_flag)
			{
				AskEnvNoiseData();			//发出获取噪声数据的指令
				Dir_Speed_flag=0;
			}
			WindSpeedGetFlag=0;		
		}

		/*每隔Xmin读取更新一次温度、湿度值(1对应约3.2s)*/
		TemperUpDateFlag++;  	    	
		if(TemperUpDateFlag==10)	  
		{			
			All_Results_Float_Buffer[4]=gettemper;
			All_Results_Float_Buffer[5]=getrh;
			TemperUpDateFlag=0;
		}
		

	}
}



