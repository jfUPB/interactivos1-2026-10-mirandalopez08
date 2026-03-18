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



## Bitácora de reflexión
