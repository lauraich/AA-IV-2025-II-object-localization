### Integrantes:
- Ricardo Chicangana
- Fabian Ortiz
- Laura Isabel Chaparro

Arquitecturas
=============

Se evaluaron diferentes configuraciones arquitectónicas tomando como base ResNet50_2 y ResNet50_FPN.

En el caso de ResNet50_2, se utilizaron las siguientes cabezas:

-   Clasificación: Backbone → 768 → 256 → 2

-   Regresión: Backbone → 1024 → 512 → 256 → 4

El modelo ResNet50_FPN mantuvo la misma cantidad de capas y neuronas por capa; las principales diferencias radicaron en el uso de Dropout y BatchNorm, además de cerrar la cabeza de regresión con una activación Sigmoid. Esta activación al final se agregó debido a que las coordenadas de los bounding boxes están normalizadas entre 0 y 1, por lo tanto, la función sigmoide evita que la red produzca coordenadas inválidas, además, puede mejorar la estabilidad en el entrenamiento pues las salidas pueden oscilar en rangos muy grandes al inicio del entrenamiento, lo que puede ralentizar o inestabilizar la convergencia.

Por otro lado, para el modelo propuesto de diseño propio, la arquitectura inicial no proporcionó resultados satisfactorios. En consecuencia, se ajustó a la siguiente configuración:

-   Clasificación: Backbone → 1024 → 512 → 256 → 2

-   Regresión: Backbone → 1024 → 512 → 256 → 128 → 4

En ausencia de un backbone pre-entrenado, la inicialización de los pesos del modelo propio se realizó utilizando el método He Normal. Esta elección resultó en una mejora sustancial del desempeño respecto a las inicializaciones tradicionales en cero o con valores aleatorios, evidenciando la importancia de una estrategia de inicialización adecuada en el proceso de entrenamiento

La cantidad de capas en la arquitectura se definió como resultado de diferentes experimentos y comparación de resultados, buscando el mejor equilibrio entre capacidad de representación y generalización del modelo. Además, para los tres modelos siempre definimos una cabeza de clasificación con una menor cantidad de capas que la de regresión debido a que la segunda representa un problema más complejo que implica usar una arquitectura más profunda.

Función de pérdida
==================

Se implementaron variaciones en dos aspectos principales:

1.  Alpha (α): factor de ponderación para equilibrar la importancia relativa entre la tarea de clasificación y la de regresión.

2.  Pesos de clase: ajuste de ponderaciones para compensar el desbalance del conjunto de datos.

En relación con α, se evaluaron valores por debajo y por encima de 0.5. Para valores < 0.5, se observó un desempeño deficiente en la regresión, dado que la clasificación alcanzaba rápidamente un accuracy cercano a 1.0 antes de obtener un IoU aceptable. Bajo estas condiciones, la regresión quedaba subentrenada.

Para valores ≥ 0.9, el rendimiento global del sistema disminuye: tanto la clasificación como la regresión presentaban dificultades para converger adecuadamente. Los mejores resultados se obtuvieron con valores de α entre 0.6 y 0.7, logrando un balance más estable entre ambas tareas.

En cuanto a los pesos de clase (mask/no mask), se realizaron experimentos en torno a las proporciones reales del dataset (38.4% / 61.6%). Las configuraciones que proporcionaron mejor desempeño fueron 35% / 65%, mostrando que las variaciones en esta proporción afectan de manera significativa no sólo las métricas de clasificación, sino también las de regresión.

Transformación de imágenes
==========================

Para el modelo basado en ResNet50, se emplearon inicialmente únicamente dos transformaciones sobre las imágenes de entrenamiento, a partir de las cuales se inició el proceso de optimización. Al incorporar transformaciones adicionales, el rendimiento del modelo se deterioró de manera sistemática, sin observar mejoras en las métricas.

En consecuencia, los dos modelos posteriores se entrenaron desde el inicio con múltiples transformaciones, evitando las limitaciones encontradas en el enfoque inicial.

Se aplicó MotionBlur para simular imágenes borrosas, RandomBrightnessContrast para reflejar escenarios con mucha o poca iluminación y transformaciones de escalado para reconocer objetos en diferentes tamaños. Una opción descartada fueron las rotaciones, debido a que al analizar los resultados se identificaron inconsistencias en los bounding boxes.

Entrenamiento y optimización
============================

Se habilitó explícitamente el uso de model.eval() y model.train() debido a la presencia de capas Dropout y BatchNorm en las arquitecturas evaluadas.

Se utilizaron valores elevados de dropout en la cabeza de clasificación porque alcanzaba una alta precisión en pocas iteraciones, lo que indicaba un riesgo de sobreajuste. Al incrementar el dropout, se ralentiza el aprendizaje de la clasificación, permitiendo que la cabeza de regresión tenga más tiempo para optimizarse.  Así, se mejora la generalización del modelo y se evita que la clasificación domine el proceso de aprendizaje

Respecto al batch size, se exploró un rango entre 8 y 126. Los tamaños extremos (pequeños o grandes) resultaron en problemas de memoria o en modelos poco estables. Los mejores resultados se obtuvieron consistentemente con un batch size = 32, mientras que con 16 también se lograron desempeños aceptables.

El learning rate se ajustó de manera diferenciada entre el backbone, la cabeza de clasificación y la de regresión, lo cual permitió un mejor control sobre el aprendizaje. Esto fue particularmente importante debido a que la clasificación tendía a converger significativamente más rápido que la regresión.

Finalmente, se implementaron dos estrategias de optimización:

1.  Ciclos de entrenamiento dependientes del batch size, siguiendo el planteamiento del documento base.

2.  Control de ciclos basado en épocas definidas explícitamente, estrategia que introdujo mejoras en estabilidad y resultados. Durante el entrenamiento observamos que, aunque el modelo completaba el número de épocas establecido, el resultado final no siempre correspondía con el mejor desempeño alcanzado en el proceso. Este comportamiento nos llevó a implementar la técnica de Early Stopping, con una paciencia de 10 iteraciones, utilizando como criterio la minimización de la función de pérdida (Loss). De esta manera, aseguramos conservar el modelo con mejor rendimiento y evitamos un sobreentrenamiento innecesario.

En conclusión, tras realizar múltiples pruebas, el modelo que obtuvo el mejor desempeño ---reflejado en el menor valor de loss y el mayor IoU--- fue el ResNet50_FPN. En segundo lugar se ubicó el modelo ResNet50_2, mientras que el modelo propio presentó los resultados menos favorables.
