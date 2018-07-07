# PWM-based-signal-generator

Here, a sine ware generator based on PWM of an Arduino is described. Almost no additional hardware is required (except for some resistances and capacitors to build a low pass filter). The tested signal frequency is from 20Hz to 1kHz. Several signals can be also synthesized. Of course you can also arrange the max. sinusoid output voltage from 2.5V to 5V.

The theory behind this is called DDS Method (Digital Direct Synthesis). The details wouldn't be discussed here. 

To run this software on an Arduino UNO connect a potentiometer to +5V and Ground and the wiper to analog 0. The frequency appears on pin 11 where you can connect active speakers or an output filter described later.

The communication between PC and Arduino can be realized using serial port, sothat you can change the frequency of one of your synthesized sinusoid signals.

```C
#include "avr/pgmspace.h"

#define cbi(sfr, bit) (_SFR_BYTE(sfr) &= ~_BV(bit))
#define sbi(sfr, bit) (_SFR_BYTE(sfr) |= _BV(bit))
 
// table of 256 sine values / one sine period / stored in flash memory 
PROGMEM  const unsigned sine256[] = 
{63,65,66,68,69,71,72,74,75,77,78,80,81,83,84,86,87,89,90,91,93,94,95,97,98,99,101,102,103,104,105,106,108,109,110,111,112,113,
114,115,115,116,117,118,119,119,120,121,121,122,122,123,123,124,124,124,125,125,125,126,126,126,126,126,126,126,126,126,126,126,
125,125,125,124,124,124,123,123,122,122,121,121,120,119,119,118,117,116,115,115,114,113,112,111,110,109,108,106,105,104,103,102,
101,99,98,97,95,94,93,91,90,89,87,86,84,83,81,80,78,77,75,74,72,71,69,68,66,65,63,61,60,58,57,55,54,52,51,49,48,46,45,43,42,40,
39,37,36,35,33,32,31,29,28,27,25,24,23,22,21,20,18,17,16,15,14,13,12,11,11,10,9,8,7,7,6,5,5,4,4,3,3,2,2,2,1,1,1,0,0,0,0,0,0,0,0,
0,0,0,1,1,1,2,2,2,3,3,4,4,5,5,6,7,7,8,9,10,11,11,12,13,14,15,16,17,18,20,21,22,23,24,25,27,28,29,31,32,33,35,36,37,39,40,42,43,
45,46,48,49,51,52,54,55,57,58,60,61
 
};
 
const double refclk = 31367.6; //freqeucy of PWM
const double ph = 0.711112;    //256/360° pwm 256 divide into 360°
 
double freq_F;      //freqeucy of fundamental wave
double freq_O;      //freqeucy of offset wave
double offset;      //phase offset
 
volatile byte Val_F;    // var of fundamental wave inside interrupt
volatile byte Val_O;    // var of offset wave inside interrupt
volatile byte ph_O;
 
volatile unsigned long ph2_O;      // phase increment of offset wave
volatile unsigned long phaccu_F;   // pahse accumulator of fundamental wave
volatile unsigned long tword_F;    // dds tuning word m of fundamental wave
volatile unsigned long phaccu_O;   // pahse accumulator of offset wave
volatile unsigned long tword_O;    // dds tuning word m of offset wave
 
void setup() {
  Serial.begin(9600);
  pinMode(11, OUTPUT);
  Serial.setTimeout(5);
   
  Setup_timer2();
   
  sbi (TIMSK2,TOIE2);      // enable Timer2 Interrupt 
 
  freq_O = 200.0;          // initial output frequency of fundamental wave = 200.o Hz
  freq_F = 400.0;          // initial output frequency of offset wave = 400.o Hz
 
  tword_O = pow(2,32) * freq_O / refclk;  // calulate DDS new tuning word
  tword_F = pow(2,32) * freq_F / refclk;  // calulate DDS new tuning word
}

//Using interrupt to update the new value for PWM
ISR(TIMER2_OVF_vect) {  
  phaccu_O += tword_O;    // soft DDS, phase accu with 32 bits
  phaccu_F += tword_F;    // soft DDS, phase accu with 32 bits
            
  OCR2A = pgm_read_byte_near(sine256 + Val_O) + pgm_read_byte_near(sine256 + Val_F); // f1 + f2
}
 
void loop() {
  ph_O = phaccu_O >> 24;          // use upper 8 bits for phase accu as frequency information
  Val_F = phaccu_F >> 24;         // use upper 8 bits for phase accu as frequency information
  
  //Read new frequency from serial port
  if(Serial.available() > 0){
  offset = Serial.parseInt();
  ph2_O = ph * offset;
  }
   
  if(offset == 0){
   Val_O = ph_O;
  }
  else{
   Val_O = ph_O + ph2_O;
   if(Val_O > 255) Val_O -= 255;
  }
}

//Setup Timer2 to Phase Correct PWM
void Setup_timer2() {
// Timer2 Clock Prescaler to : 1
  sbi (TCCR2B, CS20);
  cbi (TCCR2B, CS21);
  cbi (TCCR2B, CS22);
 
  // Timer2 PWM Mode set to Phase Correct PWM
  cbi (TCCR2A, COM2A0);  // clear Compare Match
  sbi (TCCR2A, COM2A1);
 
  // Mode 1  / Phase Correct PWM
  sbi (TCCR2A, WGM20);  
  cbi (TCCR2A, WGM21);
  cbi (TCCR2B, WGM22);
}
```

![a][1]










[1]data:image/png;base64,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
