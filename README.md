# Practica 3 VC

Ejercicios de la tercera práctica de laboratorio de VC.

Nicolás Hernández Castro y Gabriel Godoy Navarro - EII ULPGC 20 de octubre de 2026

## Tarea 1: Contar monedas

### Imagen ideal
Inicialmente se va a realizar el conteo de monedas en la imagen ideal proporcionada. En primer lugar, se definen dos diccionarios que asocian cada moneda con su diámetro en milimetros y con su importe decimal en euros. Luego se carga la imagen de partida en formato BGR mediante `cv2.imread('Monedas.jpg')` y se convierte a escala de grises con `cv2.cvtColor`. Para atenuar el ruido de alta frecuencia y los relieves internos sin deformar los bordes exteriores, se aplica un filtro de mediana mediante `cv2.medianBlur(gris, 7)`. Dado que las monedas se encuentran sobre un fondo blanco uniforme, se binariza la imagen con `cv2.threshold(suavizada, 200, 255, cv2.THRESH_BINARY_INV)` para invertir los valores y obtener las monedas en blanco (255) y el fondo en negro (0).

A continuación, se localizan los contornos exteriores con `cv2.findContours(th, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)`. Se descartan posibles imperfecciones mediante un filtro de área mínima (`cv2.contourArea(c) > 500`) y se ajusta cada contorno al círculo mínimo con `cv2.minEnclosingCircle(c)`. Este ajuste calcula el circulo exterior real de cada moneda, almacenando sus centros `(cx, cy)` y sus radios en píxeles.

Para establecer la correspondencia métrica, se implementa una fase de calibración interactiva mediante `cv2.setMouseCallback`, que asocia la función `click_callback` a la ventana de previsualización para registrar las coordenadas `(x, y)` cuando el usuario hace clic izquierdo sobre la moneda conocida de referencia (1 Euro = 23.25 mm). Con la función `np.hypot` se calcula la distancia euclídea del punto de clic al centro de todas las monedas detectadas y se selecciona la más próxima con `np.argmin`. A partir de su radio medido en píxeles, se calcula el factor de conversión métrico (mm por pixel) dividiendo el diámetro teórico entre su diámetro en la imagen.

Finalmente, con esa escala se itera sobre cada círculo detectado calculando su diámetro real estimado en milímetros. Empleando un clasificador por mínima distancia euclídea, se compara este valor frente a una tabla con las dimensiones oficiales en milímetros mediante `min(DIAMETROS_MONEDAS.keys(), key=lambda k: abs(DIAMETROS_MONEDAS[k] - diametro_mm))`. Tras asignar a cada moneda su valor correspondiente, se marca sobre la imagen. 

En el resultado se observa como se suma de forma exacta el importe total de las 8 monedas presentes en la imagen (3.88 €):

![alt text](image.png)

### Demostrador en vivo

Para el demostrador en vivo, se inicializa la captura de fotogramas mediante `cv2.VideoCapture(0)`. Dentro del bucle continuo, cada fotograma se convierte a escala de grises y se suaviza mediante `cv2.GaussianBlur(gris, (9, 9), 2.0)` para atenuar el ruido electrónico del sensor de la cámara. 

La detección de las monedas en cada fotograma se lleva a cabo mediante la transformada de círculos de Hough con `cv2.HoughCircles(gris_blur, cv2.HOUGH_GRADIENT, dp=1.1, minDist=60, param1=100, param2=48, minRadius=30, maxRadius=110)`. Dado que puede generar falsos positivos en zonas vacías del fondo blanco por coincidencias de gradientes o sombras, se implementa un filtro de contraste interior que para cada círculo detectado genera una máscara binaria circular temporal y calcula la intensidad media de gris en su interior con `cv2.mean(gris, mask=mascara)[0]`. Si la intensidad media supera 210, se descarta por tratarse de papel blanco, conservando únicamente aquellas regiones oscuras correspondientes a metal real.

Para garantizar la estabilidad del recuento en vídeo y evitar el parpadeo constante de etiquetas entre fotogramas, se diseñó un sistema de asociación espacial y filtrado temporal:
* Se compara el centroide de cada detección actual con los centros del fotograma anterior usando `np.hypot(x - cx_ant, y - cy_ant) < 20`. Si la distancia es menor a 20 píxeles, se identifica como la misma moneda física sobre la mesa y se actualiza su posición.
* Para cada moneda se mantiene una cola o búfer deslizante con las mediciones de radio de los últimos 25 fotogramas (`len(m['radios']) > 25`). 
* En lugar de clasificar el radio instantáneo, se calcula la mediana estadística (`np.median(m['radios'])`), lo que neutraliza por completo los picos y oscilaciones aleatorias causados por el ruido de cuantización de la cámara.

Al igual que en la imagen estática, la calibración métrica se realiza haciendo clic izquierdo sobre la moneda de referencia de 1.00 €. Con la escala activa, cada moneda se clasifica por mínima distancia frente a la tabla oficial. Además, para contemplar la posible presencia de objetos que no sean monedas o monedas solapadas, se establece una tolerancia de aceptación física de `2.2 mm` (`abs(diametro_teorico - diametro_mm) < 2.2`). Si el diámetro se encuentra dentro del margen, la moneda se valida, se suma a la cuenta global `total_dinero` y se dibuja en verde `(0, 255, 0)` con su importe; si el objeto difiere en exceso de cualquier valor del catálogo, se marca en color naranja `(0, 165, 255)` como objeto no identificado o solapado. En la parte superior de la imagen se proyectan en tiempo real el valor total acumulado, las instrucciones y la escala de calibración mediante `cv2.putText`, finalizando la ejecución al presionar la tecla ESC.

Para preparar la demostración, se preparó cuidadosamente el entorno de captura para minimizar sombras y exceso de luz ambiente. Las monedas se colocaron sobre un folio blanco, la cámara se fijó en una posición cenital específica y se dispuso iluminación auxiliar lateral con linternas para eliminar por completo las sombras proyectadas.

En las prueba se puede observar que cuando las monedas se colocan desde el comienzo y separadas a una distancia parecida, el sistema las detecta y clasifica norlmalmente de forma correcta y estable. En cambio, cuando se añade cada moneda una a una se puede confundir con alguna moneda aunque acertando la mayoría.

![Demostración1](video-1-P3.gif)
![Demostración2](video-2-P3.gif)

Además, los objetos que no son monedas, como el tapón de plástico, no se detectan al ser descartados automáticamente por el filtro de brillo interior (cv2.mean < 210) y la falta de contraste con el folio. Por otro lado, las monedas parcialmente superpuestas sí se detectan y clasifican. Esto ocurre porque la Transformada de Hough no necesita el contorno cerrado completo, bastándole los arcos visibles de la circunferencia para estimar el radio real dentro del margen de tolerancia de 2.2 mm

![Demostración3](video-3-P3.gif)

Uso de la IA: Durante la fase de desarrollo surgieron dos dificultades principales, como la detección de círculos falsos sobre áreas vacías del papel blanco y la fluctuación continua de los valores monetarios entre fotogramas. Para resolver estos problemas, se recurrió a la asistencia de una IA y recomendó implementar una comprobación de contraste interno creando una máscara temporal con `cv2.circle` y evaluando la intensidad media con `cv2.mean`, descartando de forma automática regiones que superaran un umbral de 210 de brillo. También recomendó diseñar una estructura de seguimiento por proximidad espacial euclídea (`np.hypot < 20`) combinada con una cola histórica de radios estabilizada mediante la mediana (`np.median`). Esta solución eliminó los saltos de etiquetas causados por el ruido del sensor sin comprometer la fluidez de la captura en vivo.