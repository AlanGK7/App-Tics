# App de Monitoreo BLE para Lentes de Asistencia (ESP32)

Este repositorio contiene el código fuente de una aplicación móvil (React Native / Expo) diseñada para servir como puente de comunicación para un prototipo de "lentes inteligentes" para personas no videntes, basado en un ESP32.

La función principal de la app es **recibir datos (logs) enviados desde el hardware (ESP32) vía Bluetooth Low Energy (BLE)**, enriquecerlos con datos de geolocalización y hora, y enviarlos a una API web para su almacenamiento y posterior análisis.

Una característica clave es su **capacidad de operar offline**: si la app no detecta una conexión a internet, almacena los logs localmente y los envía automáticamente una vez que la conexión se restaura.

## Implementaciones Principales

Este proyecto integra varias tecnologías complejas para asegurar un flujo de datos robusto y resiliente.

### 1. Gestión Avanzada de Bluetooth (BLE)

La comunicación con el hardware ESP32 se maneja a través del hook `useBLE.ts`, que implementa:

* **Solicitud de Permisos:** Manejo de los permisos de `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` y `ACCESS_FINE_LOCATION`, adaptándose a las distintas versiones de Android (API < 31 y 31+).
* **Escaneo y Conexión:** Escaneo de dispositivos BLE cercanos y un modal (`DeviceModal.tsx`) para que el usuario seleccione el dispositivo (los lentes) al cual conectarse.
* **Suscripción a Características:** Monitoreo activo de una característica BLE específica (`CHARACTERISTIC_UUID: 6e400003...`) para recibir un *stream* de datos en tiempo real desde el ESP32.
* **Decodificación de Datos:** Los datos recibidos del ESP32 llegan en formato `base64` y son decodificados a texto plano (`UTF-8`) dentro de la app.
* **Reconexión Automática:** La app guarda el `ID` del último dispositivo conectado. Si la conexión se pierde, intentará reconectarse automáticamente a ese dispositivo cada 5 segundos.

### 2. Cola de Logs Offline (Resiliencia de Red)

El archivo `service.ts` implementa una lógica de negocio crucial para no perder datos si el usuario está sin conexión.

* **Verificación de Red:** Antes de cualquier envío, la app usa `expo-network` (`internetConection.ts`) para verificar si el dispositivo tiene una conexión a internet válida.
* **Almacenamiento Local:** Si no hay conexión, el log (junto a su hora y ubicación) se guarda en una cola de logs pendientes dentro de `AsyncStorage`.
* **Sincronización Automática:** En cuanto la app detecta que hay conexión a internet, primero procesa y envía todos los logs pendientes almacenados localmente, y solo después envía el log actual. Esto asegura un orden cronológico y que ningún dato se pierda.

### 3. Enriquecimiento de Datos (Contexto)

Para que los logs enviados por el ESP32 (ej. "obstáculo detectado distancia XX localización LLLL") tengan más valor, la app los enriquece con dos datos clave:

* **Geolocalización:** Usando `expo-location` (`location.ts`), la app captura las coordenadas GPS (latitud y longitud) exactas del usuario en el momento en que se genera el log. Esta área da paso a la escalibilidad del proyecto, como un sistema de reporte en el cual mediante geolocalización podemos hacer análisis sobre distancias críticas y lugares peligrosos.
* **Timestamp (Marca de Tiempo):** Se genera una marca de tiempo detallada (`service.ts`) que incluye la zona horaria del usuario, año, mes, día y hora exacta, asegurando que los datos puedan ser analizados en una línea de tiempo.

### 4. Envío de Datos a API

* Toda la información compilada (MAC del dispositivo, mensaje del log, hora y ubicación) se envía a una API web (en este caso, `tics-web.vercel.app`) mediante peticiones `POST` y `GET` con `axios`.

## Tecnologías Utilizadas

* **Core:** React Native (Expo)
* **Lenguaje:** TypeScript
* **Bluetooth:** `react-native-ble-plx`
* **Peticiones HTTP:** `axios`
* **Gestión de Red:** `expo-network`
* **Geolocalización:** `expo-location`
* **Almacenamiento Local:** `@react-native-async-storage/async-storage`
* **Utilidades:** `expo-device`, `react-native-base64`

## Flujo de Datos

1.  El **ESP32** detecta un evento (ej. un obstáculo) y envía un dato (`string`) codificado en `base64` a través de una característica BLE.
2.  La **App (useBLE.ts)** está suscrita a esa característica, recibe el dato y lo decodifica.
3.  Se llama a la función `enviarLog` (`service.ts`).
4.  `enviarLog` obtiene la **ubicación GPS** (`location.ts`) y la **hora actual**.
5.  `enviarLog` comprueba la **conexión a internet** (`internetConection.ts`).
    * **Caso A (Online):** La app envía todos los logs pendientes de `AsyncStorage` y luego envía el log actual a la API.
    * **Caso B (Offline):** La app guarda el nuevo log (con su ubicación y hora) en la cola de `AsyncStorage`.
6.  La **API Web** recibe los datos y los almacena en una base de datos.

## Estructura de Archivos Clave
miapp/
├── App.tsx             # (Componente raíz de la App)
├── index.ts            # Punto de entrada de la aplicación
├── package.json        # Dependencias y scripts
│
├── components/
│   └── DeviceModal.tsx # Modal de UI para seleccionar dispositivos BLE
│
├── hooks/
│   └── useBLE.ts       # Hook principal con toda la lógica de BLE
│
└── utils/
    ├── internetConection.ts # Helper para verificar conectividad
    ├── location.ts          # Helper para obtener ubicación GPS
    └── service.ts           # Lógica de negocio (API, cola offline, timestamps)
