## nRF52810 SAADC 例程

- saadc 的 channel 不用与 AIN 对应。

``` c

#include "battery_saadc.h"

#define SAADC_CALIBRATION_INTERVAL 5              				//Determines how often the SAADC should be calibrated relative to NRF_DRV_SAADC_EVT_DONE event. E.g. value 5 will make the SAADC calibrate every fifth time the NRF_DRV_SAADC_EVT_DONE is received.
#define SAADC_OVERSAMPLE 		NRF_SAADC_OVERSAMPLE_DISABLED   //Oversampling setting for the SAADC. Setting oversample to 4x This will make the SAADC output a single averaged value when the SAMPLE task is triggered 4 times. Enable BURST mode to make the SAADC sample 4 times when triggering SAMPLE task once.
#define SAADC_BURST_MODE 		1                        		//Set to 1 to enable BURST mode, otherwise set to 0.
#define BATTERY_CHANNEL			0
#deifne BATTERY_INPUT_PIN		NRF_SAADC_INPUT_VDD

#define ADC_DEBUG				1

static uint32_t                m_adc_evt_counter = 0;
static bool                    m_saadc_calibrate = false;  
static nrf_saadc_value_t       battery_adc_value = 0;

static const float VoltageTable[] = 	 {4.19, 4.05, 3.95, 3.90, 3.85, 3.80, 3.75, 3.70, 3.65, 3.6, 3.55, 3.5, 3.45, 3.0, 2.973};
static const uint8_t PercentageTable[] = {100,  97,   88,   82,   76,   69,   60,   50,   33,   20,   11,  7,   5,    1,   0};


static void saadc_callback(nrf_drv_saadc_evt_t const * p_event)
{
    if (p_event->type == NRF_DRV_SAADC_EVT_DONE)                                                    
	{
        
    }
    else if (p_event->type == NRF_DRV_SAADC_EVT_CALIBRATEDONE)
    {      
        NRF_LOG_PRINTF("#");    
    }
}

void saadc_init(void)
{
    ret_code_t err_code;
    nrf_drv_saadc_config_t saadc_config;
    nrf_saadc_channel_config_t channel_config;

    //Configure SAADC
    saadc_config.low_power_mode = true;                              
    saadc_config.resolution = NRF_SAADC_RESOLUTION_12BIT;            
    saadc_config.oversample = SAADC_OVERSAMPLE;                      
    saadc_config.interrupt_priority = APP_IRQ_PRIORITY_LOW;          
	
    //Initialize SAADC
    err_code = nrf_drv_saadc_init(&saadc_config, saadc_callback);    
    APP_ERROR_CHECK(err_code);
		
    //Configure SAADC channel
    channel_config.reference = NRF_SAADC_REFERENCE_INTERNAL;         	 
    channel_config.gain = NRF_SAADC_GAIN1_6;                         
    channel_config.acq_time = NRF_SAADC_ACQTIME_10US;                
    channel_config.mode = NRF_SAADC_MODE_SINGLE_ENDED;              
    channel_config.pin_p = BATTERY_INPUT_PIN;                      
    channel_config.pin_n = NRF_SAADC_INPUT_DISABLED;                 
    channel_config.resistor_p = NRF_SAADC_RESISTOR_DISABLED;         
    channel_config.resistor_n = NRF_SAADC_RESISTOR_DISABLED;         

	// reference: internal reference, mode: single ended. 
	// Input range = (0.6 V)/(gain) = 0.6V / 1/6 = 3.6V
	
    //Initialize SAADC channel
    err_code = nrf_drv_saadc_channel_init(BATTERY_CHANNEL, &channel_config);                            
    APP_ERROR_CHECK(err_code);
		
    if(SAADC_BURST_MODE)
    {
        NRF_SAADC->CH[0].CONFIG |= 0x01000000;		
    }
}

static void start_battery_adc_convert(void)
{
	nrf_drv_saadc_sample_convert(BATTERY_CHANNEL, &battery_adc_value);
	
	if((m_adc_evt_counter % SAADC_CALIBRATION_INTERVAL) == 0)	//Evaluate if offset calibration should be performed. Configure the SAADC_CALIBRATION_INTERVAL constant to change the calibration frequency
	{
		nrf_drv_saadc_abort();                                  // Abort all ongoing conversions. Calibration cannot be run if SAADC is busy
		m_saadc_calibrate = true;                               // Set flag to trigger calibration in main context when SAADC is stopped
	}       
	m_adc_evt_counter++; 
}

uint8_t get_battery_level(void)
{
	uint8_t length = 15;
    uint8_t i  = 0, level = 0;
	float volatage = 0;
	
	start_battery_adc_convert();
	
	// voltage = result / GAIN / reference * 2 ^(resolution)
	volatage = battery_adc_value / ((1.0f/6.0f) / 0.6f * 4095.0f);

	//find the first value which is < volatage
    for(i = 0; i< length ; ++i)
	{
        if(volatage >= VoltageTable[i]) 
		{
            break;
        }
    }

    if( i>= length) 
	{
#if NRF_LOG_BACKEND_RTT_ENABLED == 1 && defined(ADC_DEBUG)
		char string[64];
		sprintf(string, "Battery volatage: %0.4fV, %d, %d%%\r\n", volatage, battery_adc_value, 0);
		SEGGER_RTT_WriteString(0, string);
#endif
        return 0;
    }

    if(i == 0) 
	{
#if NRF_LOG_BACKEND_RTT_ENABLED == 1 && defined(ADC_DEBUG)
		char string[64];
		sprintf(string, "Battery volatage: %0.4fV, %d, %d%%\r\n", volatage, battery_adc_value, 100);
		SEGGER_RTT_WriteString(0, string);
#endif
        return 100;
    }
    
    level = (volatage - VoltageTable[i])/((VoltageTable[i-1] - VoltageTable[i])/(PercentageTable[i-1] - PercentageTable[i])) + PercentageTable[i];

#if NRF_LOG_BACKEND_RTT_ENABLED == 1 && defined(ADC_DEBUG)
	char string[64];
	sprintf(string, "Battery volatage: %0.4fV, %d, %d%%\r\n", volatage, battery_adc_value, level);
	SEGGER_RTT_WriteString(0, string);
#endif
	
	return level;
}

void saadc_calibrate_event(void)
{
	if(m_saadc_calibrate == true)
	{
		while(nrf_drv_saadc_calibrate_offset() != NRF_SUCCESS); //Trigger calibration task
		m_saadc_calibrate = false;
	}
}


```