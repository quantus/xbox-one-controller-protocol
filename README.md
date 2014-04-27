Xbox One Controller Protocol
============================


This information is result of brute force reverse engineering of XBox One Controller protocol. This is done by sending lots of different packets to the controller and observing results. This document lists some known packets and what they do. There is also a list of packets that controller can send back to computer. This document is incomplete and contains errors.

Each packet begins with single byte that tells what type of packet it is. I will call this byte packet type. This code is 5 bits long, so there are maximum of 32 different packet types. First 3 bits are dummy, so they don't matter at all.

The controller rumble effects are really complex. Some packets that don't play the same effect every time. Also delays between some packets can alter the played effect. 

**Packet notation**

D means dummy bit, that doesn't have any effect. Some packets are listed as C++ source code.

**Rumble naming:**

    lT = left trigger
    rT = right trigger
    L = left motor
    R = right motor

Different packet types + examples
=================================

0x04: Start controller (no input)
---------------------------------
> BIN: DDD0 0100 DD1D D000

> HEX: 0x04 0x20

Starts controller's light but controller will not start sending any input events.


0x05: Start controller (with input)
-----------------------------------
> BIN: DDD0 0101 DD1D D000

> HEX: 0x05 0x20

Starts controller's light and controller starts sending input events.

0x09: Activate rumble
---------------------

	buffer[0] = 0x09;
	buffer[1] = 0x08; // 0x20 bit and all bits of 0x07 prevents rumble effect. 
	buffer[2] = 0x00; // This may have something to do with how many times effect is played
	buffer[3] = 0x??; // Substructure (what substructure rest of this packet has)


If you send too long packet, overflown data is ignored. If you send too short packet, missing values are considered as 0.

Buffer[3] defines what the rest of this packets is:

**Continous rumble effect**

	buffer[3] = 0x08; // Substructure (what substructure rest of this packet has)
	buffer[4] = 0x00; // Mode
	buffer[5] = 0x0f; // Rumble mask (what motors are activated) (0000 lT rT L R)
	buffer[6] = 0x04; // lT force
	buffer[7] = 0x04; // rT force
	buffer[8] = 0x20; // L force
	buffer[9] = 0x20; // R force
	buffer[10] = 0x80; // Length of pulse
	buffer[11] = 0x00; // Period between pulses

**Single rumble effect**

	buffer[3] = 0x09; // Substructure (what substructure rest of this packet has)
	buffer[4] = 0x00; // Mode
	buffer[5] = 0x0f; // Rumble mask (what motors are activated) (0000 lT rT L R)
	buffer[6] = 0x04; // lT force
	buffer[7] = 0x04; // rT force
	buffer[8] = 0x20; // L force
	buffer[9] = 0x20; // R force
	buffer[10] = 0x80; // Length of pulse

**Play effect**

	buffer[3] = 0x04; // Substructure (what substructure rest of this packet has)
	buffer[4] = 0x00; // Mode
	buffer[5] = 0x0f; // Rumble mask (what motors are activated) (0000 lT rT L R)
	buffer[6] = 0x04; // lT force ?
	buffer[7] = 0x04; // rT force ?

**Short rumble packet**

	buffer[3] = 0x02; // Substructure (what substructure rest of this packet has)
	buffer[4] = 0x00; // Mode
	buffer[5] = 0x0f; // Rumble mask (what motors are activated) (0000 lT rT L R) ?
	buffer[6] = 0x04; // lT force ?
	buffer[7] = 0x04; // rT force ?


**List of different modes**

- 0x00: Normal
- 0x20: Normal, but sometimes prevents rest of 0x00 mode effects
- 0x40: Triggerhell (= Pressing trigger starts rumbling all motors fast. I assume there is way to upload some pattern here, but don't know how.)
- 0x41: Trigger effect (if substructure = 4?) (= Pressing trigger causes single rumble effect)(if buffer[6] >= 0x0b this rumble effect can be played multiple times. if buffer[6] > 0x4F this doesn't work)
- 0x42: Fast + short rumble once


0x07: Load rumble effect
------------------------

	buffer[0] = 0x07, 
	buffer[1] = 0x85, // Atleast one of 0x07 bits must be on
	buffer[2] = 0xa0, // Dummy ?
	buffer[3] = 0x20, // L force 
	buffer[4] = 0x20, // R force 
	buffer[5] = 0x30, // length
	buffer[6] = 0x20, // period
	buffer[7] = 0x02, // Effect extra play count (0x02 means that effect is played 1+2 times)
	buffer[8] = 0x00, // Dummy ?

0x01, 0x06: Crash controller
----------------------------

**0x01 Crashes when:**

    bool is_1_packet_kill(uint8_t buffer[])
    {
    	return 
    		(
    			((buffer[1] & 0x27) == 0x20) && 
    			(buffer[4] == 0x00) &&
    			(buffer[9] > 1 || (buffer[10] & 0x0F))
    		) || (
    			(buffer[1] & 0x20) &&
    			(buffer[4] == 0x00) &&
    			(buffer[5] == 0x04) &&
    			(buffer[9] > 1 || (buffer[10] & 0x0F))
    		);
    }

Simple crash packet (that I use to reset my controller):

    0x01, 0x20, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0E

**0x06 crash packet example**

    0x06, 0x20, 0x0, 0x10, 0x0, 0x08, 0x0, 0x0, 0x38 

**Another 0x06 crash packet example**

    0x06, 0x20, 0x0, 0x02, 0x0, 0x1 (if 0x02 is replaced with 0x06 it renders controller immune of this packet)
    

Different packets that controller sends back
============================================

0x20: Button data
-----------------
Sent everytime controllor input values change.
Following code is from taken from [Chrome web browser](https://code.google.com/p/chromium/codesearch#chromium/src/content/browser/gamepad/xbox_data_fetcher_mac.cc).

    struct XboxOneButtonData {
    	uint8_t type;
    	uint8_t const_0;
    	uint16_t id;
    
    	bool sync : 1;
    	bool dummy1 : 1;  // Always 0.
    	bool start : 1;
    	bool back : 1;
    
    	bool a : 1;
    	bool b : 1;
    	bool x : 1;
    	bool y : 1;
    
    	bool dpad_up : 1;
    	bool dpad_down : 1;
    	bool dpad_left : 1;
    	bool dpad_right : 1;
    
    	bool bumper_left : 1;
    	bool bumper_right : 1;
    	bool stick_left_click : 1;
    	bool stick_right_click : 1;
    
    	uint16_t trigger_left;
    	uint16_t trigger_right;
    
    	int16_t stick_left_x;
    	int16_t stick_left_y;
    	int16_t stick_right_x;
    	int16_t stick_right_y;
    };
    
0x03: Heartbeat
---------------
    struct XboxOneHeartbeatData{
    	uint8_t type;
    	uint8_t const_20;
    	uint16_t id;
    
    	uint8_t dummy_const_80;
    	uint8_t first_after_controller;
    	uint8_t dummy1;
    	uint8_t dummy2;
    };
    
0x07: Guide button status
-------------------------
Sent when guide button status changes.

    struct XboxOneGuideData{
    	uint8_t type;
    	uint8_t const_20;
    	uint16_t id;
    
    	uint8_t down;
    	uint8_t dummy_const_5b;
    };
    
0x01: Invalid Op data ?
-----------------------
Sent when you send something multiple start packets or something. 
query + query2 variables contain first 2 bytes of invalid packet?
    
    struct XboxOneInvalidOpData{
    	uint8_t type;
    	uint8_t const_20;
    	uint16_t const_ff_09;
    
    	uint8_t const_0;
    	uint8_t query;
    	uint8_t query2;
    	uint16_t dummy;
    	uint32_t dummy_;
    };

0x02: Waiting connection
------------------------
Sent periodically when controller is connected but not started.

    struct XboxOneWaitingConnectionData{
    	uint8_t type;
    	uint8_t const_20;
    	uint16_t id;
    
    	uint8_t dummy[28];
    };

0x00: Unknown
-------------
This is received if you send this

    {
    	0x1, 0x1, 0x0, 0xbe, 0x10, 0x10, 0x21, 0x80, 0x80, 0x0, 0x10, 0xc0,
		0x0, 0x8, 0x61, 0x4, 0x0, 0x48, 0x0, 0x90, 0x1, 0x0, 0x20, 0x0,
		0x80, 0x60, 0x0, 0x0, 0x2, 0x0, 0x8, 0x8, 0x0, 0x28, 0x20, 0x10,
		0x0, 0x4, 0x8c, 0x0, 0x10, 0x31, 0x94, 0x4, 0x22, 0x80, 0x0, 0x8,
		0x8, 0x10, 0x0, 
	}
    	
or this packet

	{
    	0x0, 0xc0, 0x0, 0x20, 0x0, 0xbe, 0x1, 0x8a, 0x49, 0x1, 0x0, 0x0,
    	0x40, 0x0, 0x2, 0x33, 0x0, 0x8, 0x40, 0x8, 0x0, 0x0, 0x4, 0x8,
    	0x0, 0x0, 0x0, 0x5, 0x0, 0x0, 0x0, 0x0, 0x0, 0x1, 0x0, 0x50,
    	0x81, 0x0, 0x0, 0x26, 0x0, 0x2, 0x0, 0xa4, 0x4c, 0x0, 0x48, 0x40,
    	0x0, 0x0, 0x14, 
	}

0x04: Unknown
-------------
    struct XboxOneUnknown4Data{
        uint8_t type;
    
    	uint8_t unknown[0x3F];
    };
    
If received packet is longer, it contains some other packet after this one. There might also be smaller packet with this same type.

This is received if you send this

    {
    	0x1, 0xa8, 0x0, 0xbe, 0x0, 0x0, 0x10, 0x40, 0x0, 0x0, 0x0, 0x64,
    	0x1, 0x0, 0x4, 0x10, 0x2, 0x4, 0x0, 0x8, 0x4, 0x80, 0x40, 0x4d,
    	0x14, 0x4, 0x0, 0x2, 0x10, 0x0, 0x12, 0x80, 0x42, 0x0, 0x69, 0x0,
    	0x4, 0x40, 0x0, 0x0, 0x5, 0x0, 0xa, 0x0, 0x0, 0x0, 0xa, 0x49,
    	0x1, 0x20, 0x0, 
	}


0x06: Unknown
-------------
    struct XboxOneUnknown6Data{
    	uint8_t type;
    	uint8_t const_30;
    	uint16_t id;
    
    	uint8_t const_0;
    	uint8_t dummy[5];
    };
    


    
