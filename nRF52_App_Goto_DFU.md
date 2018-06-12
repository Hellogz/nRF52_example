## nRF52810 的 APP 里跳转到 DFU 模式例程

- 该方法是通过查看 DFU 工程而来，已验证可以正常跳转到 DFU 模式。

``` c
#include "nrf_pwr_mgmt.h"
#include "nrf_sdm.h"
#include "nrf_power.h"

#define BOOTLOADER_DFU_GPREGRET_MASK            (0xB0)      /**< Magic pattern written to GPREGRET register to signal between main app and DFU. The 3 lower bits are assumed to be used for signalling purposes.*/
#define BOOTLOADER_DFU_START_BIT_MASK           (0x01)      /**< Bit mask to signal from main application to enter DFU mode using a buttonless service. */

#define BOOTLOADER_DFU_GPREGRET2_MASK           (0xA8)      /**< Magic pattern written to GPREGRET2 register to signal between main app and DFU. The 3 lower bits are assumed to be used for signalling purposes.*/
#define BOOTLOADER_DFU_SKIP_CRC_BIT_MASK        (0x01)      /**< Bit mask to signal from main application that CRC-check is not needed for image verification. */


#define BOOTLOADER_DFU_START    (BOOTLOADER_DFU_GPREGRET_MASK | BOOTLOADER_DFU_START_BIT_MASK)      /**< Magic number to signal that bootloader should enter DFU mode because of signal from Buttonless DFU in main app.*/
#define BOOTLOADER_DFU_SKIP_CRC (BOOTLOADER_DFU_GPREGRET2_MASK | BOOTLOADER_DFU_SKIP_CRC_BIT_MASK)  /**< Magic number to signal that CRC can be skipped due to low power modes.*/


static uint32_t go_to_bootload(void)
{
	uint32_t err_code;
	
	// Softdevice was disabled before going into reset. Inform bootloader to skip CRC on next boot.
    nrf_power_gpregret2_set(BOOTLOADER_DFU_SKIP_CRC);
	
	err_code = sd_power_gpregret_clr(0, 0xffffffff);
    VERIFY_SUCCESS(err_code);

    err_code = sd_power_gpregret_set(0, BOOTLOADER_DFU_START);
    VERIFY_SUCCESS(err_code);
	
	//Disable SoftDevice. It is required to be able to write to GPREGRET2 register (SoftDevice API blocks it).
    //GPREGRET2 register holds the information about skipping CRC check on next boot.
    err_code = nrf_sdh_disable_request();
    APP_ERROR_CHECK(err_code);
	
	if (NRF_UICR->NRFFW[0] != 0xFFFFFFFF)
    {
        NRF_LOG_INFO("Setting vector table to bootloader: 0x%08x", NRF_UICR->NRFFW[0]);
        err_code = sd_softdevice_vector_table_base_set(NRF_UICR->NRFFW[0]);
        if (err_code != NRF_SUCCESS)
        {
            NRF_LOG_ERROR("Failed running sd_softdevice_vector_table_base_set");
            return err_code;
        }
    }
	
	// Signal that DFU mode is to be enter to the power management module
    nrf_pwr_mgmt_shutdown(NRF_PWR_MGMT_SHUTDOWN_GOTO_DFU);
	
	return NRF_SUCCESS;
}


```