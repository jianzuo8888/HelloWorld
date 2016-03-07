

/* Includes ------------------------------------------------------------------*/

#include "includes.h"

/* Private define ------------------------------------------------------------*/
		TemperUpDateFlag++;  	    	
		if(TemperUpDateFlag==10)	  
		{			
			All_Results_Float_Buffer[4]=gettemper;
			All_Results_Float_Buffer[5]=getrh;
			TemperUpDateFlag=0;
		}
		

	}
}



