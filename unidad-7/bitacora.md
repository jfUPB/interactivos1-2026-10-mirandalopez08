# Unidad 7

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

Creé un archivo en OSC con los respectivos sliders
<img width="370" height="806" alt="image" src="https://github.com/user-attachments/assets/c99071d6-6fd1-4cce-bd95-d5c879b709c4" />

Estas son sus configuraciones: 

RGB - COLOR
<img width="959" height="1018" alt="image" src="https://github.com/user-attachments/assets/503bb4d3-4bee-4cb7-85a3-4dfb6c6b6738" />

FADER - TAMAÑO
<img width="956" height="1008" alt="image" src="https://github.com/user-attachments/assets/609019c1-88a6-4476-ad9a-48fd2cf2ef10" />

FONDO - RASTROS DE COLOR
<img width="952" height="1031" alt="image" src="https://github.com/user-attachments/assets/b6f71f0b-b4af-4b81-976e-e81374337142" />

KNOB - VELOCIDAD
<img width="956" height="1017" alt="image" src="https://github.com/user-attachments/assets/1824374b-de14-476b-86b5-1a34b0e54af4" />

Los mensajes se reciben de esta forma: 
<img width="919" height="421" alt="image" src="https://github.com/user-attachments/assets/a465cf3e-225c-4d48-b9df-e53ae5c5fcb4" />


el `servidor` se abre con:  `node bridgeServer.js --device strudel --wsPort 8081 --strudelPort 8080 --oscPort 9000`

**OSC ADAPTER**
```js
// OpenStageControlAdapter.js
// Recibe mensajes OSC por UDP desde Open Stage Control,
// los normaliza al contrato del sistema y los entrega
// a bridgeServer mediante onData.
// No decide nada visual ni musical.

const { Server: OscServer } = require("node-osc");
const BaseAdapter = require("./BaseAdapter");

class OpenStageControlAdapter extends BaseAdapter {
  constructor({ port = 8082, verbose = false } = {}) {
    super();
    this.port    = port;
    this.verbose = verbose;
    this._server = null;
  }

  // Abre el servidor UDP/OSC donde Open Stage Control enviará mensajes.
  // Equivalente a abrir el WebSocketServer en StrudelAdapter.
  connect() {

    return new Promise((resolve) => {
      this._server = new OscServer(this.port, "0.0.0.0", () => {
        this.connected = true;
        this.onConnected?.(`OSC escuchando en udp://0.0.0.0:${this.port}`);
        resolve();
      });

      // Cada mensaje OSC que llega pasa por _onMessage.
      /*this._server.on("message", (msg) => {
        // node-osc entrega [address, arg1, arg2, ...]
        this._onMessage(msg);
      });*/

      this._server.on("message", (msg, rinfo) => {
    console.log("[OSC RAW]", msg); // temporal para debug
    this._onMessage(msg);
});

      this._server.on("error", (err) => {
        this.onError?.(err.message);
        resolve(); // no bloquear el arranque del bridge
      });
    });
  }

  disconnect() {
    return new Promise((resolve) => {
      if (!this._server) { resolve(); return; }
      this._server.close(() => {
        this.connected = false;
        this._server = null;
        this.onDisconnected?.("OSC server cerrado");
        resolve();
      });
    });
  }

  getConnectionDetail() {
    return `osc udp://0.0.0.0:${this.port}`;
  }

  handleCommand(_cmd) {}

  // Recibe el array crudo de node-osc y lo normaliza.
  _onMessage(msg) {
    const normalized = this._normalize(msg);
    if (!normalized) return;

    if (this.verbose) console.log("[OSCAdapter]", normalized);
    this.onData?.(normalized);
  }

  // node-osc entrega: ["/address", arg1, arg2, ...]
  // Contrato de salida estable que bridgeServer reenvía sin transformar.
  _normalize(msg) {
    const address = msg[0];
    const args    = msg.slice(1);

    // Descarta mensajes sin address válida.
    if (typeof address !== "string" || !address.startsWith("/")) return null;

    return {
      type: "osc",
      payload: {
        address,
        args,
      },
    };
  }
}

module.exports = OpenStageControlAdapter;
```

**BRIDGE CLIENT**

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
    
    if (msg.type === "osc") {
    this._onData?.(msg);
    return;
    }
    };
```

**BRIDGE SERVER**

```JS
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");
const OSC_PORT = parseInt(getArg("oscPort", "8082"), 10);
```

```JS
async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }
  if (DEVICE === "microbit2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit 2 found at ${path}`);
    return new MicrobitAscii2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }
  if (DEVICE === "microbit-bin") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    return new MicrobitBinaryAdapter({ path, baud: BAUD });
  }
  if (DEVICE === "strudel") {
  return new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE });
  }

  return new SimAdapter({ hz: SIM_HZ }); // fallback obligatorio
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected    = (detail) => { log.info(`[ADAPTER] Device Connected: ${detail}`); status(wss, "connected", detail); };
  adapter.onDisconnected = (detail) => { log.warn(`[ADAPTER] Device Disconnected: ${detail}`); status(wss, "disconnected", detail); };
  adapter.onError        = (detail) => { log.error(`[ADAPTER] Device Error: ${detail}`); status(wss, "error", detail); };

  adapter.onData = (d) => {
    if (d.type === "strudel") { broadcast(wss, d); return; }
    if (d.type === "osc")     { broadcast(wss, d); return; }
    broadcast(wss, { type: "microbit", x: d.x, y: d.y, btnA: !!d.btnA, btnB: !!d.btnB, t: nowMs() });
  };

  // OSC siempre corre en paralelo — instancia única
  const oscAdapter = new OpenStageControlAdapter({ port: OSC_PORT, verbose: VERBOSE });
  oscAdapter.onData         = (d) => broadcast(wss, d);
  oscAdapter.onConnected    = (d) => log.info(`[OSC] ${d}`);
  oscAdapter.onDisconnected = (d) => log.warn(`[OSC] ${d}`);
  oscAdapter.onError        = (d) => log.error(`[OSC] ${d}`);
  await oscAdapter.connect();

  status(wss, "ready", `bridge up (${DEVICE})`);
```

## COMPLETO: 
```JS

//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitAscii2Adapter = require("./adapters/MicrobitASCII2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);
// Ya tienes esto arriba — está bien:
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");
const OSC_PORT = parseInt(getArg("oscPort", "8082"), 10);

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};


function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }
  if (DEVICE === "microbit2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit 2 found at ${path}`);
    return new MicrobitAscii2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }
  if (DEVICE === "microbit-bin") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    return new MicrobitBinaryAdapter({ path, baud: BAUD });
  }
  if (DEVICE === "strudel") {
  return new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE });
  }

  return new SimAdapter({ hz: SIM_HZ }); // fallback obligatorio
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected    = (detail) => { log.info(`[ADAPTER] Device Connected: ${detail}`); status(wss, "connected", detail); };
  adapter.onDisconnected = (detail) => { log.warn(`[ADAPTER] Device Disconnected: ${detail}`); status(wss, "disconnected", detail); };
  adapter.onError        = (detail) => { log.error(`[ADAPTER] Device Error: ${detail}`); status(wss, "error", detail); };

  adapter.onData = (d) => {
    if (d.type === "strudel") { broadcast(wss, d); return; }
    if (d.type === "osc")     { broadcast(wss, d); return; }
    broadcast(wss, { type: "microbit", x: d.x, y: d.y, btnA: !!d.btnA, btnB: !!d.btnB, t: nowMs() });
  };

  // OSC siempre corre en paralelo — instancia única
  const oscAdapter = new OpenStageControlAdapter({ port: OSC_PORT, verbose: VERBOSE });
  oscAdapter.onData         = (d) => broadcast(wss, d);
  oscAdapter.onConnected    = (d) => log.info(`[OSC] ${d}`);
  oscAdapter.onDisconnected = (d) => log.warn(`[OSC] ${d}`);
  oscAdapter.onError        = (d) => log.error(`[OSC] ${d}`);
  await oscAdapter.connect();

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state  = adapter.connected ? "connected" : "ready";
    const detail = adapter.connected ? adapter.getConnectionDetail() : `bridge (${DEVICE})`;
    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);
        if (adapter.connected) {
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        try { await adapter.connect(); }
        catch (e) { status(wss, "error", `connect failed: ${e.message || e}`); }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);
        if (wss.clients.size > 1) {
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        try { await adapter.disconnect(); }
        catch (e) { status(wss, "error", `disconnect failed: ${e.message || e}`); }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try { await adapter.handleCommand?.(msg); }
        catch (e) { status(wss, "error", `command failed: ${e.message || e}`); }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) adapter.disconnect();
    });
  });

  if (DEVICE === "sim") await adapter.connect();
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```

**SKETCH**

```JS
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

        this.eventQueue = [];
        this.activeAnimations = [];
        this.mode = null;

        // ── estado OSC — NUEVO ─────────────────────────────
        // Variables persistentes que los controles de OSC modifican.
        // No se resetean entre frames: son estado continuo, no eventos.

        // Control 1: color base RGB — /rgb_1 [r, g, b] rango 0-255
        // Sobreescribe el color por defecto de getColorForSound()
        // cuando oscColorOverride está activo.
        this.oscColor         = [220, 60, 50];
        this.oscColorOverride = false; // empieza desactivado: cada sonido mantiene su color

        // Control 2: escala de tamaño global — /size [0-1]
        // Multiplica el tamaño de todas las formas.
        this.oscSize = 0.5;

        // Control 3: capa de fondo — /bg_layer [0 o 1]
        // Si está activo, el fondo deja un trail del color OSC
        // en vez del negro neutro.
        this.oscBgLayer = false;

        this.oscSpeed = 1.0; // velocidad normal por defecto

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

                // NUEVO: si oscColorOverride está activo, usa el color OSC.
                // Si no, mantiene el color original por tipo de sonido.
                const col = this.oscColorOverride
                    ? this.oscColor
                    : getColorForSound(d.s);

                // NUEVO: oscSize escala la duración visual de la animación.
                // Un size mayor hace las formas más grandes y duraderas.
                const scaledDelta = (d.delta * map(this.oscSize, 0, 1, 0.3, 2.0)) / this.oscSpeed;
                this.activeAnimations.push({
                    startTime: ev.timestamp,
                    duration:  scaledDelta * 1000,
                    type:      d.s,
                    x:         random(width * 0.2, width * 0.8),
                    y:         random(height * 0.2, height * 0.8),
                    color:     col,
                    size:      this.oscSize, // guardamos el size en la animación
                });

                this.eventQueue.shift();
            } else break;
        }
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            background(0);
            console.log("Connected. Waiting for data...");
        } else if (ev.type === EVENTS.DISCONNECT) {
            this.mode = null;
            this.transitionTo(this.estado_esperando);
        } else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        } else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        } else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        } else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        if (data.type === "strudel") {
            this.mode = "strudel";
            this.eventQueue.push(data);
            return;
        }

        // ── caso OSC — NUEVO ───────────────────────────────
        // Actualiza variables persistentes. No encola nada.
        // El efecto es inmediato y continuo entre frames.
        if (data.type === "osc") {
            this._applyOscControl(data.payload);
            return;
        }
    }

    // NUEVO: traduce cada address OSC a una variable de estado.
    // updateLogic delega aquí para mantener el routing limpio.
    _applyOscControl({ address, args }) {
        if (address === "/color") 
        {
            this.oscColor = [
            Math.round(args[0]),
            Math.round(args[1]),
            Math.round(args[2])
            ];
            this.oscColorOverride = true;
            return;
        }

        if (address === "/size") {
            this.oscSize = constrain(args[0] ?? 0.5, 0, 1);
            return;
        }

        if (address === "/bg_layer") {
            this.oscBgLayer = args[0] === 1;
            return;
        }

        if (address === "/speed") {
            this.oscSpeed = constrain(args[0] ?? 1.0, 0.1, 2.0);
            return;
        }
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
    bridge = new BridgeClient("ws://127.0.0.1:8081");

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => console.log("BRIDGE STATUS:", s.state, s.detail ?? ""));

    bridge.onData((data) => {
        if (data.type === "strudel") {
            painter.postEvent({ type: EVENTS.DATA, payload: data });
            return;
        }
        if (data.type === "osc") {
            painter.postEvent({ type: EVENTS.DATA, payload: data });
            return;
        }
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
    if (painter.mode === "strudel") {

        // NUEVO: si oscBgLayer está activo, el trail usa el color OSC.
        // Si no, usa el negro neutro original.
        if (painter.oscBgLayer) {
            const [r, g, b] = painter.oscColor;
            background(r * 0.1, g * 0.1, b * 0.1, 30); // trail teñido
        } else {
            background(0, 30); // comportamiento original
        }

        const now = Date.now();

        for (let i = painter.activeAnimations.length - 1; i >= 0; i--) {
            const anim = painter.activeAnimations[i];
            const elapsed = now - anim.startTime;
            const p = elapsed / anim.duration;

            if (p <= 1) {
                dibujarElemento(anim, p);
            } else {
                painter.activeAnimations.splice(i, 1);
            }
        }

        return;
    }
}

// ── funciones de dibujo — sin cambios ─────────────────────
// drawRunning no interpreta mensajes: solo pasa anim a estas funciones.
// El color y el size ya fueron resueltos en _flushQueue.

function dibujarElemento(anim, p) {
    push();
    switch (anim.type) {
        case 'tr909bd': dibujarBombo(anim, p, anim.color); break;
        case 'tr909sd': dibujarCaja(anim, p, anim.color);  break;
        case 'tr909hh':
        case 'tr909oh': dibujarHat(anim, p, anim.color);   break;
        default:        dibujarDefault(anim, p, anim.color); break;
    }
    pop();
}

function dibujarBombo(anim, p, c) {
    // NUEVO: el tamaño máximo escala con anim.size (resuelto en _flushQueue)
    const maxD = map(anim.size, 0, 1, 200, 800);
    let d     = lerp(100, maxD, p);
    let alpha = lerp(255, 0, p);
    noStroke();
    fill(c[0], c[1], c[2], alpha);
    circle(width / 2, height / 2, d);
}

function dibujarCaja(anim, p, c) {
    // NUEVO: el ancho máximo escala con anim.size
    const maxW = map(anim.size, 0, 1, width * 0.3, width);
    let w     = lerp(maxW, 0, p);
    let alpha = lerp(255, 0, p);
    noStroke();
    fill(c[0], c[1], c[2], alpha);
    rectMode(CENTER);
    rect(width / 2, height / 2, w, 50);
}

function dibujarHat(anim, p, c) {
    // NUEVO: el tamaño escala con anim.size
    const maxSz = map(anim.size, 0, 1, 20, 80);
    let sz = lerp(maxSz, 0, p);
    noStroke();
    fill(c[0], c[1], c[2]);
    rectMode(CENTER);
    rect(anim.x, anim.y, sz, sz);
}

function dibujarDefault(anim, p, c) {
    // NUEVO: el tamaño escala con anim.size
    const maxSz = map(anim.size, 0, 1, 50, 200);
    let size  = lerp(maxSz, 0, p);
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
## Bitácora de reflexión
**1. Realiza un diagrama detallado del flujo de datos completo de tu sistema. Debe incluir al menos:**

*Strudel;
Open Stage Control;
Adapter de Strudel;
Adapter de Open Stage Control;
bridgeServer.js;
bridgeClient.js o cliente equivalente;
Cola de eventos;
Estado persistente de controles;
Render visual.*

<img width="795" height="667" alt="image" src="https://github.com/user-attachments/assets/57e63e43-9cd3-40ef-8f0e-daacbc16e095" />


**2.Compara en una tabla las fuentes de datos que has trabajado en las unidades 4, 5, 6 y 7. Compara al menos:**

*Fuente de datos;
Formato del mensaje;
Tipo de dato que produce;
Problema técnico principal;
Lugar donde ocurre la traducción del dato;
Papel del tiempo;
Relación con el estado del sistema.*

| Unidad | Fuente de datos    | Formato                | Tipo de dato           | Problema técnico                | Traducción      | Tiempo               | Estado         |
| ------ | ------------------ | ---------------------- | ---------------------- | ------------------------------- | --------------- | -------------------- | -------------- |
| 4      | micro:bit         | ASCII                  | Botones x, y, ac       | Sicronización de datos          | SimAdapter      | Tiempo real          | Activo cuando llegan mensajes |
| 5      | micro:bit         | binario                | Botones x, y, ac       | Parseo de los datos             | MicrobitAdapter | Tiempo real          | Activo cuando llegan mensajes        |
| 6      | Strudel           | Sonidos strudel        | Eventos musicales      | Sincronización temporal precisa | StrudelAdapter  | Timestamp            | Activo cuando hay música en strudel |
| 7      | Open Stage Control | OSC                   | Controles continuos    | Funcionamiento de dos servidores| OSC Adapter     | No depende de tiempo | Persistente    |


**3.Explica por qué Open Stage Control no debe tratarse igual que Strudel dentro de la arquitectura.**

Debido a que el servidor de OSC debe mantenerse abiert constantemente y el de strudel aunque este abierto solo funciona cuando llegan datos si ambos funcionaran de la misma manera se producirían leves restrasos en la ejecución del sistema pues primero tendria que entrar el código a leer los datos de strudel y luego volver al mismo lugar por los datos de osc y no permitiría la sincronización de ambos servidores

**4.Justifica los tres controles que elegiste:**

*Qué parámetro modifican;
Por qué ese parámetro es relevante;
Cómo se refleja en el comportamiento visual.
Si tuvieras que integrar una tercera fuente de control en el futuro, ¿Qué partes de tu arquitectura actual conservarías y cuáles extenderías?*

modifiqué 3 parámetros, color que impacta en la identidad visual de el proyecto y lo hace más o menos llamativo dependiendo del color escogido, el tamaño que hace que las figuras destaquen más o menos y el fondo que deja un leve rastro de color dependiendo del color que fue escogido en el primer slider, por ultimo añadí uno de tiempo pero no está sincronizado con la música

Conservaría la mayoría de la arquitectura y adaptaría el bridge server a un nuevo dispositivo al igual que el sketch, permitiendo que pueda conectarse otro dispositivo
