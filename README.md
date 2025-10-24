# gazlaarm

#include <DHT.h> // Isı nem
#include <DHT_U.h> //Isı nem 2

//#include <LiquidCrystal_I2C.h> // LCD Ekran
//LiquidCrystal_I2C lcd(0x3F, 16, 2); // LCD Ekran, I2C adresi 0x3F (diğer adresler için 0x27 de olabilir)
#include <SoftwareSerial.h>


#define DHTPIN 8
DHT dht(DHTPIN, DHT11);



#define LEDCARBON 2
#define LEDNATURAL 7
#define BUTTON 4
#define CARBON A0
#define NATURALGAS A4
#define BUZZER 9

SoftwareSerial esp(10, 11); // RX, TX

// WiFi Ayarları
const char* ssid = "HUAWEI P20 lite";
const char* password = "hepsikucuk";
const char* host = "maker.ifttt.com";
const int httpPort = 80;

// Değişkenler
unsigned long lastDebounceTime = 0;
const unsigned long debounceDelay = 50; // Debounce süresi (ms)

bool mailgonderildimi = false;

int carbonmax = 400;
int naturalgasmax = 220;

bool systemState = false;  // Sistem açık/kapalı durumu
bool buttonPressed = false;
unsigned long buttonPressTime = 0;




void setup() {
  Serial.begin(9600);

  pinMode(LEDCARBON, OUTPUT);
  pinMode(LEDNATURAL, OUTPUT);
  pinMode(BUTTON, INPUT_PULLUP);
  pinMode(BUZZER, OUTPUT);
  pinMode(CARBON, INPUT);
  pinMode(NATURALGAS, INPUT);


  esp.begin(115200);

  // Buton Girişi

  //ISI NEM BAŞLANGIÇ
  dht.begin(); //Iıs nem başlangıç



  initializeESP(); //BURAYI AÇ ------------------------------------------------------------------------------------------------------------




  Serial.println("Gaz Alarm Sistemi Başlatıldı");
}

void loop() {
  isinem();







  // Buton durum kontrolü
  handleButton();

  // Sistem açıksa gaz seviyelerini kontrol et
  if (systemState) {
    checkGasLevels();
  }
}

void handleButton() {
  // Buton basıldığında
  if (digitalRead(BUTTON) == LOW && !buttonPressed) {
    buttonPressed = true;
    buttonPressTime = millis();
  }

  // Buton bırakıldığında
  if (digitalRead(BUTTON) == HIGH) {
    buttonPressed = false;


  }

  // 3 saniye basılı tutma kontrolü
  if (buttonPressed && (millis() - buttonPressTime >= 3000)) {
    systemState = !systemState; // Sistem durumunu tersine çevir
    Serial.print("BASILDI2");




    // LED'leri güncelle
    digitalWrite(LEDCARBON, systemState ? HIGH : LOW);
    digitalWrite(LEDNATURAL, systemState ? HIGH : LOW);
    delay(500);
    digitalWrite(LEDCARBON, systemState ? HIGH : LOW);
    digitalWrite(LEDNATURAL, systemState ? HIGH : LOW);

    Serial.print("Sistem ");

  }
}

void checkGasLevels() {
  static unsigned long lastReadTime = 0;
  bool naturalGasAlarm = false;    // Değişkenler EN ÜSTTE tanımlandı
  bool carbonAlarm = false;        // Değişkenler EN ÜSTTE tanımlandı

  if (millis() - lastReadTime >= 500) {
    int carbonValue = analogRead(CARBON);
    int naturalGasValue = analogRead(NATURALGAS);

    Serial.print("Karbon: ");
    Serial.print(carbonValue);
    Serial.print(" | Doğalgaz: ");
    Serial.println(naturalGasValue);

    // Carbon alarm kontrolü
    if (carbonValue >= carbonmax) {
      carbonAlarm = true;
    } else {
      carbonAlarm = false;
    }

    // Doğalgaz alarm kontrolü (HATA DÜZELTİLDİ)
    if (naturalGasValue >= naturalgasmax) {
      naturalGasAlarm = true;
    } else {                      // Süslü parantez eklendi
      naturalGasAlarm = false;
    }

    // Alarm kontrolü (Bu blok, sensör okuma IF'inin İÇİNDE olmalı)
    if (carbonAlarm || naturalGasAlarm) {
      tone(BUZZER, 800);
      if (carbonAlarm) {
        digitalWrite(LEDCARBON, 1);
        //Mail Gönderme
        lastDebounceTime = millis();
        Serial.println("İstek gönderiliyor...");
        sendEmailRequest();
      }
      else {
        digitalWrite(LEDCARBON, 0);
      }
      if (naturalGasAlarm) {
        digitalWrite(LEDNATURAL, 1);
        //Mail Gönderme

        if (mailgonderildimi == false) {
          lastDebounceTime = millis();
          Serial.println("İstek gönderiliyor...");
          sendEmailRequest();
          mailgonderildimi = true;
        }

      }
      else {
        digitalWrite(LEDNATURAL, 0);
      }
    } else {
      noTone(BUZZER);
      digitalWrite(LEDNATURAL, LOW);
      digitalWrite(LEDCARBON, LOW);
    }

    lastReadTime = millis();
  } // IF bloğu burada kapatıldı
} // Fonksiyon burada kapatıldı





//---------------------------------------------------------------------------------------------------------------------------------------
void initializeESP() {
  sendCommand("AT", "OK", 2000);
  sendCommand("AT+CWMODE=1", "OK", 2000);

  Serial.println("Ağa bağlanıyor...");
  String connectCmd = "AT+CWJAP=\"" + String(ssid) + "\",\"" + String(password) + "\"";
  if (sendCommand(connectCmd, "OK", 10000)) {
    Serial.println("Ağa bağlandı");
    tone(BUZZER,800);
    delay(200);
    noTone(BUZZER);
    
  } else {
    Serial.println("Ağa bağlanamadı!");
    for(int i; i<4; i++){tone(BUZZER,800);
    delay(400);
    noTone(BUZZER);
    delay(400);}
  }
}

void sendEmailRequest() {
  // TCP bağlantısı kur
  String tcpCommand = "AT+CIPSTART=\"TCP\",\"" + String(host) + "\",80";
  if (!sendCommand(tcpCommand, "OK", 5000)) {
    Serial.println("IFTTT sunucusuna bağlanılamadı");
    return;
  }

  // GET isteğini hazırla
  String url = "/trigger/sendmail/with/key/mBdp6CjBwBnYhG3-fQftWACC3DqZDxmdhqQe_EAxiu9";
  String request = "GET " + url + " HTTP/1.1\r\n"
                   "Host: " + String(host) + "\r\n"
                   "Connection: close\r\n\r\n";

  // Veri gönder
  String sendCmd = "AT+CIPSEND=" + String(request.length());
  if (!sendCommand(sendCmd, ">", 3000)) {
    Serial.println("Gönderim başlatılamadı");
    return;
  }

  esp.print(request);
  Serial.println("İstek gönderildi");

  // Yanıtı oku
  if (waitForResponse("200 OK", 5000)) {
    Serial.println("E-posta gönderimi başarılı!");
  } else {
    Serial.println("Sunucu yanıtı alınamadı");
  }

  // Bağlantıyı kapat
  sendCommand("AT+CIPCLOSE", "CLOSED", 2000);
  Serial.println("Bağlantı kapatıldı");
}

// Yardımcı Fonksiyonlar
bool waitForResponse(const String & target, int timeout) {
  unsigned long start = millis();
  String response = "";

  while (millis() - start < timeout) {
    while (esp.available()) {
      char c = esp.read();
      response += c;
      if (response.indexOf(target) != -1) {
        Serial.println("[ESP] " + response);
        return true;
      }
    }
  }
  Serial.println("[Timeout] " + response);
  return false;
}

bool sendCommand(const String & cmd, const String & expected, int timeout) {
  Serial.println("[Gönderilen] " + cmd);
  esp.println(cmd);
  return waitForResponse(expected, timeout);
}



unsigned long isinemtime = 0;
void isinem () {
  if (millis() - isinemtime >= 2000) {
    //----------------------------------------------------DHT11-------------------------------------------------------

    float hum = dht.readHumidity();
    float temp = dht.readTemperature();
    Serial.print("Isı:");
    Serial.print(temp);
    Serial.print("|  Nem:");
    Serial.println(hum);



    isinemtime = millis();


  }


}
