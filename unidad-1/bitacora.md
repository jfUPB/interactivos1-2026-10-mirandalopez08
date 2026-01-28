# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 01

Texto de prueba: Este es mi primer texto de la actividad 1. Estoy continuando con la actividad 1

#### ¿Qué es un sistema físico interactivo?
Un sistema fisico interactivo es un producto con el cual los usuarios están envueltos y deben participar en la experiencia paa que tenga sentido

#### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Por medio de los sistemas fisicos interactivos es posible ampliar las posibilidades en una gran variedad de ámbitos, en cuanto al entretenimiento digital es posible crear experiencias envolventes para los usuarios, además es posible integrarlo en muchos campos por lo que es muy positivo tener la capacidad de hacerlo, esto permite traer innovacion a tareas que se han hecho de una manera tradicional y convertirlas en una experiencia novedosa

### Actividad 02

#### ¿Qué es el diseño/arte generativo?
Es una forma de arte en la que a partir de las interacciones del usuario una maquina puede generar una respuesta artistica relacionada con esta interaccion de manera que se convierta en una experiencia que envuelva al usuario y sea completamente personalizada 

#### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Como profesional seria posible crear expewriencias envolventes e innovadoras a partir del arte generatrivo usando la programacion e incluso la inteligencia artificial para permitir que los usuarios interactuen en tiempo real y obtengan los resultadfos inmediatamente

### Actividad 04

#### ¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.

## Bitácora de aplicación 

### Actividad 05

Programa *p5.js*
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
  // recibo datos del micro bit
  if (port.availableBytes() > 0) {
    let dataRx = port.read(1);

    if (dataRx == "A") {
      x -= 10;
      fill("pink");
    } else if (dataRx == "B") {
      x += 10;
      fill("purple");
    }
  }
  //establece x como la variable de posicion en el circulo
  background(220);
  ellipse(x, height / 2, 100, 100);

  // mostrar el boton de conectar al microbit
  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  }
  // mostrar el boton de desconectar del microbit
  else {
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

```
Programa de *micro:bit*
```py
from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
    if accelerometer.was_gesture('shake'):
        uart.write('C')
        sleep(500)
    if uart.any():
        data = uart.read(1)
        if data:
            if data[0] == ord('h'):
                display.show(Image.HEART)
                sleep(500)
                display.show(Image.HAPPY)
```

#### Explica detalladamente cómo funciona el sistema físico interactivo que has creado.
Antes de desarrollar el código es necesario preparar el del micro:bit 

Para comenzar en el codigo *p5.js* se declaran las variables globales que serán utilizadas a lo largo del código, una de las más importantes será x pues será utilizado más adelanter para definir la posición del circulo, después se creará un canvas al que se le asignará ancho, alturta y un color de fondo y posteriormente se crea un botón para conectarse al *micro:bit*. Después crearemos un circulo que estará ubicado en la parte central del canvas y lo rellenareos con el color blanco.
```js
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
```
Luego comenzamos a crear las condiciones para el movimiento del círculo, debemos establecer que para que esto ocurra se debe recibir una señal del *micro:bit* y comenzamos a crear los casos, el caso *A* será que si presionamos el botón *A* en el *micro:bit* nuestro circulo se moverá hacia la izquierda en el canvas, esto se logra utilizando *x--* como un contador que moverá la posición inicial x hacia la izquierda, en mi caso decidí utilizar *x-=10* debido a que el movimiento era muy leve en *x--*, por el contrario si presionamos el botón *B* el circulo se moverá hacia la derecha en el canvas. esto se logra utilizando la misma logica pero en en vez de usar *x--* usaremos *x++*, mi decision personal fue establecer un color para cuando se mueve a la izquierda y otro para la derecha utilizando la función fill 
```js
function draw() {
  // recibo datos del micro bit
  if (port.availableBytes() > 0) {
    let dataRx = port.read(1);

    if (dataRx == "A") {
      x -= 10;
      fill("pink");
    } else if (dataRx == "B") {
      x += 10;
      fill("purple");
    }
```
Después de esto es necesario establecer que la posición inicial del circulo será dada por x para que los contadores que usamos previamente funcionen correctamente
```js
 background(220);
  ellipse(x, height / 2, 100, 100);
```
Finalmente estableceremos la visualizacion de el botón *Connect to micro:bit* y *Disconnect* y también le daremeos sus funciones correspondientes, es importante saber que 115200 es la velocidad de transmision de datos y este numero es comunmente utilizado
```js
 // mostrar el boton de conectar al microbit
  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  }
  // mostrar el boton de desconectar del microbit
  else {
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
```
Por otra parte está el código del micro:bit, para empezar, al igual que en el anterior es necesario establecer la conexion con el micro:bit, inmediabtermente después se proyectará una mariposa. Después comienzan los condicionales, si se opirime el botón *A* se enviará información a la pantalla, en este caso haciendo caso a las indicaciones del código anterior el círculo se moverá a la izquierda y si se oprime el botón B también se enviará información de moverlo a la derecha

``` py
while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
```
El resto del código fue usado en una actividad anterior pero en esta no va a tener participacion pues solo necesitamos que el circulo se mueva a la derecha e izquierda

## Bitácora de reflexión








