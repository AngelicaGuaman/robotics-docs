# Práctica 1 - Follow line

En esta práctica he desarrollado un **sistema de control PD** para permitir que un coche de F1 siga una línea roja en el suelo utilizando visión artificial.
La cámara del vehículo se encuentra ligeramente desplazada hacia la derecha, por lo que el sistema compesará ese desfase al calcular el error.
El objetivo principal es percibir la línea, estimar su trayectoria y ajustar la velocidad del vehículo en función del recorrido: aumentar la aceleración en tramos rectos y reducirla cuando se aproxima una curva para garantizar estabilidad y precisión.
Para abordar la solución de este problema, he dividido el sistema en los siguientes procesos:

- Segmentación de la línea
- Cálculo del error utilizando ROIs
- Detección de curvas mediante desviación lateral
- Construcción de la referencia de trayectoria
- Control PD

## 1. Segmentación de la línea
El primer paso consiste en **detectar la línea roja en la imagen de la cámara**.

- Se convierte la imagen de **RGB a HSV**, ya que este espacio de color facilita la segmentación por color.
- Se utilizan **dos rangos de rojo** para cubrir la discontinuidad del tono rojo en HSV.
- Se aplica una **operación morfológica** para eliminar huecos y ruido.

Resultado: una **máscara binaria estable** que representa la línea.


# 2. Cálculo del error utilizando ROIs

La imagen se divide en **tres regiones horizontales (ROIs)**:

- **Top ROI** → anticipa la trayectoria futura. (Franja azul)
- **Mid ROI** → representa la trayectoria inmediata. (Franja amarilla)
- **Bottom ROI** → representa la posición actual del coche respecto a la línea. (Franja verde)

Cada ROI permite calcular el **centroide del mayor contorno detectado** mediante:

- `cv2.findContours`
- `cv2.moments`

Esto permite obtener tres puntos clave:

- cx_top 
- cx_mid
- cx_bot

A continuación, se muestra la imagen segmentada y dividida en las 3 regiones que se utilizarán más adelante para el seguimiento de la línea.

![Segmentación de línea roja](recursos/imagen_divida_simple.png)

A pesar que se ha divido la imagen en 3 partes, según se iba mejorando el sistema, la sección de top no se usa porque el coche de F1 no percibe línea (no hay) en esa franja. Esto es buena señal a la hora de hacer el seguimiento; ya que no hay cambios muy bruscos llegando a perder la línea.

---

# 3. Detección de curvas mediante desviación lateral

Para determinar si el F1 se encuentra en una **recta o en una curva**, se calcula la **desviación lateral** entre los centroides detectados en diferentes regiones de la imagen.

En concreto, se mide la diferencia horizontal entre puntos de la línea en distintas ROIs. Una gran diferencia entre estos puntos indica que la línea está cambiando de dirección, lo que sugiere la presencia de una curva.

Para evitar cambios de estado bruscos entre recta y curva, se utiliza una variable global **`curve_mode`** que almacena el estado anterior del sistema.

El cambio de estado se controla mediante los siguientes umbrales:

- **`CURVE_ON_TH`** → umbral a partir del cual el sistema entra en **modo curva**.
- **`CURVE_OFF_TH`** → umbral por debajo del cual el sistema vuelve a **modo recta**.

Esto me ayudó a reducir las oscilaciones entre ambos estados, proporcionando un comportamiento más estable del controlador.

---

# 4. Construcción de la referencia de trayectoria

La posición objetivo (`ref`) se calcula mediante una **media ponderada de los centroides**.

Las ponderaciones utilizadas dependen del **estado de conducción del coche F1**, es decir, si el sistema detecta que el coche se encuentra en **recta** o en **curva**.

## En rectas

Cuando el F1 percibe una recta, se da mayor peso al **centroide de la región inferior (bottom)**.  
Esta región corresponde a la parte de la línea más cercana al coche y proporciona una estimación más estable de la posición actual respecto a la trayectoria.

Priorizar este punto permite:
- mejorar la estabilidad del control
- reducir pequeñas oscilaciones laterales
- mantener el vehículo centrado en la línea a altas velocidades

## En curvas
Cuando se detecta una curva, se incrementa el peso de las regiones **mid y top**, que representan puntos más adelantados de la línea.

Esto permite **anticipar la dirección futura de la trayectoria**, ayudando al controlador a reducir la velocidad e iniciar el giro antes de que la curva sea demasiado pronunciada.

Como consecuencia:
- el vehículo comienza a girar con mayor antelación
- se reduce el riesgo de perder la línea
- se mejora el comportamiento del coche al reducir la velocidad durante la curva

---

# 5. Control PD

Una vez calculado el error lateral respecto a la línea, utilizo un **controlador PD (Proporcional + Derivativo)**. 

El controlador utiliza dos componentes principales:

- **Término proporcional (Kp)**
- **Término derivativo (Kd)**

Para adaptar el comportamiento del coche a diferentes situaciones de conducción, se definen **dos parejas de ganancias**, uno para **rectas** y otro para **curvas**.

Además, el cálculo de la velocidad se realiza teniendo en cuenta la **velocidad anterior del vehículo** y el **estado anterior**, lo que permite evitar cambios bruscos de aceleración o frenado y conseguir transiciones más suaves durante el seguimiento del circuito.

## Rectas
En rectas se utiliza un control **más agresivo**, ya que el vehículo puede mantener velocidades más altas sin comprometer la estabilidad.

Parámetros utilizados:

- `Kp_straight`
- `Kd_straight`

Velocidad permitida:

- `MAX_SPEED`
- `MIN_SPEED`

## Curvas
En curvas se emplea un control **más suave**, reduciendo las ganancias del controlador para evitar oscilaciones bruscas mientras el coche gira.

Parámetros utilizados:

- `Kp_curve`
- `Kd_curve`

Velocidad permitida:

- `MAX_SPEED_CURVE`
- `MIN_SPEED_CURVE`


---

# Demo de la solución del problema 

## Circuito Simple

En la siguiente imagen se puede observar la segmentación de la línea con las ROIs elegidas.

![Segmentación de línea roja circuito simple](recursos/imagen_divida_simple.png)

A continuación se muestra un **video demostrativo** del funcionamiento del sistema:

[![Demo circuito simple](https://img.youtube.com/vi/BMsKdHyreCo/0.jpg)](https://youtu.be/BMsKdHyreCo)


## Circuito Montmelo

En la siguiente imagen se puede observar la segmentación de la línea con las ROIs elegidas.

![Segmentación de línea roja circuito Montmelo](recursos/imagen_divida_montmelo.png)

A continuación se muestra un **video ilustrativo** del funcionamiento del sistema:

[![Demo circuito Montmelo](https://img.youtube.com/vi/qJKz3H181Ec/0.jpg)](https://youtu.be/qJKz3H181Ec)


---

# Conclusiones

Durante el desarrollo de esta práctica tuve varios desafíos tanto a nivel técnico como experimental.

Uno de los principales problemas fue la **dificultad para medir con precisión el tiempo real que tarda el F1 en completar el circuito**. Debido a la ausencia de una GPU dedicada y a la dependencia de la conexión a internet para ejecutar el simulador, en ocasiones el rendimiento variaba. Esto provocaba que algunas ejecuciones fueran más lentas, lo que inicialmente me generaba confusión al evaluar el rendimiento del algoritmo, ya que parecía que ciertos cambios empeoraban el comportamiento del sistema cuando en realidad se trataba de variaciones en el tiempo de ejecución.

Otro aspecto importante fue el **diseño del mecanismo de recuperación de la línea**. Implementar una estrategia que permitiera al vehículo recuperar la trayectoria cuando la línea dejaba de detectarse resultó fundamental para mejorar la robustez del sistema.

Finalmente, el **ajuste de los parámetros del controlador** requirió un proceso iterativo basado en experimentación. Pequeñas variaciones en las ganancias del controlador (`Kp` y `Kd`) podían alterar significativamente el comportamiento del vehículo, llegando incluso a provocar que el sistema vaya muy lento o tenga más oscilaciones. Por esta razón, el proceso de ajuste se realizó mediante múltiples pruebas hasta encontrar un equilibrio adecuado entre estabilidad, capacidad de respuesta y velocidad de recorrido del circuito.
