# Unidad 4

## Bitácora de proceso de aprendizaje

## Bitácora de aplicación 

Código MicrobitASCII2Adapter: 

```js
function parseCsvLine(line) {
  const values = line.trim().split("|");
  if (values.length !== 6) throw new ParseError(`Expected 6 values, got ${values.length}`);

  const t = Number(values[0].split(":")[1]);
  const x = Number(values[1].split(":")[1]);
  const y = Number(values[2].split(":")[1]);
  const btnA = Number(values[3].split(":")[1]);
  const btnB = Number(values[4].split(":")[1]);
  const CHK = Number(values[5].split(":")[1]) % 1000;
  const calcCHK = Math.abs(x) + Math.abs(y) + btnA + btnB;
  if (calcCHK !== CHK)
  throw new ParseError("Checksum mismatch");

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  if (![1, 0].includes(btnA) || ![1, 0].includes(btnB)) throw new ParseError("Invalid button data");

  return { x: x | 0, y: y | 0, btnA: btnA === 1, btnB: btnB === 1 };
}
```

añadir en el bridgeServer 

``` js
async function createAdapter()
if (DEVICE === "microbit2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit 2 found at ${path}`);
    return new MicrobitAscii2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }
```
``` js
const MicrobitAscii2Adapter = require("./adapters/MicrobitASCII2Adapter");
```

Codigo del micro:bit
```py
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    t = running_time()
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0
    chk = (abs(xValue) + abs(yValue) + aState + bState) % 1000

    data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(
        t, xValue, yValue, aState, bState, chk
    )
    uart.write(data)
    sleep(100) # Envia datos a 10 Hz
```

Código en el sketch.js drawRunning
```js
function drawRunning() {
    let mb = painter.rxData;

    if (!mb.ready) return;

    // equivalente a mouseIsPressed
    if (mb.btnA == 1) {

        push();
        translate(width / 2, height / 2);

        // equivalente a mouseY
        let circleResolution = int(map(mb.y + 100, 0, height, 2, 10));

        // equivalente a mouseX
        let radius = mb.x - width / 2;

        let angle = TAU / circleResolution;

        // equivalente a keyIsPressed
        if (mb.btnB == 1) {
            fill(34, 45, 122, 50);
        } else {
            noFill();
        }
        
        beginShape();
        for (let i = 0; i <= circleResolution; i++) {
            let x = cos(angle * i) * radius;
            let y = sin(angle * i) * radius;
            vertex(x, y);
        }
        endShape();

        pop();
    }
}
```

## Bitácora de reflexión

