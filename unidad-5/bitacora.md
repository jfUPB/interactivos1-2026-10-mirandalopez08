# Unidad 5
## Bitácora de proceso de aprendizaje
Al utilizar ascii no tengo certeza de cuantos bites estoy usando, mientras que al usar binario puedo optimizar definiendo que cantidad de datos voy a recibir, por ejemplo, si en el código que usamos en la unidad para enviar datos desde el microbit lo modifico de la siguiente manera:
```py
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = 567
    yValue = 231
    aState = true
    bState = false
    data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
    uart.write(data)
    sleep(100)
```
los datos enviados a la aplicacion de consexion serial serán: 

02 37 00 e7 01 00

**02 37** = 567
**00 e7** = 231
**01** = 1 = true
**00** = 0 = false

*Los numeros enviados en binario tinene dos clasificaciones*

**BIG ENDIAN:** que es enviar primero el dato de mayor peso y luego el dato de menor peso, por ejemplo *02 37*

**LITTLE ENDIAN:** enviar primero el dato de menor peso y luego el de mayor peso por ejemplii *37 02* 

Es importante saber cuál es porque pueden ser dos números diferentes

Un ejemplo 

`05 F3 ¿Que numero te envié?`

Tienes dos opciones:

`Opción 1:` si me enviaron Little endian: 62213 (armo el numero como F305)

`Opción 2:` si me enviaron Big endian: 62213 (armo el numero como 05F3)

## ¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?
Es mucho más fácil controlar los datos que entran y los bites que ocupan, lo que significa más seguridad al momento de analizar los datos y la posibilidad de optimizar el sistema, por otra parte una de las desventajas más claras es que los datos son más dificiles de leer e interpretar por lo que es importante saber como manejarlos 

Si xValue=500, yValue=524, aState=True, bState=False, ¿cómo se vería el paquete en hexadecimal? (Pista: convierte cada valor según su tipo y anota los bytes en orden.) Respuesta esperada: 01 F4 02 0C 01 00

## Bitácora de aplicación 


## Bitácora de reflexión
