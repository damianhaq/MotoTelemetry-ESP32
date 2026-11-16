# 🏍️ MotoTelemetry ESP32: Customowy Dashboard dla klasycznych motocykli

Projekt budowy własnego, zaawansowanego komputera pokładowego (telemetrii) dla starszych motocykli (np. Kawasaki ZZR 600), które nie posiadają szyny CAN ani złącza OBD2. 

Cały system oparty jest na mikrokontrolerze **ESP32** i wykorzystuje analogowe sygnały z instalacji motocykla oraz dedykowane czujniki zewnętrzne, aby wyliczać zaawansowane parametry jazdy w czasie rzeczywistym. Co najważniejsze – projekt jest przystosowany do montażu **całkowicie bez lutowania** (z wykorzystaniem terminali śrubowych).

## 🚀 Funkcje systemu

* **Kąt pochylenia (Lean Angle):** Precyzyjny odczyt głębokości złożenia w zakręcie.
* **Przeciążenia (G-Force):** Rejestrowanie siły hamowania i przyspieszania (wzdłużnego oraz bocznego).
* **Obroty silnika (RPM):** Pobierane bezpośrednio z cewki zapłonowej z pełną izolacją galwaniczną.
* **Prędkość rzeczywista (Speed):** Mierzona z koła (czujnik Halla), niezależna od mechanicznych linkomani i uślizgów.
* **Wskaźnik zapiętego biegu:** Wyliczany dynamicznie w locie za pomocą prostej matematyki (Stosunek Obrotów do Prędkości).
* **Wskaźnik Luzu (Neutral):** Odczyt z fabrycznej kontrolki w motocyklu.

---

## 🛒 Hardware (BOM - Bill of Materials)

Lista podzespołów została dobrana z myślą o pancernej niezawodności i pracy w środowisku z dużymi wibracjami oraz zakłóceniami elektromagnetycznymi. 

### 1. Jednostka centralna i obraz
* **ESP32 DevKit V1 (38-pin):** Dwurdzeniowy mikrokontroler z zapasem mocy do płynnej grafiki.
* **ESP32 Terminal Adapter 38-pin:** Płytka rozszerzeń z zaciskami śrubowymi (eliminuje konieczność lutowania).
* **Wyświetlacz IPS TFT (1.69" lub 2.0"):** Nowoczesny, gładki prostokątny ekran oparty na sterowniku **ST7789** (interfejs SPI).
* **Obudowa:** Dowolna puszka instalacyjna IP65 z okienkiem z pleksy lub wydruk 3D.

### 2. Sensoryka i pomiary
* **Bosch BNO055 (Moduł IMU 9-DOF):** Posiada własny koprocesor realizujący Sensor Fusion. Oblicza kąty i odfiltrowuje drgania ramy bez obciążania ESP32.
* **Czujnik Halla NJK-5002C:** Przemysłowy, gwintowany czujnik zliczający obroty koła.
* **Magnes neodymowy:** (Biegunowy) Wklejany w śrubę mocującą tarczy hamulcowej lub zębatki.

### 3. Zabezpieczenia i zasilanie (Ważne!)
* **Przetwornica LM2596 (Step-down) z zaciskami ARK:** Redukuje napięcie z motocyklowego 12V do bezpiecznych 5V.
* **Moduł Optoizolatora 2-kanałowego (PC817):** Galwanicznie oddziela "brudne", pełne przepięć sygnały 12V (Cewka zapłonowa i kontrolka N) od delikatnych pinów 3.3V w ESP32. Przekazuje sygnał wyłącznie za pomocą światła LED.
* **Przewody łączące:** Kable zworkowe Dupont (Żeńsko-Męskie) do połączenia ekranu/czujnika BNO z terminalem śrubowym.

---

## 🔧 Architektura i fizyka (Jak to działa?)

### 1. Dlaczego BNO055, a nie tani MPU-6050?
Zwykły akcelerometr gubi orientację w zakręcie, ponieważ siła odśrodkowa wypadkowa naciska idealnie w oś motocykla. BNO055 wykorzystuje zintegrowany żyroskop i własne algorytmy matematyczne do odzyskania tego położenia. 
**Krytyczne ustawienie w kodzie:** Moduł musi pracować w trybie `OPERATION_MODE_IMU` (z wyłączonym kompasem), aby rama i prądy z alternatora motocykla nie zakłócały odczytów!

### 2. Obroty (RPM) i separacja szpilek
Motocykle takie jak ZZR 600 stosują system *Wasted Spark* (jedna cewka przypada na dwa cylindry i podaje iskrę raz na jeden obrót wału korbowego). Pin niskiego napięcia (12V) cewki podłączamy do diody LED w module PC817. Fototranzystor w module pobudza pin przerwaniowy (Hardware Interrupt) w ESP32. Zliczając przerwania w jednostce czasu, uzyskujemy perfekcyjne obroty silnika, całkowicie separując mikrokontroler przed wstecznymi impulsami rzędu 100V+.

### 3. Obliczanie zapiętego biegu (Math Magic)
System nie ma czujnika położenia wybieraka w skrzyni biegów. Zamiast tego dzieli aktualne **Obroty (RPM)** przez **Prędkość (km/h)**. 
Każdy bieg ma swoje stałe przełożenie mechaniczne, więc to działanie dla konkretnego biegu daje zawsze stały współczynnik (niezależnie od tego, czy jedziesz wolno, czy odkręcasz manetkę). Kod sprawdza współczynnik w instrukcji warunkowej i wyrzuca na ekran konkretną cyfrę (od 1 do 6).

---

## 🛠️ Wskazówki montażowe (Mechanika)

1. **Wybór i montaż magnesu (Czujnik Halla):** Magnesy neodymowe tracą właściwości przy ok. 80°C. *NIGDY* nie wklejaj magnesu na zewnętrzny (roboczy) wieniec tarczy hamulcowej. Wklej go w gniazdo imbusowe śruby mocującej tarczę do nośnika lub zamontuj na tylnej zębatce.
2. **Polaryzacja:** Przed wklejeniem magnesu przetestuj, na który biegun (stronę) reaguje NJK-5002C! Jest to czujnik jednobiegunowy.
3. **Ustawienie szczeliny:** Odległość czoła czujnika Halla od magnesu powinna wynosić 2-4 mm. 
4. **Wibracje:** Do zabezpieczenia gwintów NJK-5002C na aluminiowym wsporniku *bezwzględnie* użyj niebieskiego kleju do gwintów (Loctite). Po zmontowaniu instalacji elektrycznej w puszce, łączenia zworkowe Dupont polej klejem na gorąco, aby zapobiec ich rozpięciu od wibracji silnika.
5. **Zasilanie układu:** 12V wprowadzane na przetwornicę LM2596 bierzemy zawsze z kabla "Po stacyjce". W przeciwnym razie system powoli rozładuje akumulator na postoju w garażu.

---

## 💻 Kod oprogramowania (Wkrótce)

*(Miejsce na kod programu. Główne wykorzystywane biblioteki to `TFT_eSPI` dla sprzętowej obsługi nowoczesnej grafiki z antyaliasingiem oraz `Adafruit_BNO055` dla czujnika przechyłu).*

```cpp
// Tu w przyszłości wkleję gotowy kod z obsługą TFT, Przerwań sprzętowych i logiki liczącej.
```

---
*Zbudowane w garażu. Z pasji do miedzianych zwojów i ostrych zakrętów.*