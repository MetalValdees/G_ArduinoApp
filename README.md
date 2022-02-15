#include <Stepper.h>

//#define Y_z 0 // Устранение люфта по оси Y
#define Vector_delay 1000 // Задержка в векторном режиме  

Stepper XStepper = Stepper(300, 8, 9, 10, 11);
Stepper YStepper = Stepper(300, 2, 3, 4, 5);

int X = 0;
int Y = 0;
bool Z = true;
boolean V = false;
boolean F = false;

int Data[7];
boolean recievedFlag;
boolean getStarted;
byte index;
String string_convert = "";

//------------setup()-----------------
void setup()
{
  Serial.begin(9600);
  pinMode(7, OUTPUT);
  XStepper.setSpeed(50);
  YStepper.setSpeed(50);
}

//------------parsing()---------------
void parsing() {
  if (Serial.available() > 0) {
    char incomingByte = Serial.read();
    if (getStarted) {
      if (incomingByte != ' ' && incomingByte != ';') {
        string_convert += incomingByte;
      } else {
        Data[index] = string_convert.toInt();
        string_convert = "";
        index++;
      }
    }
    if (incomingByte == '#') {
      getStarted = true;
      index = 0;
      string_convert = "";
    }
    if (incomingByte == ';') {
      getStarted = false;
      recievedFlag = true;
    }
    if (incomingByte == 'T') { // ТЕСТ ЧПУ
      YStepper.step(-50);
      delay(500);
      YStepper.step(50);
      delay(500);
      XStepper.step(50);
      delay(500);
      XStepper.step(-50);
      digitalWrite(7, HIGH);
      delay(500);
      digitalWrite(7, LOW);
      Serial.print('#');
    }
    if (incomingByte == 'V') { // ПЕРЕВОД ЧПУ В ВЕКТОРНЫЙ РЕЖИМ
      V = true;
      Serial.print('#');
    }
    if (incomingByte == 'R') { // ПЕРЕВОД ЧПУ В РАСТОВЫЙ РЕЖИМ
      V = false;
      Serial.print('#');
    }
    if (incomingByte == 'F') { // ОТКЛУЧЕНИЕ ЛАЗЕРА
      XStepper.setSpeed(20);
      digitalWrite(7, LOW);
      F = true;
      Serial.print('#');
    }
    if (incomingByte == '.') { // ЗАВЕРШЕНИЕ РАБОТЫ
      restart();
    }
  }
}

//------------restart()---------------
void restart()
{
  XStepper.setSpeed(50);
  YStepper.setSpeed(50);
  YStepper.step(-Y);
  XStepper.step(-X);
  digitalWrite(7, LOW);
  X = 0; Y = 0; F = false;
}

//------------implementation()--------
void implementation()
{
  if (V == false)
  {
    XStepper.setSpeed(20);
    YStepper.step(Data[4] - Y); // Подходит по Y/i
    XStepper.step(Data[0] - X); // Подходит по X/G/j
    XStepper.setSpeed(Data[6] + 1); // Устанавливает скорость прохождения
    digitalWrite(7, HIGH);
    XStepper.step(Data[2] - Data[0]); // Проходит по вектору
    digitalWrite(7, LOW);
    Y = Data[4]; // Новые координаты ЛАЗЕРА по Y
    X = Data[2]; // Новые координаты ЛАЗЕРА по X
  }
  else
  {
    /*if (Data[2] - Y > 0 && Z == false)
    {
      YStepper.setSpeed(20);
      YStepper.step(-5);
      YStepper.step(5 + Y_z);
      YStepper.setSpeed(1);
      Z = true;
    }
    if (Data[2] - Y < 0 && Z == true)
    {
      YStepper.setSpeed(20);
      YStepper.step(10);
      YStepper.step(-10 - Y_z);
      YStepper.setSpeed(1);
      Z = false;
    }*/
    if (F == true)
    {
      F = false;
      YStepper.step(Data[2] - Y); // Подходит по Y/i
      XStepper.step(Data[0] - X); // Подходит по X/G/j
      digitalWrite(7, HIGH);
      delay(Vector_delay);
      Y = Data[2]; // Новые координаты ЛАЗЕРА по Y
      X = Data[0]; // Новые координаты ЛАЗЕРА по X
    }
    else
    {
      digitalWrite(7, HIGH);
      XStepper.setSpeed(1);
      YStepper.setSpeed(1);
      YStepper.step(Data[2] - Y); // Подходит по Y/i
      XStepper.step(Data[0] - X); // Подходит по X/G/j
      delay(Vector_delay);
      Y = Data[2]; // Новые координаты ЛАЗЕРА по Y
      X = Data[0]; // Новые координаты ЛАЗЕРА по X
    }
  }
}

//------------loop()------------------
void loop()
{
  parsing();
  if (recievedFlag) {
    recievedFlag = false;
    Data[0] = Data[0] * 100 + Data[1]; // X/G/j
    Data[2] = Data[2] * 100 + Data[3]; // X/E/j в случае вектора Y/i
    if (V == false)
    {
      Data[4] = Data[4] * 100 + Data[5]; // Y/i
      for (int RGB = 28, i = 0; RGB < 281; RGB += 28, i++)
      {
        if (Data[6] <= RGB && Data[6] >= RGB - 28) Data[6] = i;
      }
    }
    implementation(); // Выполнение прожига
    Serial.print('#');
  }
}
