## General Info
- ___Config.h__ file must be modified to select which protocols will be available, change protocols/sub_protocols/settings associated with dial for PPM input, different TX channel orders and timing, Telemetry or not, ... This is mandatory since all available protocols will not fit in the ATmega328. You need to pick and choose what you want.
- __CYRF6936_SPI.ino__ is the low level registers interface driver to communicate with the CYRF6936 physical radio transceiver chip.
- __iface_cyrf6936.h__ is the header file where chip registers are defined using mnemonic names
- __Channel selection:__
	- Binding phase is realized using a fixed channel
	- Control phases use a different channel at each loop according to frequency hopping pattern. <br/>
	_Note: In frequency hopping systems, the transmitter changes the carrier frequency according to a certain "hopping" pattern . The advantage is that the signal sees a different channel (range of frequencies where the signal is transmitted) and a different set of interfering signals during each hop. This avoids the problem of	failing communication at a particular frequency, because of a fade or a particular interferer._
- __Binding notes:__
	- __RX Num__: number your different RXs and make sure only one model will react to the commands
	- __Autobind__: Yes or No. At the model selection (or power applied to the TX) a bind sequence will be initiated
	- __TX ID__: The multiprotocol TX module is using a 32bits ID generated randomly at first power up. This global ID is used by nearly all protocols. There are little chances to get a duplicated ID. __For DSM2/X and Devo the CYRF6936 unique manufacturer ID is used.__ It's possible to generate a new ID using bind button on the Hubsan protocol during power up.
	- __Bind__:	
		- To bind a model in _PPM Mode_ press the physical bind button, apply power and then release.
		- In _Serial Mode_ you have 2 options:
			- Use the GUI, access the model protocol page and long press on Bind. This operation can be done at anytime.
			- Press the physical bind button, apply power and then release. It will request a bind of the first loaded model protocol.<br/>
			Notes: the physical bind button is only effective at power up. Pressing the button later has no effects. A bind in progress is indicated by the LED fast blinking. Make sure to bind during this period.
- __DSMX protocol:__ In the DSMX protocol the transmitter and receiver both use the transmitter radio chip ID,which is send during the binding process, for generating 23 channels. Each time the transmitter transmits a packet or the receiver receives a packet they will hop to the next channel. __TO BE CONTINUED...__
- __ReadDsm() flow:__ when a loop is executed and a certain phase is called, then 2 different situations can happen:
	- The phase remains the same until a change phase event happens, software keeps looping the same phase over and over
	- Some events can make the phase switch (usually phase++) so that code execution continues and a different phase is called
- __Transmission interaction__: every interaction with the receiver is carried out in 3 phases _WRITE + CHECK + READ_:
	1. WRITE: Send out packet to receiver; 
	2. CHECK: Wait for response from the receiver;
	3. READ: Read response data if any answer has been received.
- __FW compilation:__ the code seems to compile ONLY on _windows_, using _Arduino IDE_. Some specific IDE settings must be set, check [Compiling and Programming guide](https://github.com/pascallanger/DIY-Multiprotocol-TX-Module/blob/master/docs/Compiling.md).

## Raw Flow Analysis (no WAIT_FOR_BIND)
```c
setup() {
	...
	if( IS_BIND_BUTTON_on )	{BIND_BUTTON_FLAG_on;}
	...
	SERIAL {
		Mprotocol_serial_init();	// serial setup end
	}
	PPM {
		protocolInit();	->
			initDsm();
			remoteCallback = ReadDsm;	// ppm setup end
	}
}

loop() {
	...
	// SERIAL calls UpdateAll() until protocolInit() is called and remote_callback is defined
	// PPM jumps directly to remote_callback() invocation
	while (!remote_callback) {
		UpdateAll() ->
			update_serial_data(); ->
				CHANGE_PROTOCOL_FLAG_on;
			if (CHANGE_PROTOCOL_FLAG_on) {
				protocolInit();
			}
	}
	// at this point remote_callback() is ReadDsm()
	next_callback=remote_callback();
}
```				


## Main Snippets:
#### setup code
```c
/* ##MultiProtocol.ino## */	
typedef uint16_t (*void_function_t) (void);	//pointer to a function with no parameters which return an uint16_t integer
void_function_t remote_callback = 0;
		
setup() {	// ##MultiProtocol.ino-->setup()##		
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
	#endif 
	...
	#ifdef ENABLE_SERIAL	// Serial
		...
		Mprotocol_serial_init(); 	// Configure serial and enable RX interrupt
	#endif
	...
}

static void protocol_init() {	// ##Multiprotocol.ino-->protocolInit()##
	static uint16_t next_callback;
	if(IS_WAIT_BIND_off) {
		...
		// #alx# global TxId to bind Multi-module with Receiver
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
	// #alx# if AUTOBIND is DISABLED and no binding operation is performed (by pressing the button etc) WAIT_BIND flag is raised
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
// #alx# DSM protocol init, this method sets the chip to TX mode and performs preliminary binding operations
uint16_t initDsm()	{	// ##DSM_cyrf6936.ino-->initDsm()##
	// #alx# TxId related operations
	CYRF_GetMfgData(cyrfmfg_id);
	//Model match
	cyrfmfg_id[3]^=RX_num;
	//Calc sop_col
	sop_col = (cyrfmfg_id[0] + cyrfmfg_id[1] + cyrfmfg_id[2] + 2) & 0x07;
	//Fix for OrangeRX using wrong pncodes by preventing access to "Col 8"
	if(sop_col==0)
	{
	   cyrfmfg_id[rx_tx_addr[0]%3]^=0x01;					//Change a bit so sop_col will be different from 0
	   sop_col = (cyrfmfg_id[0] + cyrfmfg_id[1] + cyrfmfg_id[2] + 2) & 0x07;
	}
	//Hopping frequencies
	// #alx# the transmitter changes channel at each loop according to a certain "hopping" pattern. Channel list calculation is based on TxId
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
	// #alx# setting CYRF chip straight into TX mode in order send BIND_PACKET or CONTROL_PACKETS the receiver
	CYRF_SetTxRxMode(TX_EN);
	//
	DSM_update_channels();
	// #alx# at "setup" time select the first phase to be executed when "looping" [readDSM()]
	if(IS_AUTOBIND_FLAG_on )
	{
		BIND_IN_PROGRESS;
		DSM_initialize_bind_phase();		
		phase = DSM_BIND_WRITE;			// #alx# selected if AUTOBIND is ON or if bind button is pressed at startup
		bind_counter=DSM_BIND_COUNT;
	}
	else
		phase = DSM_CHANSEL;			// #alx# selected if binding is already done
	return 10000;
}

// #alx# initialize bindig phase
static void __attribute__((unused)) DSM_initialize_bind_phase() {	// ##DSM_cyrf6936.ino-->DSM_initialize_bind_phase()##
	CYRF_ConfigRFChannel(DSM_BIND_CHANNEL); //This seems to be random?		#alx# set fixed channel for binding (channel 13)
	//64 SDR Mode is configured so only the 8 first values are needed but need to write 16 values...
	CYRF_ConfigDataCode((const uint8_t*)"\xD7\xA1\x54\xB1\x5E\x89\xAE\x86\xc6\x94\x22\xfe\x48\xe6\x57\x4e", 16);
	DSM_build_bind_packet();	// #alx# build fixed BIND_PACKET that will be sent to the receiver
}
```
#### loop code
```c
/* ##MultiProtocol.ino## */
// Main #alx# this method is the main program loop
// Protocol scheduler
void loop() { 	// ##Multiprotocol.ino-->loop()##
	// #alx# next_callback is the number of microSecs to wait before executing next remote_callback
	uint16_t next_callback,diff=0xFFFF;
	// #alx# main loop cycle
	while(1) {
		// #alx# execute Update_All() that will retry protocolInit() until remote_callback is not defined and binding is completed
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

uint8_t Update_All() {	// ##Multiprotocol.ino-->Update_All()##
	#ifdef ENABLE_SERIAL
		if(mode_select==MODE_SERIAL && IS_RX_FLAG_on) {		// Serial mode and something has been received
			update_serial_data();							// Update protocol and data
			...
		}
	#endif //ENABLE_SERIAL
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

void update_serial_data() {		// ##Multiprotocol.ino-->update_serial_data()##
	...
	if( (rx_ok_buff[0] != cur_protocol[0]) || ... )	{ // New model has been selected
		CHANGE_PROTOCOL_FLAG_on;				//change protocol
		WAIT_BIND_off;
		...
	} else if( ((rx_ok_buff[1]&0x80)!=0) && ((cur_protocol[1]&0x80)==0) ) {		// Bind flag has been set
			CHANGE_PROTOCOL_FLAG_on;
	} else {
		...
	}
	...	
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

// #alx# main operations method called during loop phase
uint16_t ReadDsm() {	// ##DSM_cyrf6936.ino-->ReadDsm()##
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
		...
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
