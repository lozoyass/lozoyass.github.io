---
layout: single
title: "Práctica 3: Marker Visual Loc"
mathjax: true
classes: wide
date: 2026-04-13
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

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/escenario_localizacion.png" alt="Escenario de localización" style="width: 100%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Escenario de la vivienda con balizas AprilTag distribuidas en las paredes.
    </figcaption>
  </figure>
</div>

## 1. Introducción
El objetivo de esta tercera práctica consiste en lograr que nuestro robot aspiradora sea capaz de autolocalizarse de manera fiable en un entorno doméstico. Para ello, contamos con dos fuentes de información principales: la **visión**, mediante la detección de balizas visuales de tipo AprilTag, y la **odometría** del propio robot.

El reto principal reside en fusionar ambas informaciones. Mientras que la visión nos ofrece una posición absoluta muy precisa pero intermitente (solo cuando vemos una baliza), la odometría nos proporciona un flujo continuo de datos sobre el desplazamiento, aunque con un error acumulativo (ruido) que degrada la estimación con el tiempo.

En la interfaz de simulación, podemos distinguir tres representaciones del robot:
* **Robot Verde:** Representa la posición real (*Ground Truth*).
* **Robot Azul:** Representa la posición calculada únicamente mediante odometría (con ruido acumulado).
* **Robot Rojo:** Es nuestra estimación de usuario, el resultado de nuestro algoritmo de localización.

## 2. Configuración del sistema de visión
Para que el robot pueda entender lo que ve, primero debemos definir matemáticamente su cámara y el mundo que le rodea.

### Carga del mapa de balizas
Las balizas AprilTag están situadas a lo largo de la casa. Cada una tiene un identificador único y una posición (X, Y, Z) y orientación (Yaw) fija en el mapa del mundo. Estos datos se cargan desde un archivo YAML que nuestro código procesa al inicio para conocer la ubicación exacta de cada "faro" visual.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/apriltag_ejemplo.png" alt="Ejemplo de AprilTag" style="width: 300px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 1. Marcador visual AprilTag (familia tag36h11) utilizado como baliza.
    </figcaption>
  </figure>
</div>

### Modelo de cámara Pinhole
Al igual que en prácticas anteriores, utilizamos el modelo de cámara *pinhole* para proyectar los puntos 3D del mundo en el plano 2D de la imagen. Basándonos en la resolución de la cámara del robot ($1920\times1080$), extraemos los parámetros intrínsecos fundamentales:
* **Distancia focal ($f$):** Aproximada al ancho de la imagen en píxeles ($1920$).
* **Centro óptico ($c_x, c_y$):** Situado en el centro geométrico de la imagen ($960, 540$).

Además, definimos las dimensiones reales de los marcadores. Aunque el tamaño total es de $0.3\times0.3$ metros, la parte negra que detecta el algoritmo mide exactamente **0.24 metros** de lado, dato crucial para que el cálculo de la distancia sea preciso.

## 3. Detección y Localización Visual (PnP)
Una vez conocemos las características de nuestra cámara y dónde están situadas las balizas en el mundo, podemos empezar a procesar las imágenes.

### 3.1. Detección y optimización del rendimiento
Para detectar los marcadores en la imagen utilizamos la librería `pyapriltags`. Sin embargo, aquí nos encontramos con el primer desafío técnico: procesar imágenes a resolución nativa ($1920\times1080$) a 15-30 frames por segundo requiere una enorme capacidad de cómputo, lo que saturaba el simulador.

Para solucionar este cuello de botella y hacer que el robot se moviese de forma fluida, implementamos un **escalado previo de la imagen**. Reduciendo la imagen a escala de grises a la mitad de su tamaño original (`FACTOR_ESCALA = 0.5`), el algoritmo procesa 4 veces menos píxeles. Una vez detectadas las esquinas del AprilTag en esta imagen reducida, simplemente multiplicamos sus coordenadas por la escala inversa para devolverlas a su tamaño original antes de realizar los cálculos matemáticos.

Además, para cumplir con los requisitos de la práctica, si el robot visualiza varias balizas simultáneamente, calculamos la diagonal de cada una en píxeles y **nos quedamos únicamente con la más grande** (la más cercana), descartando el resto para minimizar el error por lejanía.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/bounding_box_amarilla.png" alt="Detección de AprilTag con bounding box" style="width: 100%; max-width: 800px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 2. Bounding box dibujada sobre la baliza detectada en tiempo real.
    </figcaption>
  </figure>
</div>

### 3.2. Perspective-n-Point (PnP) y matrices de transformación
Con las 4 esquinas del marcador en coordenadas 2D de la imagen (`image_points`) y conociendo su tamaño físico en 3D (`object_points`), aplicamos el algoritmo **Perspective-n-Point (PnP)** mediante la función `cv2.solvePnP` de OpenCV. 

Este algoritmo nos devuelve la rotación (`rvec`) y la traslación (`tvec`) del marcador *respecto a la cámara*. Para poder operar con estos vectores de manera robusta, los convertimos a una **matriz de transformación homogénea (RT)** de $4\times4$:

$$
RT = \begin{bmatrix}
R_{3\times3} & T_{3\times1} \\
0_{1\times3} & 1
\end{bmatrix}
$$

Como lo que realmente necesitamos no es dónde está la baliza respecto a la cámara, sino **dónde está la cámara respecto a la baliza**, calculamos la matriz inversa ($RT_{tag \to camara} = RT_{camara \to tag}^{-1}$).

### 3.3. Los sistemas de coordenadas
Durante el desarrollo, nos dimos cuenta de que nuestra estimación (el robot rojo) aparecía desplazada o "dentro de las paredes" formando un efecto espejo.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/1_robot rojo detrás(error).png" alt="Robot rojo reflejado fuera del mapa." style="width: 100%; max-width: 800px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 3. Robot rojo localizado fuera del mapa, en espejo.
    </figcaption>
  </figure>
</div>

Esto ocurre porque OpenCV y el simulador Gazebo (el mundo 3D) hablan "idiomas" distintos:
* **OpenCV:** Asume que la cámara mira hacia adelante en el eje $Z$, el eje $X$ va hacia la derecha y el eje $Y$ hacia abajo.
* **Gazebo:** El eje $X$ suele representar el avance (frente), el eje $Y$ la izquierda y el eje $Z$ la altura (arriba).

Para solucionar esto, intercalamos una **matriz de corrección** que traduce las coordenadas de OpenCV al entorno tridimensional del mapa:

$$
T_{correccion} = \begin{bmatrix}
0 & 0 & 1 & 0 \\
-1 & 0 & 0 & 0 \\
0 & -1 & 0 & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

### 3.4. Cálculo de la posición absoluta
El último paso de la fase visual es la obtención de nuestras coordenadas globales. Consultando nuestro archivo YAML, extraemos la matriz de transformación absoluta de la baliza que estamos viendo ($RT_{mundo \to tag}$). 

La posición y orientación final de nuestra cámara en el mundo real se calcula mediante una concatenación (multiplicación matricial) de todas las transformaciones obtenidas:

$$
RT_{mundo \to camara} = RT_{mundo \to tag} \cdot T_{correccion} \cdot RT_{tag \to camara}
$$

De esta matriz resultante extraemos la componente $X$ e $Y$ absolutas. Para la orientación global del robot (el *Yaw*), extraemos el ángulo utilizando el arcotangente de los componentes direccionales del eje $Z$ de la matriz de rotación resultante (`np.arctan2(RT[1, 2], RT[0, 2])`). En este punto, si vemos una baliza, nuestro robot rojo se posicionará muy cerca del ground truth.

## 4. Fusión de Visión y Odometría
Mientras que la localización visual nos ofrece una posición absoluta muy precisa y sin acumulación de error, solo está disponible de forma intermitente (cuando el robot tiene una baliza en su campo de visión). Por el contrario, la odometría es constante, pero degrada la estimación rápidamente debido al ruido acumulado.

Para resolver esto, implementamos un sistema de localización que fusiona ambos sensores:

1. **Localización visual**: Cuando detectamos un AprilTag, calculamos la posición absoluta mediante PnP y actualizamos nuestra "memoria" interna. En ese preciso instante, guardamos tanto la pose visual estimada como la lectura actual de la odometría del robot para que sirvan como punto de referencia.
2. **Localización odométrica**: Si el robot pierde de vista la baliza. En lugar de confiar en la posición global que reporta la odometría (que sería errónea respecto a nuestro mapa), calculamos únicamente el **incremento de movimiento** (delta) desde la última vez que estuvimos localizados visualmente.

Este incremento relativo se suma a nuestra posición almacenada.

## 5. Estrategia de navegación y exploración
Para garantizar que el robot recorra el entorno y visite diversas estancias, implementamos una **máquina de estados simple** basada en la presencia de señales visuales:

* **Estado de búsqueda (sin balizas)**: Si el robot no detecta ningún AprilTag, detiene su avance lineal y comienza a rotar sobre su eje. Este giro constante permite a la cámara "barrer" las paredes de la habitación hasta encontrar una referencia.
* **Estado de Avance (baliza detectada)**: En cuanto el algoritmo de visión detecta un marcador, el robot deja de rotar y avanza directamente hacia adelante. 

## 6. Resultados y conclusiones

### Demostración en vídeo
En los siguientes vídeos se puede observar el comportamiento final del robot. Debido a la carga computacional que supone el procesamiento de imagen y la simulación física, la grabación original de la sesión deambulando por la casa ha sido acelerada.

Es importante mencionar que aunque en el vídeo de demostración se pinta la bounding box de todas las balizas visibles, en el código final solamente trabajamos con la más cercana (como nos pide el enunciado).

Se muestran dos vídeos, cada uno con una ruta distinta:

<video width="100%" controls>
  <source src="/assets/videos/resultado1.mp4" type="video/mp4">
  Tu navegador no soporta la etiqueta de vídeo.
</video>

<video width="100%" controls>
  <source src="/assets/videos/resultado2.mp4" type="video/mp4">
  Tu navegador no soporta la etiqueta de vídeo.
</video>

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