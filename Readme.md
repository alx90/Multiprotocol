# Multiprotocol Tx Module software
### Forked from [DIY-Multiprotocol-TX-Module](https://github.com/pascallanger/DIY-Multiprotocol-TX-Module)

## Quicklinks
* [Compiling and Programming guide](https://github.com/pascallanger/DIY-Multiprotocol-TX-Module/blob/master/docs/Compiling.md)
* [The old documentation](https://github.com/pascallanger/DIY-Multiprotocol-TX-Module/blob/master/docs/README-old.md)
* [CYRF6939 chip datasheet](http://www.cypress.com/file/126466/download)
* [Forum on rcgroups](http://www.rcgroups.com/forums/showthread.php?t=2165676)

## Main Flow Snippets:

### setup code
```c
/* ##MultiProtocol.ino## */	
typedef uint16_t (*void_function_t) (void);//pointer to a function with no parameters which return an uint16_t integer
void_function_t remote_callback = 0;
		
setup() {		// ##MultiProtocol.ino-->setup()##		
	...
	// Read status of bind button
	if( IS_BIND_BUTTON_on )		// #alx# if pressing the BIND_BUTTON at startup, BIND_BUTTON_FLAG is set to ON
		BIND_BUTTON_FLAG_on;	// If bind button pressed save the status for protocol id reset under hubsan
	...
	#ifdef ENABLE_PPM
		if(mode_select != MODE_SERIAL) {
			...
			protocolInit();		// ##Multiprotocol.ino-->protocolInit()##
			...
		}
	#endif //ENABLE_PPM
	...
}

static void protocol_init() {
	static uint16_t next_callback;
	if(IS_WAIT_BIND_off) {
		...
		// #alx# global TX/RX_ID to bind Multi-module with Receiver
		//Set global ID and rx_tx_addr
		MProtocol_id = RX_num + MProtocol_id_master;
		set_rx_tx_addr(MProtocol_id);
		...
		// #alx# if the bind button was pressed at startup, BIND_IN_PROGRESS flag will be set
		if(IS_BIND_BUTTON_FLAG_on)
			AUTOBIND_FLAG_on;
		if(IS_AUTOBIND_FLAG_on)
			BIND_IN_PROGRESS;			// Indicates bind in progress for blinking bind led
		else
			BIND_DONE;
		...
		switch(protocol) {
			...
			#if defined(DSM_CYRF6936_INO)
				case MODE_DSM:
					PE2_on;	//antenna RF4
					// #alx# next_callback is the number of ms for next loop?
					next_callback = initDsm(); // #alx# DSM_CYRF6936_INO.initDsm()
					remote_callback = ReadDsm; //puts the whole ReadDsm function into remote_callback #alx# DSM_CYRF6936_INO.ReadDsm
					break;
			#endif
			...
		}
	}
	#if defined(WAIT_FOR_BIND) && defined(ENABLE_BIND_CH)
		if( IS_AUTOBIND_FLAG_on && ! ( IS_BIND_CH_PREV_on || IS_BIND_BUTTON_FLAG_on || (cur_protocol[1]&0x80)!=0 ) )
		{
			WAIT_BIND_on;
			return;
		}
	#endif
	WAIT_BIND_off;
	...
}

/* ##DSM_cyrf6936.ino## */
uint16_t initDsm()	{	// #alx# DSM protocol init, this method sets the chip to TX mode and performs binding operations
	...
	//Hopping frequencies #alx# TODO
	if (sub_protocol == DSMX_11 || sub_protocol == DSMX_22)
		DSM_calc_dsmx_channel();
	else
	{ 
		uint8_t tmpch[10];
		CYRF_FindBestChannels(tmpch, 10, 5, 3, 75);
		//
		uint8_t idx = random(0xfefefefe) % 10;
		hopping_frequency[0] = tmpch[idx];
		while(1)
		{
			idx = random(0xfefefefe) % 10;
			if (tmpch[idx] != hopping_frequency[0])
				break;
		}
		hopping_frequency[1] = tmpch[idx];
	}
	//
	DSM_cyrf_config();
	// #alx# setting CYRF chip straight into TX mode in order to control the receiver
	CYRF_SetTxRxMode(TX_EN);
	//
	DSM_update_channels();
	// if AUTOBIND is disabled,
	// #alx# at "setup" time select the first phase to be executed when "looping" [readDSM()]
	if(IS_AUTOBIND_FLAG_on )
	{
		BIND_IN_PROGRESS;
		DSM_initialize_bind_phase();		
		phase = DSM_BIND_WRITE;			// #alx# selected if AUTOBIND is ON or if bind button is pressed at startup
		bind_counter=DSM_BIND_COUNT;
	}
	else
		phase = DSM_CHANSEL;			// #alx# selected if no binding operation is performed
	return 10000;
}

// #alx# initialize bindig phase
static void __attribute__((unused)) DSM_initialize_bind_phase() {
	CYRF_ConfigRFChannel(DSM_BIND_CHANNEL); //This seems to be random?		#alx# set fixed channel for binding (channel 13)
	//64 SDR Mode is configured so only the 8 first values are needed but need to write 16 values...
	CYRF_ConfigDataCode((const uint8_t*)"\xD7\xA1\x54\xB1\x5E\x89\xAE\x86\xc6\x94\x22\xfe\x48\xe6\x57\x4e", 16);
	DSM_build_bind_packet();	// #alx# build fixed BIND_PACKET that will be sent to the receiver
}
```
### loop code
```c
/* ##MultiProtocol.ino## */
// Main #alx# this method is the main program loop
// Protocol scheduler
void loop() { 
	// #alx# next_callback is the number of microSecs to wait before executing next remote_callback
	uint16_t next_callback,diff=0xFFFF;
	// #alx# main loop cycle
	while(1) {
		// #alx# execute Update_All() that will retry protocolInit() until remote_callback is not defined
		if(remote_callback==0 || IS_WAIT_BIND_on || diff>2*200)	{
			do {
				Update_All();
			} while(remote_callback==0 || IS_WAIT_BIND_on);
		}
		...
		do {
			TX_MAIN_PAUSE_on;
			tx_pause();
			if(IS_INPUT_SIGNAL_on && remote_callback!=0)
				next_callback=remote_callback();	// #alx# this is where remote_callback() gets actually called - note that after setup remote_callback() is readDSM()
			else
				next_callback=2000;					// No PPM/serial signal check again in 2ms...
			TX_MAIN_PAUSE_off;
			tx_resume();
			...
		} while(diff&0x8000);	 					// Callback did not took more than requested time for next callback
													// so we can launch Update_All before next callback
	}
}

/* ##DSM_cyrf6936.ino## */
// #alx# readDSM phases enum
enum {
	DSM_BIND_WRITE=0,
	DSM_BIND_CHECK,
	DSM_BIND_READ,
	DSM_CHANSEL,
	DSM_CH1_WRITE_A,
	DSM_CH1_CHECK_A,
	DSM_CH2_WRITE_A,
	DSM_CH2_CHECK_A,
	DSM_CH2_READ_A,
	DSM_CH1_WRITE_B,
	DSM_CH1_CHECK_B,
	DSM_CH2_WRITE_B,
	DSM_CH2_CHECK_B,
	DSM_CH2_READ_B,
};

uint8_t Update_All()
{
	...
	#ifdef ENABLE_BIND_CH
		if(IS_AUTOBIND_FLAG_on && IS_BIND_CH_PREV_off && Servo_data[BIND_CH-1]>PPM_MAX_COMMAND && Servo_data[THROTTLE]<(servo_min_100+25))
		{ // Autobind is on and BIND_CH went up and Throttle is low
			CHANGE_PROTOCOL_FLAG_on;							//reload protocol to rebind
			BIND_CH_PREV_on;
		}
		...
	#endif //ENABLE_BIND_CH
	if(IS_CHANGE_PROTOCOL_FLAG_on)
	{ // Protocol needs to be changed or relaunched for bind
		protocol_init();									//init new protocol
		return 1;
	}
	return 0;
}

// #alx# main operations method called during loop phase
uint16_t ReadDsm() {
	...
	switch(phase) {
		// #alx# fixed BIND_PACKET send phase
		case DSM_BIND_WRITE:
			if(bind_counter--==0)
			#if defined DSM_TELEMETRY
				phase=DSM_BIND_CHECK;						//Check RX answer
			#else
				phase=DSM_CHANSEL;							//Switch to normal mode
			#endif
			CYRF_WriteDataPacket(packet);					// #alx# send pre built BIND_PACKET to receiver (chip is in TX mode at this point)
			return 10000;
		#if defined DSM_TELEMETRY
			// #alx# set chip into receive mode, and check if any answer from possible receivers is found (sets a timeout before checking again)
			case DSM_BIND_CHECK:
				//64 SDR Mode is configured so only the 8 first values are needed but we need to write 16 values...
				CYRF_ConfigDataCode((const uint8_t *)"\x98\x88\x1B\xE4\x30\x79\x03\x84\xC9\x2C\x06\x93\x86\xB9\x9E\xD7", 16);
				CYRF_SetTxRxMode(RX_EN);						//Receive mode
				CYRF_WriteRegister(CYRF_05_RX_CTRL, 0x87);		//Prepare to receive #alx# sets to 1 RX_GO bit into 0x05 RX_CTRL_ADR registry
				bind_counter=2*DSM_BIND_COUNT;					//Timeout of 4.2s if no packet received
				phase++;										// change from BIND_CHECK to BIND_READ
				return 2000;
			// #alx# read any eventual answer packet received
			case DSM_BIND_READ:
				//Read data from RX
				rx_phase = CYRF_ReadRegister(CYRF_07_RX_IRQ_STATUS);
				if((rx_phase & 0x03) == 0x02)  					// RXC=1, RXE=0 then 2nd check is required (debouncing)
					rx_phase |= CYRF_ReadRegister(CYRF_07_RX_IRQ_STATUS);
				if((rx_phase & 0x07) == 0x02)
				{ // data received with no errors
					CYRF_WriteRegister(CYRF_07_RX_IRQ_STATUS, 0x80);	// need to set RXOW before data read
					len=CYRF_ReadRegister(CYRF_09_RX_COUNT);			// #alx# checks if any data has been received
					if(len>MAX_PKT-2)
						len=MAX_PKT-2;
					CYRF_ReadDataPacketLen(pkt+1, len);
					if(len==10 && DSM_Check_RX_packet())
					{
						pkt[0]=0x80;
						telemetry_link=1;						// send received data on serial
						phase++;								// #alx# change to CHANSEL
						return 2000;
					}
				}
				else
					if((rx_phase & 0x02) != 0x02)
					{ // data received with errors
						CYRF_WriteRegister(CYRF_29_RX_ABORT, 0x20);	// Abort RX operation
						CYRF_SetTxRxMode(RX_EN);					// Force end state read
						CYRF_WriteRegister(CYRF_29_RX_ABORT, 0x00);	// Clear abort RX operation
						CYRF_WriteRegister(CYRF_05_RX_CTRL, 0x83);	// Prepare to receive
					}
				if( --bind_counter == 0 )
				{ // Exit if no answer has been received for some time
					phase++;									// DSM_CHANSEL
					return 7000 ;
				}
				return 7000;
		#endif
		// #alx# this case gets called when BINDING is completed (WRITE + CHECK + READ)
		case DSM_CHANSEL:
			BIND_DONE;
			DSM_cyrf_configdata();
			CYRF_SetTxRxMode(TX_EN); 						// #alx# binding is completed, can switch to Transmission mode
			hopping_frequency_no = 0;
			phase = DSM_CH1_WRITE_A;						// in fact phase++
			DSM_set_sop_data_crc();
			return 10000;
		// #alx# the following cases get called after CHANSEL in order to send control data to the receiver
		...
	}
}
```

## NOTES
### Multi-Protocol
- __CYRF6936:__ physical radio transceiver chip
- __CYRF6936_SPI.ino__ is the low level "registry writer" driver class
- __iface_cyrf6936.h__ is the header file where chip registries are defined using mnemonic names
- __Frequency Hopping__: In frequency hopping systems, the transmitter changes the carrier frequency according to a certain "hopping" pattern . The advantage is that the signal sees a different channel (range of frequencies where the signal is transmitted) and a different set of interfering signals during each hop. This avoids the problem of failing communication at a particular frequency, because of a fade or a particular interferer.
- __Binding notes:__
	- If __Autobind__ is set to Yes, At the model selection (or power applied to the TX) a bind sequence will be initiated
	- __TX ID__: 
	The multiprotocol TX module is using a 32bits ID generated randomly at first power up. This global ID is used by nearly all protocols. There are little chances to get a duplicated ID. 
	For DSM2/X and Devo the CYRF6936 unique manufacturer ID is used. 
	It's possible to generate a new ID using bind button on the Hubsan protocol during power up.
	- __Bind__:	To bind a model in PPM Mode press the physical bind button, apply power and then release.
	In Serial Mode you have 2 options:
	use the GUI, access the model protocol page and long press on Bind. This operation can be done at anytime.
	press the physical bind button, apply power and then release. It will request a bind of the first loaded model protocol. <br/>
	Notes: the physical bind button is only effective at power up. Pressing the button later has no effects.
	a bind in progress is indicated by the LED fast blinking. Make sure to bind during this period.
- __readDsm() cases flow:__ when a loop is executed and a certain phase is called, then 2 different situations can happen:
	- The phase remains the same until a change phase event happens, software keeps looping the same phase over and over
	- Some events can make the phase switch (usually phase++) so that code execution continues and a different phase is called
- Every interaction with the receiver is carried out in 3 phases (__WRITE + CHECK + READ__):
	1. Send out packet to receiver; 
	2. Wait for response from the receiver;
	3. Read response data if any answer has been received.

### General
- Arduino Sketch:
	- A sketch is simply a file containing an arduino program and is defined with the .ino extension.
	- There are two special functions that are a part of every Arduino sketch: setup() and loop(). 
		The setup() is called once, when the sketch starts. It's a good place to do setup tasks like setting pin modes or initializing libraries.
		The loop() function is called over and over and is heart of most sketches. You need to include both functions in your sketch, even if you don't need them for anything.
- intN_t types: simple integers types composed by N bits [ex uint8_t = 0 to 2^8 (0 to 255)]
- x >> y means shift x var bits of y positions rightwards (can be applied lefwards using <<)
- cli() and sei() methods are used respectively to disable and re-enable interrupts so that instructions included between cli() and sei() cannot be interrrupted ex:
	``` c
	void loop(){
	  cli();
	  foobar(); // critical, time-sensitive code here
	  sei();
	}
	```

## TODO
- CHECK WHERE GLOBAL __TX/RX_ID__ IS USED - note: ReadDSM()->CYRF_WriteDataPacket(__packet__); // __packet__ should contain the TX/RX_ID somewhere... 
- __FW COMPILATION:__ try to import the project into Arduino IDE in windows env and make it compile. If it compiles, try to make the same in Sloeber under windows. Check [Compiling and Programming guide](https://github.com/pascallanger/DIY-Multiprotocol-TX-Module/blob/master/docs/Compiling.md) for IDE settings.
- __FW IMPLEMENTATION:__ A PRE_BIND phase must be added before the binding phase. In this phase the software should:
	- Set in receive mode and check if any other module is already transmitting with a given ID
	- If any module is detected:
		- Read the ID from the already transmitting module
		- Send the ID to the receiver and perform the binding phase with the detected ID
	<br/>
	- Change initDSM() so that the new PRE_BIND phase is selected before looping
	- Add the new PRE_BIND case into readDSM();
	- Make the PRE_BIND phase selectable from _Config.h
- __PPM or SERIAL:__ 
- ___BV()__: "BV" stands for "bit value" and _BV() is a function that associates a numeric bit value to....
- __bitwise assignment operators__ ^= etc...