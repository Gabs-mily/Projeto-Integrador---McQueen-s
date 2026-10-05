#include <BluetoothSerial.h>
#define REMOTE_BLUETOOTH_NAME "carrinho"; // nome do bluetooth
#include <RemoteXY.h>

#pragma pack(push, 1)
uint8_t RemoteXY_CONF[] = {
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
} // numeros do app
#pragma(pop) // interface do remoteXY

struck {
  int8_t joystick_01_x;
  int8_t joystick_01_y;
  uint8_t connect_flag;
} RemoteXY;

const int xx = xx;
const int xx = xx;
const int xx = xx;
const int xx = xx; // portas do esp

void setup() {
  RemoteXY_Init();
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
}

void loop() {
  RemoteXY_Handler();

  int8_t x = RemoteXY.joystick_01_x;
  int8_t y = RemoteXY.joystick_01_y;

  if ( y > 30){
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW );
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW ;
  } // frenteee
  else if(y < -30){
    digitalWrite(IN1, LOW );
    digitalWrite(IN2, HIGH);
    digitalWrite(IN3, LOW );
    digitalWrite(IN4, HIGH);
  } // trasss
  else if(x > 30){
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW );
    digitalWrite(IN3, LOW );
    digitalWrite(IN4, LOW );
  } // direitaaaa
  else if(x < -30){
    digitalWrite(IN1, LOW );
    digitalWrite(IN2, LOW );
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW );
  } // esquerdaaa
  else{
    digitalWrite(IN1, LOW );
    digitalWrite(IN2, LOW );
    digitalWrite(IN3, LOW );
    digitalWrite(IN4, LOW );
  } // paradooo
}
