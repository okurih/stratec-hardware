# STRATEC Biomedical hardware challenge

## Assignment
As understood, lcd displays current position, desired number of turns ( with decimal accuracy ), angular velocity, and ON/OFF state for movement activated by a button press. The two potentiometers address the number of turns (displayed) and the step delay, therefore adjusting the velocity (not displayed, too cramped).


| Component | Part |
| - | - |
| Microcontroller | Arduino nano |
| Motor | Nema 17 Stepper |
| Supply | generic 12v36W |
| Driver | A4988 |
| Display | HY1602E LCD |
| Auxiliary | Generic button, 2x potentiometer |

## Schematic
![schematic](./img/schematic.png)

## Code
``` 
// nema17 stepper
#include <LiquidCrystal.h>

const int dirPin = 6;
const int stepPin = 5;
const int buttonPin = 2;

const int rs = 7, en = 8, d4 = 9, d5 =10, d6 =11, d7 = 12;
LiquidCrystal lcd(rs, en, d4, d5, d6, d7);

int count=0;
float pos=0;
unsigned long lastUpdate = 0;
bool active = false;

  enum State {IDLE, TURNING, HOMING}; // motor state
  State state = IDLE;
  float startPos = 0;

void setup() {

  pinMode(dirPin, OUTPUT);
  pinMode(stepPin, OUTPUT);
  pinMode(buttonPin, INPUT_PULLUP); // switches logic to default high

  digitalWrite(dirPin, HIGH);
  digitalWrite(stepPin, LOW);

  lcd.begin(20, 2); // column, row
}


void loop() {

  
  int potTurns = analogRead(A0); 
  float turns = (potTurns/1023.0) * 10.0; // max 10 turns

  int potSpeed = analogRead(A1);
  int stepDelay = map(potSpeed, 0, 1023, 10, 50); // step delay from 10 to 50 ms


  if (digitalRead(2) == LOW && state == IDLE){ // forces to finish movement first
  active = true; // LOW and HIGH inverted, line 24
  state = TURNING;
  startPos = pos;
  }
  float target =0;
  if (state == TURNING) target = startPos + (turns +1) * 360;
  if (state == HOMING) target = startPos + turns * 360;
  if (state == IDLE) target = pos;



  //movement
  if (pos >= target + 1.7 || pos <= target - 1.7) { 
    if (target > pos) digitalWrite(dirPin, HIGH); else digitalWrite(dirPin, LOW);
    digitalWrite(stepPin, HIGH);
    delay(stepDelay);
    digitalWrite(stepPin, LOW);
    delay(stepDelay);

    if (digitalRead(dirPin) == HIGH) count+=1; else count-=1; // step counter
    pos = count*1.8; // position

  } else if ( state == TURNING ) state = HOMING;
    else if ( state == HOMING ) {state = IDLE; active = false; } // idle // active lcd indicator off once movement is finished

  

if (millis() - lastUpdate >= 100) { // 10/s lcd update
    lastUpdate = millis();

  lcd.setCursor(0,0); //row 1
  lcd.print("Pos>");   
  lcd.print(pos);  
  lcd.print("   T>");  
  lcd.print(turns); 
  lcd.print("   "); // position and selected turns


  lcd.setCursor(0,1); //row 2
  lcd.print("Vel>");   
  lcd.print(1.8 / ((stepDelay*2)/1000.0));  // "*2" because 2 delays
  lcd.print("deg/s ");  
  lcd.print(active ? "[ON] " : "[OFF]"); //velocity and 'home'-call active
}
}