# Unidad 3

## Bitácora de proceso de aprendizaje

## Actividad 01

Regresemos a la actividad del semáforo de la unidad anterior. Te pediré que hagas las siguientes modificaciones:

Si el botón A se presiona el semáforo debe cambiar a modo “peatonal”. En este modo el semáforo se pone en rojo, pero antes debe pasar por amarillo para darle tiempo a los autos a detenerse.

Si el botón B se presiona el semáforo pasa a modo nocturno. En este modo el semáforo parpadea en amarillo. Si el botón A se presiona en este modo, el semáforo vuelve a modo normal.

``` js
from microbit import *
import utime

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration

        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)


class Semaforo:
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        self.event_queue = []
        self.timers = []
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.createTimer("Timeout",self.timeInRed)

        self.estado_actual = None
        self.transicion_a(self.estado_waitInRed)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)

    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transicion_a(self.estado_waitInGreen)
        if ev == "B":
            self.transicion_a(self.estado_nocturno)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)

        if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transicion_a(self.estado_waitInYellow)

        if ev == "A":
            display.set_pixel(self.x,self.y+2,0)
            self.transicion_a(self.estado_waitInYellow)
            
        if ev == "B":
            self.transicion_a(self.estado_nocturno)

    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transicion_a(self.estado_waitInRed)
        
            
    def estado_nocturno(self,ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            pixelState = display.get_pixel(self.x,self.y+1)
            if pixelState == 9: display.set_pixel(self.x,self.y+1,0)
            else: display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "A":
            self.transicion_a(self.estado_waitInRed)
            
semaforo1 = Semaforo(0,0,2000,1000,500)

while True:
    if button_a.was_pressed():
        semaforo1.post_event("A")

    if button_b.was_pressed():
        semaforo1.post_event("B")
    semaforo1.update()
    utime.sleep_ms(20)
```

## Actividad 02

Vas a modificar el código del temporizador de la unidad anterior así:

Si está corriendo el temporizador, al presionar el botón A se debe pausar el conteo. Si se vuelve a presionar el botón A, el conteo se reanuda en donde se había pausado.

Si el temporizador está corriendo, si se presiona una secuencia de botones A-B-A, el temporizador debo volver a modo de configuración.

```js
from microbit import *
import utime
import music

def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()
# Para mostrar usas display.show(FILL[n]) donde n será
# un valor de 0 a 25

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

            
class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.password = ["A","B","A"]
        self.stopSequence = []
        # Personalizas el nombre del evento y la duración
        self.myTimer = self.createTimer("Timeout",1000)
        self.counter = 20
        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def estado_config(self, ev):
        if ev == "ENTRY":
           self.counter = 20
           display.show(FILL[self.counter])
            
        if ev == "B":
            if self.counter<25:
              self.counter = self.counter + 1
              display.show(FILL[self.counter])     
        
        if ev == "A":
           if self.counter>15:
              self.counter = self.counter - 1
              display.show(FILL[self.counter])    
           
        if ev == "S":
            self.transicion_a(self.estado_timerStart)

    def estado_timerStart(self, ev):
        if ev == "ENTRY":
            self.myTimer.start(1000)
            
        if ev == "Timeout":
            if self.counter > 0: 
               self.counter = self.counter - 1
               display.show(FILL[self.counter])
               if self.counter == 0:     
                   self.transicion_a(self.estado_timerFinish)
               else:
                    self.myTimer.start(1000)
        if ev == "S":
            self.transicion_a(self.estado_paused)
                              
        if ev == "A" or ev == "B":
           self.stopSequence.append(ev)
           if len(self.stopSequence) == 3:
               if self.stopSequence == self.password:
                   self.transicion_a(self.estado_config)
               else:
                   self.stopSequence.clear()
                
    def estado_paused(self,ev):
       if "ENTRY":
           self.myTimer.stop()
       if ev == "S":
           self.transicion_a(self.estado_timerStart)
       if ev == "A" or ev == "B":
           self.stopSequence.append(ev)
           if len(self.stopSequence) == 3:
               if self.stopSequence == self.password:
                   self.transicion_a(self.estado_config)
               else:
                   self.stopSequence.clear() 
                   
    def estado_timerFinish(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            music.play(music.NYAN)
            
        if ev == "A":
           self.transicion_a(self.estado_config)        
                
task = Task()
while True:
    # Aquí generas los eventos de los botones y el gesto
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)
```
## Bitácora de aplicación 
## Actividad 04

sketch.js
```js
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};

const UI = {
  dialSize: 250,
  ringWeight: 20,
  bigText: 100,
  configText: 120,
  helpText: 18,
};


class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;
    this.password = [EVENTS.DEC, EVENTS.INC, EVENTS.DEC];
    this.stopSequence = [];

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);

  }

  get currentState() {
    return this.state;
  }

  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
    }
    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };


  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start();
    } else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      }
    } else if (ev === EXIT) {
      this.myTimer.stop();
    }
      else if (ev === EVENTS.START) { // O el evento que decidas para pausar
      this.transitionTo(this.estado_paused);
    }
  };

  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  }

  estado_paused = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.stop();
    }
    
    // Si presionas Start (S), vuelve a armar el temporizador
    if (ev === EVENTS.START) {
      this.transitionTo(this.estado_armed);
    }

    // Lógica de la secuencia secreta (A o B)
    if (ev === EVENTS.DEC || ev === EVENTS.INC) {
      this.stopSequence.push(ev);

      if (this.stopSequence.length === 3) {
        // Comparamos la secuencia con el password ["A", "B", "A"]
        if (this.stopSequence.join() === this.password.join()) {
          this.stopSequence = []; // Limpiamos para la próxima vez
          this.transitionTo(this.estado_config);
        } else {
          // Si falló, borramos lo que llevaba para que intente de nuevo
          this.stopSequence = [];
        }
      }
    }
  };
}

let temporizador;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  port = createSerial();

  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
  textAlign(CENTER, CENTER);

  renderer.set(temporizador.estado_config, () => drawConfig(temporizador.configValue));
  renderer.set(temporizador.estado_armed, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds));
  renderer.set(temporizador.estado_timeout, () => drawTimeout());
  connectBtn = createButton('Connect to micro:bit');
  connectBtn.position(width/2-50, height-100);
  connectBtn.mousePressed(connectBtnClick);
}

function draw() {
  portAva();
  temporizador.update();
  renderer.get(temporizador.currentState)?.();
}


function drawConfig(val) {
  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);
  textSize(18);
  fill(200);
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total) {
  background(20, 20, 20);
  let pulse = sin(frameCount * 0.1) * 10;

  noFill();
  strokeWeight(20);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, 250);

  stroke(255, 150, 0);
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100 + pulse);
  text(val, width / 2, height / 2);
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent("A");
  if (key === "b" || key === "B") temporizador.postEvent("B");
  if (key === "s" || key === "S") temporizador.postEvent("S");
}

function portAva() {
    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
          temporizador.postEvent("A");
        }
        else if(dataRx == 'B'){
            temporizador.postEvent("B");
        }
        else if (dataRx == 'S'){
            temporizador.postEvent("S");
        }
      }
    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    }
    else {
        connectBtn.html('Disconnect');
    }

}
function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
  }

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}

```
index.html
añadir 
```html
<script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script>
```

micro:bit 
```py
from microbit import *

uart.init(baudrate=115200)
display.show(Image.HAPPY)

while True:
    if button_a.was_pressed():
        uart.write('A')
        sleep(500)
    if button_b.was_pressed():
        uart.write('B')
        sleep(500)
    if accelerometer.was_gesture('shake'):
        uart.write('S')
        sleep(500)
```

## Bitácora de reflexión

## Actividad 05 
Mi compañera Tatiana y yo realizamos el trabajo juntas, comenzamos probando como funcionaba enviando mensajes simples, pasamos texto y sonidos, luego programamos el recibir señales que funcionaran junto al temporizador y finalmente unimos los dos códigos para que sirviera desde ambos micro:bits

```py
from microbit import *
import radio
import music

radio.config(group=250)
uart.init(baudrate=115200)
display.show(Image.HAPPY)
radio.on()


while True:
    if button_a.was_pressed():
        uart.write('A')
    if button_b.was_pressed():
        uart.write('B')
    if accelerometer.was_gesture('shake'):
        uart.write('S')

    message = radio.receive()
    if message:
        if message == "A":
            uart.write('A')
        if message == "B":
            uart.write('B')
        if message == "S":
            uart.write('S')
```



