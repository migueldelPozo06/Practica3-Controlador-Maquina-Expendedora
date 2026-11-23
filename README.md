# Práctica 3: Controlador Máquina Expendedora
## 1. Introducción
La practica 3: Máquina expendedora automática con Arduino UNO, consiste en programar con Arduino IDE un software, que pueda ejecutarse en un arduino UNO, ademas se ha tenido que realizar un montaje con los componentes necesarios. La maquina expendedora que se ha pedido implementar, debe cargar, detectar al cliente, mostrar información ambiental, permitir la selección del producto deseado y proporcionarlo, ademas de un modo admin que permite acceder a los sensores y un sub-menu para modificar los precios.

### 1.1. Funcionamiento del sistema:

[Video modo servicio](https://youtu.be/LYOKA6V9D0s)

[Video modo administrador y reboot](https://youtu.be/f2h-99fnkrk)

## 2. Componentes hardware utilizados
Se han utilizado los siguientes componentes hardware:
- Pantalla lcd: ha sido utilizada para mostrar toda la información de la maquina expendedora.
- Un joystick para poder moverse entre los menús y seleccionar el producto deseado por el usuario.

- Un sensor de ultrasonido que detecta al cliente.

- Un sensor de temperatura DHT para obtener la información medioambiental.

- Dos leds para indicar estados y un botón para poder acceder al modo admin y reiniciar el sistema.

- Un arduino UNO.

Fotos del montaje del circuito
<img width="1052" height="799" alt="imagen" src="https://github.com/user-attachments/assets/a0fb7b38-5010-4704-9f50-65123bf4acd6" />

<img width="1052" height="799" alt="imagen" src="https://github.com/user-attachments/assets/4b2c29bb-a956-4eb5-bf6a-27bd3cc82340" />


## 3. Diseño y montaje del circuito
Para montar el circuito se ha utilizado una protoboard en la que se han puesto los componentes que se necesita tener fijos, como el joystick(para evitar falsas lecturas) y la pantalla(debido al numero de pines que necesita para funcionar), el botón y los leds, en cambio el sensor de ultrasonidos y el sensor DHT se han podido conectar con cables sin la protoboard ya que son sensores menos sensibles a movimientos.

Diseño del circuito.
<img width="1166" height="695" alt="imagen" src="https://github.com/user-attachments/assets/39515cac-224c-46f0-a522-f027d5c48500" />

## 4. Arquitectura y funcionamiento del software
### 4.1. Organización general del programa
Para organizar el programa se ha dividido en tres partes:
* States: Los archivos States.hpp y States.cpp encapsulan las clases que corresponden a cada estado, para que todas las clases tengan la misma estructura se ha creado una clase abstracta llamada state de la que heredan todos los estados, ademas de implementar la función isFinished necesaria para determinar si el estado ha terminado o no.
* Threads: Los archivos Threads.hpp y Threads.cpp, encapsulan las clases que corresponden a todos los eventos que tienen que ocurrir cada cierto tiempo, estos eventos son, la lectura del sensor DHT, la lectura del sensor de distancia, el parpadeo del led 1 en el estado de carga del sistema y el sistema de lectura del botón para acceder al modo administrador o reiniciar el sistema.
* Device: Los archivos Device.hpp y Device.cpp, encapsulan las clases que corresponden a todos los sensores y actuadores que no requieren una lectura o escritura cada cierto tiempo, sino que su sistema de lectura y escritura permite que el arduino pueda leer y escribir lo mas rápido posible, estos sensores y actuadores son, una clase que encapsula el funcionamiento de los leds y una clase que encapsula el funcionamiento del joystick, ademas también se ha encapsulado la clase que corresponde con un timer que usa timerOne en esta sección.
Por ultimo en el .ino esta definida la maquina de estados que orquesta todo el sistema, aparte del sistema que permite reiniciar el sistema y entrar en el modo admin.

Para que los estados puedan transitar, se ha definido una variable finish, que dentro de cada estado se pone a true o false, dependiendo de si se desea terminar o no el estado, así se puede conseguir una mayor abstracción en la maquina de estados y definir una estructura que funciona para todos los estados.


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

La ultima función implementada para este modo es la posibilidad de reiniciar el sistema, para ello la clase buttonThread, se ejecuta con una frecuencia de 100 ms, esta clase tiene un método para obtener el tiempo que el botón lleva pulsado, para reiniciar el sistema es necesario pulsar el botón durante un tiempo comprendido en el intervalo de 2s a 3s, en el apartado 4.5. se explica mejor el mecanismo implementado. 

### 4.4. Modo administración
El modo administración sirve para obtener las lecturas del sensor DHT, distancia, obtener el tiempo desde que el programa comenzó y cambiar el precio de los productos. Para acceder al modo administrador es necesario mantener el botón externo pulsado durante mas de 5 segundos, para ello un método de la clase buttonThread, devuelve el tiempo que el botón ha estado pulsado. Una vez se entra en el modo administrador una maquina de estados establece un estado para mostrar el menú de opciones, desde ese menú el usuario accede a la opción deseada, si quiere volver para atrás, se comprueba el valor del eje x del joystick y si disminuye de cierto valor se establece la maquina de estados en el estado de mostrar opciones, los estados de mostrar la temperatura y humedad, contador de tiempo y distancia son prácticamente iguales en cuanto a código, el estado con un código muy diferente es el de cambiar los precios, que usa el mismo código que el estado de seleccionar producto, pero al seleccionar un producto guarda su identificador y establece el modo de edición a true con una variable bool, después se establece el precio para ello se ha reutilizado el mismo código pero ahora no modifica el identificador sino el precio del producto, si se pulsa el botón integrado el modo de edición se desactiva y en la siguiente iteración la maquina de estados vuelve al estado anterior, si el joystick se mueve hacia la izquierda hace lo mismo pero ya no reasigna el nuevo valor al array que se ha pasado como puntero, simplemente vuelve al menú anterior, la variable que en realidad se modifica es una variable temporal que guarda el valor del producto en el momento en el que se ha decidido cambiar, así si se quieren descartar cambios es suficiente con no reasignar ese valor. Por ultimo en el menú principal si se vuelve a mover el joystick a la izquierda se establece la variable finish a true y se sale del modo administrador volviendo al flujo normal de ejecución.

### 4.5. Gestión de pulsaciones
Para gestionar las pulsaciones del botón se ha implementado la clase buttonThread, que cada 100 ms obtiene la lectura del botón, si en algún momento se detecta la pulsación se activa una interrupción, que activa una variable booleana volátil para evitar que se apliquen optimizaciones y el valor de esa variable no se lea de manera correcta, ademas para evitar los rebotes se ha implementado un mecanismo que cuenta ticks cada vez que el thread del buttonThread ejecuta, cuando se cuentan 3 ticks que corresponde con unos 300 ms a 400 ms, se vuelve a leer el estado del botón, si sigue en HIGH se puede estimar que el usuario quiere mantener el botón pulsado, si esta e bajo quiere decir que el botón no se esta manteniendo y por tanto se sigue con la ejecución normal del programa.
En el caso de la pulsación del joystick se ha utilizado un mecanismo de conteo con millis mas sencillo, ya que el botón del joystick ha presentado menos rebotes que el botón externo.

## 5. Técnicas de programación utilizadas
Para la organización del código de esta maquina de estados se ha utilizado la programación orientada a objetos y herencia, los sensores y actuadores están encapsulados en clases que se gestionan de diferentes maneras, dependiendo de si necesitan ejecutar cada cierto tiempo (heredar de thread) o ejecutarse lo mas rápido posible, ademas se ha integrado un reloj que utiliza timerOne para calcular el tiempo que el arduino lleva encendido, en una clase independiente. Para organizar todos los estados se ha creado una clase abstracta State que tiene cuatro métodos, tres virtuales para obligar a todas las clases que hereden de este método a implementarlos y un método isFinished y setFinished que cada clase activa y desactiva dependiendo de si se necesita transaccionar o no a otro estado. Para que las clases puedan comunicarse con los sensores y con los actuadores del sistema, se proporcionan a las clases los punteros a las clases que gestionan cada elemento hardware dependiendo de si se necesita o no en ese estado un hardware u otro.  
Otra técnica de programación usada es utilizar interrupciones para comprobar si el botón se ha pulsado o no, modificando una variable dentro de la función que se ejecuta en la interrupción, las variables que se modifiquen en una interrupción es necesario que sean volátiles, para que no se apliquen optimizaciones y el microcontrolador lea constantemente el valor de esa variable en memoria.
Para la ejecución de tareas periódicas se ha utilizado la librería de arduino threads, que sirve para organizar el código y ejecutar tareas de manera periódica cada cierto intervalo, es importante saber que esta librería no proporciona un sistema de ejecución paralela real, ya que en el microcontrolador atmega del arduino carecemos de un sistema operativo que gestione los recursos y varios núcleos que permitan ejecutar mas de un flujo de ejecución a la vez, pero si que es muy útil para organizar el código y manejar de manera cómoda las tareas que se tienen que ejecutar cada cierto tiempo pero no de manera estricta sino cuando se llegue al controlador en el bucle principal del código.

Ademas, para calcular el tiempo que el sistema lleva encendido se ha utilizado timerOne, esta decisión se ha tomado para que el sistema de conteo del tiempo no dependa de millis() y de timer0 teniendo un sistema de conteo independiente de la ejecución principal ya que las interrupciones generadas por timerOne no dependen del numero de tareas que se estén ejecutando en ese momento, es importante si se utiliza timerOne no usar los pines 9 y 10 para PWM ya que pueden producirse comportamientos no deseados.

Por ultimo se ha implementado un mecanismo de watchdog, que para este sistema se reinicia cada vez que el código dentro de loop se ejecuta por completo, si en algún momento hubiera algún bloqueo de alguno de los estados, después de 8 segundos el watchdog reiniciaría el sistema, solo hay un reset ya que no debería bloquearse ninguna función run() de ningún estado, reiniciándose el watchdog en cada iteración.

## 6. Problemas encontrados y soluciones
A lo largo del desarrollo de la practica se han encontrado varios problemas que se han ido solucionando a lo largo del desarrollo de la practica:

* Un problema que se ha tenido con el sensor DHT es que en algunas lecturas el sensor devolvía Nan, para solucionarlo lo que se ha hecho es leer el sensor en intervalos aproximados de 1,2 segundos en vez del intervalo inicial de 300 ms, esto producía que en la primera lectura del sensor no se pudiera leer y devolviera un error.

* Otro problema también ha sido con el sensor DHT, que estaba mal conectado debido a que las conexiones no están impresas en la placa y no ha sido posible encontrar el modelo exacto de sensor que se tiene en este kit, se ha solucionado cambiando las conexiones hasta que ha dejado de calentarse y funcionar perfectamente.

* Otro problema ha sido leer el joystick, ya que analogRead devuelve el valor que tenga el joystick en ese momento y en el momento de navegar en los menús, se pasaba de una opción a otra demasiado rápido, para ello lo que se ha hecho es comprobar donde esta el joystick para saber si esta o no en el centro del joystick y así solo cambiar de opción si se ha movido el joystick y ha vuelto a la posición central.

* Por ultimo, un problema que ha sido difícil de detectar es que no se completaban algunos estados pero solo en las ejecuciones siguientes a la primer ejecución, para resolverlo se ha tenido que establecer en todos los onExit de cada estado la variable finish a false para evitar que en la siguiente iteración el código contenido en el if que se activa con finish igual a true no se volviera a ejecutar.

## 7. Conclusiones
En conclusión en esta practica se ha implementado una maquina expendedora utilizando la plataforma arduino, se han implementado varias técnicas de organización y desarrollo del código como las librerías arduino threads y timerOne, ademas se han utilizado varios componentes hardware que han requerido implementar el código necesario para ser controladas manera adecuada, combinando estos componentes mediante software para conseguir programar una maquina expendedora funcional.
