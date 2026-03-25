# Unidad 5
## Bitácora de proceso de aprendizaje
Al utilizar ascii no tengo certeza de cuantos bites estoy usando, mientras que al usar binario puedo optimizar definiendo que cantidad de datos voy a recibir, por ejemplo, si en el código que usamos en la unidad para enviar datos desde el microbit lo modifico de la siguiente manera:
```py
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = 567
    yValue = 231
    aState = true
    bState = false
    data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
    uart.write(data)
    sleep(100)
```
los datos enviados a la aplicacion de consexion serial serán: 

02 37 00 e7 01 00

**02 37** = 567
**00 e7** = 231
**01** = 1 = true
**00** = 0 = false

*Los numeros enviados en binario tinene dos clasificaciones*

**BIG ENDIAN:** que es enviar primero el dato de mayor peso y luego el dato de menor peso, por ejemplo *02 37*

**LITTLE ENDIAN:** enviar primero el dato de menor peso y luego el de mayor peso por ejemplii *37 02* 

Es importante saber cuál es porque pueden ser dos números diferentes

Un ejemplo 

`05 F3 ¿Que numero te envié?`

Tienes dos opciones:

`Opción 1:` si me enviaron Little endian: 62213 (armo el numero como F305)

`Opción 2:` si me enviaron Big endian: 62213 (armo el numero como 05F3)

## ¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?
Es mucho más fácil controlar los datos que entran y los bites que ocupan, lo que significa más seguridad al momento de analizar los datos y la posibilidad de optimizar el sistema, por otra parte una de las desventajas más claras es que los datos son más dificiles de leer e interpretar por lo que es importante saber como manejarlos 

Si xValue=500, yValue=524, aState=True, bState=False, ¿cómo se vería el paquete en hexadecimal? (Pista: convierte cada valor según su tipo y anota los bytes en orden.) Respuesta esperada: 01 F4 02 0C 01 00

## Bitácora de aplicación 

**binaryAdapter:**
```js
// adapters/MicrobitBinaryAdapter.js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error {}

const SYNC_BYTE = 0xaa;
const PACKET_SIZE = 8;

/**
 * Packet structure (8 bytes):
 *   [0] 0xAA        sync header
 *   [1-2] int16 BE  accelerometer X
 *   [3-4] int16 BE  accelerometer Y
 *   [5] uint8       button A (1=pressed)
 *   [6] uint8       button B (1=pressed)
 *   [7] uint8       checksum = sum(bytes 1..6) % 256
 */
function parsePacket(buf) {
  if (buf[0] !== SYNC_BYTE) throw new ParseError("Missing sync byte");

  const checksum = buf[7];
  const calcChecksum = (buf[1] + buf[2] + buf[3] + buf[4] + buf[5] + buf[6]) % 256;

  if (calcChecksum !== checksum) {
    throw new ParseError(`Checksum mismatch: expected ${calcChecksum}, got ${checksum}`);
  }

  const x = buf.readInt16BE(1);
  const y = buf.readInt16BE(3);
  const btnA = buf[5] === 1;
  const btnB = buf[6] === 1;

  return { x, y, btnA, btnB };
}

class MicrobitBinaryAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort path is required for microbit binary mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port?.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => (err ? reject(err) : resolve()));
      });
    }

    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path} @${this.baud}`;
  }

  _onChunk(chunk) {
    this.buf = Buffer.concat([this.buf, chunk]);

    while (this.buf.length >= PACKET_SIZE) {
      // Buscar sync byte 0xAA
      const syncIdx = this.buf.indexOf(SYNC_BYTE);

      if (syncIdx < 0) {
        // No hay sync byte en todo el buffer, descartar todo
        if (this.verbose) console.warn(`[MicrobitBinary] No sync byte found, discarding ${this.buf.length} bytes`);
        this.buf = Buffer.alloc(0);
        break;
      }

      if (syncIdx > 0) {
        // Hay basura antes del sync byte, descartarla
        if (this.verbose) console.warn(`[MicrobitBinary] Discarding ${syncIdx} garbage bytes before sync`);
        this.buf = this.buf.slice(syncIdx);
      }

      // Esperar a tener el paquete completo
      if (this.buf.length < PACKET_SIZE) break;

      const packet = this.buf.slice(0, PACKET_SIZE);

      try {
        const parsed = parsePacket(packet);
        this.buf = this.buf.slice(PACKET_SIZE);
        this.onData?.(parsed);
      } catch (e) {
        if (e instanceof ParseError) {
          // El 0xAA era un falso positivo — avanzar 1 byte y re-sincronizar
          console.warn(`[MicrobitBinary] ${e.message} — re-syncing`);
          this.buf = this.buf.slice(1);
        } else {
          this._fail(e);
        }
      }
    }

    // Guardia contra buffer desbordado (datos sin sync por mucho tiempo)
    if (this.buf.length > 4096) {
      console.warn("[MicrobitBinary] Buffer overflow, flushing");
      this.buf = Buffer.alloc(0);
    }
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port?.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitBinaryAdapter;
```

**Añadir en el bridge server**
```js
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
```

```js
 if (DEVICE === "microbit-bin") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    return new MicrobitBinaryAdapter({ path, baud: BAUD });
  }
```
**código del microbit**

```py
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0, 0, 9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
    checksum = sum(data) % 256
    packet = b'\xAA' + data + bytes([checksum])
    uart.write(packet)
    sleep(100)
```


## Bitácora de reflexión

**1. Realiza una tabla comparativa entre el adapter ASCII que creaste en la Unidad 4 (MicrobitV2Adapter.js) y el adapter binario de esta unidad (MicrobitBinaryAdapter.js). Compara al menos:**

**Tamaño de cada paquete en bytes.
Mecanismo de delimitación/framing.
Mecanismo de verificación de integridad (checksum).
Complejidad de implementación del parser.
Facilidad de depuración (¿cuál es más fácil de leer con un terminal serial?).**

## ASCII
*Tamaño*: el tamaño puede ser variable ya que no se determinan la cantidad de bytes usados
*Framing*: Cada paquete comienza con un carácter identificador como $ y termina con un salto de línea (\n). El parser simplemente acumula caracteres del puerto serial hasta encontrar el fin de línea, momento en el cual considera que ha recibido un paquete completo y procede a procesarlo.
*checksum*: chk = (abs(xValue) + abs(yValue) + aState + bState) % 1000 
*parser*: los datos se reciben como texto. El programa solo necesita leer la línea completa hasta el carácter de fin de línea, separar los campos usando los delimitadores y convertir las cadenas a valores numéricos. Este proceso es directo y fácil de programar.
*facilidad*: es más simple pues los datos vienen como texto y se leen literalmente

## Binario: 
*Tamaño*: es posible regular el tamaño de el paquete de los bytes lo que permite optimizar el código, en este caso los bytes serán 8 y no podrán pasarse
*Framing*: Es importante realizar framing porque el paquete de datos puede desconfigurarse si entra un byte de más pero esto ayuda a asegurar que los datos llegan en el orden correcto, aqui se determinará con el inicio de aa y terminará cuando se cumplan los 8 bytes
*checksum*: checksum = sum(data) % 256 este numero es el rango, el checksum se realiza con este límite 
*parser*: Debe trabajar directamente con bytes, asegurarse de que se han recibido exactamente los bytes esperados, interpretar correctamente el orden y tamaño de cada campo dentro del paquete y verificar el checksum antes de considerar válidos los datos.
*facilidad*: este es más complejo pues los paquetes de datos vienen decodificados y es necesario interpretarlos

**2. ¿Por qué la arquitectura desacoplada (patrón Adapter + Bridge + FSM) te permite añadir soporte para un protocolo completamente diferente sin modificar el frontend (sketch.js) ni el transporte (bridgeClient.js)?**
Ya que es posible adaptar diversos protocolos de lectura de datos a el mismo procedimiento por lo que no será necesario crear nuevos sketchs o bridgeclients porque estos solo recibirán la información y la procesarán sin importar como el dispositivo envía la información

**3. ¿En qué situaciones del mundo real preferirías un protocolo binario sobre uno ASCII y viceversa? Justifica con ejemplos concretos.**
Preferiría un **protocolo binario** en situaciones donde la eficiencia, la velocidad y la confiabilidad son críticas. Por ejemplo, en sistemas embebidos o de tiempo real como drones, robots o sensores industriales que envían datos a alta frecuencia, el uso de paquetes binarios reduce el tamaño de transmisión y permite procesar los datos más rápido. Esto disminuye la latencia y el uso del ancho de banda, lo cual es importante cuando el dispositivo tiene recursos limitados o cuando se transmiten muchos datos por segundo.

En cambio, elegiría un **protocolo ASCII** en etapas de desarrollo, prototipado o en sistemas donde la facilidad de depuración y la legibilidad humana son más importantes que la eficiencia. Por ejemplo, al trabajar con un micro:bit en un entorno educativo o durante pruebas iniciales de un sistema serial, es mucho más sencillo leer directamente en un monitor serial mensajes como `X:120|Y:-45|A:1` que interpretar una secuencia de bytes en hexadecimal. También es útil en aplicaciones donde los datos se registran en archivos de texto o deben ser revisados manualmente por desarrolladores o técnicos.


**4. Actualiza el diagrama de flujo de datos de la Unidad 4 para reflejar el protocolo binario. ¿Qué componentes cambiaron? ¿Qué componentes permanecieron intactos?**
[Ver diagrama](https://excalidraw.com/#json=qSKuMX8nLh8JUp3s88MAI,S2dItJzIkd44Uw5G6UOZZg)

## pruebas extra

para confirmar que el micro:bit todavía funciona con los códigos anteriores realicé pruebas:

ASCII adapter 1: 

<img width="903" height="488" alt="image" src="https://github.com/user-attachments/assets/982b2abd-6f02-4b80-91cf-0164b948b47e" />
<img width="837" height="131" alt="image" src="https://github.com/user-attachments/assets/32c533e5-c148-4cd4-922b-085cbe81729e" />
<img width="1902" height="910" alt="image" src="https://github.com/user-attachments/assets/8ae73b96-9007-4ae6-8541-26bf62219ca0" />




ASCII adapter 2 / micro:bit 2

<img width="850" height="616" alt="image" src="https://github.com/user-attachments/assets/5a6a74df-061b-4021-9ee3-2601640b4196" />
<img width="817" height="206" alt="image" src="https://github.com/user-attachments/assets/28c695ec-d14c-4238-91b3-145a8ffb4cf8" />
<img width="1883" height="895" alt="image" src="https://github.com/user-attachments/assets/b7f9fba5-71ff-44d4-8bcf-3fb54e58944b" />


