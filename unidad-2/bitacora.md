# Unidad 2

## Bitácora de proceso de aprendizaje

## Actividad 01

#### ¿Cuáles son los estados en el programa?

estado_waitTimeout (es el único estado del Pixel).

#### ¿Cuáles son los eventos en el programa?
- ENTRY
- EXIT
- Timeout (generado por el temporizador cuando se cumple el tiempo)

#### ¿Cuáles son las acciones en el programa?
- Encender el pixel (pixelState = 9 y display.set_pixel(...)).
- Apagar el pixel (pixelState = 0 y display.set_pixel(...)).
- Iniciar el temporizador (myTimer.start()).
- Cambiar de estado (transicion_a(...)).
- Publicar eventos (post_event(...)).

## Actividad 02

Cuando se está realizando un código y estamos comenzando a poner clases o estados que aun no funcionan podemos usar la palabra *pass* para que no afecte el código
<img width="949" height="144" alt="image" src="https://github.com/user-attachments/assets/2f928987-5f98-4c04-9887-2a5fc5b598e4" />

Es posible crear un dioagrama UML que explica visualmente el código realizado, el diagrama se vería de la siguiente forma: 
<img width="473" height="465" alt="image" src="https://github.com/user-attachments/assets/a3f122f7-1229-416c-b8c7-b763d718510b" />

Vas a realizar una modificación. Cuando el semáforo esté en verde, si se presiona el botón A, el semáforo debe cambiar inmediatamente a amarillo (sin esperar a que termine el tiempo de verde). El evento que se debe postear es “A” (post_event(“A”)). Añadimos en clase una opcion para cuando se sacuda pase a modo noche

``` Py
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

class semaforo: 
    def __init__(self, _x,_y):
        self.redX = _x
        self.redY = _y

        self.event_queue = [] # donde se colocan los eventos que tiene que procesar la maquina de estado
        self.timers = [] #lista de temporizadores

        self.myTimer = self.createTimer("Timeout", 2000)
        self.estado_actual = None
        self.transicion_a(self.estado_WaitRed)
        
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

    def estado_WaitRed(self,ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY,9)
            self.myTimer.start(2000)
        if ev == "Timeout":
            display.set_pixel(self.redX,self.redY, 0)
            self.transicion_a(self.estado_WaitGreen)
        if ev == "S":
            display.set_pixel(self.redX,self.redY, 0)
            self.myTimer.stop()
            self.transicion_a(self.estado_WaitNightOn)
            
    def estado_WaitGreen(self,ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+2,9)
            self.myTimer.start(1000)
        if ev == "Timeout":
            display.set_pixel(self.redX,self.redY+2, 0)
            self.transicion_a(self.estado_WaitYellow)
        if ev == "A":
            self.myTimer.stop()
            display.set_pixel(self.redX,self.redY+2, 0)
            self.transicion_a(self.estado_WaitYellow)
            
    def estado_WaitYellow(self,ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+1,9)
            self.myTimer.start(1000)
        if ev == "Timeout":
            display.set_pixel(self.redX,self.redY+1, 0)
            self.transicion_a(self.estado_WaitRed)
            
    def estado_WaitNightOn(self,ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+1,9)
            self.myTimer.start(1000)
        if ev == "Timeout":
            self.transicion_a(self.estado_WaitNightOff)
            
    def estado_WaitNightOff(self,ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+1,0)
            self.myTimer.start(1000)
        if ev == "Timeout":
            self.transicion_a(self.estado_WaitNightOn)
                     
semaforo1 = semaforo(0,0)

while True:

    if button_a.was_pressed():
        semaforo1.post_event("A")

    if accelerometer.was_gesture("shake"):
        semaforo1.post_event("S")
        
    semaforo1.update()
    utime.sleep_ms(20) #pueden ser 17 si lo estamos corriendo a 40 fps
```

## Bitácora de aplicación 

## Actividad 04

```py
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
            
        if ev == "A":
            if self.counter<25:
              self.counter = self.counter + 1
              display.show(FILL[self.counter])     
        
        if ev == "B":
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

#### Construye la máquina de estados que modela el problema usando PlantUML.

<img width="449" height="452" alt="image" src="https://github.com/user-attachments/assets/807776e3-cb57-4a45-b2d8-c522a83e665a" />

```
@startuml

[*] --> Config

Config : entry / 20 leds ON
Config : A (si es menor a 25 aumento el contador)
Config : B (si es mayor a 15 disminuyo el contador)

Config --> Timer : si sacudo (S)

Timer : entry / timer.start(1000)
Timer : cuando pasa 1 segundo -1 lo muestro
Timer : si todavía no llegó a 0 -> timer.start(1000)

Timer --> Fin : cuando llega a 0

Fin : entry /display.show(Image.SKULL) / music.play(music.NYAN)
Fin --> Config : si aprieto A

@enduml
```

## Bitácora de reflexión

## Actividad 05

#### Explica cómo resolviste el reto.
Inicialmente me sentía perdida con el reto pues no entendía como enviar información desde el teclado a el programa p5.js, por lo que decidí usar chat gpt para que me apoyara en la realización del código, le hice varias preguntas y usé de referencia códigos de la unidad 1 para p5.js, intenté que la solución fuera similar a las que yo entendía para no confundirme, tuve una conversación corrigiendo errores y cambiando parte del código hasta que finalmente logré un código que enviaba la informaci´ón pero tenía ciertos errores, alli le pedí ayuda al profesor y logramos corregirlos

#### Codigos 

micro:bit
``` py
from microbit import *
import utime
import music
uart.init(baudrate=115200)

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
            
        if ev == "A":
            if self.counter<25:
                self.counter = self.counter + 1
                display.show(FILL[self.counter])     
        
        if ev == "B":
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
                   
    def estado_timerFinish(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            music.play(music.NYAN)
            
        if ev == "A":
           self.transicion_a(self.estado_config)        
                

task = Task()
while True:
    if uart.any():
        data = uart.read(1)  # leer 1 byte
        if data:
            letra = chr(data[0])  # convertir a caracter
            if letra in ["A", "B", "S"]:
                task.post_event(letra)  # enviar al Task
    
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

p5.js

``` js
let port;
let connectBtn;
// declarar x como variable global para utilizarla en la posicion del circulo
let x = 200;

function setup() {
  //crear canvas
  createCanvas(400, 400);
  background(220);
  //crear boton para conectar al micro bit
  port = createSerial();
  connectBtn = createButton("Connect to micro:bit");
  connectBtn.position(width / 3, 300);
  connectBtn.mousePressed(connectBtnClick);

  //crear la elipse y rellenarla de blanco
  fill("white");
  ellipse(width / 2, height / 2, 100, 100);
}
//crear las situcaiones para los botones a y b

function draw() {
  background(220);
  // Dibujar círculo
  fill("white");
  ellipse(x, height / 2, 100, 100);

  // Mostrar estado del botón
  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  } else {
    connectBtn.html("Disconnect");
  }
}

function connectBtnClick() {
  //conectar al microbit
  if (!port.opened()) {
    port.open("MicroPython", 115200);
    //desconectar del microbit
  } else {
    port.close();
  }
}

function keyPressed() {
  if (port.opened()) {
    if (key === "a") {
      port.write("A"); // enviar evento A
    } else if (key === "b") {
      port.write("B"); // enviar evento B
    } else if (key === "s" || key === "S") {
      port.write("S"); // enviar evento S
    }
  }
}
```


