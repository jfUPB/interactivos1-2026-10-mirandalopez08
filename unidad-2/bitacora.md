# Unidad 2

## Bitácora de proceso de aprendizaje

## Actividad 01

¿Cuáles son los estados en el programa?
¿Cuáles son los eventos en el programa?
¿Cuáles son las acciones en el programa?

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

        self.event_queue = [] # donde se colocan los eventos que tiene que prosesar la maquina de estado
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

#### Construye la máquina de estados que modela el problema usando PlantUML.

## Bitácora de aplicación 



## Bitácora de reflexión
