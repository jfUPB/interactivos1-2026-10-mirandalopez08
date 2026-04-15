# Unidad 6

## Bitácora de proceso de aprendizaje

Será necesario tener 2 web browsers, el de strudel y el de nuestra pagina de p5, ambos con su servidor correspondiente, para pasar de uno al otro no es posible hacerlo directamente si no que será necesario pasar por el bridge server, esto por cuestiones de seguridad. 


**¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?**
Recibir un mensaje es simplemente información que el servidor tendrá que interpretar, no necesariamete recibir el mensaje significará procesarlo pues hay mensajes que no son válidos y por ende no funcionarán para el sistema, por otra parte ejecutar el mensaje será enviar el mensaje a otro servidor que procesará la información.

**¿Por qué un sistema audiovisual puede necesitar timestamp además de los datos del evento?**
Es necesario debido a que al reproducir eventos 

**¿Qué aspectos de la arquitectura de las unidades 4 y 5 permanecen intactos aunque ahora la fuente de datos ya no sea hardware?**


## Bitácora de aplicación 

StrudelAdapter: 

```js
const { WebSocketServer } = require("ws");
const BaseAdapter = require("./BaseAdapter");

class StrudelAdapter extends BaseAdapter {
  constructor({ port = 8080, verbose = false } = {}) {
    super();
    this.port    = port;
    this.verbose = verbose;
    this._wss    = null;
  }

  // Abre el servidor WebSocket donde Strudel se conectará.
  // Equivalente a abrir el SerialPort en MicrobitAsciiAdapter.
  async connect() {
    if (this.connected) return;

    this._wss = new WebSocketServer({ port: this.port });

    this._wss.on("listening", () => {
      this.connected = true;
      this.onConnected?.(`strudel ws escuchando en :${this.port}`);
    });

    this._wss.on("connection", (ws) => {
      if (this.verbose) console.log("[StrudelAdapter] Strudel conectado");

      // Cada mensaje que llega de Strudel pasa por _onMessage.
      // Equivalente al evento "data" del chunk serial.
      ws.on("message", (raw) => this._onMessage(raw.toString("utf8")));
      ws.on("error",   (err) => this._fail(err));
    });

    this._wss.on("error", (err) => this._fail(err));
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;
    await new Promise((resolve) => this._wss.close(resolve));
    this._wss = null;
    this.onDisconnected?.("strudel ws cerrado");
  }

  getConnectionDetail() {
    return `strudel ws :${this.port}`;
  }

  // Recibe el string crudo, lo parsea y lo manda a normalizar.
  _onMessage(raw) {
    let parsed;
    try {
      parsed = JSON.parse(raw);
    } catch (e) {
      if (this.verbose) console.log("[StrudelAdapter] JSON inválido:", raw);
      return;
    }

    const normalized = this._normalize(parsed);

    // Si el mensaje no tiene un sonido reconocible, lo descartamos silenciosamente.
    if (!normalized) return;

    // Entrega el evento normalizado a bridgeServer mediante el hook onData.
    // A partir de aquí el adapter no sabe qué pasa con el mensaje.
    this.onData?.(normalized);
  }

  // Convierte el formato interno de Strudel al contrato del sistema.
  // args llega como array plano: ['cps', 0.5, 'cycle', 15.25, 's', 'tr909bd', ...]
  // Lo convertimos a objeto para acceder por nombre de parámetro.
  _normalize(msg) {
    const args   = msg.args ?? [];
    const params = {};
    for (let i = 0; i + 1 < args.length; i += 2) {
      params[args[i]] = args[i + 1];
    }

    // Sin campo 's' no hay sonido; descartamos el evento.
    if (!params.s) return null;

    // Contrato de salida estable.
    // type:"strudel" permite identificarlo en bridgeClient sin ambigüedad.
    return {
      type:      "strudel",
      timestamp: msg.timestamp ?? Date.now(),
      payload: {
        eventType: "noteEvent",
        s:         params.s,
        bank:      params.bank  ?? null,
        delta:     params.delta ?? 0.5,
        cycle:     params.cycle ?? 0,
        cps:       params.cps   ?? 0.5,
      },
    };
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  // No hay comandos de vuelta hacia Strudel en esta versión.
  async handleCommand(_cmd) {}
}

module.exports = StrudelAdapter;
```

Sketch: 
```js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        // ===== MICROBIT =====
        this.c = color(181, 157, 0);
        this.lineSize = 100;
        this.angle = 0;
        this.clickPosX = 0;
        this.clickPosY = 0;

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false
        };

        // ===== STRUDEL =====
        this.eventQueue = [];
        this.activeAnimations = [];

        // ===== MODO =====
        this.mode = null; // "microbit" | "strudel"

        this.transitionTo(this.estado_esperando);
    }

    update() {
        super.update();
        this._flushQueue();
    }

    _flushQueue() {
        if (this.mode !== "strudel") return;

        const now = Date.now();

        this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);

        while (this.eventQueue.length > 0) {
            const ev = this.eventQueue[0];

            if (ev.timestamp <= now) {

                const d = ev.payload;

                this.activeAnimations.push({
                    startTime: ev.timestamp,
                    duration: d.delta * 1000,
                    type: d.s,
                    x: random(width * 0.2, width * 0.8),
                    y: random(height * 0.2, height * 0.8),
                    color: getColorForSound(d.s)
                });

                this.eventQueue.shift();

            } else break;
        }
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } 
        else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            background(0);

            console.log("Connected. Waiting for data...");
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.mode = null;
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {

        // =========================
        // 🎵 STRUDEL
        // =========================
        if (data.type === "strudel") {
            this.mode = "strudel";
            this.eventQueue.push(data);
            return;
        }

        // =========================
        // 🎮 MICROBIT
        // =========================
        this.mode = "microbit";

        this.rxData.ready = true;
        this.rxData.x = map(data.x, -2048, 2047, 0, width);
        this.rxData.y = map(data.y, -2048, 2047, 0, height);
        this.rxData.btnA = data.btnA;
        this.rxData.btnB = data.btnB;

        if (this.rxData.btnA && !this.prevA) {
            this.lineSize = random(50, 160);
            this.clickPosX = this.rxData.x;
            this.clickPosY = this.rxData.y;
        }

        if (!this.rxData.btnB && this.prevB) {
            this.c = color(random(255), random(255), random(255), random(80, 100));
        }

        this.prevA = this.rxData.btnA;
        this.prevB = this.rxData.btnB;
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(0);

    painter = new PainterTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {

        // 🎵 STRUDEL
        if (data.type === "strudel") {
            painter.postEvent({ type: EVENTS.DATA, payload: data });
            return;
        }

        // 🎮 MICROBIT
        painter.postEvent({
            type: EVENTS.DATA,
            payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
        });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {

    // =========================
    // 🎵 STRUDEL
    // =========================
    if (painter.mode === "strudel") {

        background(0, 30);

        let now = Date.now();

        for (let i = painter.activeAnimations.length - 1; i >= 0; i--) {
            let anim = painter.activeAnimations[i];

            let elapsed = now - anim.startTime;
            let p = elapsed / anim.duration;

            if (p <= 1) {
                dibujarElemento(anim, p);
            } else {
                painter.activeAnimations.splice(i, 1);
            }
        }

        return;
    }

    // =========================
    // 🎮 MICROBIT
    // =========================
    else {

        let mb = painter.rxData;

        if (!mb.ready) return;

        if (mb.btnA == 1) {

            push();
            translate(width / 2, height / 2);

            let circleResolution = int(map(mb.y + 100, 0, height, 2, 10));
            let radius = mb.x - width / 2;
            let angle = TAU / circleResolution;

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

        return;
    }
}

// ========================
// 🎨 STRUDEL VISUALES
// ========================

function dibujarElemento(anim, p) {
    push();

    switch (anim.type) {
        case 'tr909bd':
            dibujarBombo(p, anim.color);
            break;

        case 'tr909sd':
            dibujarCaja(p, anim.color);
            break;

        case 'tr909hh':
        case 'tr909oh':
            dibujarHat(anim, p, anim.color);
            break;

        default:
            dibujarDefault(anim, p, anim.color);
            break;
    }

    pop();
}

function dibujarBombo(p, c) {
    let d = lerp(100, 600, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    circle(width / 2, height / 2, d);
}

function dibujarCaja(p, c) {
    let w = lerp(width, 0, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    rect(width / 2, height / 2, w, 50);
}

function dibujarHat(anim, p, c) {
    let sz = lerp(40, 0, p);
    fill(c[0], c[1], c[2]);
    rect(anim.x, anim.y, sz, sz);
}

function dibujarDefault(anim, p, c) {
    let size = lerp(100, 0, p);
    let angle = p * TWO_PI;

    translate(anim.x, anim.y);
    rotate(angle);

    stroke(c[0], c[1], c[2]);
    noFill();

    rect(0, 0, size, size);
    line(-size, 0, size, 0);
    line(0, -size, 0, size);
}

function getColorForSound(s) {
    const colors = {
        'tr909bd': [255, 0, 80],
        'tr909sd': [0, 200, 255],
        'tr909hh': [255, 255, 0],
        'tr909oh': [255, 150, 0]
    };

    if (colors[s]) return colors[s];

    let charCode = s.charCodeAt(0) || 0;

    return [
        (charCode * 123) % 255,
        (charCode * 456) % 255,
        (charCode * 789) % 255
    ];
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```

Brdige server: 

```js
const StrudelAdapter = require("./adapters/StrudelAdapter");
const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);
```

```js
if (DEVICE === "strudel") {
  return new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE });
  }
```

```js
adapter.onData = (d) => {
    if (d.type === "strudel") {
      broadcast(wss, d); // ya viene normalizado del adapter
      return;
    }
    // mensaje microbit (comportamiento original)
    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  }
```

Bridge client

```js
 if (msg.type === "microbit") {
    // sin cambios
    this._onData?.(msg);
    return;
    }

    if (msg.type === "strudel") {
    this._onData?.(msg);
    return;
    }
```
## Bitácora de reflexión

**Realiza un diagrama detallado del flujo de datos de tu sistema. Debe incluir al menos:**

Strudel;
Adapter de Strudel;
WebSocket de salida;
bridgeServer.js;
bridgeClient.js;
Cola de eventos;
FSM o capa de estado;
Render visual.

<img width="813" height="479" alt="image" src="https://github.com/user-attachments/assets/25a62696-c863-4bd6-915b-b1f705137b78" />

**Compara las unidades 4, 5 y 6 en una tabla. Compara al menos:**

## Unidad 4: 
**Fuente de datos**: micro-bit protocolo ASCII
**Formato del mensaje**: $T:tiempo|X:acel_x|Y:acel_y|A:estado_a|B:estado_b|CHK:checksum\n 
**Problema técnico principal**: leer los datos y enviarlos al nuevo arte del sketch
**Mecanismo de validación o control**:  si la suma de las variables no coincide con el valor CHK, la trama está corrupta y tu sistema debe descartarla silenciosamente sin actualizar la vista, pero deberías registrar un mensaje de advertencia en la consola indicando que se recibió una trama corrupta.
**Lugar donde ocurre la traducción del dato**:
**Papel del tiempo o sincronización**: al usar \n se pasaba de renglón y se interpretaba como un nuevo mensaje

## Unidad 5: 
**Fuente de datos**: micro-bit binario
**Formato del mensaje**: Paquete de bytes iniciando con AA -> ejemplo: AA 01 F4 02 0C 01 00 FE
**Problema técnico principal**: recibir los datos en binario e interpretarlos para entender el mensaje como movimiento o botones presioandos
**Mecanismo de validación o control**: AA al inicio y son 8 bytes por paquete entonces no recibia más que eso, además el cheksum debía ser igual como en la unidad 4
**Lugar donde ocurre la traducción del dato**:
**Papel del tiempo o sincronización**: Es importante usar el framing por el problema de sincronización, si no tenemos esto el bufer podría empezar a leer un mensaje a la mitad y de ahí en adelante el resto de mensajes leidos perderían su validez

## Unidad 6
**Fuente de datos**: Strudel
**Formato del mensaje**: 
{
  timestamp: 1710000000000,
  args: ["cps", 0.5, "cycle", 15.25, "s", "tr909bd", "delta", 0.25]
}
**Problema técnico principal**: Crear un servidor que conecte strudel con nuestras visuales e integrar un nuevo sketch con animaciones que corrieran al mismo tiempo que la musica en strudel
**Mecanismo de validación o control**: cuando llegan los mensajes de Strudel, se intenta convertir el string recibido a JSON usando JSON.parse. Si el formato es inválido, el mensaje se descarta inmediatamente.
**Lugar donde ocurre la traducción del dato**:
**Papel del tiempo o sincronización**: es importante el tiempo y la sincronizacion para que los sonidos y las animaciones se reproduzcan al mismo tiempo


**Explica por qué esta unidad sigue perteneciendo a la misma arquitectura del curso, aunque la fuente de datos ya no sea hardware físico.**

Aunque el dispositivo que envia los datos ya no es físico todavía es un problema de recibir e interpretar datos, se trata de crear un adaptador que pueda recibir e interpretar los datos y que para el resto del programa simplemente sea correrlo, por esto es posible integrarlo con las unidades anteriores

**Explica qué decisiones tomaste para traducir eventos musicales en visualidad. Justifica por qué tu mapeo visual tiene sentido.**
Para traducir los eventos musicales en visuales usé principalmente el tipo de sonido, la duración (delta) y el tiempo (timestamp). Asigné formas según cómo se percibe cada sonido: por ejemplo, el bombo es un círculo grande porque es fuerte y central, la caja es un rectángulo porque es más seca, y los hi-hats son formas pequeñas porque son rápidos y agudos.

También usé el delta para que las animaciones duren lo mismo que el sonido, haciendo que crezcan o desaparezcan con el tiempo, y el timestamp para que todo esté sincronizado con la música.

**Si tuvieras que integrar una tercera aplicación en el futuro, ¿Qué partes de tu arquitectura actual conservarías y cuáles cambiarías?**

Conservaría todos los adaptadores y sus llamados, el sketch podria ser modificado pero podría conservar su arquitectura original y agregar el tipo de dispositivo que alterará cierto evento, agregaría un nuevo adaptador para el nuevo dispositivo y simplemente lo importaría y añadiría lo que es necesario en el bridge server y en el bridge client de ser necesario. 

No modificaría el resto de la arquitectura pues con estos cambios debería ser suficiente para añadir una nueva aplicación
