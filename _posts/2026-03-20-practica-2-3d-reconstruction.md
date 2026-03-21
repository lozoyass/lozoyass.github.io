---
layout: single
title: "Práctica 2: Reconstrucción 3D"
mathjax: true
classes: wide
date: 2026-03-20
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
    <img src="/assets/images/escenario.png" alt="Escenario de la reconstrucción" style="width: 100%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Escenario del simulador que vamos a reconstruir.
    </figcaption>
  </figure>
</div>

## 1. Introducción
En esta práctica, el objetivo principal es dotar a nuestro robot de percepción de profundidad. A partir de dos imágenes bidimensionales obtenidas simultáneamente por sus cámaras estéreo (izquierda y derecha), programaremos la lógica matemática necesaria para generar una nube de puntos y reconstruir en 3D la escena que el robot tiene en frente.

Para llevar a cabo este desarrollo, volvemos a utilizar el entorno de simulación y programación de Unibotics. Todo el algoritmo de visión artificial se ha implementado íntegramente en Python, apoyándonos en las matrices de las cámaras y en las herramientas de procesamiento de imagen de OpenCV.

## 2. Fundamentos de la geometría estéreo
Para poder reconstruir el mundo en 3D, primero necesitamos entender cómo están configuradas las cámaras de nuestro robot. En la visión estéreo, es fundamental conocer la posición espacial y las características de cada cámara para poder trazar y cruzar los rayos de luz.

Aunque la propia documentación de la práctica nos indica que el diseño de las cámaras es coplanario, el objetivo es desarrollar un algoritmo computacional robusto que funcione mediante matrices de proyección, calculando la geometría epipolar real de la escena.

Realizamos una pequeña depuración imprimiendo por terminal los datos del driver de ROS (`HAL.getCameraPosition()`). A partir de las matrices intrínseca ($K$) y extrínseca ($RT$) devueltas por la terminal, extrajimos nuestras constantes fundamentales:

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/parametros_terminal.png" alt="Parámetros mostrados por la terminal" style="width: 100%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 1. Constantes fundamentales mostradas por terminal.
    </figcaption>
  </figure>
</div>

* **Focal ($f$):** $240.0$ píxeles
* **Centro óptico ($c\_x, c\_y$):** $(320.0, 240.0)$, justo en el centro de nuestras imágenes de $640\times480$
* **Línea base o _baseline_ ($B$):** $220.0$ mm. Es la distancia física que separa ambas cámaras, deducida al observar que cada cámara está trasladada $110$ mm desde el centro del robot en ejes opuestos.

Con estos parámetros construimos nuestras matrices de proyección ($P_1$ y $P_2$), estableciendo la cámara izquierda como el origen de coordenadas del mundo y la cámara derecha trasladada espacialmente en el eje X en función del _baseline_.

## 3. Preprocesamiento de las imágenes
Antes de buscar correspondencias entre las imágenes izquierda y derecha, es crucial "limpiar" la información visual. Trabajar con las imágenes crudas capturadas por el robot (``` HAL.getImage() ```) nos expondría a variaciones de iluminación y ruido del sensor que arruinarían las comparaciones posteriores.

* **Filtro bilateral:** El primer paso es aplicar un filtro bilateral (``` cv2.bilateralFilter ```). El filtro bilateral suaviza las texturas planas y elimina el ruido, preservando intactos los bordes de los objetos. Como la propia documentación de la práctica recomienda en sus hints, esto elimina detalles innecesarios que solo estorbarían en la reconstrucción 3D.
* **Detector de bordes de Canny:** Una imagen de 640x480 píxeles contiene más de 300.000 puntos. Procesar iterativamente cada uno de ellos hundiría el rendimiento del algoritmo. Para optimizar el proceso, utilizamos el algoritmo de Canny (` cv2.Canny `) sobre las imágenes en escala de grises. Esto reduce nuestra área de trabajo únicamente a los píxeles que forman los contornos de los objetos (donde el valor es 255). Estos bordes se convierten en nuestros puntos de interés y así reducimos drásticamente el coste computacional.

## 4. Correspondencia y geometría epipolar
Una vez extraídos los píxeles de interés en la cámara izquierda, necesitamos encontrar su píxel gemelo exacto en la cámara derecha. Para ello aplicamos la teoría de la geometría epipolar.

* **Proyección y retroproyección:** Para buscar el punto homólogo de un píxel de la imagen izquierda, trazamos matemáticamente su línea epipolar correspondiente en la cámara derecha. Tomamos el píxel de interés y lanzamos un rayo tridimensional imaginario acotado entre una distancia mínima ($Z=2\text{ m}$) y una máxima ($Z=20\text{ m}$). Proyectando los extremos de este rayo 3D sobre el plano de la segunda cámara, obtenemos las coordenadas de un segmento exacto en 2D.

* **Búsqueda a lo largo de la línea:** Una vez calculado el segmento epipolar limitamos nuestra búsqueda a los puntos que forman dicha línea. Evaluamos iterativamente las coordenadas, pero solo nos detenemos a procesar aquellos píxeles que pertenezcan a los bordes detectados por el algoritmo de Canny en la imagen derecha. Con esto evitamos procesar áreas vacías.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/lineas_epipolares.png" alt="Líneas epipolares dibujadas" style="width: 100%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 2. Líneas epipolares de búsqueda.
    </figcaption>
  </figure>
</div>

* **_Template matching_:** Usamos una ventana de $15\times15$ píxeles de la imagen izquierda para buscar su homólogo en la derecha mediante ``` cv2.matchTemplate ```. Solo aceptamos emparejamientos con una similitud superior al $70$%.

## 5. Triangulación matemática y filtrado de ruido
Una vez encontrado el píxel gemelo, el siguiente paso es transformar esas coordenadas 2D en un punto tridimensional $(X, Y, Z)$ real.

* **Triangulación espacial:** Cruzamos los rayos visuales de ambas cámaras en el espacio 3D. Utilizando las matrices de proyección ($P_1$ y $P_2$) definidas en la configuración, empleamos la función algorítmica `cv2.triangulatePoints`. Este método matemático busca el punto de intersección óptimo minimizando el error, devolviendo una coordenada homogénea en 4D ($P_{4D}$).

* **Conversión a cartesianas:** Para recuperar nuestras métricas reales en milímetros, dividimos los tres primeros componentes del vector 4D por su cuarta coordenada (la escala):

$$
\begin{align*}
X &= \frac{P_{4D}[0]}{P_{4D}[3]} \\
Y &= \frac{P_{4D}[1]}{P_{4D}[3]} \\
Z &= \frac{P_{4D}[2]}{P_{4D}[3]}
\end{align*}
$$

* **Limpieza de puntos espúreos:** A pesar del umbral de similitud, el _template matching_ puede generar falsos positivos. Estos errores producen disparidades absurdas. Para limpiar la nube de puntos, aplicamos un "_clipping_": descartamos cualquier punto que matemáticamente quede a más de 20 metros ($Z > 20000 \text{ mm}$) o a menos de 2 metros ($Z < 2000 \text{ mm}$). Esto elimina algo del ruido, pero no todo.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/espureos_pintados.png" alt="Puntos espúreos (malos emparejamientos)" style="width: 100%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 3. Puntos espúreos (emparejamientos erróneos).
    </figcaption>
  </figure>
</div>

## 6. Proyección final
* **Ajuste de coordenadas:** El sistema de coordenadas de una imagen plana sitúa el origen $(0,0)$ arriba a la izquierda (la Y crece hacia abajo). En un entorno 3D, el eje Y suele crecer hacia el cielo. Para evitar renderizar la escena boca abajo y en espejo, invertimos los signos al empaquetar el punto visual ($-X$, $-Y$).

### Resultado 

Se muestra un vídeo del escenario renderizado.

<video width="100%" controls>
  <source src="/assets/videos/resultado_editado.mp4" type="video/mp4">
  Tu navegador no soporta la etiqueta de vídeo.
</video>

## 7. Conclusiones

Al principio, la teoría de la visión estéreo me resultaba abstracta y difícil de asimilar. Sin embargo, el punto de inflexión fue la programación: ver cómo la focal y la disparidad daban forma al entorno del robot me hizo comprenderlo mejor. La implementación práctica fue el puente necesario para comprender cómo funciona realmente la reconstrucción 3D. Para sorpresa de nadie, la práctica hace al maestro (aunque no me considero maestro ni de lejos en este ámbito).

La visión estéreo no era el tema que más me entusiasmaba, pero mi perspectiva ha cambiado radicalmente. Esto debo atribuirlo también a las clases complementarias de la asignatura de visión tridimensional, en las cuales hemos profundizado mucho más y me han permitido tener un entendimiento mayor del tema. Me parece un mundo apasionante.

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