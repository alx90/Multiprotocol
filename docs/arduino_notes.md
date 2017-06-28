## Arduino technical notes
- __Arduino Sketch:__ A sketch is simply a file containing an arduino program and is defined with the .ino extension. <br/>
There are two special functions that are a part of every Arduino sketch: setup() and loop():
	- _setup()_: is called once, when the sketch starts. It's a good place to do setup tasks like setting pin modes or initializing libraries.
	- _loop()_: is called over and over and is heart of most sketches. You need to include both functions in your sketch, even if you don't need them for anything.
- __uintN_t types:__ simple integers types composed by N bits [ex uint8_t = 0 to 2^8 (0 to 255)]
- __x >> y__ means shift x var bits of y positions rightwards (can be applied lefwards using <<)
- __cli()__ and __sei()__ methods are used respectively to disable and re-enable interrupts so that instructions included between cli() and sei() cannot be interrupted ex:
	``` c
	void loop(){
	  cli();
	  foobar(); // critical, time-sensitive code here
	  sei();
	}
	```
- ___BV()__ is a macro used in Arduino to manipulate single bits. Here is the _BV() macro definition:
	``` c
	#define _BV(bit) (1 << (bit))
	```
	So _BV(7) simply shifts the 1 left of 7 places. Now there are some peculiar instructions used to manipulate single bits:
	``` c
	PORTB |= _BV(5);	// SET bit 5 of PORTB
	PORTB &= ~_BV(5);	// CLEAR bit 5 of PORTB
	```
	The compiler recognizes the |= () and the &= ~() forms and reduces them to a single instruction, sbi or cbi.
- __#ifdef vs #if defined__: The difference between the two is that #ifdef can only use a single condition, while #if defined(NAME) can do compound conditionals like for instance:
	``` c
	#if defined(WIN32) && !defined(UNIX)
		/* Do windows stuff */
	#elif defined(UNIX) && !defined(WIN32)
		/* Do linux stuff */
	#else
		/* Error, both can't be defined or undefined same time */
	#endif
	```