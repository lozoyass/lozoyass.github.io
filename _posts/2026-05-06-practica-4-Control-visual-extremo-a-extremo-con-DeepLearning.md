---
layout: single
title: "Práctica 4: Control visual extremo a extremo con DeepLearning"
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
    <img src="/assets/images/Portada_P4.png" alt="Portada" style="width: 100%; max-width: 1400px; height: auto;">
  </figure>
</div>

## Definición del problema y objetivos

En esta última práctica propuesta en la asignatura nos piden resolver exactamente el mismo problema que en la primera: conseguir que el vehículo del simulador, equipado con una cámara, complete el circuito utilizando visión. La diferencia es que en la primera práctica, éramos nosotros los que teníamos que implementar el algoritmo de visión para que el coche siguiera la línea roja (mediante máscaras de color, umbralización y control PID). Pues bien, ahora tenemos que conseguir lo mismo **entrenando una red neuronal**. Vamos a delegar el proceso de la extracción de características y cálculo de la velocidad lineal y angular a esta red.

Para conseguir este objetivo, los pasos que se han seguido en esta práctica son el balanceo del dataset, la elección del modelo de la red, el entrenamiento del modelo y la inferencia para probar el modelo entrenado. Se explican a continuación cada uno de los pasos.

## 1. Balanceo del dataset

Lo primero que se tuvo que hacer fue visualizar la distribución de los datos en nuestro dataset. En el entrenamiento de modelos, es muy importante no tener desbalanceo entre las clases; esto puede derivar en un modelo "vago" que responde según la clase que más probabilidad tenga de aparecer (según su entrenamiento).

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/histograma_desbalanceo.png" alt="Histograma de desbalanceo" style="width: 100%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Histograma del dataset original donde se aprecia el desbalanceo en el valor de w (giro)
    </figcaption>
  </figure>
</div>

En el histograma de distribución de la velocidad angular ($w$), se aprecia un pico gigantesco justo en el valor 0. Esto nos indica que la inmensa mayoría de las imágenes del dataset corresponden a tramos en los que el coche circula en línea recta. También existen otros picos menores cerca del -1 y del 1 que representan curvas pronunciadas hacia un lado u otro.

Para solucionar el desequilibrio del dataset, primero dividimos las imágenes en dos grupos (rectas y curvas) y, como había demasiadas rectas, descartamos el exceso aleatoriamente hasta igualar la cantidad de ambos tipos. Finalmente, juntamos y mezclamos todas las imágenes al azar para obtener un conjunto de datos equilibrado que obligue al modelo a aprender a conducir correctamente, sin sesgos ni patrones de orden.

## 2. Elección del modelo

Para esta práctica se ha elegido **PilotNet** como modelo de red neuronal para realizar el control del vehículo. La elección de este modelo frente a otras alternativas se fundamenta en los siguientes pilares técnicos:
  - Se trata de la arquitectura recomendada por el profesor debido a su fiabilidad y éxito contrastado en el ámbito de la robótica.
  - Su estructura de red neuronal convolucional está diseñada específicamente para aprender a mapear píxeles de entrada directamente en comandos de dirección.
  - El diseño de esta arquitectura coincide plenamente con nuestra solución técnica al permitir una transición fluida entre la percepción visual y la toma de decisiones motrices.
  - Su diseño ligero garantiza el tiempo de inferencia mínimo y constante que es estrictamente necesario para que el vehículo reaccione en tiempo real dentro del simulador.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/pilotnet_architecture.png" alt="Arquitectura interna de la red PilotNet" style="width: 100%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Arquitectura interna de la red PilotNet
    </figcaption>
  </figure>
</div>

## 3. Entrenamiento

El entrenamiento comienza con la preparación técnica de los datos. Por cada imagen del conjunto balanceado, se lee el archivo en local, se ajusta el espacio de color a RGB y se aplica el mismo preprocesamiento que utilizará el coche en la pista: se recorta la región de interés para descartar el cielo (del horizonte para abajo) y se redimensiona la imagen a 200x66 píxeles. Tras normalizar los píxeles y convertirlos a tensores, las imágenes se emparejan con sus etiquetas correspondientes (velocidad lineal y angular). Finalmente, el dataset se divide de forma aleatoria, reservando un 80% de los datos para enseñar a la red y un 20% exclusivamente para validar su aprendizaje.

<div style="text-align: center; margin: 2em 0;">
  <figure style="display: inline-block; margin: 0; padding: 0;">
    <img src="/assets/images/preprocesamiento_dataset.png" alt="Pipeline de preprocesamiento" style="width: 100%; max-width: 1400px; height: auto;">
    <figcaption style="text-align: center; margin-top: 0.5em; font-style: italic; color: #666;">
      Pipeline de preprocesamiento
    </figcaption>
  </figure>
</div>

Una vez preparada la información, se inicializa el modelo PilotNet. Se utiliza la función **MSELoss (Error Cuadrático Medio)** para evaluar los fallos. Esta función calcula la diferencia entre lo que la red decide hacer y lo que realmente debería haber hecho en el dataset original, guiando al modelo para que ajuste sus criterios de conducción a lo largo de las 20 épocas programadas.

En cada época, el sistema alterna entre una fase de entrenamiento y otra de validación. En esta última, se pone a prueba el rendimiento utilizando ese 20% de imágenes reservadas que el modelo nunca ha visto. Durante esta evaluación, el algoritmo monitorea los resultados y se encarga de guardar los pesos de la red únicamente cuando detecta una mejora en la precisión.

## 4. Testing del modelo

Por último, se puso a prueba el modelo en distintos circuitos dentro de la plataforma Unibotics.

En los vídeos (acelerados x16) que se muestran a continuación se puede ver que el modelo desempeña bien su tarea, completando los distintos circuitos (solo se muestra un fragmento en los vídeos) de la manera esperada, en consonancia con las imágenes del dataset utilizado para su entrenamiento.

#### Circuito simple

<video width="100%" controls>
  <source src="/assets/videos/Simple_circuit.mp4" type="video/mp4">
  Tu navegador no soporta la etiqueta de vídeo.
</video>

#### Circuito de Montreal

<video width="100%" controls>
  <source src="/assets/videos/Montreal_circuit.mp4" type="video/mp4">
  Tu navegador no soporta la etiqueta de vídeo.
</video>

#### Circuito de Montmeló

<video width="100%" controls>
  <source src="/assets/videos/Montmelo_circuit.mp4" type="video/mp4">
  Tu navegador no soporta la etiqueta de vídeo.
</video>

#### Circuito de Nürburgring

<video width="100%" controls>
  <source src="/assets/videos/Nurburgring_circuit.mp4" type="video/mp4">
  Tu navegador no soporta la etiqueta de vídeo.
</video>

## 5. Conclusiones y líneas futuras

Finalizada con éxito esta práctica, puedo decir que ha sido interesante trabajar con un modelo de red neuronal extremo a extremo o "end-to-end". Resulta fascinante comprobar cómo el sistema aprende a conducir directamente desde los píxeles de la cámara, sin necesidad de programar a mano ningún detector para que entienda por dónde va la línea roja.

Como líneas de mejora futuras destacan dos enfoques clave: aplicar aumento de datos (variando artificialmente brillos y contrastes) para que el modelo sea resistente a los cambios de iluminación del entorno, y enseñarle a recuperarse de errores añadiendo imágenes al dataset que le muestren cómo girar para volver al centro si el coche se desvía ligeramente de la trayectoria ideal.

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