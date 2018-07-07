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


<img src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg==" />
