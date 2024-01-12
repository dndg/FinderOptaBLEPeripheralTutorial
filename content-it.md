---
title: 'Peripheral BLE su Finder Opta'
description: "Imparare ad utilizzare Finder Opta come Peripheral BLE,
              creando una Characteristic che permetta ad un Central di cambiare
              lo stato dei LED del dispositivo."
author: 'Fabrizio Trovato'
libraries:
  - name: 'ArduinoBLE'
    url: https://www.arduino.cc/reference/en/libraries/arduinoble/
difficulty: intermediate
tags:
  - BLE
software:
  - ide-v1
  - ide-v2
  - arduino-cli
  - web-editor
hardware:
  - hardware/07.opta/opta-family/opta
---

## Panoramica

In questo tutorial, mostreremo come configurare Finder Opta per far si che
agisca da Peripheral BLE, a cui un Central si possa connetterre per pilotare i
LED del dispositivo. In particolare, andremo a configurare un Service contente
una Characteristic in lettura e scrittura, da cui un Central possa scrivere o
leggere dei valori esadecimali corrispondenti a diversi stati dei LED.

## Obiettivi

* Imparare a creare un Service ed una Characteristic BLE su Finder Opta.
* Imparare a leggere i dati scritti dai Central nelle Characteristic di Finder
  Opta.

## Requisiti hardware e software

### Requisiti hardware

* PLC Finder Opta (x1).
* Cavo USB-C® (x1).

### Requisiti Software

* [Arduino IDE 1.8.10+](https://www.arduino.cc/en/software), [Arduino IDE
2.0+](https://www.arduino.cc/en/software) o [Arduino Web
Editor](https://create.arduino.cc/editor).
* Se si utilizza Arduino IDE offline, è necessario installare la libreria
  `ArduinoBLE` utilizzando il Library Manager dell'Arduino IDE.
* [Codice di esempio](assets/OptaBLEPeripheralExample.zip).

## Finder Opta e il BLE

Grazie alla libreria `ArduinoBLE`, Finder Opta può comportarsi come Peripheral
BLE e configurare Service con una o più Characteristic in modalità lettura e/o
scrittura. Sempre sfruttando questa libreria, il Finder Opta potrà fare
advertising e attendere la connessione di un Central con cui scambiare comandi.

## Istruzioni

### Configurazione dell'Arduino IDE

Per seguire questo tutorial, sarà necessaria [l'ultima versione dell'Arduino
IDE](https://www.arduino.cc/en/software). Se è la prima volta che configuri il
Finder Opta, dai un'occhiata al tutorial [Getting Started with
Opta](/tutorials/opta/getting-started).

Assicurati di installare l'ultima versione delle librerie
[`ArduinoBLE`](https://www.arduino.cc/reference/en/libraries/arduinoble/)
poiché verrà utilizzata per la comunicazione BLE con il Central.

Per ulteriori dettagli su come installare manualmente le librerie, consulta
[questo
articolo](https://support.arduino.cc/hc/en-us/articles/5145457742236-Add-libraries-to-Arduino-IDE).

### Panoramica del codice

Lo scopo del seguente esempio è configurare Finder Opta come Peripheral BLE,
andando poi ad esporre una Characteristic in lettura e scrittura dentro cui un
Central possa scrivere per accendere o spegnere i LED del Finder Opta. In
particolare, configureremo un Service con una Characteristic ed avvieremo
l'advertising, attendendo scritture del Central contenenti comandi esadecimali
che pilotino i LED.

#### Setup dello sketch

Nella parte iniziale dello sketch dichiaramo il Service e la Characteristic BLE
con i relativi UUID. Inoltre, la Characteristic sarà configurata in modalità
_read/write_:

```cpp
BLEService ledService("b4bf4c90-a3eb-413f-9698-cfaaa9a428cc");
BLEByteCharacteristic ledCharacteristic("b4bf4c91-a3eb-413f-9698-cfaaa9a428cc", BLERead | BLEWrite);
```

Nella funzione di `setup()` andremo a configurare il nome del device usato
durante l'advertising, ed in seguito ad abbinare Service e Characteristic,
settandone anche il valore iniziale; al termine di queste operazioni avvieremo
l'advertising. Il codice della funzione è mostrato di seguito:

```cpp
void setup()
{
    Serial.begin(9600);

    // Init the BLE service.
    if (BLE.begin() == 0)
    {
        while (1)
        {
        }
    }

    // Set local name and service UUID.
    BLE.setLocalName("Finder Opta");
    BLE.setAdvertisedService(ledService);

    // Add service and characeristic.
    ledService.addCharacteristic(ledCharacteristic);
    BLE.addService(ledService);

    // Set initial value.
    ledCharacteristic.writeValue(0x00);

    // Start advertising.
    BLE.advertise();
}
```

#### Loop principale

La funzione `loop()` di questo sketch rimane in ascolto in attesa di
connessioni da parte di Central, fino a quando non se ne connette uno: in caso
di connessione il Finder Opta controllerà se la `ledCharacteristic` sia stata
scritta o meno, ed in caso affermativo leggerà il valore per pilotare i LED dal
numero 0 al numero 3. Il condice del loop principale è riportato di seguito:

```cpp
void loop()
{
    // Check if any central is available.
    BLEDevice central = BLE.central();
    if (central)
    {
        Serial.println("Central connected.");
        while (central.connected())
        {
            // If central wrote to characeristic.
            if (ledCharacteristic.written())
            {
                uint8_t state = ledCharacteristic.value();
                digitalWrite(LED_D0, state & 0x01);
                digitalWrite(LED_D1, (state & 0x02) >> 1);
                digitalWrite(LED_D2, (state & 0x04) >> 2);
                digitalWrite(LED_D3, (state & 0x08) >> 3);
            }
        }
        Serial.println("Central disconnected.");
    }
}
```

Si noti che lo stato del LED n-esimo dipenderà dal valore del bit n-esimo nel
comando scritto dal Central all'interno della `ledCharacteristic`: se il bit
_n_ vale 1 il LED _n_ verrà acceso, se il bit _n_ vale 0 il LED _n_ verrà
spento. Nei commenti del codice è disponibile una tabella contenente la
mappatura tra codice esadecimale del comando e comportamento dei LED.

### Esempio di interazione

Una volta compilato e caricato lo sketch sul Finder Opta, è possibile
utilizzare l'app [nRF
Connect](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-mobile)
per connettersi alla Peripheral BLE e scrivere un comando nella Characteristic.
Al termine della scansione il Finder Opta comparirà tra i dispositivi
disponibili:

<img src="assets/nrf1.jpg" width=250 alt="Screenshot di nRF Connect">

Procediamo connettendoci e vedremo apparire un Service e una Characteristic con
gli UUID assegnati dallo sketch:

<img src="assets/nrf2.jpg" width=250 alt="Screenshot di nRF Connect">

A questo punto clicchiamo sull'icona con freccia che punta verso l'alto per
effetuare una scrittura. Gli screenshot seguenti mostrano ad esempio come
accendere il LED 0 e il LED 3:

<img src="assets/nrf3.jpg" width=250 alt="Screenshot di nRF Connect">

<img src="assets/nrf5.jpg" width=250 alt="Screenshot di nRF Connect">

Infine questo ultimo screenshot mostra come accendere tutti i LED del Finder
Opta con un singolo comando:

<img src="assets/nrf4.jpg" width=250 alt="Screenshot di nRF Connect">

## Conclusioni

Questo tutorial mostra come configurare Finder Opta da Peripheral BLE, per
esporre una Characteristic in cui un Central possa scrivere un comando
esadecimale per pilotare lo stato dei LED del dispositivo.
