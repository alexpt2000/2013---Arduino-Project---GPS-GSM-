# 2013---Arduino-Project---GPS-GSM-


# [Project - Report PDF](https://github.com/alexpt2000/2013---Arduino-Project---GPS-GSM-/blob/master/Project%20Report%20Projeto%20GPS%20GSM%20V2.pdf)

## The Code

```
/*
 GPS/GSM/SP/Display Arduino V4
 
 Alexander Souza 11/06/2014
 alexpt2000@gmail.com
 
 Configuracao da Ligacoes
 ********************** GPS
 RXD Pin 3
 TXD Pin 2
 GND Ground
 VCC +5
  ********************** GSM
 RXD Pin 5
 TXD Pin 5
  *********************** LCD
 SDA    A4
 SCL    A5
  *********************** SD
 MOSI   A11
 MISO   A12
 CLK    A13
 CS     A10
  *********************** Others
 Button  D6
 Buzzer  D7
 Temp    A0
  */

// Bibliotecas Obrigatorias
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <TinyGPS.h>
#include <SoftwareSerial.h>
#include <SPI.h>
#include <SD.h>

// Endereco LCD
LiquidCrystal_I2C lcd(0x27, 2, 1, 0, 4, 5, 6, 7, 3, POSITIVE);


// Variaveis
SoftwareSerial GPS(2, 3);
TinyGPS gps;
void gpsdump(TinyGPS &gps);
bool feedgps();
void getGPS();
float LAT, LON;
long lat, lon, year, month, day, hour, minute, second, velocity;
unsigned long fix_age, time, date, speed, course, chars;
unsigned short sentences, failed_checksum;

float temp;
int tempPin = 0;

//Configuracao  SD shields para pin 10
const int chipSelect = 10;


void setup()
{
  GPS.begin(9600);
  Serial.begin(115200);
  Serial.begin(9600);
  lcd.begin(20, 4);
  /*
while (!Serial) {
   ; // wait for serial port to connect. Needed for Leonardo only
   }
   
   */

  pinMode(10, OUTPUT);

  if (!SD.begin(chipSelect)) {
    lcd.setCursor(0, 0);
    lcd.print("initialization failed!");
    return;
  }
  lcd.setCursor(0, 0);
  lcd.print("initialization done.");
  delay(1000);

  // Informacoes de LCD
  lcd.setCursor(0, 0);
  lcd.print("Project GPS/GSM   V4");
  lcd.setCursor(0, 1);
  lcd.print("Alexander Souza");
  delay(1000);
  lcd.backlight();
}

void loop()
{
  // Adquirir Posicao, data e hora GPS
  gps.get_position(&lat, &lon, &fix_age);
  gps.get_datetime(&date, &time, &fix_age);
  gps.f_speed_kmph();

  year = date % 100;
  month = (date / 100) % 100;
  day = date / 10000;
  hour = time / 1000000;
  minute = (time / 10000) % 100;
  second = (time / 100) % 100;
  
  
  temp = analogRead(tempPin); // Adquirindo Temperatura
  temp = temp * 0.48828125;



  getGPS();

  // Impressao das coodenadas GPS na tela
  Serial.print("https://maps.google.com/maps?q=");
  Serial.print(LAT / 100000, 7);
  Serial.print("+");
  Serial.println(LON / 100000, 7);
  // Serial.println((int)(velocity * 1852 / 100000));


  // coodenadas GPS no LCD
  
  lcd.setCursor(0, 0);
  lcd.print("                    ");
  lcd.setCursor(0, 1);
  lcd.print("                    ");
  lcd.setCursor(0, 2);
  lcd.print("                    ");
  lcd.setCursor(0, 3);
  lcd.print("                    ");
  
  lcd.setCursor(0, 0);
  lcd.print("Speed");
  lcd.setCursor(6, 0);
  lcd.print(gps.f_speed_kmph()); // Velocidade Kmph
  lcd.setCursor(11, 0);
  lcd.print("Kmph");

  
  
  // lcd.setCursor(0, 1);
  // lcd.print("Recording...      ");  
  lcd.setCursor(0, 2);
  //lcd.print("Lat:");
  //lcd.setCursor(4, 2);
  lcd.print(LAT / 100000, 7);
  lcd.setCursor(0, 3);
  //lcd.print("Lon:");
  //lcd.setCursor(4, 3);
  lcd.print(LON / 100000, 7);

  lcd.setCursor(11, 2);
  lcd.print("TEMP  SAT");
  lcd.setCursor(17, 3);
  lcd.print(gps.satellites());  // Numero de satelites disponivel
  
  lcd.setCursor(11, 3);
  lcd.print(temp-0.5);


  if (day != 0.00) { // Condicao para gravar o GPSLOG somente quando adquirir sinal GPS, Verificao feita pela a variavel dia

    // Prosseco de Gravacao e Verificacao do arquivo GPSLOG.csv
    File myFile = SD.open("gpslog.csv", FILE_WRITE);
    // Alterar o estado do Arquivo para OPEN
    if (myFile) {
      feedgps(); // If we don't feed the gps during this long routine, we may drop characters and get checksum errors
      // Campos a serem gravados no arquivo gpslog.csv
      myFile.print(month);
      myFile.print("/");
      myFile.print(day);
      myFile.print("/");
      myFile.print(year);
      myFile.print(",");

      myFile.print(hour+1); 
      myFile.print(":");
      myFile.print(minute); 
      myFile.print(":");
      myFile.print(second);
      myFile.print(",");

      myFile.print(LAT / 100000, 7);
      myFile.print(",");
      myFile.print(LON / 100000, 7);
      myFile.print(",");
      myFile.print(gps.f_speed_kmph());
      myFile.print(",");
      myFile.print(temp);
      myFile.print(",");
      myFile.print("https://maps.google.com/maps?q=");
      myFile.print(LAT / 100000, 7);
      myFile.print("+");
      myFile.print(LON / 100000, 7);

      myFile.println("");
      myFile.close();
      lcd.setCursor(17, 0);
      lcd.print("Rec"); 
    } 
    else {
      //Serial.println("error opening gpslog.csv");
      lcd.setCursor(17, 0);
      lcd.print("Err"); 
    }
  }

  delay(5000); // Tempo de Recolha para os dados esta definido para 5 segundos
}


//****************** Function GPS *******************************************
void getGPS() {
  bool newdata = false;
  unsigned long start = millis();
  // Every 1 seconds we print an update
  while (millis() - start < 1000)
  {
    if (feedgps ()) {
      newdata = true;
    }
  }
  if (newdata)
  {
    gpsdump(gps);
  }
}


bool feedgps() {
  while (GPS.available())
  {
    if (gps.encode(GPS.read()))
      return true;
  }
  return 0;
}


void gpsdump(TinyGPS &gps)
{
  //byte month, day, hour, minute, second, hundredths;
  gps.get_position(&lat, &lon);
  LAT = lat;
  LON = lon;
  //gps.f_speed_kmph();
  velocity = gps.speed();

  {
    feedgps(); // If we don't feed the gps during this long routine, we may drop characters and get checksum errors
  }
}



/*
void setup() { 
 pinMode(7, OUTPUT); 
 pinMode(6, INPUT); 
}
void loop() {
 if(digitalRead(6)){
 tone(7, 1500, 1000); 
 delay(1000);
  }else{ 
 tone(7, 400, 1000);
  delay(1000);
 }
}
*/

```
