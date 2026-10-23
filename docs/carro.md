# Proyecto del carro
Martin Urbina Revuelta

Introduccion a la mecatronica

fecha: 16/10/26

Se usa codigo, para el funcionamiento del carro y corte laser, un sp32, motores, cableado e impresión 3D para hacer el chasis y resto del carro

# Objetivos
Aplicar lo aplicado en la carrera de una manera practica a traves del proyecto

# Alcance
Sirve de muchas maneras, por la parte del codigo y la preparacion del chasis

# Requisitos
motores, sp32, , llantas, material para el cuerpo del coche, un control para el mismo

# Realizacion
Se corto un pedazo de madera de 13x14cm con muescas de 2 cm de profundidad y 7 de largo en el centro de los 2 lados de 13, se colocan los motores flotando en los espacios de las muescas atornillados a una pieza impresa con forma de L.

se pone una rueda loca, el circuito con el sp32 para el codigo, que permite al carro moverse en línea recta así como en diagonal y cambiar de velocidad.

Luego se coloco las paredes y el techo que es una gran pieza también de impresión 3D para proteger el carro, esta a su ves tiene la garra para poder tener un buen control de la pelota.

El código usado para el robot es el siguiente: 

#include "BluetoothSerial.h"
BluetoothSerial BT;

// Pines ESP32 -> TB6612
const int AIN1 = 26, AIN2 = 27, PWMA = 25;  // Motor izquierdo
const int BIN1 = 32, BIN2 = 33, PWMB = 14;  // Motor derecho
const int STBY = 13;

int velocidad = 170;  
char movimiento = 'S';     // ultimo movimiento activo

void pwm(int pin, int valor) {
#if ESP_ARDUINO_VERSION_MAJOR >= 3
  ledcWrite(pin, valor);
#else
  ledcWrite(pin == PWMA ? 0 : 1, valor);
#endif
}

// Curvitas
void motores(int a1, int a2, int b1, int b2, int velA, int velB) {
  digitalWrite(AIN1, a1);
  digitalWrite(AIN2, a2);
  digitalWrite(BIN1, b1);
  digitalWrite(BIN2, b2);
  pwm(PWMA, velA);
  pwm(PWMB, velB);
}
// Dirrecciones
void adelante()  { motores(HIGH, LOW, HIGH, LOW, velocidad, velocidad); }
void atras()     { motores(LOW, HIGH, LOW, HIGH, velocidad, velocidad); }
void derecha()   { motores(HIGH, LOW, LOW, HIGH, velocidad, velocidad); }
void izquierda() { motores(LOW, HIGH, HIGH, LOW, velocidad, velocidad); }
void alto()      { motores(LOW, LOW, LOW, LOW, 0, 0); }

// Curvas
void adelanteIzq() { motores(HIGH, LOW, HIGH, LOW, velocidad / 2, velocidad); }
void adelanteDer() { motores(HIGH, LOW, HIGH, LOW, velocidad, velocidad / 2); }
void atrasIzq()    { motores(LOW, HIGH, LOW, HIGH, velocidad / 2, velocidad); }
void atrasDer()    { motores(LOW, HIGH, LOW, HIGH, velocidad, velocidad / 2); }

void ejecutar(char m) {
  movimiento = m;
  switch (m) {
    case 'F': adelante();     break;
    case 'B': atras();        break;
    case 'L': izquierda();    break;
    case 'R': derecha();      break;
    case 'G': adelanteIzq();  break;   
    case 'H': adelanteDer();  break;   
    case 'I': atrasDer();     break;   
    case 'J': atrasIzq();     break;   
    default:  alto(); movimiento = 'S'; break;
  }
}

void setup() {
  Serial.begin(115200);                    // Monitor Serie (115200 baud)
  BT.begin("CarroRC_ATOM");

  int pines[] = {AIN1, AIN2, BIN1, BIN2, STBY};
  for (int p : pines) pinMode(p, OUTPUT);
  digitalWrite(STBY, HIGH);

#if ESP_ARDUINO_VERSION_MAJOR >= 3
  ledcAttach(PWMA, 1000, 8);
  ledcAttach(PWMB, 1000, 8);
#else
  ledcSetup(0, 1000, 8); ledcAttachPin(PWMA, 0);
  ledcSetup(1, 1000, 8); ledcAttachPin(PWMB, 1);
#endif
  alto();
}

void loop() {
  if (BT.available()) {
    char c = BT.read();

    if (c != '\r' && c != '\n') {          // ignora saltos de linea
      Serial.print("Recibido: ");          // para pruebas
      Serial.println(c);
    }

    if (c >= '0' && c <= '9') {            
      velocidad = map(c - '0', 0, 9, 90, 255);
      ejecutar(movimiento);                
    }
    else if (c == 'q') {                   
      velocidad = 255;
      ejecutar(movimiento);
    }
    else if (c != '\0' && strchr("FBLRGHIJS", c)) {   // movimientos y alto
      ejecutar(c);
    }
  }

  // Failsafe: si se pierde la conexion, alto
  if (!BT.hasClient()) alto();
}