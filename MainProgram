
#include <TinyGPSPlus.h>
//#include <AltSoftSerial.h>

//KOMUNIKACIJA
#define rxPin 17
#define txPin 16
HardwareSerial sim800(1);

#define RXD2 32
#define TXD2 27
HardwareSerial neogps(2);

String pravilnoGeslo = "voda";

TinyGPSPlus gps;

//VKLOP ali IZKLOP naprave
bool stanjeVklopa = 1;

//PREVERJANJE HITROSTI
uint32_t prejsnoPreverjanjeHitorsti = 0;
const uint32_t intervalPreverjanjaHitrosti = 30000;

//GSM-spremenljivke
//static const uint32_t SIMBaud = 9600;
String prejetoSporocilo = "";
String klicateljID = "";
String nastavljenKlicatelj = "+38640886754";

byte resetSIMPin = 04;

String pomocSMS = "POMOC - Izpiše vse možne ukaze, ki jih lahko uporabiš.\nTEST - Preveri, ali naprava pravilno deluje.\nZVOK - Pokliče napravo in omogoča poslušanje okolice.\nGPS - Pošlje trenutne GPS koordinate za pridobitev lokacije.\nSLEDENJE - Pošilja GPS koordinate vsakih 30 sekund, prekinitev z ukazom STOP.\nSTOP - Prekine sledenje in pošiljanje lokacije.\nBATERIJA - Izpiše napetost baterije, ob prazni bateriji prejmeš obvestilo.\nVKLOP%VAŠE_GESLO - Vklopi napravo za varovanje s tvojim geslom.\nIZKLOP%VAŠE_GESLO - Izklopi napravo za varovanje s tvojim geslom.";


boolean pridobiOdgovor(String pricakovani_odgovor, unsigned int cas_izteka = 1000, boolean ponastavi = false) {
  boolean zastava = false;
  String odgovor = "";
  unsigned long prejsnji;
  for (prejsnji = millis(); (millis() - prejsnji) < cas_izteka;) {
    while (sim800.available()) {
      odgovor = sim800.readString();
      //----------------------------------------
      // Uporabljeno v funkciji resetSIM800L
      // Če obstajajo nekateri odgovori
      // if (odgovor != "") {
      //   Serial.println(odgovor);
      //   if (ponastavi == true)
      //     return true;
      // }
      //----------------------------------------
      if (odgovor.indexOf(pricakovani_odgovor) > -1) {
        return true;
      }
    }
  }
  return false;
}

boolean preveriDelovanjeModula() {
  sim800.println("AT\r\n");
  return pridobiOdgovor("OK", 1000);
}

void setup() {

  Serial.begin(115200);

  delay(1000);
  //Nastavljanje resetSIM - Pina
  pinMode(resetSIMPin, OUTPUT);
  delay(100);
  digitalWrite(resetSIMPin, 0);

  //VZPOSTAVLJANJE KOMUNIKACIJE
  sim800.begin(9600, SERIAL_8N1, rxPin, txPin);
  Serial.println("Zagon SIM800 komunikacije");
  delay(1000);
  neogps.begin(9600, SERIAL_8N1, RXD2, TXD2);
  delay(1000);
  Serial.println("Zagon NEO7M komunikacije");

  Serial.println("zagon");
  sim800.println("AT");  // Preveri, ali je modul SIM800 pripravljen za sprejem ukazov. Pričakuje se odgovor "OK".
  pridobiOdgovor("OK", 1000);
  sim800.println("AT+CMGF=1");  // Nastavi način SMS sporočil na besedilni način. Pričakuje se odgovor "OK".
  pridobiOdgovor("OK", 1000);
  sim800.println("AT+CNMI=1,2,0,0,0");  // Nastavi parametre za prejemanje novih SMS sporočil. Pričakuje se odgovor "OK".
  pridobiOdgovor("OK", 1000);
  sim800.println("AT+CMGD=1,4");  // Izbriše vsa prejeta SMS sporočila. Pričakuje se odwgovor "OK".
  delay(1000);
  sim800.println("AT+CMGDA=\"DEL ALL\"");  // Izbriše vsa shranjena SMS sporočila. Pričakuje se odgovor "OK".
  delay(1000);

  preveriDelovanjeModula() ? Serial.println("Pravilo deluje") : Serial.println("Ne deluje");

  Serial.println("Zagon Opralvjen \n .................................................");
}

void loop() {
  //preveriDelovanjeModula() ? Serial.println("Pravilo deluje") : Serial.println("Ne deluje");
  if(Serial.available()){
    Serial.print(Serial.read());
    digitalWrite(resetSIMPin, 1);
    delay(200);
    digitalWrite(resetSIMPin, 0);
  }
  //KONSTANTNO PREVERJANJA DELOVANJA
  if (millis() - prejsnoPreverjanjeHitorsti >= intervalPreverjanjaHitrosti && stanjeVklopa == 1) {
    while (neogps.available()) {
      if (gps.encode(neogps.read())) {
        Serial.println(gps.speed.kmph());
        if (gps.speed.kmph() >= 2.0) {
          //posljiSMS("Čolen je v premikanju", nastavljenKlicatelj);
          klicVSili();
          break;
        }
      }
      else{
        continue;
      }
    }
    if (!neogps.available()){
      Serial.println("Ni Podatka o Hitrosti");
    }

    if (!preveriDelovanjeModula()) {
      digitalWrite(resetSIMPin, 1);
      delay(200);
      digitalWrite(resetSIMPin, 0);
    }
    prejsnoPreverjanjeHitorsti = millis();
  }

  //SPREJEMANJE SMS ukazov
  while (sim800.available()) {
    prejetoSporocilo = sim800.readString();
    Serial.println("Dobil sem SMS" + prejetoSporocilo);

    if (prejetoSporocilo.indexOf("+CMT:") > -1) {
      klicateljID = pridobiKlicateljID(prejetoSporocilo);
      String ukaz = pridobiVsebinoSporocila(prejetoSporocilo);

      Serial.println("Prejet ukaz: " + ukaz + "\n ............................................");


      //--PREVERJANJE UKAZA--:
      if (ukaz == "pomoc") {
        posljiSMS("PomocSMS - poslan", klicateljID);
      }
      if (ukaz == "test") {
        if (preveriDelovanjeModula()) {
          while (neogps.available()) {
            if (gps.encode(neogps.read())) {
              Serial.print("Pravilno deluje, --POŠILJAM SMS");
              //posljiSMS("Test je bil opravljen,\nnaprava deluje pravilno", klicateljID);
            }
            break;
          }
          posljiSMS("Ni Lokacije", klicateljID);
        } else {
          Serial.println("Sistem ne deluje");
        }
      }

      // case "zvok":
      //   break;
      else if (ukaz == "gps") {
        String kordinate = pridobiLokacijo();
        Serial.println(kordinate);
        posljiSMS(kordinate, klicateljID);
      } else if (ukaz == "sledenje") {
        unsigned long millisSledenja = millis();
        unsigned long intervalPosiljanja = 30000;
        byte steviloIntervalov = 12;
        byte i;
        while (ukaz != "stop" || i <= steviloIntervalov) {  //30s

          while (sim800.available()) {
            prejetoSporocilo = sim800.readString();
            if (prejetoSporocilo.indexOf("+CMT:") > -1) {
              klicateljID = pridobiKlicateljID(prejetoSporocilo);
              ukaz = pridobiVsebinoSporocila(prejetoSporocilo);
            }
          }

          while (millis() - millisSledenja > intervalPosiljanja) {
            String kordinate = pridobiLokacijo();
            Serial.println(kordinate);
            millisSledenja = millis();
            //posljiSMS(kordinate, klicateljID);
            i++;
          }
        }
      }
      // case "stop":
      //   break;
      // case "baterija":
      //   break;

      else if (ukaz == "izklop") {
        stanjeVklopa == 0;
        Serial.println("Sistem je izklopljen");
        posljiSMS("Sistem je izklopljen, za vklop napisite: VKLOP%geslo", klicateljID);
      }

      else if (ukaz == "vklop") {
        stanjeVklopa == 1;
        Serial.println("Sistem je vklopljen");
        posljiSMS("Sistem je vklopljen, za izklop napisite: IZKLOP%geslo", klicateljID);
      }

      // }
    }
  }
}



void klicVSili() {
  sim800.print(F("ATD"));
  sim800.print(nastavljenKlicatelj);
  sim800.print(F(";\r\n"));
  Serial.println("Klicem");

  if (sim800.find("CONNECT") || sim800.find("BUSY") || sim800.find("NO CARRIER"))
    ;
  posljiSMS("Vaše plovilo je v nevarnosti!", nastavljenKlicatelj);
}
String pridobiKlicateljID(String buff) {
  unsigned int indeks = buff.indexOf("\"");
  unsigned int indeks2 = buff.indexOf("\"", indeks + 1);
  String klicateljID = buff.substring(indeks + 1, indeks2);
  klicateljID.trim();
  Serial.println("Klicatelj ID: " + klicateljID);
  return klicateljID;
}

String pridobiVsebinoSporocila(String buff) {
  unsigned int indeksUkaza = buff.lastIndexOf("\"");
  unsigned int indeksGesla = buff.lastIndexOf("%");

  if (indeksUkaza != -1) {
    String vnosUkaza = buff.substring(indeksUkaza + 1, indeksGesla);
    vnosUkaza.toLowerCase();
    vnosUkaza.trim();
    Serial.println("Uporabnikov Ukaz: " + vnosUkaza);

    if (vnosUkaza.equals("vklop") || vnosUkaza.equals("izklop")) {
      
      String vnosGesla = buff.substring(indeksGesla + 1);
      vnosGesla.trim();
      pravilnoGeslo.trim();
      if (indeksGesla != -1) {
      Serial.println("Uporanikov vnos gesla: " + vnosGesla);
      }
      if (vnosGesla.equals(pravilnoGeslo)) {
        Serial.println("Geslo je pravilno");
        return vnosUkaza;
      } else if (indeksUkaza != -1 && indeksGesla == -1) {
        return "ni-gesla";
        Serial.println("Niste vpisali gesla");
      } else {
        Serial.println("Napačno geslo");
        return "napačno-geslo";
      }
    }
    // Če "%" ni najden vrnemo ukaz uporabnika
    return vnosUkaza;
  }
}

void posljiSMS(String besedilo, String klicateljID) {
  sim800.print("AT+CMGF=1\r");
  delay(1000);
  sim800.print("AT+CMGS=\"" + klicateljID + "\"\r");
  delay(1000);
  sim800.print(besedilo);
  delay(100);
  sim800.write(0x1A);  // ASCII koda za ctrl-26
  delay(1000);
  Serial.println("SMS poslano uspešno.");
}

//GPS MODUL (NEO-7M)

String pridobiLokacijo() {
  bool flag = 0;
  String lokacija;
  Serial.print(F("Lokacija: "));  // F - shrani v flash memory saj je konstanta
  // Can take up to 60 seconds
  bool PodatkiLokacije = false;
  for (unsigned long start = millis(); millis() - start < 2000;) {
    while (neogps.available()) {
      if (gps.encode(neogps.read())) {
        float sirina = gps.location.lat();
        float dolzina = gps.location.lng();

        Serial.print(sirina, 6);
        Serial.print(F(","));
        Serial.print(dolzina, 6);
        lokacija = "http://maps.google.com/maps?q=loc:" + String(sirina, 6) + "," + String(dolzina, 6);
        return lokacija;
      }
    }
  }
  return "ni lokacije";
}
