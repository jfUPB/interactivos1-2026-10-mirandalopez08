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

El código no funcionaba con *was_pressed()* porque esa función solo devuelve verdadero una sola vez en el instante exacto en que ocurre el clic, por lo que se enviaba la información solo en ese momento y, si el programa receptor no la leía justo entonces, el dato se perdía. En cambio, con *is_pressed()* el estado del botón se envía de forma continua mientras esté presionado, lo que garantiza que el otro programa siempre reciba la información y pueda detectarla correctamente, haciendo la comunicación mucho más confiable.

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
En el código del micro:bit se inicia importando la biblioteca con la info del microbit y se establece la linea para la comunicación con el mismo, posteriormente se establece un condicional, si el botón *A* está siendo presionado en este momento entonces se enviará la letra *A* para copnfirmar, de no ser así se enviará la letra *N* para que el programa sepa que no está ocurriendo nada en este momento 
``` py
from microbit import * 

uart.init(baudrate=115200)  

while True:

    if button_a.is_pressed():
        uart.write('A')
    else:
        uart.write('N')

    sleep(100)
```
Después está el código *p5.j* que lo explicaré por partes, para empezar se inicializan las variables generales que serán utilizadas a lo largo del código, después se creará un canvas y se le asignará un color a el mismo, luego se creará un botón llamado *Connect to micro:bit* con el propósito de establecer la conexión con el mismo antes de comenzar, este botón funcionará cuando el mouse lo oprima
 ``` js
  let port;
  let connectBtn;
  let connectionInitialized = false;

  function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial(); // inicializa la conexión con el micro:bit
    connectBtn = createButton("Connect to micro:bit");
    connectBtn.position(80, 300); // posicion del botón
    connectBtn.mousePressed(connectBtnClick); // iniciar la acción de conectar después de que el mosuse presione el botón
  }
```
Luego se crea un espacio para limpiar el liemzo constantemente, además realiza la limpieza del puerto
```js
  function draw() {
    background(220); // exister para refrescar el fondo y que cada que haya un cambio no queden rastros del dibujo anterior

    if (port.opened() && !connectionInitialized) {
      port.clear();
      connectionInitialized = true;
    }
```

Para continuar comienza con la lectura de los datos enviados desde la micro:bit, si recibe una *A* va a pintar el cuadrado de rojo, si recibe una *N* que en este caso significa que nada está siendo presionado va a pintarlo de verde 
```js
    if (port.availableBytes() > 0) {
      let dataRx = port.read(1);
      if (dataRx == "A") {
        fill("red");
      } else if (dataRx == "N") {
        fill("green");
      }
    }
```
Después va a dibujar un cuadrado que va a estar ubicado en el centro de el lienzo, este es el que va a ser pintado según la información que el código reciba. Además va a revisar la conexión con el micro:bit si no está conectado entonces va a mostrar *Connect to micro:bit* por otro lado si sí se encuentra conectado va a mostrar la opción *Disconnect* y finalmente si no hay conexión va a abrir el puerto para establecer la conexión con el, pero si ya se encuentra conectado entonces va a cerrar el puerto y desconectarlo del micro:bit, esta parte final del código está diseñada para el botón que se creó en el principio que permite conectar y desconectar. 
```js
    rectMode(CENTER);
    rect(width / 2, height / 2, 50, 50);

    if (!port.opened()) {
      connectBtn.html("Connect to micro:bit");
    } else {
      connectBtn.html("Disconnect");
    }
  }

  function connectBtnClick() {
    if (!port.opened()) {
      port.open("MicroPython", 115200);
      connectionInitialized = false;
    } else {
      port.close();
    }
  }
```
Es necesario establecer la opción de que el *micro:bit* no está enviando ningún tipo de información con el propósito de que el sistema no se confuna cuando no esté recibiendo ninguna información








