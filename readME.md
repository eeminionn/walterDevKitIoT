# 🔐 Collar Inteligente de Seguridad — Sistema Embebido con Walter DevKit  
### *Alerta de emergencia mediante SMS + GPS en un wearable estético y discreto*

---

## 📘 Resumen del Proyecto

Este repositorio documenta el desarrollo completo de un **sistema embebido de seguridad personal** integrado en un **collar inteligente**, capaz de enviar un **SMS de emergencia** acompañado de la **ubicación GPS** del usuario cuando este presiona un botón oculto.

El corazón del sistema es el **Walter DevKit**, una plataforma IoT con:

- Microcontrolador **ESP32-S3**
- Módem celular **Sequans GM02SP** compatible con LTE-M / NB-IoT
- **GNSS** integrado (GPS)
- APIs avanzadas para SMS, sockets, GNSS y gestión de energía

El dispositivo está encapsulado en una **carcasa 3D personalizada** diseñada para ocultarse en un collar de piedras preciosas, permitiendo un sistema estético, discreto y altamente funcional.

---

# 🧩 Descripción General

El proyecto consiste en integrar un **sistema electrónico completo dentro de un collar**, combinando estética y funcionalidad. El usuario, al presionar discretamente un botón escondido, activa el sistema de emergencia:

1. El DevKit despierta del modo de bajo consumo  
2. Se inicializa el módem celular  
3. Se obtiene un **GNSS fix** con las coordenadas actuales  
4. Se envía un **SMS de alerta** al contacto deseado  
5. El sistema vuelve al modo de bajo consumo

Este tipo de dispositivo es útil en:

- Casos de violencia intrafamiliar  
- Seguimiento de adultos mayores  
- Protección personal en salidas nocturnas  
- Rutas deportivas en zonas aisladas  

Su diseño en formato de collar lo hace **discreto, elegante y no intrusivo**.

---

# 🎯 Objetivos del Sistema

- Crear un **dispositivo de emergencia portable** sin apariencia tecnológica.  
- Enviar un **SMS automático** con ubicación GPS en un evento crítico.  
- Ejecutar la alarma con **un solo botón físico oculto**.  
- Integrar tutto dentro de una **carcasa 3D embebida en el collar**.  
- Mantener un consumo energético reducido para autonomía prolongada.  
- Asegurar **compatibilidad con redes celulares LTE-M y NB-IoT**.

---

# ⚙️ Arquitectura del Sistema Embebido

## 🧱 Componentes Principales

| Componente | Función |
|-----------|---------|
| **Walter DevKit (ESP32-S3 + GM02SP)** | Procesamiento, comunicación celular y GNSS |
| **Botón de emergencia (GPIO)** | Activa el sistema |
| **Antena LTE-M** | Comunicación con la red |
| **Antena GNSS** | Obtención de ubicación satelital |
| **Batería Li-Po** | Alimentación portable |
| **Carcasa 3D personalizada** | Integración mecánica dentro del collar |


---

# 🔌 Flujo de Funcionamiento

1. Usuario presiona el **botón oculto**  
2. ESP32-S3 sale de *deep sleep*  
3. Se inicializa el módem GM02SP  
4. Se establece conexión LTE-M / NB-IoT  
5. Se obtiene posición GNSS  
6. Se envía SMS con el mensaje y coordenadas  
7. Sistema vuelve a modo de bajo consumo  

---

# 🛠️ Características del Walter DevKit

Este módulo fue seleccionado porque:

- Tiene **LTE-M/NB-IoT integrado**, ideal para dispositivos móviles.  
- Permite **envío nativo de SMS** vía AT commands.  
- Posee **GNSS integrado**, simplificando hardware adicional.  
- Ofrece **bajo consumo energético**.  
- Es compacto y fácil de integrar en wearables.  
- Su API es avanzada, estable y con soporte profesional.

---

# 💻 Programación del Sistema

El firmware se desarrolló en **ESP-32**, usando:

- `WalterModem.h` para manejar el módem y GNSS  
- `HardwareSerial` para UART  
- Interrupciones de GPIO  
- Modo de bajo consumo del ESP32-S3  

El sistema valida:

- Estado de SIM (`AT+CPIN?`)  
- Registro de red (`AT+CEREG?`)  
- Calidad de señal  
- Estado de GNSS  
- Obtención del fix  
- Ensamble de datos  
- Envío por SMS (modificable a socket TCP/UDP)

---

# 🧱 Diseño del Modelo 3D

El sistema se integra en una carcasa diseñada en CAD que:

- Permite flujo de señal para LTE/GNSS  
- Evita apantallamiento  
- Asegura firmeza mecánica  
- Oculta la electrónica elegantemente  
- Alberga batería, PCB y antenas  

```cpp
#include <WalterModem.h>
#include <HardwareSerial.h>

WalterModem modem;

// GNSS
volatile bool gnssFixRcvd = false;
WalterModemGNSSFix latestFix = {};

void setup() {
    Serial.begin(115200);
    delay(3000);
    if (!WalterModem::begin(&Serial2)) {
        Serial.println("Modem init failed");
        ESP.restart();
    }
    modem.gnssSetEventHandler(gnssEventHandler, NULL);
    modem.socketSetEventHandler(socketEventHandler, NULL);
}
// ---------------- LTE Connection ----------------

bool lteConnected() {
    auto reg = modem.getNetworkRegState();
    return (reg == WALTER_MODEM_NETWORK_REG_REGISTERED_HOME ||
            reg == WALTER_MODEM_NETWORK_REG_REGISTERED_ROAMING);
}

bool lteConnect() {
    modem.setOpState(WALTER_MODEM_OPSTATE_NO_RF);
    modem.definePDPContext(1, "");
    modem.setOpState(WALTER_MODEM_OPSTATE_FULL);
    modem.setNetworkSelectionMode(WALTER_MODEM_NETWORK_SEL_MODE_AUTOMATIC);
    Serial.print("Connecting to LTE...");
    while (!lteConnected()) {
        Serial.print(".");
        delay(1000);
    }
    Serial.println("\nLTE connected");
    return true;
}

// ---------------- GNSS Event Handler ----------------

void gnssEventHandler(const WalterModemGNSSFix* fix, void* args) {
    memcpy(&latestFix, fix, sizeof(WalterModemGNSSFix));
    gnssFixRcvd = true;
    Serial.printf("Fix: Lat=%.6f Lon=%.6f Conf=%.2f\n",
                  fix->latitude, fix->longitude, fix->estimatedConfidence);
}

// ---------------- GNSS Fix Request ----------------

bool attemptGNSSFix() {
    gnssFixRcvd = false;
    modem.gnssPerformAction();
    Serial.println("Requesting GNSS fix...");
    while (!gnssFixRcvd) {
        Serial.print(".");
        delay(500);
    }
    Serial.println();
    return latestFix.estimatedConfidence <= 100.0;
}

// ---------------- Data Packet (Demo / Adaptable a SMS) ----------------

uint8_t dataBuf[30] = {0};

void sendPacket() {
    float lat = latestFix.latitude;
    float lon = latestFix.longitude;
    memcpy(dataBuf + 10, &lat, 4);
    memcpy(dataBuf + 14, &lon, 4);
    if (modem.socketDial("walterdemo.quickspot.io", 1999)) {
        Serial.println("Socket connected");
        if (modem.socketSend(dataBuf, sizeof(dataBuf))) {
            Serial.println("Packet sent");
        } else {
            Serial.println("Send error");
        }
        modem.socketClose();
    } else {
        Serial.println("Dial error");
    }
}

// ---------------- Socket Handler ----------------

void socketEventHandler(WalterModemSocketEvent ev, int socketId,
                        uint16_t bytes, uint8_t* buffer, void* args) {
    if (ev == WALTER_MODEM_SOCKET_EVENT_RING) {
        Serial.printf("Socket %d (%u bytes):\n", socketId, bytes);
        Serial.printf("%.*s\n", bytes, buffer);
    }
}

// ---------------- Main Loop ----------------
void loop() {
    if (attemptGNSSFix()) {
        Serial.println("GNSS OK");
    }
    lteConnect();
    sendPacket();
    delay(60000);
}
```

```markdown



