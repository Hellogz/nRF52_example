
### nRF52810 的广播例程

- 两种广播模式，一种短广播间隔并且可连接，一种长广播间隔并且不可连接。
- 长广播间隔可以更新广播数据。
- 有限发现模式 BLE_GAP_ADV_FLAGS_LE_ONLY_LIMITED_DISC_MODE 的两个关键宏定义：BLE_GAP_ADV_TIMEOUT_LIMITED_MAX（182 秒） 、BLE_GAP_ADV_TIMEOUT_HIGH_DUTY_MAX（1.28秒）


``` c

#define	APP_COMPANY_IDENTIFIER			0x0201										/* Bluetooth sig company identifier */
#define APP_ADVERTISING_INFO_LENGTH		16

#define ADV_ENCODED_AD_TYPE_LEN         1                                             /**< Length of encoded ad type in advertisement data. */
#define ADV_ENCODED_AD_TYPE_LEN_LEN     1                                             /**< Length of the 'length field' of each ad type in advertisement data. */
#define ADV_FLAGS_LEN                   1                                             /**< Length of flags field that will be placed in advertisement data. */
#define ADV_ENCODED_FLAGS_LEN           (ADV_ENCODED_AD_TYPE_LEN +       \
                                        ADV_ENCODED_AD_TYPE_LEN_LEN +   \
                                        ADV_FLAGS_LEN)                               /**< Length of flags field in advertisement packet. (1 byte for encoded ad type plus 1 byte for length of flags plus the length of the flags itself). */
#define ADV_ENCODED_COMPANY_ID_LEN      2                                            /**< Length of the encoded Company Identifier in the Manufacturer Specific Data part of the advertisement data. */
#define ADV_ADDL_MANUF_DATA_LEN         (APP_CFG_ADV_DATA_LEN -                \
                                        (                                     \
                                            ADV_ENCODED_FLAGS_LEN +           \
                                            (                                 \
                                                ADV_ENCODED_AD_TYPE_LEN +     \
                                                ADV_ENCODED_AD_TYPE_LEN_LEN + \
                                                ADV_ENCODED_COMPANY_ID_LEN    \
                                            )                                 \
                                        )                                     \
                                        )                                             /**< Length of Manufacturer Specific Data field that will be placed on the air during advertisement. This is computed based on the value of APP_CFG_ADV_DATA_LEN (required advertisement data length). */


#define CONNECTABLE_ADV_INTERVAL		MSEC_TO_UNITS(400, UNIT_0_625_MS)
#define CONNECTABLE_ADV_DURATION		0

static uint8_t m_advertising_info[APP_ADVERTISING_INFO_LENGTH] = {0};
static ble_gap_adv_params_t     m_adv_params;                                       /**< Parameters to be passed to the stack when starting advertising. */
static uint8_t m_adv_handle = BLE_GAP_ADV_SET_HANDLE_NOT_SET;                       /**< Advertising handle used to identify an advertising set. */
static uint8_t m_enc_advdata[BLE_GAP_ADV_SET_DATA_SIZE_MAX];                        /**< Buffer for storing an encoded advertising set. */

static bool m_is_non_connectable_mode = true;


bool now_is_connectable_mode(void)
{
	return (m_is_non_connectable_mode == false);
}

/**@brief Struct that contains pointers to the encoded advertising data. */
static ble_gap_adv_data_t m_adv_data =
{
    .adv_data =
    {
        .p_data = m_enc_advdata,
        .len    = BLE_GAP_ADV_SET_DATA_SIZE_MAX
    },
    .scan_rsp_data =
    {
        .p_data = NULL,
        .len    = 0

    }
};

/**@brief Function for initializing the connectable advertisement parameters.
 *
 * @details This function initializes the advertisement parameters to values that will put
 *          the application in connectable mode.
 *
 */
static void connectable_adv_init(void)
{
    // Initialize advertising parameters (used when starting advertising).
    memset(&m_adv_params, 0, sizeof(m_adv_params));

    m_adv_params.properties.type = BLE_GAP_ADV_TYPE_CONNECTABLE_SCANNABLE_UNDIRECTED;
    m_adv_params.duration        = CONNECTABLE_ADV_DURATION;

    m_adv_params.p_peer_addr   = NULL;
    m_adv_params.filter_policy = BLE_GAP_ADV_FP_ANY;
    m_adv_params.interval      = CONNECTABLE_ADV_INTERVAL;
    m_adv_params.primary_phy   = BLE_GAP_PHY_1MBPS;
}


/**@brief Function for initializing the non-connectable advertisement parameters.
 *
 * @details This function initializes the advertisement parameters to values that will put
 *          the application in non-connectable mode.
 *
 */
static void non_connectable_adv_init(void)
{
    // Initialize advertising parameters (used when starting advertising).
    memset(&m_adv_params, 0, sizeof(m_adv_params));

    m_adv_params.properties.type = BLE_GAP_ADV_TYPE_NONCONNECTABLE_SCANNABLE_UNDIRECTED; 
    m_adv_params.duration        = BLE_GAP_ADV_TIMEOUT_LIMITED_MAX ;
	
    m_adv_params.p_peer_addr     = NULL;
    m_adv_params.filter_policy   = BLE_GAP_ADV_FP_ANY;
    m_adv_params.interval        = MSEC_TO_UNITS(4000, UNIT_0_625_MS);
    m_adv_params.primary_phy     = BLE_GAP_PHY_1MBPS;
}

#if 1
/**@brief Function for initializing the Advertisement packet.
 *
 * @details This function initializes the data that is to be placed in an advertisement packet in
 *          both connectable and non-connectable modes.
 *
 */
static void advertising_init(void)
{
    ret_code_t               err_code;
    ble_advdata_t            advdata;
    ble_advdata_manuf_data_t manuf_data;
    uint8_t                  flags;

    APP_ERROR_CHECK_BOOL(sizeof(flags) == ADV_FLAGS_LEN);  // Assert that these two values of the same.

	if(now_is_connectable_mode())
	{
		flags = BLE_GAP_ADV_FLAGS_LE_ONLY_GENERAL_DISC_MODE;
	}
	else
	{
		flags = BLE_GAP_ADV_FLAGS_LE_ONLY_LIMITED_DISC_MODE;
	}
	
    // Build and set advertising data is here
    memset(&advdata, 0, sizeof(advdata));

    manuf_data.company_identifier = APP_COMPANY_IDENTIFIER;
    manuf_data.data.size          = APP_ADVERTISING_INFO_LENGTH;
    manuf_data.data.p_data        = m_advertising_info;
	
	advdata.name_type			  = BLE_ADVDATA_FULL_NAME;
	advdata.include_appearance 	  = false;
    advdata.flags                 = flags;
    advdata.p_manuf_specific_data = &manuf_data;

    err_code = ble_advdata_encode(&advdata, m_adv_data.adv_data.p_data, &m_adv_data.adv_data.len);
    if(err_code) { NRF_LOG_INFO("ble_advdata_encode: %d", err_code); }
	APP_ERROR_CHECK(err_code);
	
	err_code = sd_ble_gap_adv_set_configure(&m_adv_handle, &m_adv_data, &m_adv_params);
	if(err_code) { NRF_LOG_INFO("sd_ble_gap_adv_set_configure: %d", err_code); }
	APP_ERROR_CHECK(err_code);
}	

/**@brief Function for starting advertising.
 */
void advertising_start(void)
{
    ret_code_t err_code;

    err_code = sd_ble_gap_adv_start(m_adv_handle, APP_BLE_CONN_CFG_TAG);
	if(err_code) { NRF_LOG_INFO("sd_ble_gap_adv_start: %d", err_code); }
    APP_ERROR_CHECK(err_code);
}

/**@brief Function for stop advertising.
 */
void advertising_stop(void)
{
	uint32_t err_code = sd_ble_gap_adv_stop(m_adv_handle);
	if(err_code) { NRF_LOG_INFO("sd_ble_gap_adv_stop: %d", err_code); }
	APP_ERROR_CHECK(err_code);
}

void advertising_update(void)
{
	if(now_is_connectable_mode())
	{
		connectable_adv_init();
	}
	else
	{
		non_connectable_adv_init();
	}
	advertising_init();
}

```