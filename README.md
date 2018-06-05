# STM32_CAN
CAN Project on STM32


## CAN分析仪用例
淘宝按销量买的 [CAN分析仪](https://item.taobao.com/item.htm?spm=a230r.1.14.4.3f2f5de7StiJHH&id=18286496283&ns=1&abbucket=18#detail), 测试的产品是Benewake的 [TF02](http://www.benewake.com/tf02.html), 有串口和CAN接口, 这里用的当然是CAN接口.  

TF02数据手册上给出的CAN的相关信息如下:  
![tf02_1](/Assets/tf02_1.png)  
![tf02_2](/Assets/tf02_2.png)  
也就是1Mbps, ID为0x00090002, 帧格式为扩展帧, Byte0为DIST高8位, Byte1为DIST低8位.  

给TF02供5V电源, CANH和CANL分别接 CAN分析仪CAN1接口的H和L, 接上120Ω终端电阻, 如图所示:  
![can1](/Assets/can1.png)  

打开CAN分析仪的上位机 USB-CAN Tool V2.12, 设备操作 -> 启动设备:  
![can2](/Assets/can2.png)  

确定, 就可以看到数据的输出了:  
![can3](/Assets/can3.png)  

图中可以看到帧率103.8, ID号0x009002, 帧格式为扩展帧, 长度8字节, 然后是8字节16进制的数据.  

## STM32读取CAN
参考 [can2uart](/can2uart) 工程, STM32读取TF02 CAN接口的数据, 通过串口发送给上位机.   
![tf02_3](/Assets/tf02_3.png)  

STM32F103TBU6, VP230(接到PA11 CAN_RX, PA12 CAN_TX), USB转串口(PA9, PA10).  
打开STM32CubeMX, Pinout选项卡配置如下:  
![cube1](/Assets/cube1.png)  

Clock Configuration选项卡配置如下:  
![cube2](/Assets/cube2.png)  

Configuration选项卡配置如下:  
![cube3](/Assets/cube3.png)  

![cube4](/Assets/cube4.png)  

CAN挂在APB1时钟上, CAN波特率 36M/Pre/(BS1+BS2+SJW) = 36M/3/(5+6+1) = 1Mbps, 刚好对应图中 Time for one Bit 的 1000ns. 要用FIFO0的接收中断, 所以USB low priority or CAN RX0 interrupts 这个勾上.  

main.c中主要添加的代码如下:  

```C
/* USER CODE BEGIN PV */
/* Private variables ---------------------------------------------------------*/
CanRxMsgTypeDef RxMessage;
/* USER CODE END PV */

/* USER CODE BEGIN PFP */
/* Private function prototypes -----------------------------------------------*/
uint8_t CAN_Init() {
    hcan.pRxMsg = &RxMessage;

    CAN_FilterConfTypeDef filterConfig;
      
    filterConfig.FilterNumber = 0;
    filterConfig.FilterMode = CAN_FILTERMODE_IDMASK;
    filterConfig.FilterScale = CAN_FILTERSCALE_32BIT;
    filterConfig.FilterIdHigh = 0x0000;
    filterConfig.FilterIdLow = 0x0000;
    filterConfig.FilterMaskIdHigh = 0x0000;
    filterConfig.FilterMaskIdLow = 0x0000;
    filterConfig.FilterFIFOAssignment = CAN_FILTER_FIFO0;
    filterConfig.FilterActivation = ENABLE;
    filterConfig.BankNumber = 14;
      
    if(HAL_CAN_ConfigFilter(&hcan,&filterConfig) != HAL_OK) return 1;
    return 0;
}

void HAL_CAN_RxCpltCallback(CAN_HandleTypeDef* hcan1) {
    __HAL_CAN_ENABLE_IT(&hcan, CAN_IT_FMP0);
    printf("(0x%08x, %d,", hcan.pRxMsg->ExtId, hcan.pRxMsg->DLC);
    for(int i = 0; i < hcan.pRxMsg->DLC; i++) {
        printf(" %02x", hcan.pRxMsg->Data[i]);
    }
    if(hcan.pRxMsg->DLC >= 2) {
        printf(", %d", (hcan.pRxMsg->Data[0]<<8) + hcan.pRxMsg->Data[1]);	//distance
    }
    printf(")\r\n");
}
/* USER CODE END PFP */

    //main函数中
    /* USER CODE BEGIN 2 */
    printf("Hello, world!\r\n");
    CAN_Init();
    __HAL_CAN_ENABLE_IT(&hcan, CAN_IT_FMP0);
    /* USER CODE END 2 */
```  

下完程序后打开串口调试助手, 可以看到发送的信息为:  
![sscom](/Assets/sscom1.png)  

一行的数据为 (扩展帧ID, 字节数, 十六进制原始数据, 计算的TF02距离值). 