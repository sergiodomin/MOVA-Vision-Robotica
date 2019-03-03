<hr />

<p><strong>Introducción</strong></p>

<p>En esta práctica se pretende realizar un algoritmo que sea capaz de que un coche de Fórmula 1 simulado siga una línea dibujada sobre un circuito. Esta línea es de color rojo y está en el centro de la pista. En esta práctica se pondrá en uso técnicas de procesado de imagen para encontrar la línea así como técnicas de controladores PID.</p>

<p><img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/Follow_line/circuito.png" alt="Foto Circuito" /></p>

<p><strong>Detección de la línea roja</strong></p>

<p>Para detectar la línea roja sobre el circuito, lo primero que hacemos es pasar al espacio de color HSV, con esto tendremos mayor independencia respecto a la iluminación y podremos detectar mejor el color rojo. Posteriormente aplicamos un filtro de color para quedarnos únicamente con la línea roja del circuito.</p>

<p><code>image_HSV = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)</code></p>

<p><code>value_min_HSV = np.array([0, 235, 60])</code></p>

<p><code>value_max_HSV = np.array([180, 255, 255])</code></p>

<p><code>mask = cv2.inRange(image_HSV, value_min_HSV, value_max_HSV)</code></p>

<p><strong>¿Cómo conducir?</strong></p>

<p>Para calcular el error que cometíamos, ver en qué punto estábamos y aplicar consecuentemente cierta velocidad y giro, hemos realizado diferentes pruebas:</p>

<hr />

<p>En un primer momento, probamos a dividir verticalmente la imagen que ve el coche de Fórmula 1 y contar la cantidad de línea (píxeles) que había en cada lado. Probamos a dividirla en 3 y 5 regiones. Con esto, si había muchos píxeles a la derecha girábamos al a izquierda y viceversa. Por otro lado, si había mayor cantidad de píxeles en el centro que en los laterales considerábamos que estábamos centrados. En este caso, no consideramos controladores PID, simplemente nos guiábamos por unas sentencias 'if's' en función de unos umbrales para cada caso. Con esta opción conseguíamos seguir el circuito y acabarlo pero teníamos varios inconvenientes: El coche no seguía fielmente la línea, pegaba muchos tirones en los cambios de umbrales y por último no estábamos aplicando la teoría vista en clase sobre controladores PID.</p>

<hr />

<p>Posteriormente, probamos a eliminar los 'if's', aplicar un controlador PD y cambiar la forma de analizar el error. En este caso, para ver el error que estábamos cometiendo, buscábamos el primer pixel que aparecía en la parte superior de la imagen, con este punto calculábamos la diferencia que tenía respecto a la parte central de la imagen. Con este error, lo aplicábamos a nuestros controladores PD y conseguimos unos tiempos muy bajos (en torno a 40 segundos por vuelta) pero no era del todo correcto debido a que el coche conducía por el circuito de una forma rápida pero al tener el punto de vista tan alto, en muchos casos, no iba sobre la línea, se adelantaba demasiado a los cambios.</p>

<hr />

<p>Por último, decidimos desarrollar un método mucho más conservador y robusto que siguiera la línea más fielmente a pesar de tener que reducir la velocidad. Para este método, además de buscar el punto más alto, también calculábamos donde se encontraba un punto intermedio de la imagen y un punto en la parte inferior de la imagen. En función del valor de estos tres puntos teníamos diferentes 'situaciones': fuera de línea, recta y curva </p>

<ol>
<li>Para empezar evaluamos si hay suficientes píxeles en la imagen, si no superamos un cierto umbral de píxeles en la imagen decidimos que no estamos viendo la línea, que estamos fuera de linea, en este caso retrocedemos de forma rápida con intención de encontrar otra vez la línea lo antes posible.</li>

<li>Un vez que sabemos que no estamos fuera de línea calculamos tres puntos, estos puntos hemos probado con diferentes opciones: usando sólo el punto más alto, probando puntos puestos a mano y lo último que hemos probado y que mejor rendimiento nos ha dado es lo siguiente.
Primero buscamos el punto más alto en el que hay línea, posteriormente buscamos el punto bajo donde hay línea. Con estos dos puntos <em>calculamos el punto intermedio por el cual nos guiaremos posteriormente para calcular el error</em>. Este punto intermedio lo calculamos de la siguiente forma:
<code>row_p_medium = int(((row_p_bottom-row_p_up)/4)+row_p_up)</code></li>

<li>Cuando los puntos están bastante alineados, es decir, están los tres en un rango de 40 píxeles respecto del centro de la imagen, decidimos que estamos en 'modo recta' en este caso a nuestro controlador PD de velocidad de pasamos unos valores que permiten llegar a una velocidad más alta y sin embargo al controlador PD que controla el giro le pasamos unos valores para que gire muy poco(si estamos centrados, ¿para qué vamos a girar mucho?). </li>

<li>Si cumplimos el mínimo de píxeles y no cumplimos el 'modo recta', estamos en 'modo curva' (hubo una versión intermedia en la que definimos 'curva suave' y 'curva fuerte' pero al final desestimamos esta opción). En este caso al controlador PD de velocidad le pasamos unos parámetros para que tenga una velocidad más baja que en caso de recta y al controlador PD de giro le pasamos unos parámetros para que su rango de giro sea mayor.</li>
</ol>

<p><strong>El controlador PD para calcular la velocidad:</strong></p>

<p><code>def calVel(diff_prev, diff, kpv, kdv, minvel):</code></p>

<pre><code>diff = abs(abs(diff)-320)
diff_prev = abs(abs(diff_prev)-320)
vel = minvel + diff/kpv + (diff-diff_prev)/kdv
return vel
</code></pre>

<p>En este controlador, le pasamos: la diferencia previa, la diferencia actual, las k's del controlador P y D y por último le pasamos una velocidad mínima para que el controlador no llegue a velocidades muy bajas, limitamos de alguna forma la mínima velocidad que pueda tomar el controlador PD.</p>

<p><strong>El controlador PD para calcular el giro:</strong></p>

<p><code>def calGiro(diff_prev, diff, kpg, kdg):</code></p>

<pre><code>giro_kpg = diff*kpg
giro_kdg = (diff-diff_prev)*kdg
giro = giro_kpg + giro_kdg

return giro, giro_kpg, giro_kdg
</code></pre>

<p>Para el controlador PD de giro, al contrario que en el caso anterior, que en el caso anterior, no limitamos el giro para que así pueda corregir mejor el controlador D respecto al P.</p>

<p>Por último mostramos un vídeo en el cual se muestra una vuelta completa al circuito con el último método explicado:</p>

<p><img src="https://github.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/blob/master/docs/src/Follow_line/F1_v4.gif?raw=true" alt="Vídeo vuelta completa" /></p>
