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



## Bitácora de reflexión

