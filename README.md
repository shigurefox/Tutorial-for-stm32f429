# Tutorial for stm32f429

## 1.Setup/Installation

### (1) ARM Toolchain : 

我是照這網址裝  
https://launchpad.net/~team-gcc-arm-embedded/+archive/ubuntu/ppa  
  
	sudo add-apt-repository ppa:team-gcc-arm-embedded/ppa  
	sudo apt-get update  
	sudo apt-get install gcc-arm-embedded  

compile : arm-none-eabi-gcc ...
		
### (2) STlink : 
		
安裝stlink需要的東西  
CMake (minimal v2.8.7)  

	sudo apt install cmake  

C compiler (gcc, clang, mingw)  

	sudo apt install gcc  
	sudo apt install clang  
	sudo apt install mingw-w64    

Libusb 1.0 (minimal v1.0.9)  

	sudo apt-get install libusb-1.0-0-dev  
  
stlink github網址 https://github.com/texane/stlink  
	git clone https://github.com/texane/stlink.git  
	cd stlink && make release  
	cd build/Release && sudo make install  
	export LD_LIBRARY_PATH=/usr/local/lib  

然後應該就可以用 st-info 、 st-flash 、 st-util  
e.g.  

	st-flash write example.bin 0x08000000  

將 stm32f429板子與另外一個usart用的小東西接上電腦後用 lsusb 列出裝置有  
Bus 001 Device 004: ID 0483:3748 STMicroelectronics ST-LINK/V2  (stm32板子，上面的stlink裝置)  
和  
Bus 001 Device 003: ID 0403:6001 Future Technology Devices International, Ltd FT232 USB-Serial (UART) IC  (那個小東西)  
	
我用dmesg | grep tty 後看到 FTDI USB Serial Device converter now attached to "ttyUSB0"	## 等等設定minicom的路徑

### (3) minicom :
		
我找的透過 USART 跟板子溝通的軟體，也有其他軟體可以用，不過目前我只會用這個  

	sudo apt-get install minicom  
  
打開minicom之後，按 ctrl+a 再按 o 進入設定，  
進入 Serial port setup，按  
	a，把路徑改成 /dev/ttyUSB0  	
	e，Bps/Par/Bits : 115200 8N1  預設這樣，不用改  
	f，Hardware Flow Control : 改成 NO  
設定完後 Save setup as dfl 存成預設，然後離開設定畫面就可以開始用  

### (4) pqm4 : https://github.com/mupq/pqm4
  	
require:  
python3 (>= 3.6)  
Ubuntu 18.04 LTS 自帶 3.6.8，或者也可以自行升級3.7  

	sudo apt-get install python3.7
  
pyserial  

	pip3 install pyserial  


## 2.用 stm32f4 測試 pqm4 上的 scheme
  
installation:  

	git clone --recursive https://github.com/mupq/pgm4.git  
  
在執行 python3 build_everything.py 之前，因為我們要用的 USART 的腳位 PA9、PA10，跟 pqm4 預設用的 PA2、PA3 不一樣，所以要先去 /pqm4/common/hal-stm32f4.c 裡面更改 USART 的設定，
把裡面的 "USART2" 都改成 "USART1"  
"RCC_USART2" 改成 "RCC_USART1"  
"GPIO2 | GPIO3" 改成 "GPIO9 | GPIO10"  
裡面用來輸出字串的 function : send_USART_srt(const char* in) ， 字串輸出完後只有 '\n' ，可以在 usart_send_blocking(USART1, '\n'); 下面加一行
	usart_send_blocking(USART1, '\r');  
，用來靠左對齊  

之後執行  
	python3 build_everything.py  
然後就會對每個 scheme 生成 6 個測試檔案，然後就可以把 bin 裡面的檔案燒到 stm32f4 上測試  
全部編完要很久，可以先編出前面幾個 .bin 檔後中斷，測試能不能跑  
  
測試方式:  
1. 將 stm32f4 板子上的 PA9(USART_TX) 接到 FT232 USB-Serial (UART) IC (那塊小東西) 的 RXD ， PA10(USART_RX) 接到 TXD  
2. 將 .bin 檔燒錄到板子上  
e.g.  
	st-flash write crypto_kem_r5nd-1kemcca-0d_m4_speed.bin 0x08000000  
3. 用 minicom 看執行結果  
(1) 開啟 minicom  
(2) 按板子上的 RESET button 就會顯示執行結果  
或是 python3 pqm4/hostside/host_unidirectional.py 也可以看結果  
  
## 3.stm32f4板子操作

libopencm3 : Open source ARM Cortex-M microcontroller library  
http://libopencm3.org/  
libopencm3-example : https://github.com/libopencm3/libopencm3  
裡面示範一些板子周邊設備的基本操作  
libopencm3-template : https://github.com/libopencm3/libopencm3-template  
已經連結好libopencm3的空白範本，可以自己加東西進去測試功能，直接 make 就會生成 .bin 檔  
  
### (1) 用USART對stm32f4輸入輸出  
可以參考 libopencm3-example/examples/stm32/f4/stm32f429i-discovery/usart_console/usart_console.c
  
假設用 USART1 (對應板子上的 PA9 : TX  ，  PA10 : RX)，要在code裡設定 :
下面是從example裡面抓出來的部分code
		
	//////////////////這部分是用來enable要用到的設備，以及設定一些參數/////////////////////////////////////
	static void clock_setup(void)   		//這個function用來把一些libopencm3寫好的function包成一個，下面同理
	{
	rcc_periph_clock_enable(RCC_GPIOA);		//stm32f4有幾組GPIO pins，一組16個，如 PA9、PG13。有用到 PA9, PA10 (A組)，所以要enable GPIOA 的 clock，如果要點燈(如 PG13)，就要rcc_periph_clock_enable(RCC_GPIOG);
	rcc_periph_clock_enable(RCC_USART1);		//在 libopencm3/include 裡面有定義好暫存器的起始位置，然後看情況取得，通常是32bit，該暫存器的內容。 USART1 表示那個暫存器，USART_MODE_TX_RX 為常數，代表暫存器的第幾位要設成1，以開啟對應功能。
	}  
	
	static void usart_setup(void)			//USART的一些參數設定，baud / databits /...	
	{						//USART1 ,USART_MODE_TX_RX 這些東西有些指的是暫存器，有些是常數，詳細還是要去include裡面看	
	usart_set_baudrate(USART1, 115200);			
	usart_set_databits(USART1, 8);
	usart_set_stopbits(USART1, USART_STOPBITS_1);	
	usart_set_mode(USART1, USART_MODE_TX_RX);		
	usart_set_parity(USART1, USART_PARITY_NONE);					    
	usart_set_flow_control(USART1, USART_FLOWCONTROL_NONE);
	
	usart_enable(USART1);
	}
	
	static void gpio_setup(void)
	{
	/* Setup GPIO pins for USART1 transmit. */
	gpio_mode_setup(GPIOA, GPIO_MODE_AF, GPIO_PUPD_NONE, GPIO9 | GPIO10); 
	
	/* Setup USART1 TX/RX pins as alternate function. */
	gpio_set_af(GPIOA, GPIO_AF7, GPIO9 | GPIO10);         //AF7 : USART1~3
	}

然後在main裡面call上面寫的function來設定要用的設備

***  

基本上就是想要甚麼板子提供的基本功能，就用libopencm3寫好的function去enalbe那個設備、clock、設定參數等等。  
然後視情況再另外寫function來達成想要的功能。  
  
libopencm3-example/examples/stm32/f4/stm32f429i-discovery/usart_console/usart_console.c 的例子裡
  
	#define CONSOLE_UART USART1 				//看用的是哪個USART  
	void console_putc(char c)				//用來把字元寫入 USART1 的 data register ，然後用監視 USART 的軟體就會看到輸出的字元  
	{
	uint32_t	reg;					//宣告一個 32bit 的變數，用來放 USART1 的 status resister 的內容
	do {
		reg = USART_SR(CONSOLE_UART);
	} while ((reg & USART_SR_TXE) == 0);			// USART_SR_TXE : 是一個常數 (1 << 7)，定義在  libopencm3/include/libopencm3/stm32/common/usart_common_f124.h 裡，表示 USART_SR 的第 7 位，當該位是 1 的時候，表示 transmit data buffer empty  
		USART_DR(CONSOLE_UART) = (uint16_t) c & 0xff;	// 此時會跳出迴圈，將字元寫入 data register
	}
上面的function效果同 libopencm3 裡面寫好的 usart_send_blocking(uint32_t usart, uint16_t data)  
libopencm3 只有幫我們寫好收發字元的function，若要收發字串，要另外自己寫  
如  

	void console_puts(char *s)  
	{  
		while (*s != '\000') {  
			console_putc(*s);  
			/* Add in a carriage return, after sending line feed */  
			if (*s == '\n') {  
				console_putc('\r');  
			}  
			s++;  
		}
	}
  
  
### (2) 用 Systick 量 cycles 

可以參考 pqm4/common/hal-stm32f4.c裡 systick 的設定方法以及 pqm4/mupq/crypto_kem(or crypto_sign)裡面的6個測試檔的方法  
  
Systick 是一個會隨著 system clock 跳動而倒數的一個 counter ， 每次倒數至 0 後會中斷，重新reload至設定好的值後繼續倒數，並call sys_tick_handler() 這個 function，這個function的內容由使用者自己定義(預設是空的)  
  
以下是 hal-stm32f4.c 裡的部分code
  
	static void systick_setup(void)
	{
		// assumes clock_setup was called with CLOCK_BENCHMARK
		systick_set_clocksource(STK_CSR_CLKSOURCE_AHB);			//用來設定clocksource
		systick_set_reload(2399999);					//用來設定 reload 值 ， reload 至 2399999 ，倒數至 0 ，所以每次reload之後到數到 0 然後執行中斷，會經過 2400000 個 cycles
		systick_interrupt_enable();
		systick_counter_enable();
	}
	
	static unsigned long long overflowcnt = 0;				//用來記 systick 倒數到0並reload幾次 
	void sys_tick_handler(void)
	{
		++overflowcnt;
	}
	uint64_t hal_get_time()							//用來記從一開始到現在總過經過了幾個 cycles
	{
		return (overflowcnt+1)*2400000llu - systick_get_value();		
	}
  
然後參考 pqm4/mupq/crypto_kem(or crypto_sign) 裡面的6個 .c 檔
  
用 t0 及 t1 分別記錄想要測量的動作的起始時間(cycles)以及結束時間(cycles)，相減之後就可以得到經過多少cycles  
e.g.  

	t0 = hal_get_time();
	yourfunction();
	t1 = hal_get_time();
	print(t1-t0);
