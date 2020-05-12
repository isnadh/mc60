#ifdef __CUSTOMER_CODE__
#include "custom_feature_def.h"
#include "ril.h"
#include "ril_util.h"
#include "ril_telephony.h"
#include "ql_stdlib.h"
#include "ql_error.h"
#include "ql_trace.h"
#include "ql_uart.h"
#include "ql_system.h"
#include "ril_network.h"
#include "ril_http.h"
#include "ql_timer.h"
 
#define APN_NAME "mcinet\0"
#define APN_USERID ""
#define APN_PASSWD ""
 
s32 stable = 0;
s32 SENDING_TO_SERVER = 0;
s32 rep = 0;
s32 RIL_STATUS = 0;
s32 iRet = 0;
 
static u32 Stack_timer = 0x102;
static u32 ST_Interval = 3000;
static s32 m_param1 = 0;
static u32 m_rcvDataLen = 0;
 
char HTTP_URL_ADDR[100] = "http:// \0";
 
u8 RMC_BUFFER[1000];
u8 postMsg[300] = "";
u8 arrHttpRcvBuf[10 * 1024];
 
Enum_PinName LED_1 = PINNAME_RI;
 
#define SERIAL_RX_BUFFER_LEN 2048
char m_RxBuf_Uart[SERIAL_RX_BUFFER_LEN];
 
#define DEBUG_ENABLE 1
#if DEBUG_ENABLE > 0
#define DEBUG_PORT UART_PORT1
#define DBG_BUF_LEN 512
static char DBG_BUFFER[DBG_BUF_LEN];
#define APP_DEBUG(FORMAT, ...)                                                                                       \
    {                                                                                                                \
        Ql_memset(DBG_BUFFER, 0, DBG_BUF_LEN);                                                                       \
        Ql_sprintf(DBG_BUFFER, FORMAT, ##__VA_ARGS__);                                                               \
        if (UART_PORT2 == (DEBUG_PORT))                                                                              \
        {                                                                                                            \
            Ql_Debug_Trace(DBG_BUFFER);                                                                              \
        }                                                                                                            \
        else                                                                                                         \
        {                                                                                                            \
            Ql_UART_Write((Enum_SerialPort)(DEBUG_PORT), (u8 *)(DBG_BUFFER), Ql_strlen((const char *)(DBG_BUFFER))); \
        }                                                                                                            \
    }
#else
#define APP_DEBUG(FORMAT, ...)
#endif
#define SERIAL_RX_BUFFER_LEN 2048
static Enum_SerialPort m_myUartPort = UART_PORT1;
static u8 m_RxBuf_Uart1[SERIAL_RX_BUFFER_LEN];
static void Timer_handler(u32 timerId, void *param);
static void CallBack_UART_Hdlr(Enum_SerialPort port, Enum_UARTEventType msg, bool level, void *customizedPara);
static s32 ATResponse_Handler(char *line, u32 len, void *userData);
static void HTTP_Program();
 
void proc_main_task(s32 taskId)
{
    s32 ret;
    ST_MSG msg;
 
    Ql_GPIO_Init(LED_1, PINDIRECTION_OUT, PINLEVEL_LOW, PINPULLSEL_PULLUP);
    Ql_GPIO_SetLevel(LED_1, PINLEVEL_LOW);
 
    while (TRUE)
    {
        Ql_OS_GetMessage(&msg);
        switch (msg.message)
        {
        case MSG_ID_RIL_READY:
            APP_DEBUG("LOAD LEVEL 1 (RIL READY)\r\n");
            Ql_RIL_Initialize();
            RIL_STATUS = 1;
            GPSPower(1);
            break;
        case MSG_ID_URC_INDICATION:
            APP_DEBUG("Received URC: type: %d\r\n", msg.param1);
            switch (msg.param1)
            {
            case URC_GSM_NW_STATE_IND:
                APP_DEBUG("GSM Network Status:%d\r\n", msg.param2);
                break;
 
            case URC_GPRS_NW_STATE_IND:
                APP_DEBUG("GPRS Network Status:%d\r\n", msg.param2);
                if (NW_STAT_REGISTERED == msg.param2 || NW_STAT_REGISTERED_ROAMING == msg.param2)
                {
                    APP_DEBUG("LOAD LEVEL 2 (NETWORK REGISTERED)\r\n");
                    stable = 1;
                }
                break;
            }
            break;
        }
    }
}
 
//Read serial port
void proc_subtask1(s32 TaskId)
{
    s32 ret;
    ST_MSG msg;
    ST_UARTDCB dcb;
    Enum_SerialPort mySerialPort = UART_PORT1;
    dcb.baudrate = 115200;
    dcb.dataBits = DB_8BIT;
    dcb.stopBits = SB_ONE;
    dcb.parity = PB_NONE;
    dcb.flowCtrl = FC_NONE;
    Ql_UART_Register(mySerialPort, CallBack_UART_Hdlr, NULL);
    Ql_UART_OpenEx(mySerialPort, &dcb);
    Ql_UART_ClrRxBuffer(mySerialPort);
    APP_DEBUG("START PROGRAM \r\n");
    while (TRUE)
    {
        Ql_OS_GetMessage(&msg);
        switch (msg.message)
        {
        case MSG_ID_USER_START:
            break;
        default:
            break;
        }
    }
}
 
//Update location data
proc_subtask2(s32 TaskId)
{
    while (1)
    {
        Ql_Sleep(1000);
        if (RIL_STATUS = 1)
        {
            iRet = RIL_GPS_Read("RMC", RMC_BUFFER);
            if (RIL_AT_SUCCESS != iRet)
            {
                APP_DEBUG("Read %s information failed.\r\n", "RMC");
            }
            else
            {
                if (RMC_BUFFER[30] == 'A')
                {
                    Ql_GPIO_SetLevel(LED_1, PINLEVEL_HIGH);
                    Ql_Sleep(50);
                    Ql_GPIO_SetLevel(LED_1, PINLEVEL_LOW);
                    Ql_Sleep(50);
                    Ql_GPIO_SetLevel(LED_1, PINLEVEL_HIGH);
                    Ql_Sleep(50);
                    Ql_GPIO_SetLevel(LED_1, PINLEVEL_LOW);
                }
                else if (RMC_BUFFER[30] == 'V')
                {
                    Ql_GPIO_SetLevel(LED_1, PINLEVEL_HIGH);
                    Ql_Sleep(50);
                    Ql_GPIO_SetLevel(LED_1, PINLEVEL_LOW);
                }
            }
        }
    }
}
 
// Send location data to server
proc_subtask3(s32 TaskId)
{
    while (1)
    {
           Ql_Sleep(3000);
        if (stable == 1)
        {
            Ql_strcpy(postMsg, "location=");
            Ql_strcat(postMsg, RMC_BUFFER);
            Ql_strcat(postMsg, "\0");
            HTTP_Program();
        }
    }
}
 
static void HTTP_RcvData(u8 *ptrData, u32 dataLen, void *reserved)
{
    APP_DEBUG("<-- Data coming on http, total len:%d -->\r\n", m_rcvDataLen + dataLen);
    if ((m_rcvDataLen + dataLen) <= sizeof(arrHttpRcvBuf))
    {
        Ql_memcpy((void *)(arrHttpRcvBuf + m_rcvDataLen), (const void *)ptrData, dataLen);
    }
    else
    {
        if (m_rcvDataLen < sizeof(arrHttpRcvBuf))
        {
            u32 realAcceptLen = sizeof(arrHttpRcvBuf) - m_rcvDataLen;
            Ql_memcpy((void *)(arrHttpRcvBuf + m_rcvDataLen), (const void *)ptrData, realAcceptLen);
            APP_DEBUG("<-- Rcv-buffer is not enough, discard part of data (len:%d/%d) -->\r\n", dataLen - realAcceptLen, dataLen);
        }
        else
        {
            APP_DEBUG("<-- No more buffer, discard data (len:%d) -->\r\n", dataLen);
        }
    }
    m_rcvDataLen += dataLen;
}
 
static void HTTP_Program()
{
    s32 ret;
    m_rcvDataLen = 0;
    ret = RIL_NW_SetGPRSContext(Ql_GPRS_GetPDPContextId());
    APP_DEBUG("START SEND TO SERVER\r\n");
    ret = RIL_NW_SetAPN(1, APN_NAME, APN_USERID, APN_PASSWD);
    APP_DEBUG("<-- Set GPRS APN, ret=%d -->\r\n", ret);
    ret = RIL_NW_OpenPDPContext();
    APP_DEBUG("<-- Open PDP context, ret=%d -->\r\n", ret);
    ret = RIL_HTTP_SetServerURL(HTTP_URL_ADDR, Ql_strlen(HTTP_URL_ADDR));
    APP_DEBUG("<-- Set http server URL, ret=%d -->\r\n", ret);
    ret = RIL_HTTP_RequestToPost(postMsg, Ql_strlen((char *)postMsg));
    APP_DEBUG("<-- Send post-request,  ret=%d -->\r\n", ret);
    ret = RIL_HTTP_ReadResponse(120, HTTP_RcvData);
    APP_DEBUG("READ SERVER RESPONSE (dataLen=%d) \r\n", m_rcvDataLen);
    ret = RIL_NW_ClosePDPContext();
    APP_DEBUG("<-- Close PDP context, ret=%d -->\r\n", ret);
}
 
static s32 ReadSerialPort(Enum_SerialPort port, /*[out]*/ u8 *pBuffer, /*[in]*/ u32 bufLen)
{
    s32 rdLen = 0;
    s32 rdTotalLen = 0;
    if (NULL == pBuffer || 0 == bufLen)
    {
        return -1;
    }
    Ql_memset(pBuffer, 0x0, bufLen);
    while (1)
    {
        rdLen = Ql_UART_Read(port, pBuffer + rdTotalLen, bufLen - rdTotalLen);
        if (rdLen <= 0)
        {
            break;
        }
        rdTotalLen += rdLen;
    }
    return rdTotalLen;
}
 
void GPSPower(int status)
{
    if (status == 1)
    {
        iRet = RIL_GPS_Open(1);
        if (RIL_AT_SUCCESS != iRet)
        {
            APP_DEBUG("GPS is on \r\n");
        }
        else
        {
            APP_DEBUG("Power on GPS Successful.\r\n");
        }
    }
    else if (status == 0)
    {
        iRet = RIL_GPS_Open(0);
        if (RIL_AT_SUCCESS != iRet)
        {
            APP_DEBUG("GPS is off \r\n");
        }
        else
        {
            APP_DEBUG("Power off GPS Successful.\r\n");
        }
    }
}
 
static void CallBack_UART_Hdlr(Enum_SerialPort port, Enum_UARTEventType msg, bool level, void *customizedPara)
{
    switch (msg)
    {
    case EVENT_UART_READY_TO_READ:
    {
        char *p = NULL;
        s32 totalBytes = ReadSerialPort(port, m_RxBuf_Uart, sizeof(m_RxBuf_Uart));
        if (totalBytes <= 0)
        {
            break;
        }
        if (Ql_strstr(m_RxBuf_Uart, "GPSOn"))
        {
            APP_DEBUG("ok\r\n");
            GPSPower(1);
            break;
        }
        if (Ql_strstr(m_RxBuf_Uart, "GPSOff"))
        {
            APP_DEBUG("ok\r\n");
            GPSPower(0);
            break;
        }
        if (Ql_strstr(m_RxBuf_Uart, "location"))
        {
            APP_DEBUG("ok\r\n");
            APP_DEBUG("%s \r\n", RMC_BUFFER);
            break;
        }
        break;
    }
    }
}
#endif // __CUSTOMER_CODE__
