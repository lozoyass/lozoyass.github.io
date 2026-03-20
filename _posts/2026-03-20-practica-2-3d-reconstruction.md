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

## 1. Introducción
En esta práctica, el objetivo principal es dotar a nuestro robot de percepción de profundidad. A partir de dos imágenes bidimensionales obtenidas simultáneamente por sus cámaras estéreo (izquierda y derecha), programaremos la lógica matemática necesaria para generar una nube de puntos y reconstruir en 3D la escena que el robot tiene en frente.

Para llevar a cabo este desarrollo, volvemos a utilizar el entorno de simulación y programación de Unibotics. Todo el algoritmo de visión artificial se ha implementado íntegramente en Python, apoyándonos en las matrices de las cámaras y en las herramientas de procesamiento de imagen de OpenCV.

## 2. Fundamentos del estéreo canónico
Para poder reconstruir el mundo en 3D, primero necesitamos entender cómo están configurados los "ojos" de nuestro robot. En nuestro caso, trabajamos con un par estéreo canónico. Esto significa que las dos cámaras están perfectamente coplanarias (en el mismo plano) y sus ejes ópticos son estrictamente paralelos. ¿Cómo sabemos que esto es así y no hay distorsiones físicas? Porque nos apoyamos en la propia documentación teórica de la práctica, que establece esta configuración por diseño: 

> "Stereo reconstruction is a special case of the above 3d reconstruction where the two image planes are parallel to each other and equally distant from the 3d point we want to plot. In this case the epipolar line for both the image planes are same, and are parallel to the width of the planes, simplifying our constraints better."

Trabajar en un simulador perfecto nos permite asumir esta geometría ideal, lo que simplificará enormemente los cálculos epipolares más adelante.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/estereo_canonico_cropped.jpg" alt="Configuración estéreo canónica" style="width: 60%; max-width: 600px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 1. Representación del par estéreo canónico.
    </figcaption>
  </figure>
</div>

Además de la geometría general, el algoritmo necesita las medidas exactas del modelo de la cámara. Realizamos una pequeña depuración imprimiendo por terminal los datos del driver de ROS (``` HAL.getCameraPosition() ```). A partir de las matrices intrínseca ($K$) y extrínseca ($RT$) devueltas por la terminal, extrajimos nuestras constantes fundamentales:

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/parametros_terminal.png" alt="Parámetros mostrados por la terminal" style="width: 60%; max-width: 800px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 2. Constantes fundamentales mostradas por terminal.
    </figcaption>
  </figure>
</div>

* **Focal ($f$):** $240.0$ píxeles
* **Centro óptico ($c\_x, c\_y$):** $(320.0, 240.0)$, justo en el centro de nuestras imágenes de $640\times480$
* **Línea base o Baseline ($B$):** $220.0$ mm. Es la distancia física que separa ambas cámaras, deducida al observar que cada cámara está trasladada $110$ mm desde el centro del robot en ejes opuestos.

## 3. Preprocesamiento de las imágenes
Antes de buscar correspondencias entre las imágenes izquierda y derecha, es crucial "limpiar" la información visual. Trabajar con las imágenes crudas capturadas por el robot (``` HAL.getImage() ```) nos expondría a variaciones de iluminación y ruido del sensor que arruinarían las comparaciones posteriores.

* **Filtro bilateral:** El primer paso es aplicar un filtro bilateral (``` cv2.bilateralFilter ```). El filtro bilateral suaviza las texturas planas y elimina el ruido, preservando intactos los bordes de los objetos. Como la propia documentación de la práctica recomienda en sus Hints, esto elimina detalles innecesarios que solo estorbarían en la reconstrucción 3D.
* **Detector de bordes de Canny:** Una imagen de 640x480 píxeles contiene más de 300.000 puntos. Procesar iterativamente cada uno de ellos hundiría el rendimiento del algoritmo. Para lograr una optimizar el proceso, utilizamos el algoritmo de Canny (` cv2.Canny `) sobre las imágenes en escala de grises. Esto reduce nuestra área de trabajo únicamente a los píxeles que forman los contornos de los objetos (donde el valor es 255). Estos bordes se convierten en nuestros puntos de interés y así reducimos drásticamente el coste computacional.

## 4. Correspondencia y geometría Epipolar
Una vez extraídos los píxeles de interés en la cámara izquierda, necesitamos encontrar su píxel gemelo exacto en la cámara derecha. Aquí es donde la teoría de la geometría epipolar choca con la realidad del código y nos permite un atajo matemático.

* **La ventaja canónica:** En un sistema estéreo cualquiera, para buscar el punto homólogo deberíamos proyectar un rayo 3D para hallar una línea epipolar en la segunda cámara. Sin embargo, como establece la documentación: 

> "Stereo reconstruction is a special case... the epipolar line for both the image planes are same, and are parallel to the width of the planes".

Al tener un par estéreo canónico perfecto, la línea epipolar es perfectamente horizontal. Por lo tanto, nos ahorramos todos los cálculos de backprojection: si nuestro píxel está en la fila $Y$ de la imagen izquierda, su gemelo estará exactamente en la misma fila $Y$ de la imagen derecha.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/lineas_epipolares.png" alt="Líneas epipolares dibujadas" style="width: 60%; max-width: 600px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 3. Líneas epipolares dibujadas.
    </figcaption>
  </figure>
</div>

* **Acotando la búsqueda (rango en $X$):** Ya sabemos que la búsqueda es unidimensional (solo nos movemos en el eje $X$), pero no hace falta recorrer toda la fila. Geométricamente, un objeto visto desde la cámara derecha siempre aparecerá desplazado hacia la izquierda en comparación con la cámara izquierda. Así, nuestro límite máximo de búsqueda (rango_max) es la posición X original del píxel. El límite mínimo (rango_min) lo definimos como el radio_parche de nuestra ventana de comparación, puramente para evitar salirnos de los límites de la matriz de la imagen.
* **Template matching:** Usamos una ventana de $15\times15$ píxeles de la imagen izquierda para buscar su homólogo en la derecha mediante ``` cv2.matchTemplate ```. Solo aceptamos emparejamientos con una similitud superior al $70$%.

## 5. Triangulación matemática y filtrado de ruido
Una vez encontrado el píxel gemelo, el siguiente paso es transformar esas coordenadas 2D en un punto tridimensional $(X, Y, Z)$ real.

* **Cálculo de la disparidad:** La disparidad ($d$) es la diferencia en píxeles entre la posición de un objeto en la cámara izquierda y en la derecha ($d = x\_{izq} - x\_{der}$). Cuanto más cerca está el objeto, mayor es este salto visual. Si un punto está en el infinito, su disparidad es cero.

* **Triángulos semejantes:** Gracias a la geometría del estéreo canónico, podemos usar la relación de triángulos semejantes para calcular la profundidad exacta ($Z$) y las coordenadas espaciales ($X$, $Y$) aplicando el modelo pinhole:

$$
\begin{align*}
Z &= \frac{f \cdot B}{d} \\
X &= \frac{(x_{izq} - c_x) \cdot Z}{f} \\
Y &= \frac{(y_{izq} - c_y) \cdot Z}{f}
\end{align*}
$$

* **Limpieza de puntos espúreos:** A pesar del umbral de similitud, el template matching puede generar falsos positivos. Estos errores producen disparidades absurdas. Para limpiar la nube de puntos, aplicamos un "clipping": descartamos cualquier punto que matemáticamente quede a más de 20 metros ($Z > 20000 \text{ mm}$) o a menos de 2 metros ($Z < 2000 \text{ mm}$). Esto elimina algo del ruido, pero no todo.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/espureos_pintados.png" alt="Puntos espúreos (malos emparejamientos)" style="width: 80%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Figura 4. Puntos espúreos (emparejamientos erróneos).
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