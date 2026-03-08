---
layout: single
title: "Práctica 1: Follow Line"
mathjax: true
classes: wide
date: 2026-03-07
categories: vision-robotica
author_profile: true
sidebar:
  nav: ""
---

<style>
  .page__content {
    width: 100% !important;
    max-width: 1000px !important;
    margin: 0 auto !important;
    float: none !important;
  }
  .page__inner-wrap {
    margin-left: auto !important;
    margin-right: auto !important;
    width: 100% !important;
  }
</style>

## 1. Introducción

### a. Resumen
En este post detallo el proceso evolutivo que he seguido para resolver la práctica 1 **"Follow Line"** de la asignatura Visión Robótica. El documento está estructurado para guiar al lector desde los conceptos teóricos fundamentales sobre los bucles de control y algoritmos PID, pasando por el *pipeline* de procesamiento de imagen hasta llegar a la arquitectura final del software. 

Por último, analizaré la evolución de los resultados, donde logré optimizar el tiempo por vuelta de forma drástica tras replantear el algoritmo de visión, y expondré las conclusiones extraídas de este desafío.

### b. Enfoque
Mi filosofía de trabajo para este proyecto se basó en una premisa innegociable: **estabilidad antes que velocidad**. Durante las primeras pruebas comprobé empíricamente que intentar exprimir el cronómetro con un coche que sufría de oscilaciones severas o cabeceos era completamente contraproducente. Un sistema de control inestable no se puede optimizar; cualquier aumento de velocidad solo magnificaba el error y acababa estrellando al coche en el lateral de la pista.



Por ello, mi prioridad absoluta fue conseguir una trazada fluida. Mi desesperación por intentar reducir los cabeceos del coche sin éxito alguno me llevó a descartar los enfoques más básicos (como calcular el centro de masa de toda la línea, que provocaba adoptar valores de giro demasiado bruscos) e investigar alternativas para darle al controlador una mayor anticipación. 

> **Estrategia clave:** Adoptar una visión de "futuro" para calcular el error.

Como detallaré a lo largo de este blog, este cambio de estrategia no solo erradicó el cabeceo en las curvas, sino que me proporcionó un chasis digital tan estable que me permitió ser mucho más agresivo con las velocidades, logrando bajar mi marca personal hasta los **25 segundos** en el circuito simple.

## 2. Teoría y Fundamentos de Control

El seguimiento de la línea se basa en un **enfoque reactivo en bucle cerrado**. A diferencia de un sistema deliberativo (que sigue un plan rígido sin mirar el entorno), un sistema reactivo utiliza la información de la cámara para calcular constantemente la diferencia entre la posición actual y el estado ideal (centro de la imagen). 

| Controlador de bucle abierto | Controlador de bucle cerrado |
| :---: | :---: |
| ![Bucle abierto](/assets/images/Bucle_abierto.png) | ![Bucle cerrado](/assets/images/Bucle_cerrado.png) |
| *La actuación se decide a priori.* | *Compara continuamente el estado deseado y el actual del robot.* |

Este cálculo del error permite al robot corregir su trayectoria a una frecuencia de **50 Hz**, ajustando el volante no en función de un mapa previo, sino de lo que percibe en cada instante.



Para procesar este error, el estándar industrial es el controlador **PID** (proporcional, integral y derivativo):

* **Componente proporcional (P):** Aplica una fuerza de giro proporcional a la magnitud del error actual; es el "músculo" que lleva al coche hacia la línea. Sin embargo, un control puramente proporcional provoca que el vehículo se pase de largo y oscile.
* **Componente derivativo (D):** Analiza la velocidad de cambio del error. Actúa como un amortiguador que predice la llegada al centro y aplica un contravolante suave, eliminando los bandazos o "cabeceos".



En esta práctica, se optó por un controlador **PD**, prescindiendo de la parte integral ya que se consiguen resultados lo suficientemente buenos sin aplicar dicha componente.

## 3. Adquisición y tratamiento de las imágenes

Para que el robot pueda navegar, primero debe transformar los píxeles capturados por su cámara en información geométrica útil mediante un proceso de segmentación y filtrado.

### a. Filtro HSV
Tras capturar la imagen y enviarla a la interfaz WebGUI, el primer paso es aislar la línea roja del resto del entorno. Para ello, convertimos el espacio de color de BGR a **HSV** (hue, saturation, value).

Debido a que el color rojo se encuentra en los extremos del espectro circular del matiz (alrededor de los 0° y 360°), es necesario definir **dos rangos de color** para capturarlo por completo. Utilizando esta herramienta de [selector de color online](https://imagecolorpicker.online/es/), identifiqué que la línea abarca desde los 358° hasta los 2° más o menos.

![Cilindro de color HSV](/assets/images/Cilindro_hsv.jpg)

En la implementación con OpenCV, debemos ajustar estas escalas debido a la codificación de 8 bits (0-255): 
* El valor del **Matiz (H)** se divide por 2 para encajar en el rango 0-179.
* La **Saturación (S)** y el **Brillo (V)** se escalan multiplicando por 2.55 para cubrir el rango 0-255.

Finalmente, aplicamos estas máscaras para generar una **imagen binaria** donde los píxeles rojos se muestran en blanco y el resto en negro, facilitando así la extracción de las coordenadas de la línea.

| Imagen de cámara original | Máscara binaria (HSV) |
| :---: | :---: |
| ![Imagen en meta](/assets/images/imagen_meta.png) | ![Máscara roja](/assets/images/mascara_rojo.png) |
| *Captura de la cámara frontal en línea de meta.* | *Segmentación HSV que aísla la línea de referencia.* |

### b. Cálculo del error: del centro de masa al enfoque "Topmost"

Una vez segmentada la línea, el reto consiste en extraer un valor numérico para el error. En clase se plantearon dos estrategias principales: el cálculo del **centro de masa** (vía momentos de la imagen) o el **análisis de filas individuales**.

Inicialmente implementé el enfoque de momentos, pero pronto descubrí que el modelo era conceptualmente erróneo para este problema de control reactivo. Al promediar toda la mancha roja, el controlador se volvía extremadamente estricto: si en una curva la línea quedaba en un lateral, el sistema forzaba un giro excesivo para intentar centrar todo ese "bloque" de píxeles en el eje central de la imagen. Esto provocaba que el coche sobrepasara la línea, invirtiendo el error y entrando en un **ciclo infinito de correcciones bruscas**, lo que generaba el persistente cabeceo.



Tras comprobar que era imposible optimizar un modelo de error defectuoso, cambié al segundo enfoque sugerido en clase: el análisis de filas para buscar el punto más alejado o **"topmost"**. Esta idea, reforzada tras investigar el [blog de mi compañera Sandra Montejano](https://sanmcr.github.io/follow-line-fuera/), permite al robot ignorar la línea que ya tiene bajo las ruedas y centrarse exclusivamente en la dirección de la curva futura.

Para la implementación utilicé **NumPy**, localizando la coordenada "y" más pequeña de los píxeles blancos dentro de una **región de interés (ROI)**. Como mejora adicional, añadí un promedio de las coordenadas "x" en esa fila superior; esto garantiza que, en tramos rectos donde la línea tiene grosor, el coche apunte siempre al centro exacto del carril. 



Este cambio de paradigma erradicó el cabeceo por completo, permitiéndome alcanzar una trazada fluida y estable incluso a velocidades muy altas, bajando mi tiempo hasta los **25 segundos**.

### c. Análisis técnico: ¿Por qué el enfoque "Topmost" eliminó el cabeceo?

Tras analizar el comportamiento del vehículo, identifiqué que el éxito del punto *topmost* frente al centro de masa no fue una simple cuestión de ajuste de parámetros, sino un cambio en la **latencia espacial** del error:

1.  **Eliminación del "efecto ancla":** Al usar el centro de masa, los píxeles más cercanos al coche (que aún están en la recta) promedian con los lejanos (que ya están en la curva). Esto genera un error pequeño y retardado que obliga al coche a entrar en la curva con poco giro. Cuando el error finalmente aumenta, la inercia es tan grande que el coche sobrepasa la línea para compensar, iniciando el ciclo de oscilación.
2.  **Anticipación pura:** El *topmost* ignora el "pasado" (la línea bajo las ruedas) y se centra en el "futuro". Aunque este error es numéricamente mayor al estar más desplazado por la perspectiva, llega al controlador mucho antes. Esto permite que el coche trace la curva de forma proactiva, no reactiva.
3.  **Optimización de la componente Derivativa ($K_d$):** La $K_d$ actúa sobre la velocidad de cambio del error. Con el centro de masa, el error cambia de forma lenta por el promedio de toda la mancha roja. Con el *topmost*, el salto lateral del punto al inicio de la curva genera una derivada muy alta, permitiendo que la $K_d$ detecte la curva al instante y aplique el contravolante necesario para suavizar la entrada.

> **Conclusión:** El Centro de Masa da un error más "preciso" respecto a la posición actual, pero llega demasiado tarde. El *topmost* da un error "exagerado", pero en el momento exacto para vencer a la inercia.

## 4. Actuadores y lógica de control

Una vez calculado el error mediante el punto *topmost*, el siguiente paso es traducir esa información en órdenes de movimiento precisas para los motores del robot (velocidad lineal y velocidad angular).

### a. Implementación del controlador P (Ganancia)
La componente **proporcional** es la encargada de generar el giro del volante en función del error actual. Esta ganancia determina la agresividad con la que el coche busca recuperar el centro de la línea. 

Al ajustarla, busqué un equilibrio donde el coche tuviera el "nervio" suficiente para iniciar el giro de forma inmediata sin llegar a ser tan violento que desestabilizara la lectura del enfoque *topmost*. Una $K_p$ bien calibrada permite que el vehículo sea reactivo pero mantenga la suavidad necesaria para no perder la referencia visual.

### b. Implementación del controlador D (Derivativo)
La componente **derivativa** resultó ser la pieza clave para la estabilidad a altas velocidades. Su función es analizar la tasa de cambio del error para anticiparse a la inercia del vehículo. 



Un hallazgo fundamental durante el proceso de ajuste fue que, al subir notablemente el valor de $K_d$, conseguí que el coche empezara a girar mucho antes, justo en el instante en que el error comenzaba a crecer al inicio de una curva. Esta "amortiguación" matemática no solo permitió una trazada mucho más técnica y fluida, sino que eliminó los volantazos bruscos.

### c. Implementación del controlador de la velocidad ($K_v$)
Para optimizar el tiempo por vuelta, implementé una **velocidad adaptativa** mediante una ecuación que vincula la velocidad lineal con el valor absoluto del error detectado:

$$V = V_{max} - (|error| \cdot K_v)$$

El objetivo es que el coche ruede al límite en las rectas y frene de forma proporcional al entrar en las curvas. Para calibrar la constante $K_v$, seguí un método deductivo: definía la velocidad punta deseada para las rectas ($V_{max}$) y la velocidad de paso por curva ($V$), asumiendo un error de cálculo razonable en el vértice (aproximadamente 150 píxeles). 



Sustituyendo estos valores en la ecuación, pude despejar y ajustar la $K_v$ ideal. Este enfoque permite que el coche "lea" la dificultad de la curva a través del error del punto *topmost* y adapte su inercia de forma automática antes de girar.

## 5. Vacío de decisión

Un aspecto crítico en la robustez del software es la gestión del **vacío de decisión**, es decir, qué debe hacer el coche cuando la cámara pierde por completo la referencia de la línea roja. Generalmente, esto ocurre en curvas muy cerradas donde el coche no ha girado lo suficiente o con la antelación necesaria, quedando la línea fuera del campo de visión (ROI).



Para solucionar esto, implementé una lógica basada en la **memoria del último estado conocido**. Cuando el algoritmo detecta que no hay píxeles rojos en el ROI, reduce la velocidad drásticamente a $V_{min}$ y aplica un giro constante en la dirección donde se vio la línea por última vez. Si la última posición estaba a la derecha, el coche pivotará hacia ese lado hasta que la línea vuelva a entrar en su campo visual. Este enfoque evita que el vehículo siga recto por inercia y permite una recuperación autónoma en las situaciones en las que pierda la línea.

## 6. Resultados y evaluación de robustez

Para evaluar el rendimiento del algoritmo, se han realizado pruebas de validación en diferentes trazados. El objetivo era doble: conseguir el menor tiempo posible y garantizar que el controlador no estuviera sobreajustado (*overfitted*) a un único circuito.

### Resultados en el circuito simple
El comportamiento en el circuito base ha sido excepcional, demostrando la eficacia del enfoque *topmost* combinado con el control PD y la velocidad adaptativa. He documentado dos configuraciones distintas:

#### a. Configuración agresiva (Tiempo: ~25 segundos)
Con una $V_{max}$ alta, el coche traza el circuito al límite del agarre. El algoritmo anticipa perfectamente las curvas, frenando lo justo y necesario.

<div style="width: 100%; max-width: 800px; margin: 0 auto;">
  <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.15);">
    <iframe 
      src="https://www.youtube.com/embed/T0ddJ6jPpbQ" 
      style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
      frameborder="0" 
      allowfullscreen>
    </iframe>
  </div>
  <p align="center"><small><em>Vídeo demostrativo del vehículo completando el circuito simple en 26s.</em></small></p>
</div>

#### b. Configuración conservadora (Tiempo: ~1 minuto)
Reduciendo la $V_{max}$, el coche demuestra que el control es estable independientemente de la velocidad punta configurada.

<div style="width: 100%; max-width: 800px; margin: 0 auto;">
  <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.15);">
    <iframe 
      src="https://www.youtube.com/embed/MuQpxBm7BKo" 
      style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
      frameborder="0" 
      allowfullscreen>
    </iframe>
  </div>
  <p align="center"><small><em>Vídeo demostrativo del vehículo completando el circuito simple en 1 min.</em></small></p>
</div>

### Validación de robustez: Montmeló y Nürburgring
Para avalar la robustez de la solución , se probó el código en los circuitos de Montmeló y Nürburgring.

#### Montmeló

En el caso de Montmeló, el algoritmo demostró su adaptabilidad completando sin problemas y de forma muy fluida aproximadamente el **75% del trazado**. Sin embargo, en el último sector de este circuito y durante las pruebas en Nürburgring, me topo con limitaciones técnicas propias del entorno de simulación. Al alcanzar ciertas zonas del mapa, el motor de físicas del simulador experimenta anomalías en el cálculo de colisiones, provocando que el vehículo salga despedido por los aires y abandone el mapa de forma errática. Muestro el recorrido de mi coche en el fragmento inicial del circuito.

<div style="width: 100%; max-width: 800px; margin: 0 auto;">
  <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.15);">
    <iframe 
      src="https://www.youtube.com/embed/9HV75X7DLs0" 
      style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
      frameborder="0" 
      allowfullscreen>
    </iframe>
  </div>
  <p align="center"><small><em>Vídeo demostrativo del comportamiento del vehículo en el circuito de Montmeló.</em></small></p>
</div>

#### Nürburgring

Muestro ahora el fallo persistente en el circuito de Nürburgring, el cual me impide siquiera avanzar unos metros:

<div style="width: 100%; max-width: 800px; margin: 0 auto;">
  <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.15);">
    <iframe 
      src="https://www.youtube.com/embed/gauoVx4d1RQ" 
      style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
      frameborder="0" 
      allowfullscreen>
    </iframe>
  </div>
  <p align="center"><small><em>Vídeo demostrativo del error de físicas en el circuito de Nürburgring.</em></small></p>
</div>

Monitorizando la telemetría interna, se comprobó que los cálculos del error, la velocidad lineal ($V$) e angular ($W$) generados por mi controlador eran **numéricamente correctos** y coherentes con la trazada en el momento del fallo. Por tanto, se concluye que el rendimiento final queda limitado exclusivamente por la estabilidad física del simulador.

## 7. Limitaciones y áreas de mejora

Aunque el sistema desarrollado ha demostrado ser altamente competitivo, el enfoque actual presenta ciertas limitaciones de control que abren la puerta a futuras optimizaciones y versiones más avanzadas del software.

### a. El problema de la "chicane" (limitación del enfoque topmost)
La mayor vulnerabilidad del enfoque *topmost* reside en los trazados que se pliegan sobre sí mismos sin oclusiones visuales. Un ejemplo clásico sería una **chicane cerrada** (giros sucesivos de 90º izquierda, derecha, derecha, izquierda).



Como el algoritmo busca incansablemente el píxel rojo con la coordenada "Y" más pequeña (el más lejano en el horizonte), en una chicane plana el coche detectaría directamente la **recta de salida al fondo**, ignorando por completo las curvas intermedias y trazando una línea recta, recortando la curva y saliéndose inevitablemente de la pista al no respetar el trazado inmediato.

<figure align="center">
  <img src="{{ site.baseurl }}/assets/images/Chicane.jpeg" alt="Trayectoria en chicane" style="border-radius: 8px; max-width: 90%; box-shadow: 0 4px 8px rgba(0,0,0,0.1);">
  <figcaption>Esquema de la trayectoria (flecha negra) que seguiría el coche en una chicane cerrada.</figcaption>
</figure>

### b. ROI estática
Actualmente, el horizonte de visión del coche está delimitado por un recorte rígido en la imagen (`inicio_y_ROI = 250`). Aunque este ajuste funciona perfectamente en los circuitos planos del simulador, esta rigidez supondría un problema en entornos tridimensionales con **rasantes, cuestas o peraltes**. 

Si el coche se enfrenta a una pendiente pronunciada, el morro se levanta y la línea roja podría quedar por debajo del límite inferior de nuestro ROI estático, provocando un "falso vacío de decisión" a pesar de que la línea sigue estando presente en la imagen original de la cámara.

### c. Comportamiento de búsqueda por tiempo limitado
En el estado actual del código, cuando el algoritmo pierde la referencia visual (vacío de decisión), el vehículo entra en un bucle de búsqueda girando sobre su propio eje basándose en el `error_anterior`. Si bien esto permite recuperar la trazada en pérdidas puntuales, el coche mantendría este giro indefinidamente si la línea desaparece por completo.

Una optimización futura necesaria sería la implementación de un **temporizador de seguridad**. Si tras un intervalo de tiempo $X$ (por ejemplo, 2 segundos) el sensor de visión no vuelve a detectar píxeles rojos, el controlador debería reducir la velocidad lineal a cero de forma progresiva. Esto evitaría comportamientos erráticos o que el vehículo se aleje excesivamente de la pista en caso de un fallo crítico de segmentación o un cambio drástico en las condiciones del entorno.

## 8. Conclusiones y opinión personal

Esta práctica ha supuesto mi primer contacto real con la robótica de control y la visión artificial aplicada. Aunque anteriormente había trabajado en proyectos con microcontroladores como Arduino, la complejidad de gestionar un sistema dinámico en tiempo real a 50 Hz ha sido un desafío de una escala completamente diferente.

Tras finalizar el proyecto, me llevo una lección fundamental: **no se puede optimizar una base que es conceptualmente errónea**. Mi insistencia inicial en el enfoque de momentos (centro de masa) fue el mayor lastre del desarrollo. Al toparme con el mismo problema de cabeceo una y otra vez, comprendí que no era cuestión de ajustar constantes $K_p$ o $K_d$, sino de volver a los cimientos y cambiar el modelo de percepción. El cambio al enfoque *topmost* no fue solo una mejora técnica; fue lo que me permitió "despegar" y centrarme en buscar el límite de velocidad del vehículo.

> "A veces, la mejor forma de avanzar no es acelerar, sino detenerse a revisar si el camino elegido es el correcto."



Finalmente, me gustaría dedicar unas palabras al entorno de trabajo. Soy consciente del enorme esfuerzo que conlleva desarrollar y mantener una plataforma como **Unibotics**. Aunque el trabajo con el simulador ha sido en ocasiones tedioso debido a la carga computacional y a ciertas inestabilidades físicas, considero que es la herramienta adecuada para esta asignatura. La práctica está perfectamente alineada con los contenidos teóricos y es un proyecto ideal para comprender la aplicación real de los controladores en el mundo de la automatización.

Quiero agradecer especialmente el trabajo del equipo desarrollador del entorno: [Carlos del Águila](https://github.com/CDAM2020) y [Javier Izquierdo](https://github.com/javizqh). Su labor, permitiéndonos experimentar en esta sandobx virtual, ha sido esencial para que podamos aplicar la teoría a la práctica.

Me gustaría recalcar, además, la predisposición de mi compañero Javier para resolver de forma individualizada cualquier duda sobre la instalación, descarga y ejecución del entorno. Su compromiso con nostros, atendiendo cada incidencia técnica con una actitud profesional, ha sido clave para que muchos de nosotros pudiéramos centrarnos en lo que realmente importa: la programación del robot.

## 9. Referencias y enlaces de interés

En esta sección se detallan las herramientas y fuentes de consulta que han sido fundamentales para el desarrollo de la práctica:

* **Herramientas de visión:**
    * [Selector de color online (ImageColorPicker)](https://imagecolorpicker.online/es/): Utilizado para la identificación precisa de los rangos HSV de la línea roja.
* **Documentación y referentes:**
    * [Blog de Sandra Montejano Cánovas](https://sanmcr.github.io/follow-line-fuera/): Referencia para la comparación de arquitecturas de control y lógica de filtrado.
* **Colaboradores y desarrolladores:**
    * [Javier Izquierdo Hernández (GitHub)](https://github.com/javizqh): Integrante del equipo de desarrollo de Unibotics y soporte técnico del entorno.
    * [Carlos del Águila (GitHub)](https://github.com/CDAM2020): Integrante del equipo de desarrollo de Unibotics y soporte técnico del entorno.
* **Entorno de prácticas:**
    * [Repositorio Oficial de Unibotics](https://github.com/JdeRobot/RoboticsAcademy): Infraestructura sobre la que se ha ejecutado el simulador y el framework de control.

---

<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.7/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
<script type="text/x-mathjax-config">
  MathJax.Hub.Config({
    tex2jax: {
      inlineMath: [['$','$'], ['\\(','\\)']],
      displayMath: [['$$','$$'], ['\\[','\\]']],
      processEscapes: true
    }
  });
</script>