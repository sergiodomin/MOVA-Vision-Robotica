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

<p>En un primer momento, probamos a dividir verticalmente la imagen que ve el coche de Fórmula 1 y contar la cantidad de línea (píxeles) que había en cada lado. Probamos a dividirla en 3 y 5 regiones. Con esto, si había muchos píxeles a la derecha girábamos al a izquierda y viceversa, con una velocidad más baja. Por otro lado, si había mayor cantidad de píxeles en el centro que en los laterales considerábamos que estábamos centrados, por lo que nuestro giro era menor y nuestra velocidad más alta. En este caso, no consideramos controladores PID, simplemente nos guiábamos por unas sentencias 'if's' en función de unos umbrales para cada caso. Con esta opción conseguíamos seguir el circuito y acabarlo pero teníamos varios inconvenientes: El coche no seguía fielmente la línea, pegaba muchos tirones en los cambios de umbrales y por último no estábamos aplicando la teoría vista en clase sobre controladores PID.</p>

<hr />

<p>Posteriormente, probamos a eliminar los 'if's', aplicar un controlador PD y cambiar la forma de analizar el error. En este caso, para ver el error que estábamos cometiendo, buscábamos el primer pixel que aparecía en la parte superior de la imagen, con este punto calculábamos la diferencia que tenía respecto a la parte central de la imagen. Con este error, lo aplicábamos a nuestros controladores PD y conseguimos unos tiempos muy bajos (en torno a 30 segundos por vuelta) pero no era del todo correcto debido a que el coche conducía por el circuito de una forma rápida pero al tener el punto de vista tan alto, en muchos casos, no iba sobre la línea, se adelantaba demasiado a los cambios.</p>

<hr />

<p>Por último, decidimos desarrollar un método mucho más conservador y robusto que siguiera la línea más fielmente a pesar de tener que reducir la velocidad. Para este método, además de buscar el punto más alto, también calculábamos donde se encontraba un punto intermedio de la imagen y un punto en la parte inferior de la imagen. En función del valor de estos tres puntos teníamos diferentes 'situaciones': fuera de línea, recta y curva </p>

<ol>
<li>Para empezar evaluamos si hay suficientes píxeles en la imagen, si no superamos un cierto umbral de píxeles en la imagen decidimos que no estamos viendo la línea, que estamos fuera de linea, en este caso retrocedemos de forma rápida con intención de encontrar otra vez la línea lo antes posible.</li>

<li>Un vez que sabemos que no estamos fuera de línea calculamos tres puntos, estos puntos hemos probado con diferentes opciones: usando sólo el punto más alto, probando puntos puestos a mano y lo último que hemos probado y que mejor rendimiento nos ha dado es lo siguiente.
Primero buscamos el punto más alto en el que hay línea, posteriormente buscamos el punto más bajo donde hay línea. Con estos dos puntos <em>calculamos un punto intermedio por el cual nos guiaremos posteriormente para calcular el error</em>. Para afinar este punto intermedio hemos probado diferentes valores. Para empezar, para este caso, probamos a usar el punto medio entre el punto más alto y el punto más bajo, esto nos iba bien pero era demasiado brusco en algunos puntos. Fuímos subiendo el punto intermedio hasta que nos quedamos con un punto bastante alto pero no tan alto como en el caso que hemos explicado anteriormente. Por lo que finalmente el punto intermedio por el cual nos guiamos para calcular la diferencia de error es el siguiente:
<code>row_p_medium = int(((row_p_bottom-row_p_up)/25)+row_p_up)</code></li>

<li>Cuando los puntos están bastante alineados, es decir, están los tres en un rango de 40 píxeles respecto del centro de la imagen, decidimos que estamos en 'modo recta' en este caso a nuestro controlador PD de velocidad de pasamos unos valores que permiten llegar a una velocidad más alta y sin embargo al controlador PD que controla el giro le pasamos unos valores para que gire muy poco(si estamos centrados, ¿para qué vamos a girar mucho?). </li>

<li>Si cumplimos el mínimo de píxeles y no cumplimos el 'modo recta', estamos en 'modo curva' (hubo una versión intermedia en la que definimos 'curva suave' y 'curva fuerte' pero al final desestimamos esta opción debido a que se notaban demasiados saltos en los cambios de un controlador a otro). En este caso al controlador PD de velocidad le pasamos unos parámetros para que tenga una velocidad más baja que en caso de recta y al controlador PD de giro le pasamos unos parámetros para que su rango de giro sea mayor.</li>
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

<p>Por último mostramos un vídeo en el cual se muestra una vuelta completa al circuito con el último método explicado (Tiempo: 0:43):</p>

<p><img src="https://github.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/blob/master/docs/src/Follow_line/F1_v7.gif?raw=true" alt="Vídeo vuelta completa" /></p>

<p> En la carpeta <a href="https://github.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/tree/master/docs/src/Follow_line"> GIF's Follow Line</a> se pueden ver el resto de versiones probadas.

<p><strong>Conclusiones</strong></p>
<p> Tras la prueba de varios métodos y diferentes técnicas hemos apreciado que aplicando un controlador PD los movimientos del coche son mucho más suaves que simplemente con un controlador realimentado básico. 
<hr />
Por otro lado, aunque vemos necesario mirar varios puntos de la imagen para detectar en qué situación estamos, recta o curva, para calcular el error, o la desviación, que estamos cometiendo nos es útil simplemente mirar un punto en la parte alta de la línea que estamos siguiendo. Con este punto bastante alto vemos que la forma de corregir es mucho más suave que si ponemos un punto más bajo y que por lo tanto tenemos una conducción más fluida y con menos turbulencias. La contra de mirar un punto demasiado alto es que en estos casos el coche se adelanta demasiado a lo que va a suceder y corrige demasiado pronto. 
<hr />
Por último cabe destacar que en cuanto a los tiempos por vuelta, con el mismo código, observamos diferentes comportamientos debido a la 'saturación' o rendimiento del emulador. Estos cambios que hemos observado pueden hacer oscilar los tiempos en torno a 10 segundos por vuelta e incluso en algunos momentos desestabilizar el coche completamente. </p>