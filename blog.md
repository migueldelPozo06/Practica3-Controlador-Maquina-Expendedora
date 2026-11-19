# Índice del blog: Máquina expendedora automática con Arduino UNO
## 1. Introducción
La practica 3: Máquina expendedora automática con Arduino UNO, consiste en programar con Arduino IDE un software, que pueda ejecutarse en un arduino UNO, ademas se ha tenido que realizar un montaje con los componentes necesarios. La maquina expendedora que se ha pedido implementar, debe cargar, detectar al cliente, mostrar información ambiental, permitir la selección del producto deseado y proporcionarlo, ademas de un modo admin que permite acceder a los sensores y un sub-menu para modificar los precios.

## 2. Componentes hardware utilizados
Se han utilizado los siguientes componentes hardware:
- Pantalla lcd: ha sido utilizada para mostrar toda la información de la maquina expendedora.
- Un joystick para poder moverse entre los menús y seleccionar el producto deseado por el usuario.
- Un sensor de ultrasonido que detecta al cliente.
- Un sensor de temperatura DHT para obtener la información medioambiental.
- Dos leds para indicar estados y un botón para poder acceder al modo admin y reiniciar el sistema.
- Un arduino UNO.

## 3. Diseño y montaje del circuito
Para montar el circuito se ha utilizado una protoboard en la que se han puesto los componentes que se necesita tener fijos, como el joystick(para evitar falsas lecturas) y la pantalla(debido al numero de pines que necesita para funcionar), el botón y los leds, en cambio el sensor de ultrasonidos y el sensor DHT se han podido conectar con cables sin la protoboard ya que son sensores menos sensibles a movimientos.

Diseño del circuito.


## 4. Arquitectura y funcionamiento del software
### 4.1. Organización general del programa
Para organizar el programa se ha dividido en tres partes:
* States: Los archivos States.hpp y States.cpp encapsulan las clases que corresponden a cada estado, para que todas las clases tengan la misma estructura se ha creado una clase abstracta llamada state de la que heredan todos los estados, ademas de implementar la función isFinished necesaria para determinar si el estado ha terminado o no.
* Threads: Los archivos Threads.hpp y Threads.cpp, encapsulan las clases que corresponden a todos los eventos que tienen que ocurrir cada cierto tiempo, estos eventos son, la lectura del sensor DHT, la lectura del sensor de distancia y el parpadeo del led 1 en el estado de carga del sistema.
* Device: Los archivos Device.hpp y Device.cpp, encapsulan las clases que corresponden a todos los sensores y actuadores que no requieren una lectura o escritura cada cierto tiempo, sino que su sistema de lectura y escritura permite que el arduino pueda leer y escribir lo mas rápido posible, estos sensores y actuadores son, una clase que encapsula el funcionamiento de los leds y una clase que encapsula el funcionamiento del joystick.
Por ultimo en el .ino esta definida la maquina de estados que orquesta todo el sistema, aparte del sistema que permite reiniciar el sistema y entrar en el modo admin.

Para que los estados puedan transitar, se ha definido una variable finished, que dentro de cada estado se pone a true o false, dependiendo de si se desea terminar o no el estado, así se puede conseguir una mayor abstracción en la maquina de estados y definir una estructura que funciona para todos los estados.


```cpp
  if(state->isFinished()) {
    state_id = next_state_id;
    state->onExit();
    next_state->onEnter();
  }
  state->run();
```

Siendo state un puntero cualquier clase que herede de state, state_id el id del struct StateId del estado a transaccionar y next_state un puntero del siguiente estado siendo cualquier clase que herede de state.

### 4.2. Modo de arranque
Para el modo de arranque el sistema implementado es activar el thread encargado de hacer parpadear el led, este thread se encarga de hacer parpadear el led tres veces y reiniciarse, en este caso solo es necesario imprimir el mensaje por pantalla y activar en onEnter el thread blinkThread y apagarlo en onExit.

### 4.3. Modo servicio
Para ejecutar el modo servicio se han definido varios sub-estados:
* Para el estado Service, se activa el thread que lee del sensor de ultrasonidos y se comprueba que la distancia que devuelve (en cm), sea menor que la referencia de 1 metro, si es mayor lo que se hace es imprimir en la pantalla un mensaje de esperando cliente, en este caso se ha definido una variable booleana para que solo se dibuje una vez ya que al ocupar mas de 16 caracteres es necesario desplazar el texto, para ello se ha usado la siguiente función de lcd:

```cpp
lcd->scrollDisplayLeft();
```

Esta función se ha obtenido de la documentación de LiquidCrystal en la pagina de arduino, esta función lo que hace es desplazar un texto en una pantalla.

* Después de detectar al usuario se transacciona al estado siguiente WeatherDisplayState, la finalidad de este estado es informar al usuario de la temperatura y humedad ambiental durante 5 segundos, para ello en el onEnter(), se  reinicia un temporizador para que imprima por pantalla la temperatura y humedad con el siguiente formato:

```text
Temp: XX ºC
Hum: YY %
```

Para obtener esta información se ha utilizado la librería DHT que permite usar el sensor DHT que proporciona la temperatura y humedad del ambiente, DHT también esta encapsulado en una clase thread para que las mediciones se realizan cada cierto tiempo, es interesante establecer una frecuencia para evitar problemas de lectura y dar tiempo al arduino a procesar los datos revividos.

* Después de que pasen los 5 segundos, se transiciona al estado en el cual el usuario elige el producto deseado, para almacenar los productos y sus precios, se han creado dos arrays del mismo tamaño que se imprimen a la vez, en la fila de arriba de la pantalla se imprime el producto a elegir y debajo el precio del producto, para elegir un producto se obtiene la pulsación en esa iteración y en la iteración anterior para saber que es una pulsación y no que se ha mantenido pulsado el botón del joystick. Como se puede observar en el código de la clase ProductSelectionState, y en las demás clases en las que se utiliza joystick no se tratan los rebotes del botón sino que se llama a una función, dentro de ese método que pertenece a la clase joystick se ha implementado un mecanismo para evitar rebotes, que consiste en no obtener una medición hasta un timeout de unos 50 ms. Si no se ha detectado ninguna pulsación el usuario puede cambiar de producto, para ello se puede mover el joystick, en este caso se aumenta o disminuye un contador dependiendo del valor que se capture. Si el valor del joystick en el eje y es mayor a 700 se considera que se quiere subir y se aumenta el contador y para el valor menor a 300 se resta, para no saturar el array e imprimir información no deseada se reinicia el contador dependiendo de si vale mas o menos que el numero de productos. Para no recorrer la lista a la velocidad a la frecuencia a la que funciona loop, se guarda en una variable booleana el estado y hasta que el joystick no este centrado esa variable no va a cambiar de estado y por tanto obliga al usuario a mover el joystick y volver a situarlo en el centro para que asi la selección sea mas sencilla.

* Por ultimo, después de que el usuario seleccione el producto deseado se va a pasar a imprimir en la pantalla un mensaje que informe al usuario que la bebida se esta preparando y para informar acerca del progreso se calcula un valor aleatorio en el cual el led tiene que aumentar su brillo a través de PWM desde apagado hasta el brillo máximo, para aumentar el brillo de manera progresiva se ha normalizado el valor que va a recibir analogWrite entre 0 y 255 para el tiempo aleatorio que el led tarda en encenderse por completo, para normalizar se ha usado la siguiente formula:

```cpp
led_value = int(((millis() - initial_time) * 255) / random_time);
```
Esta formula permite normalizar y calcular los valores que analogWrite va a recibir, es necesario hacer un cast ya que analogWrite utiliza valores enteros.
Una vez todo este proceso termine se muestra durante tres segundos un mensaje al usuario y se vuelve al estado de servicio en el que se vuelve a detectar al cliente.

### 4.4. Modo administración

### 4.5. Gestión de pulsaciones y cancelaciones

## 5. Técnicas de programación utilizadas

## 6. Interfaz de usuario

## 7. Pruebas y resultados

## 8. Problemas encontrados y soluciones

## 9. Conclusiones
