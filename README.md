# Proyecto-de-estacionamiento
Encontraras el código en Arduino para realizar un estacionamiento inteligente 

/* Librerías correspondientes a las funcionalidades */
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>
#include <HCSR04.h>
#include "IRremote.h"

/* Cremos los Objetos */
LiquidCrystal_I2C lcd(0x27, 16, 2);  //Pantalla LCD
Servo servoE;                        //2 servos
Servo servoS;
UltraSonicDistanceSensor sensorE(12, 11);  //2 sensores
UltraSonicDistanceSensor sensorS(13, 10);

/* Asignación de Pines */
int recividor = 3;  //IR receptor en pin 3

int LED1 = 2;  //LED 1 en pin 2
//int LED2=3;  //LED 2 en pin 3
int LED3 = 4;  //LED 3 en pin 4
int LED4 = 7;  //LED 4 en pin 7
int LED5 = 8;  //LED 5 en pin 8
int LED6 = 9;  //LED 6 en pin 9

IRrecv mando(recividor);  //Mando
decode_results resultados;

/*  Variables para regular el parking */
boolean p[6] = { true, true, true, true, true, true };  //Array de plazas ocupadas
int plazas;                                             //Contador de plazas para llevar la cuenta de cuantas hay libres
int ctr_sitioE;                                         //Controlador del sitio al que tiene que aparcar
int ctr_sitioS;                                         //Controlador del sitio que abandonan

//Variables usadas para regular los tiempos
unsigned long tini1;
unsigned long tini2;
unsigned long tini3;
unsigned long tini4;
unsigned long tini5;
unsigned long tini6;

unsigned long tfin1;
unsigned long tfin2;
unsigned long tfin3;
unsigned long tfin4;
unsigned long tfin5;
unsigned long tfin6;

unsigned long ttotal;

void setup() {
  Serial.begin(9600);
  mando.enableIRIn();  // Se enciende el recibidor del mando
  for (int i = 0; i <= 5; i++) {
    p[i] = true;  //Inicializamos el array de plazas, todas disponibles
  }
  plazas = 6;             //6 plazas para empezar disponibles
  pinMode(LED1, OUTPUT);  //Establecemos los leds como salida
  //pinMode(LED2,OUTPUT); 
  pinMode(LED3, OUTPUT);  
  pinMode(LED4, OUTPUT);  
  pinMode(LED5, OUTPUT);  
  pinMode(LED6, OUTPUT);  
  servoE.attach(5);       //Servo entrada en pin 5
  servoS.attach(6);       //Servo salida en pin 6

  servoE.write(95);   //Colocamos servo de entrada en posición para no dejar pasar
  servoS.write(100);  //Colocamos servo de salida en posición para no dejar pasar

  lcd.init();                     //iniciamos la LCD
  lcd.backlight();                //Le damos luz
  lcd.setCursor(0, 0);            //
  lcd.print("Plazas libres: ");   //En primera línea de LCD escribimos "Plazas libres: 6"
  lcd.print(plazas);              //
  lcd.setCursor(0, 1);            //
  lcd.print("   Bienvenidos  ");  // En segunda línea de LCD escribimos "Bienvenidos"

  encenderLeds();  //Todos los leds encendidos, ya que todas las plazas están libres al principio
}
void loop() {

  /* Seguimiento del mando */
  if (mando.decode(&resultados)) {
    mando1();
    mando.resume();
  }

  /* Leer los valores de los sensores */
  float distanceE = sensorE.measureDistanceCm();
  float distanceS = sensorS.measureDistanceCm();

/* Mostramos por la consola de serial los valores leídos por ambos sensores */
  Serial.print("Distancia entrada: ");
  Serial.print(distanceE);
  Serial.print("    /    ");
  Serial.print("Distancia salida: ");
  Serial.println(distanceS);
  delay(1000);

  /* Funcionalidad de entradas */
  if (plazas > 0 && distanceE < 7.00) {  //Si hay plazas disponibles y detecta paso de un coche

    plazas--;
    SubirEntrada();

    if (p[0] == true) {		//Comprabamos plaza a plaza su disponibilidad

      digitalWrite(LED1, LOW);
      p[0] = false;
      tini1 = millis(); //Obtenemos el momento en el que el coche aparca en la plaza

      ctr_sitioE = 1; 
    } else if (p[1] == true) {

      //digitalWrite(LED2, LOW);
      p[1] = false;
      tini2 = millis();
      ctr_sitioE = 2;

    } else if (p[2] == true) {

      digitalWrite(LED3, LOW);
      p[2] = false;
      tini3 = millis();

      ctr_sitioE = 3;

    } else if (p[3] == true) {

      digitalWrite(LED4, LOW);
      p[3] = false;
      tini4 = millis();

      ctr_sitioE = 4;

    } else if (p[4] == true) {

      digitalWrite(LED5, LOW);
      p[4] = false;

      tini5 = millis();
      ctr_sitioE = 5;

    } else if (p[5] == true) {

      digitalWrite(LED6, LOW);
      p[5] = false;
      tini6 = millis();

      ctr_sitioE = 6;
    }

/* Modificaciones de la pantalla lcd */

    lcd.setCursor(15, 0); //Muestra las plazas disponibles
    lcd.print(plazas);

    lcd.setCursor(0, 1);
    lcd.print("Aparque en la "); //Indica el sitio en el que debe aparcar
    lcd.print(ctr_sitioE);
    delay(5000);

    lcd.setCursor(0, 1);
    lcd.print("   Bienvenidos!  ");

    BajarEntrada(); //Baja la barrera de la entrada
  }

  /* Funcionalidad de salidas */
  if (plazas < 6 && distanceS < 7.00) {

    plazas++;
    SubirSalida();
    lcd.setCursor(15, 0);
    lcd.print(plazas);
    delay(5000);
    BajarSalida();
  }
}

/* Función de gestión del mando */
/* Aquí se implementan todas las funcionalidades que van a tener 
   cada uno de los botones */
void mando1() {

  switch (resultados.value) {  //En función del botón que pulsemos, realiza una acción u otra
    case 0xFFA25D:
      break;
    case 0xFFE21D:
      break;
    case 0xFF629D:
      break;
    case 0xFF22DD:  //Al pulsar el botón de retroceso, sube la barrera de entrada
      SubirEntrada();
      break;
    case 0xFF02FD:
      break;
    case 0xFFC23D:  //Al pulsar el botón de avanzar, sube la barrera de salida
      SubirSalida();
      break;
    case 0xFFE01F:  //Al pulsar el botón de bajar, baja la barrera de entrada
      BajarEntrada();
      break;
    case 0xFFA857:
      break;
    case 0xFF906F:  //Al pulsar el botón de subir, baja la barrera de salida
      BajarSalida();
      break;
    case 0xFF9867:  //Al pulsar el botón EQ, se reinicia el parking
      encenderLeds();
      servoE.write(95);
      servoS.write(100);
      plazas = 6;
      for (int i = 0; i <= 5; i++) {
        p[i] = true;
      }
      lcd.setCursor(0, 0);
      lcd.print("Plazas libres: ");
      lcd.print(plazas);
      lcd.setCursor(0, 1);
      lcd.print("   Bienvenidos  ");
      break;
    case 0xFFB04F:
      break;
    case 0xFF6897:
      break;
    case 0xFF30CF:  //Al pulsar el botón 1, decidimos que el coche abandona la posición 1
      if (p[0] == false) { 		//Comprobación de que la plaza esté ocupada

        digitalWrite(LED1, HIGH);
        p[0] = true;

        ctr_sitioS = 1;
        Serial.print("Ha quedado libre la plaza ");
        Serial.print(ctr_sitioS);
        Serial.print("\n");

        tfin1 = millis(); //Tiempo en el que se abandona el parking
        
        imprimirTiempoTranscurrido();
        ticketSerial();
      } else {
        Serial.print("La plaza no esta ocupada\n");
      }
      break;
    case 0xFF18E7:  //Al pulsar el botón 2, decidimos que el coche abandona la posición 2
      if (p[1] == false) {
        //digitalWrite(LED2, HIGH);
        p[1] = true;
        ctr_sitioS = 2;
        Serial.print("Ha quedado libre la plaza ");
        Serial.print(ctr_sitioS);
        Serial.print("\n");
        tfin2 = millis();
        //ttotal= tfin2 - tini2;
        imprimirTiempoTranscurrido();
        ticketSerial();
      } else {
        Serial.print("La plaza no esta ocupada\n");
      }
      break;
    case 0xFF7A85:  //Al pulsar el botón 3,  decidimos que el coche abandona la posición 3
      if (p[2] == false) {
        digitalWrite(LED3, HIGH);
        p[2] = true;
        ctr_sitioS = 3;
        Serial.print("Ha quedado libre la plaza ");
        Serial.print(ctr_sitioS);
        Serial.print("\n");

        tfin3 = millis();
        
        imprimirTiempoTranscurrido();
        ticketSerial();
      } else {
        Serial.print("La plaza no esta ocupada\n");
      }
      break;
    case 0xFF10EF:  //Al pulsar el botón 4,  decidimos que el coche abandona la posición 4
      if (p[3] == false) {
        digitalWrite(LED4, HIGH);
        p[3] = true;
        ctr_sitioS = 4;
        
	Serial.print("Ha quedado libre la plaza ");
        Serial.print(ctr_sitioS);
        Serial.print("\n");
        
	tfin4 = millis();
    
        imprimirTiempoTranscurrido();
        ticketSerial();
      } else {
        Serial.print("La plaza no esta ocupada\n");
      }
      break;
    case 0xFF38C7:  //Al pulsar el botón 5,  decidimos que el coche abandona la posición 5
      if (p[4] == false) {
        digitalWrite(LED5, HIGH);
        p[4] = true;
        ctr_sitioS = 5;

        Serial.print("Ha quedado libre la plaza ");
        Serial.print(ctr_sitioS);
        Serial.print("\n");

        tfin5 = millis();
        
        imprimirTiempoTranscurrido();
        ticketSerial();
      } else {
        Serial.print("La plaza no esta ocupada\n");
      }
      break;
    case 0xFF5AA5:  //Al pulsar el botón 6, decidimos que el coche abandona la posición 6
      if(p[5]==false){
      digitalWrite(LED6, HIGH);
      p[5] = true;
      ctr_sitioS = 6;
      
      Serial.print("Ha quedado libre la plaza ");
      Serial.print(ctr_sitioS);
      Serial.print("\n");
      
	tfin6 = millis();
      
      imprimirTiempoTranscurrido();
      ticketSerial();
  }
  else {
    Serial.print("La plaza no esta ocupada\n");
  }
  break;
  case 0xFF42BD:
    break;
  case 0xFF4AB5:
    break;
  case 0xFF52AD:
    break;
  case 0xFFFFFFFF:
    break;
}
delay(500);
}

/* Función para encender todos los leds */
void encenderLeds() {
  digitalWrite(LED1, HIGH);
  //digitalWrite(LED2, HIGH);
  digitalWrite(LED3, HIGH);
  digitalWrite(LED4, HIGH);
  digitalWrite(LED5, HIGH);
  digitalWrite(LED6, HIGH);
}

/* Función de subir la entrada */
/* Esta función sube la barrera de la entrada despacio 
   subiendola un grado cada 10 ms */
void SubirEntrada() {
  for (int pos = 95; pos <= 180; pos += 1) {
    servoE.write(pos);
    delay(10);
  }
}
/* Función de bajar la entrada */
/* Esta función baja la barrera de la entrada despacio 
   bajandola un grado cada 10 ms */
void BajarEntrada() {
  for (int pos = 180; pos >= 95; pos -= 1) {
    servoE.write(pos);
    delay(10);
  }
}
/* Función de subir la salida */
/* Esta función sube la barrera de la salida despacio 
   subiéndola un grado cada 10 ms */
void SubirSalida() {
  for (int pos = 100; pos <= 180; pos += 1) {
    servoS.write(pos);
    delay(10);
  }
}
/* Función de bajar la salida */
/* Esta función baja la barrera de la salida despacio 
   bajandola un grado cada 10 ms */
void BajarSalida() {
  for (int pos = 180; pos >= 100; pos -= 1) {
    servoS.write(pos);
    delay(10);
  }
}

/* Función de mostrar tiempo */
/* Esta función se encarga de calcular el tiempo final
   que se usa una plaza concreta y a su vez imprime en la
   consola de Serial su valor expresado en segundos */
void imprimirTiempoTranscurrido() {
 
  switch (ctr_sitioS) {
    case 1:
      ttotal = tfin1 - tini1;
      break;
    case 2:
      ttotal = tfin2 - tini2;
      break;
    case 3:
      ttotal = tfin3 - tini3;
      break;
    case 4:
      ttotal = tfin4 - tini4;
      break;
    case 5:
      ttotal = tfin5 - tini5;
      break;
    case 6:
      ttotal = tfin6 - tini6;
      break;
  }
  Serial.print("La plaza ");
  Serial.print(ctr_sitioS);
  Serial.print(" ha sido ocupada durante ");
  Serial.print(ttotal / 1000);
}

/* Función que genera un ticket mostrado por consola */
/* Esta función genera un ticket con los datos de la plaza, 
   el tiempo, y el precio que le costaría al conductor. 
   Todo esto se muestra en la consola de Serial */
void ticketSerial() {
  unsigned long precioSEG = ttotal * 0.5;

  Serial.print("\n ******************* \n");
  Serial.print("    TICKET    \n\n Plaza ocupada: ");
  Serial.print(ctr_sitioS);
  Serial.print("\n Tiempo ocupada: ");
  Serial.print(ttotal / 1000);
  Serial.print("\n ---------------- \n Total a pagar: ");
  Serial.print(precioSEG / 1000);
  Serial.print("\n  ******************* \n");
}
