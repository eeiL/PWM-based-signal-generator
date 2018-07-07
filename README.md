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

![avatar][200250]
















[200250]:data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFIAAAAhCAYAAABKmvz0AAAACXBIWXMAAA7EAAAOxAGVKw4bAAAHmklEQVRoge2YW0jU2xfHPzOaDeZkiopOWoZ5TfNWYdlDiiiE2cOIJKKYZBMaSA9iYFAPCUUQGY1p2eUh0IQirR6yoJuRomneLXRykFRMpxkddbJxfuchHM4csxyTc/719wP7Zdba67f3d9bee+0tEgRBYIVfRvxfD+BPYUXIZWJFyGViRchlYkXIZWJFyGViRchlYkEhBUGgrq6Ohw8fYjQa/80x/ZbYLmSYnJzk9u3bTExMEB0dzbp166wObjAYAJBIJEsf4W+C6Ec3m4GBAQwGA5s3b0YkElkVWBAElEolXl5e7N+//5cH+r/OghkJ4OXlteTAWq2WN2/e/FKM34kFM/LLly8olUoAcnNzAbh69SofPnygoKXV/P48WPy8/MxmUyUlpby4sUL3NzccHJyAiA9PR0PDw+Ki4vZu3cv+/fvt8j0kZERzp49S0hICBkZGXR1dXHu3LnvDlwmk3H8+HEkEglKpZLW1tbv+qWnp+Pk5MTFixfJz88nODjYbJuZmeHy5cvmOXl4eJhto6OjnDlzhqioKJKTk38q5A8PG41Gg0ajQRAEVq9eTXJyMsPDw5SXlzMzM2P27ejoQKlUkpCQwMaOJj7zrXQ0FAGBwfR6/XY2NggFovx9fWd57d27VrevXuHjY0N7u7uTE5O0t7ebjHHkZERnj9/Tl1dhttps://www.jxtxzzw.comHX19fRY2tVpNfX09mzZt+qmI8JOl/U/c3d3Jy8ujsLCQHTt2EBcXh0ajQalUsmvXLuLi4hCJRMTExODt7U1dXR179uyZt0cmJSXR3d1NWVkZRUVFODo6Ultby9OnTykqKsLFxQUAb29vjhw5Yu4nCAL3798H4PDhwzg4OACwb98+i/ijo6M0NzeTlJREdHQ0JpOJgIAAOjs7MRgM5sOvp6cHqVRKVFQUjY2NREVFYWv7TZLW1lY2bNiw6K3J6joyIiICuVxOSUkJfX193LlzB41GQ3Z2Nvb29ouKYW9vT0ZGBoODg1RVVdHX18f169dJS0tj+/btC/br6+vjypUr5ObmWizRv2M0GqmsrGR2dpbs7Gzs7OyQSCRs2bIFlUrF+Pg4AF+/fqW+vp7g4GBiY2Npa2tjYmIC+FZtqFQqgoKCzNvSz7BaSFtbW1JTU5HJZBQUFHD37l2OHTuGp6enVXF8fHzIycmhqqqKgoICAgICSElJWbA60Ol0XLhwgcjISOLj4xf0e/bsGdXV1SgUClxdXc2/h4SEMDQ0xMDAAACfP3+mq6uL8PBwgoKCGBoaQqVSATA2NkZPTw8hISGsWrVqUfNZ0s3G0dGRgwcPMjw8zLZt2wgPD19KGHbv3k1UVBTDw8PI5fIFM9poNFJVVcXg4CAZGRkL+vX391NSUoJcLiciIsLCJpPJ8PT0NB9K79+/x2AwEBAQwPr16/Hx8eHt27cIgsDHjx/R6XT4+/svei5LElKv11NRUYG7uzstLS20tbUtJQxNTU3U19fj7u5OTU0NU1NT3/Vrbm6mqqqKnJwcfHx8vuszNTXFjRs3kMlkpKammve6OaRSKf7+/qhUKiYnJ3n9+jVhYWG4ubkhlUqJjIykubkZvV5PR0cHfn5+rF+/ftFzsVpIo9FIRUUFKpWKoqIiEhMTKS4uZnhhttps://www.jxtxzzw.com42Ko4/f39nD9/HrlcTlFREe3t7dTU1PDPamx0dJTy8nISEhLYs2fPd2MJgsCTJ09oamri6NGjODo6zvNZtWoVISEh9PT08P79e9ra2ggNDcXOzg6AyMhIent76ezspLOzk8DAQPNhthisFnIuO7KysvD39yc5OZk1a9ZQWlpqvhICiMVibG1tMZlM82LMZY+bmxspKSn4+fmRmZnJtWvXaG5uNvsZjUZu3rwJQGZm5rwsm+Pdu3eUlZWhUCh+uBz9/f3R6/W0tLQwPT3N1q1bzbYNGzYgk8moq6tDrVYTFhZm1W3OKiE/ffpEWVkZMTExxMfHA+Di4oJCoeDVq1fU1taaM8rV1RVfX18ePHhAb28vWq2Wz58/IwgCtbW1vHjxgtzcXJydnRGJRMTHxxMdHU1ZWRmjo6PAt4PjwYMHyOVybG1t0Wq15qbT6TCZTOh0Oi5duoSfnx/btm1Dp9NZ+P39z53bC0tKSggICLAowNetW0dkZCT37t1DLBYvun60WsiZmRmuXr2KwWAgKyvLvCTgW0mUlpbGlStXzIWtvb09aWlpjI2NceDAAeLi4rh16xYdHR0UFxejUCgsShiJRMKhQ4fQarVUVlZiNBrp7u5Gr9dTWFhIXFycRcvLy2N8fByNRkN/fz8vX74kKSlpnt+jR4/M33BwcCAwMBCA7du3WzymiEQiIiIiMBqNeHl5Wf1Is+AV0WAwcPr0aQBOnDix5Bcck8lkrs+kUili8Z/5BGrVzWYpiMXi727+fxoLpsfs7CzT09NIJBJsbGz+zTH9lszLyIaGBlpaWlCr1TQ0NHDq1KlFV/f/z8zLSJPJRGNjI1NTU5w8eXLB2m0FS374Qr7C4Uhl4kVIZeJFSGXib8Ax7wacMdUd3oAAAAASUVORK5CYII=
