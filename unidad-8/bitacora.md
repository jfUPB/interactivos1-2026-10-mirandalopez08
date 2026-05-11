# Unidad 8

## Bitácora de proceso de aprendizaje

**Cómo entra cada fuente al sistema**

El micro:bit se conecta por cable USB y envía datos por puerto serial — coordenadas x, y del acelerómetro y el estado de los dos botones. Strudel corre en el navegador y envía eventos musicales por WebSocket al puerto 8080 cada vez que hay una nota. Open Stage Control corre como aplicación de escritorio y envía mensajes OSC por UDP al puerto 9000 cada vez que mueves un control.


**Qué hace cada Adapter**

Los tres adapters hacen lo mismo: reciben el dato crudo de su fuente, lo traducen a un formato estable que el resto del sistema entiende, y lo entregan al bridge mediante el callback `onData`. Ninguno decide nada visual ni musical.

`MicrobitAsciiAdapter` lee líneas CSV del serial y las convierte dependiendo del mensaje . `StrudelAdapter` abre un servidor WebSocket, recibe el mensaje de Strudel y convierte el array plano de args en un objeto con `type: "strudel"`, `timestamp` y `payload`. `OpenStageControlAdapter` abre un servidor UDP, recibe el mensaje OSC y lo normaliza a `{type: "osc", payload: {address, args}}`.


**Qué papel cumple `bridgeServer.js`**

Es el punto central de transporte. Registra los tres adapters, les asigna callbacks y reenvía lo que recibe a todos los clientes conectados por WebSocket en el puerto 8081. No interpreta el contenido de los mensajes ni toma decisiones visuales. Su única responsabilidad es mover datos de un lado al otro.


**Cómo se conectan `bridgeClient.js`, `FSMTask`, `updateLogic` y `drawRunning`**

Es una cadena de responsabilidades donde cada capa hace una sola cosa.

`bridgeClient.js` recibe el mensaje del bridge, mira `msg.type` y llama `_onData`. No hace nada más.

En el sketch, `bridge.onData` recibe el mensaje y lo convierte en un evento para la FSM mediante `postEvent`. Aquí se distingue si es `strudel`, `osc` o `microbit`.

`FSMTask` organiza el sistema mediante estados. Cuando está en `estado_corriendo` y llega un evento `DATA`, delega a `updateLogic`.

`updateLogic` es el único lugar que toma decisiones sobre qué hacer con cada tipo de dato. Los eventos de Strudel se encolan con su timestamp. Los mensajes OSC actualizan variables persistentes. Los datos del micro:bit actualizan `rxData` y detectan flancos de botón.

`drawRunning` solo lee el estado ya calculado y dibuja. No sabe de dónde vienen los datos.


**Qué rol cumple cada fuente dentro de la obra**

Strudel es el motor temporal de la obra — genera los eventos musicales que disparan las animaciones. Es la fuente más importante porque define el ritmo.

Open Stage Control es el control paramétrico en tiempo real — permite cambiar el color, el tamaño y el efecto de fondo mientras la música suena, sin detener nada.

El micro:bit es la interacción física del intérprete — el acelerómetro mueve la posición de los elementos en pantalla y los botones ciclan entre diferentes formas visuales para el bombo y el hi-hat.


**Por qué tu sistema conserva la arquitectura del curso**

Porque cada extensión se hizo siguiendo el mismo patrón que ya existía, sin modificar lo que no era necesario tocar.

Los nuevos adapters heredan la misma interfaz que `MicrobitAsciiAdapter` — tienen `connect`, `disconnect`, `onData`, `onConnected`, `onError`. `bridgeServer.js` los registra de la misma forma que registraba el microbit. `bridgeClient.js` solo sumó dos `if` nuevos para los tipos `strudel` y `osc`. La `FSMTask` no se modificó — se extendió desde `PainterTask` con `super.update()`. `updateLogic` creció con nuevos casos pero sin romper los existentes. `drawRunning` nunca recibió lógica de red ni de timing.

La separación entre transporte, scheduling y render se mantuvo en todo momento.

## Bitácora de aplicación 

El flujo del sistema será el siguiente 
<img width="732" height="612" alt="image" src="https://github.com/user-attachments/assets/460af146-0fdd-404b-8bf0-4e6de11768c5" />

### Descripción general

La obra integra tres fuentes de datos en tiempo real para producir una experiencia audiovisual interactiva. Strudel genera los eventos musicales temporizados, Open Stage Control provee parámetros de control persistentes, y el micro:bit aporta interacción física directa del intérprete.

### Arquitectura del sistema

El sistema sigue una separación estricta de responsabilidades en cuatro capas:

**Capa de transporte** — cada fuente tiene su propio adapter que recibe el dato crudo y lo normaliza a un contrato estable antes de entregarlo al bridge. El `StrudelAdapter` escucha en `ws://localhost:8080`, el `OpenStageControlAdapter` en `udp://localhost:9000`, y el `MicrobitAdapter` por puerto serial. Los tres corren simultáneamente en `bridgeServer.js`.

**Capa de despacho** — `bridgeClient.js` recibe todos los mensajes por `ws://localhost:8081`, inspecciona `msg.type` y dispara el evento correspondiente hacia la FSM. No interpreta contenido.

**Capa de estado** — `updateLogic` organiza tres flujos distintos: los eventos de Strudel se encolan con su `timestamp` para activarse en el momento correcto; los mensajes OSC actualizan variables persistentes (`oscColor`, `oscSize`, `oscBgLayer`); los datos del micro:bit actualizan `rxData` y detectan flancos de botón para ciclar entre modos visuales.

**Capa de render** — `drawRunning` solo lee estado ya calculado. No toma decisiones sobre el origen de los datos.

### Controles implementados

| Fuente | Control | Efecto visual |
|--------|---------|---------------|
| micro:bit | Botón A | Cicla la forma del bombo: círculo → estrella → partículas |
| micro:bit | Botón B | Cicla la forma del hi-hat: línea → zigzag → cruz |
| micro:bit | Acelerómetro | Desplaza la posición de bombo y hi-hat en pantalla |
| OSC | `/color` (RGB) | Color de todas las animaciones |
| OSC | `/size` (slider) | Escala el tamaño de las formas |
| OSC | `/bg_layer` (toggle) | Activa trail de fondo teñido con el color OSC |
| Strudel | `bd sd hh oh` | Dispara animaciones temporizadas por `timestamp` |

### Decisión de diseño clave

La separación entre **transporte** y **scheduling** fue la decisión más importante. Los eventos de Strudel llegan antes de su momento de ejecución — el sistema los encola y los activa cuando `Date.now()` alcanza su `timestamp`. Esto desacopla la latencia de red del timing visual, mejorando la sincronización audiovisual.

### Comando de ejecución

Para el micro bit ascii adapter:
```bash
node bridgeServer.js --device microbit --wsPort 8081 --strudelPort 8080 --oscPort 9000
```

para el microbit ascii adapter 2: 
```bash
node bridgeServer.js --device microbit2 --wsPort 8081 --strudelPort 8080 --oscPort 9000
```

para el microbit binary adapter: 
```bash
node bridgeServer.js --device microbit-bin --wsPort 8081 --strudelPort 8080 --oscPort 9000
```

para el simulador: 
```bash
node bridgeServer.js --device sim --wsPort 8081 --strudelPort 8080 --oscPort 9000
```

Los cambios realizados en el código fueron muy pequeños puesto que en unidades anteriores habiamos trabajado teniendo en cuenta la integración del micro bit

En la funcion `createAdapter()` se eliminó la opción de strudel: 

```cs
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
  return new SimAdapter({ hz: SIM_HZ }); // fallback obligatorio
}
```
Y en el main se agregó para que el servidor de Strudel corriera en paralelo al igual que el de osc en la unidad anterior: 

```cs
 const strudelAdapter = new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE });
  strudelAdapter.onData         = (d) => broadcast(wss, d);
  strudelAdapter.onConnected    = (d) => log.info(`[STRUDEL] ${d}`);
  strudelAdapter.onDisconnected = (d) => log.warn(`[STRUDEL] ${d}`);
  strudelAdapter.onError        = (d) => log.error(`[STRUDEL] ${d}`);
  await strudelAdapter.connect();
```

Desde unidades anteriores ya estaba añadido en el bridge client en `Open()`

```cs
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
```

Los adapters no sufrieron ninguna modificación debido a que ya funcionaban correctamente. El sketch por otra parte debido a nuestras decisiónes visuales si sufrió modificaciones: 

```cs
// ── funciones de dibujo — sin cambios ─────────────────────
// drawRunning no interpreta mensajes: solo pasa anim a estas funciones.
// El color y el size ya fueron resueltos en _flushQueue.

function dibujarElemento(anim, p) {
    push();
    switch (anim.type) {
        case 'tr909bd': dibujarBombo(anim, p, anim.color); break;
        case 'tr909sd':
        case 'tr909cp': dibujarCaja(anim, p, anim.color);  break;
        case 'tr909hh':
        case 'tr909oh': dibujarHat(anim, p, anim.color);   break;
        default:        dibujarDefault(anim, p, anim.color); break;
    }
    pop();
}

function dibujarBombo(anim, p, c) {
    const maxD = map(anim.size, 0, 1, 200, 800);
    const alpha = lerp(255, 0, p);

    // Posición desplazada por el acelerómetro del microbit
    const px = map(painter.rxData.x, -2048, 2047, width  * 0.2, width  * 0.8);
    const py = map(painter.rxData.y, -2048, 2047, height * 0.2, height * 0.8);

    noStroke();
    fill(c[0], c[1], c[2], alpha);

    if (painter.modoBombo === 0) {
        // Círculo original
        const d = lerp(100, maxD, p);
        circle(px, py, d);

    } else if (painter.modoBombo === 1) {
        // Estrella
        const r1 = lerp(maxD * 0.5, 10, p);
        const r2 = r1 * 0.4;
        const puntas = 6;
        translate(px, py);
        beginShape();
        for (let i = 0; i < puntas * 2; i++) {
            const r     = i % 2 === 0 ? r1 : r2;
            const angle = (PI / puntas) * i - HALF_PI;
            vertex(cos(angle) * r, sin(angle) * r);
        }
        endShape(CLOSE);

    } else {
        // Partículas
        const n   = 12;
        const rad = lerp(maxD * 0.5, 0, p);
        for (let i = 0; i < n; i++) {
            const angle = (TWO_PI / n) * i;
            const x = px + cos(angle) * rad;
            const y = py + sin(angle) * rad;
            circle(x, y, lerp(20, 2, p));
        }
    }
}

function dibujarCaja(anim, p, c) {
    const maxW = map(anim.size, 0, 1, width * 0.3, width);
    const w     = lerp(maxW, 0, p);
    const alpha = lerp(255, 0, p);
    noStroke();
    fill(c[0], c[1], c[2], alpha);
    rectMode(CENTER);
    rect(width / 2, height / 2, w, 50);
}

function dibujarHat(anim, p, c) {
    const maxSz = map(anim.size, 0, 1, 20, 80);

    // Posición desplazada por el acelerómetro
    const px = map(painter.rxData.x, -2048, 2047, width  * 0.1, width  * 0.9);
    const py = map(painter.rxData.y, -2048, 2047, height * 0.1, height * 0.9);

    noStroke();
    fill(c[0], c[1], c[2]);

    if (painter.modoLinea === 0) {
        // Línea original
        stroke(c[0], c[1], c[2]);
        noFill();
        strokeWeight(2);
        const half = lerp(maxSz * 2, 0, p);
        line(px - half, py, px + half, py);

    } else if (painter.modoLinea === 1) {
        // Zigzag
        stroke(c[0], c[1], c[2]);
        noFill();
        strokeWeight(2);
        const len  = lerp(maxSz * 3, 0, p);
        const segs = 6;
        const segW = len / segs;
        beginShape();
        for (let i = 0; i <= segs; i++) {
            const x = px - len / 2 + segW * i;
            const y = py + (i % 2 === 0 ? -maxSz * 0.5 : maxSz * 0.5) * (1 - p);
            vertex(x, y);
        }
        endShape();

    } else {
        // Cruz
        stroke(c[0], c[1], c[2]);
        noFill();
        strokeWeight(2);
        const sz = lerp(maxSz * 2, 0, p);
        line(px - sz, py,      px + sz, py);
        line(px,      py - sz, px,      py + sz);
    }
}

function dibujarDefault(anim, p, c) {
    const maxSz = map(anim.size, 0, 1, 50, 200);
    const size  = lerp(maxSz, 0, p);
    const angle = p * TWO_PI;

    translate(anim.x, anim.y);
    rotate(angle);
    stroke(c[0], c[1], c[2]);
    noFill();
    rect(0, 0, size, size);
    line(-size, 0, size, 0);
    line(0, -size, 0, size);
}
```

Además para probarlo temporalmente se agregaron otras funciones que no tienen influencia en el código: 

```cs
function keyPressed() {
    if (key === 'a' || key === 'A') {
        painter.postEvent({
            type: EVENTS.DATA,
            payload: { type: "microbit", x: 0, y: 0, btnA: true, btnB: false }
        });
    }
    if (key === 'b' || key === 'B') {
        painter.postEvent({
            type: EVENTS.DATA,
            payload: { type: "microbit", x: 0, y: 0, btnA: false, btnB: true }
        });
    }
}

function keyReleased() {
    painter.postEvent({
        type: EVENTS.DATA,
        payload: { type: "microbit", x: 0, y: 0, btnA: false, btnB: false }
    });
}
```

## Bitácora de reflexión
