# OLED 0.96 和 1.3 寸 OLED 屏驱动程序

## 20220923 branch dev
1. 更改显示缓存 由``OLED_GRAM[132][8]`` 改为 ``OLED_GRAM[8][132]`` 同时更改画点函数 ``OLED_DrawPoint()``这样就不需要``gram_trans()``转换一次了
2. ``OLED_DrawPoint()`` 中 ``if(t){OLED_GRAM[i][x]|=n;}``只需要交换数组的i,x索引位置即可完成转换。
3. 使用SPI + DMA 后除了 SPI 传输所需时间还有一个时间开销就是DMA中断处理函数中的语句，重复8次大概需要额外消耗0.12ms左右的时间。
4. 测试HAL库的 DMA 调用好费时间大概需要4us一次。
5. 此次更改的不完整，很多涉及到操作 OLED_GRAM 的函数语句没有更改，只更改了画点函数，后续需要全部更改。
6. 如果要正常使用这个库，刷新函数`OLED_Refresh_SPI`中的 dma_busy等待应该放到函数外，而不是在里面一直等。例如：`if(!dma_busy){OLED_Refresh_SPI()}`

## 驱动芯片和相应驱动程序

1. 淘宝上的0.96寸OLED屏幕使用驱动芯片为SSD1306,而1.3寸的使用SH1106。这两款芯片的使用方法，命令字基本一样。区别主要是: 
    * SSD1306 GRAM 128\*64 bit 而 SH1106 为 132\*64 bit
    * SSD1306 可以选择三种GRAM寻址顺序，由命令字 20H 控制，分别是页寻址，水平寻址和垂直寻址，其中第一种一次只能最多写入一页(即8\*124 bit = 124 byte)，在写下一页之前，必须操作先设置写入页地址 page address。这种寻址方式的问题就是即使 MCU ram 缓冲区中的数据是连续排列的，也没办法整屏使用类似 SPI 一次传输，更不能很好地利用DMA，因为每一页之间都要软件插入page address，后两种的页或者列是可以自动加一的，也就是说可以做到一次传输所有数据而不需要额外操作。
2. 中景元的例程分析如下：
```
uint8_t OLED_GRAM[132][8];           // 显示缓存

//更新显存到OLED	
void OLED_Refresh(void)
{
	uint8_t i,n;
	for(i=0;i<8;i++)
	{
	   OLED_WR_Byte(0xb0+i,OLED_CMD); //设置页起始地址
	   OLED_WR_Byte(0x00,OLED_CMD);   //设置列起始地址低4位
	   OLED_WR_Byte(0x10,OLED_CMD);   //设置列起始地址高4位

	   OLED_CS_Clr();
	   OLED_SDA_Set();
	   for(n=0;n<130;n++)									//如果右侧有不能控制的竖线，这里由128改为130，然后调用清屏函数
	   //OLED_WR_Byte(OLED_GRAM[n][i],OLED_DATA);
	   HAL_SPI_Transmit(&hspi1,&OLED_GRAM[n][i],1,100);
  }
}

//OLED的初始化
void OLED_Init(void)
{
    ...
    OLED_WR_Byte(0x20,OLED_CMD);//-Set Memory Addressing Mode (0x00/0x01/0x02)
	OLED_WR_Byte(0x02,OLED_CMD);//-Page addressing mode (A[1:0]=10xb)
    ...
}
```
* 首先它的缓存结构是 列\*页 的结构，而初始化又设置为Page addressing mode，那么就只能按页写入，每写完一页，就需要重新设置页数，共重复8次。但是每一页的数据在缓冲区(二维数组)中是不连续的，所以只能一个一个传输而不能一次性传输比如说一页，128个字节。此种缓冲结构可以将寻址方式选择为列寻址，则可以一次性传输128*8个字节。
* SH1106 只能使用按页写入，所以中景元的这个例程也可以在1.3寸的屏幕上使用。
3. 原本的驱动程序完全使用软件写入的方式，速度很慢每刷一次屏幕需要10ms以上的时间。为了提高帧率，减少CPU占用，改为 SPI+DMA的方式，但是前提是需要传输的数据需要在内存中连续排列。因此需要将以上的二维数组缓冲区重新排列，做一下缓冲矩阵的转置，转换为 页\*列的方式 转换函数 ``gram_trans()``,转换完毕后就可以实现每一页使用DMA传输一次即可。然后在 SPI 传输完毕的回调函数中控制继续写入后面的7页。```HAL_SPI_Transmit_DMA(&hspi1,&OLED_GRAM_TRANS[dma_counter][0],132);``` + ```void HAL_SPI_TxCpltCallback(SPI_HandleTypeDef *hspi)```
4. 如果是0.96的屏幕，设置为列寻址方式，则应该可以一次DMA传输所有缓存。

## CUBEMX 的 DMA 问题
使用cubeMX自动生成的代码，DMA的初始化函数总是在外设初始化后面，这样会导致DMA不能正常工作，需要手动调整顺序，一定要将```MX_DMA_Init();```放到 ADC, SPI 等外设的初始化之前！！PS. 新版的CUBEMX似乎已经修复了这个问题。

## MDK 编译器和程序执行速度
OLED库中的字符串显示计算函数 OLED_ShowString()会耗费大量的计算时间。这个函数如果使用AC5 编译器，执行一次大概需要2ms左右，如果使用AC6,使用速度优化，可以将执行时间压缩到0.7ms，但是依然是非常耗费时间。 编译器测试数据：https://www.notion.so/AC5-vs-AC6-cb3883cc28fe4b668a1d1cff02f3bd29

2022.09.21 dev 帧率可以达到1610

