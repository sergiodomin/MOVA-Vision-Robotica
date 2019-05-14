<hr />
[Home](README.md) | [Follow Line](FollowLine.md) | [3D Reconstruction](3DReconstruction.md)
<h1>Práctica 3D Reconstruction</h1>
<p><strong>Introducción</strong></p>
<p align="justify">En esta práctica se pretende realizar un algoritmo que sea capaz de reconstruir una escena en 3D a partir de lo que ve un sistema estéreo canónico compuesto por dos cámaras.</p>

<p><strong>Preprocesado de la imagen izquierda de nuestro par estéreo</strong></p>
<p align="justify">Nuestra cámara izquierda será de la que partamos para seleccionar los puntos característicos que posteriormente queramos encontrar en la otra cámara. Para ello, la imagen de esta cámara la pasamos de un espacio de color en RGB a escala de grises, para posteriormente aplicarle un filtrado Canny.</p>
<p align="justify">Con el filtrado Canny lo que conseguimos es obtener los bordes, o contornos, de los objetos que queremos reconstruir. Para las versiones iniciales, este filtrado lo hacíamos bastante agresivo para no obtener demasiados puntos y así poder tracear mejor los errores.</p>
<p align="justify">Para la versión final, lo que se ha hecho es un filtrado Canny menos agresivo, con lo que nos hemos quedado con más puntos y posteriormente también se ha hecho una ligera dilatación de los puntos para así tener mayor cantidad de los objetos a reconstruir.</p>

<p><strong>Preprocesado de la imagen derecha de nuestro par estéreo</strong></p>
<p align="justify">En esta cámara, lo que queremos es poder encontrar aquellos puntos que hemos seleccionado en la otra cámara. Por ello también realizamos el filtrado Canny y posteriormente una dilatación. En este caso la dilatación que aplicamos en esta imagen es mayor que en la imagen izquierda. La razón de que apliquemos una dilatación mayor, es debido a que queremos tener un mayor abanico de posibilidades de puntos a buscar alrededor de los puntos característicos. De esta forma, asegurarnos de encontrar una correspondencia buena. Es decir, buscamos en los puntos característicos y en sus vecinos.</p>

<p align="center"><img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/imgCL.png" alt="Foto Canny Izquierda" />
<img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/imgCR.png" alt="Foto Canny Derecha" /></p>
<figcaption align="center">Comparativa entre la imagen izquierda con una dilatacion menor y la imagen derecha con una dilatación mayor</figcaption>
<hr />
<p><strong>Restricción de la línea epipolar</strong></p>
<p align="justify">Una vez que tenemos las dos imágenes, tenemos que encontrar donde están en la imagen derecha los puntos que hemos seleccionado en la imagen izquierda. Buscar en toda la imagen sería muy costoso, por lo que para agilizar este proceso, lo que se ha hecho es lo que se denomina como "restricción epipolar", más concretamente, en este caso, franja epipolar.</p>
<p align="justify">La restricción epipolar dice que la correspondenica de p1 en la imagen 2 se halla en la línea epipolar L21 (la línea epipolar en la imagen 2 asociada a p1). Igual para la línea epipolar de p2 en la imagen 1</p>

<p align="center"><img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/epipolar_g.png" alt="Foto Diagrama Epipolar" /></p>
<figcaption align="center">Diagrama línea epipolar</figcaption>
<hr />
<p align="justify">Para ello, hemos de calcular el rayo que parte de una de las cámaras y atraviesa un punto en el plano imagen de esta misma cámara y proyectar este rayo sobre el plano imagen de la otra cámara.</p>
<p align="justify">Para calcular el origen de coordenadas de nuestra cámara izquierda usamos la siguiente función:</p>
<p><code>origLeft = self.camLeftP.getCameraPosition()</code></p>
<p><code>origLeft = np.append(origLeft,1)</code>
<p align="justify">Proyectamos dicho punto sobre la otra imagen y cálculamos sus coordenadas sobre el plano imagen:</p>    
<p><code>pProjectLeft = self.camRightP.project(origLeft)</code></p>
<p><code>pOrigLeftToRight = self.camRightP.opticalToGrafic(pProjectLeft)</code></p>
<p align="justify">Ahora, con uno de los puntos de interés que hemos calculado anteriormente, hacemos lo mismo, lo proyectamos sobre la otra imagen y obtenémos sus coordenadas en el plano imagen:</p>
<p><code>pointLeftProj = self.camRightP.project(pointLeft)</code></p>
<p><code>pointLeftToRight = self.camRightP.opticalToGrafic(pointLeftProj)</code></p>
<p align="justify">Ya tenemos dos puntos proyectados sobre la otra imagen, con dos puntos podemos calcular una recta, esta recta será nuestra línea epipolar. Esta línea epipolar, la hicimos de un grosor de dos píxeles, franja epipolar, para así también tener un cierto margen de error a la hora de buscar su correspondencia. Dado que estamos en un sistema canónico estas líneas epipolares serán siempre líneas paralelas.</p>

<p><strong>¿Buscamos en toda la línea epipolar?</strong></p>
<p align="justify">Una vez tenemos la restricción epipolar, podríamos buscar la correspondencia en toda la línea epipolar. Como hemos dicho anteriormente, a la imagen derecha le hemos aplicado un filtrado Canny más una dilatación (más fuerte que a la imagen izquierda) por lo que para agilizar este proceso y tener menos puntos a buscar, lo que hemos hecho en un primer lugar ha sido multiplicar una máscara de nuestra imagen con la línea epipolar por una máscara de la imagen preprocesada. Con esto estamos reduciendo los puntos a buscar no sólo a la línea epipolar sino sólo a los puntos que también son caracterísicos en la otra imagen y sus vecinos.</p>
<p align="justify">Tras aplicar la opción anterior vimos que bajaba bastante el tiempo de procesado, pero aplicamos otra restricción más. Asumiendo que la correspondencia que queremos encontrar en la otra imagen debe de estar más o menos cerca "en cuanto a píxeles" en la otra imagen.
 Es decir, si queremos encontrar una correspondencia que está al inicio de la imagen (parte izquierda de la imagen) en la otra imagen, lo más normal es que si ese punto existe se encuentre tambien al inicio de la otra imagen, y no al final de la línea epipolar.</p>
<p align="justify">Partiendo de esta idea a nuestra máscara anterior la multiplicamos, otra vez, por una línea que contemplaba únicamente los diez vecinos por la derecha y por la izquierda de nuestra línea epipolar. Con esta segunda restricción bajamos aún más nuestro tiempo de cómputo.</p>

<p align="center"><img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/epipolar.png" alt="Foto Epipolar 1" /><img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/epipolar_2.png" alt="Foto Epipolar 2" />
<img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/epipolar_3.png" alt="Foto Epipolar 3" /></p>
<figcaption align="center">Muestra de varios puntos de interés sobre la línea epipolar tras aplicar las diferentes máscaras</figcaption>
<hr />

<p><strong>Matching</strong></p>
<p align="justify">Una vez que tenemos localizados en qué puntos queremos buscar la correspondencia, tenemos que aplicar un algorítmo que nos indique si estos dos puntos son los mismos o no. Para ello pasamos las imágenes a HSV y optamos por quedarnos únicamente con la componente H. Probamos también a usar sólo con escala de grises, con las tres componentes de HSV y con RGB, donde obtuvimos menos fallos de correspondencias y una mayor precisión a la hora de calcular las correspondencias fue al usar únicamente la H de HSV. Posteriormente recortamos una vecindad en cada imagen de cada uno de los pares de puntos que son candidatos a ser correspondencia, aquí probamos con diferentes tamaños de parches, y nos dimos cuenta que bien con un parche muy pequeño (3x3) o un parche muy grande (41x41) no conseguíamos buenos resultados, por lo que al final optamos por un parche de 11x11, con parches similares a este tamaño obteníamos resultados bastante similares (entre 7-25 píxeles). </p>
<p align="justify">En cuanto al algoritmo de correspondencia, primeramente probamos a usar MSE, con diferentes tamaños de parche a buscar, con esto obtuvimos unos resultados decentes, pero que tenían algunos errores.</p>
<p align="justify">Posteriormente, usamos la función de openCV "matchTemplate". A esta función se le pasan los dos recortes de nuestras dos imágenes y mediante el método "cv2.TM_CCORR_NORMED" nos devuelve como son de parecidos nuestros dos recortes. Siendo 1 el valor en el caso de que sean idénticos y 0 el valor para cuando no se parecen nada. Usando este método, obtuvimos unos resultados mejores. Nos quedaremos únicamente con las correspondencias que tengan un alto nivel de parecido, en nuestro caso, tras varias pruebas, establecimos este umbral en un 0.93</p>

<p><strong>Triangulación 3D</strong></p>
<p align="justify">Una vez que hemos obtenido la correspondencia tenemos que hallar el punto en 3D en el que está el objeto. Para ello, primeramente tenemos que pasar estos puntos de la correspondencia de píxeles (coordenadas 2D) a coordenadas 3D. Para ello usamos los siguientes métodos.</p>
<p><code>pointOptRight = self.camRightP.graficToOptical([indIntPoints[1][pNewPoint], indIntPoints[0][pNewPoint],1])</code></p>
<p><code>pointRight = self.camRightP.backproject(pointOptRight)</code></p>
<p align="justify">Obtenemos el punto en coordenadas 3D y lo retroproyectamos en la cámara. Tendríamos que hacer lo mismo para el píxel de la otra imagen.</p>
<p align="justify">Para encontrar dicho punto en 3D hemos de trazar dos líneas que crucen por cada uno de estos puntos en 3D más otro punto entre el rayo de estos con cada una de sus cámaras y encontrar donde se cruzan. El hecho de que se crucen realmente es muy poco probable, por ello, hallamos el punto medio de la línea más corta entre estos dos rayos. En nuestro caso hemos usado los puntos de origen de cada una de las cámaras y los puntos de los píxeles.</p>
<p>Para calcular dicho punto hemos aplicado la siguiente fórmula, ver <a href="http://www.homer.com.au/webdoc/geometry/lineline3d.htm">enlace</a> </p>

<p align="justify">Para evitar poder haber obtenido malas correspondencias, rechazamos aquellos puntos que tengan una disparidad muy alta.</p>

<p align="justify">Una vez obtenido los puntos en 3D, hallamos el color en RGB correspondiente a ese punto y lo pintamos en el plano 3D mediante la siguiente función:</p>
<p><code>self.drawPoint(p3D/50,pColor)</code></p>

<p align="justify">Por último mostramos unas imágenes donde se ve una reconstrucción total desde diferentes ángulos y un vídeo en el que se muestra una reconstrucción completa a tiempo real, así como el tiempo empleado, número de puntos evaluados y número de puntos pintados.</p>

<p align="center"><img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/full_reco.png" alt="Foto Reconstrucción" /><img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/full_reco_close.png" alt="Foto Reconstrucción de Cerca" />
<img src="https://raw.githubusercontent.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/master/docs/src/3D_Reconstruction/full_reco_lateral.png" alt="Foto Reconstrucción Lateral" /></p>
<figcaption align="center">Muestra de varias vistas de una reconstrucción</figcaption>
<hr />

<p><strong>Vídeo</strong></p>
<p><img src="https://github.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/blob/master/docs/src/3D_Reconstruction/3D_v2.gif?raw=true" alt="Vídeo reconstrucción completa" /></p>

<p> En el sigiente enlace <a href="https://github.com/sergiodomin/MOVA-Vision-Robotica-FollowLine/blob/master/docs/src/3D_Reconstruction/3D_v2.gif?raw=true"> GIF 3D Reconstruction</a> se puede descargar el vídeo para verlo en grande.

<p><strong>Conclusiones</strong></p>
<p align="justify">En esta práctica se ha puesto a prueba el conocimiento de cómo reconstruir una imagen 3D a partir de las imágenes obtenidas de dos cámaras de un sistema canónico estéreo. Hemos evaluado la gran ventaja, en tiempo, que tiene el usar la restricción epipolar, además de las restricciones del filtrado Canny y rango de píxeles propuestas. Por otro lado también se ha investigado el cómo evaluar las correspondencias de una forma eficiente, tanto por el tipo de método a usar, tamaño de ventana y espacio de color.</p>
<p align="justify">Con la aplicación de todo esto, se ha obtenido una reconstrucción bastante completa de la escena (11.815 puntos dibujados) en un tiempo bastante rápido (42 segundos).</p>
