# PROYECTO-FINAL-PREWORK-NUTRISCORE

README  NUTRISCORE PROJECT 3.1 
 Historial de Cambios e Historial de Versiones (v3.0 / v3.1)
Esta 3ª versión oficial del flujo de trabajo introduce mejoras sustanciales en la estabilidad, la experiencia del usuario y el manejo de excepciones mediante las siguientes correcciones y optimizaciones obligatorias:
1.	Sustitución del Trigger Inicial: Se reemplazó el antiguo nodo Webhook genérico por un nodo especializado On form submission, permitiendo que la interacción empiece directamente desde una interfaz de formulario nativa de n8n.
2.	Centralización de Notificaciones de Error y Flujo: Se corrigieron los errores previos de interrupción abrupta del flujo ocasionados por nodos selectores fragmentados. Ahora, la validación de la existencia de Nutriscore y nutrientes críticos se gestiona en un único mensaje inteligente ejecutado por IA al final del flujo.
3.	Resolución del Error HTTP 400: Se solucionó el fallo de Bad Request (400) en el nodo de conexión con la API de Open Food Facts [1] configurando un parámetro User-Agent personalizado e implementando la opción neverError: true para que el flujo no se rompa ante códigos no registrados.
4.	Simplificación de Lógica Condicional: Se reestructuraron y simplificaron los nodos If evaluadores de nutrientes críticos [1] para compactar la toma de decisiones y derivar eficientemente las clasificaciones en tres estados limpios (Saludable, Moderado y No Saludable).
________________________________________
# Arquitectura de Nodos
A continuación se detalla el comportamiento individual de cada componente implementado en el workflow bajo el estándar solicitado:
________________________________________
## NODO: On form submission

    •	ACCIÓN: Capturar datos de interfaz de usuario.
    •	PROPÓSITO: Servir de disparador (Trigger) inicial del flujo proporcionando un formulario amigable para introducir el código de barras.
    •	INPUT: Ninguno (Nodo inicial/Trigger).
    •	OUTPUT: JSON con campo barcode_number (texto introducido por el usuario).
________________________________________
## NODO: Validación Código de Barras
•	ACCIÓN: Validar sintaxis mediante Expresiones Regulares (Regex).
•	PROPÓSITO: Asegurar que el código ingresado cumpla estrictamente con ser numérico y tener una longitud válida de 8 o 13 dígitos antes de llamar a la API externa.
•	INPUT: JSON proveniente del formulario con la variable barcode_number.
•	OUTPUT:
o	ÉXITO (True): Envía el código limpio al nodo de consulta.
o	FAILURE (False): Redirecciona al nodo de error de código no válido.
Ups, algo ha salido mal
“El código de barras enviado está vacío o no es válido. Su extensión debe tener 8 o 13 caracteres y no contener ningún caracter no numérico”
Prueba con un ejemplo como este: 8480000038524
________________________________________
## NODO: ERROR. Código de Barras NO válido
•	ACCIÓN: Mapear variables fijas de contingencia.
•	PROPÓSITO: Estructurar un mensaje de error descriptivo (Código 400 simulado) y proveer un ejemplo sugerido para guiar al usuario.
•	INPUT: Activación desde la salida False del nodo de validación.
•	OUTPUT: Objeto JSON con propiedades Status, ERROR 400 
Ups, algo ha salido mal
“El código de barras enviado está vacío o no es válido. Su extensión debe tener 8 o 13 caracteres y no contener ningún caracter no numérico” 
Prueba con un ejemplo como este: 8480000038524
________________________________________
## NODO: Consulta API Open Food Facts
•	ACCIÓN: Realizar petición HTTP GET.
•	PROPÓSITO: Conectarse a la API pública de Open Food Facts para traer los metadatos y valores nutricionales asociados al producto.
•	INPUT: Variable barcode_number inyectada dinámicamente en la URL de consulta.
•	OUTPUT:
o	ÉXITO: Objeto HTTP con el cuerpo (body) completo de los datos del producto (Nutrientes, Grados, Nombre).
o	FAILURE / 404: Manejado con “neverError”, enviando el objeto de respuesta vacío o con bandera de no encontrado.
________________________________________
## NODO: Validación ¿Existe en Data Base de OFF?
•	ACCIÓN: Evaluar presencia de propiedades.
•	PROPÓSITO: Discriminar si la API de Open Food Facts devolvió un producto válido mapeado con código existente.
•	INPUT: Respuesta completa (body.code) del nodo HTTP.
•	OUTPUT:
o	ÉXITO (True): Deriva al formateador de datos generales.
o	FAILURE (False): Envía el flujo hacia el nodo de error de base de datos.
Ups, algo ha salido mal
“el código de barras parece no existir en la base de datos, ¿quieres intentar con otro producto?” 
________________________________________
## NODO: ERROR. No existe en Base de datos
•	ACCIÓN: Setear variables fijas de respuesta.
•	PROPÓSITO: Definir un mensaje informativo amigable indicando que el producto consultado no está catalogado actualmente.
•	INPUT: Redirección desde la salida False del nodo de existencia en base de datos.
•	OUTPUT: JSON con propiedades status y explicación adaptadas.
________________________________________
## NODO: RECOLECCIÓN INFORMACION NUTRICIONAL
•	ACCIÓN: Normalizar y limpiar variables (Data Cleansing).
•	PROPÓSITO: Extraer del payload crudo de la API únicamente las métricas clave necesarias (Nombre de producto, Nutriscore, Azúcares, Grasas y Sal por cada 100g) para simplificar la lectura posterior.
•	INPUT: Payload JSON exitoso de la API Open Food Facts.
•	OUTPUT: Objeto JSON simplificado con llaves directas: Producto, Nutriscore, Azúcares por 100g, Grasas por 100g y Sal por 100g.
________________________________________
## NODO: CLASIFICACIÓN NUTRISCORE ¿ES SALUDABLE?
•	ACCIÓN: Evaluar condiciones lógicas de cadenas de texto.
•	PROPÓSITO: Determinar si el valor del grado Nutriscore corresponde a las categorías óptimas superiores ("a" o "b").
•	INPUT: Variable Nutriscore normalizada.
•	OUTPUT:
o	ÉXITO (True): Dirige al nodo de asignación saludable.
o	FAILURE (False): Envía al nodo clasificador secundario (No saludable/Moderado).
________________________________________
## NODO: NSCORE CLASIF NO SALUDABLE
•	ACCIÓN: Evaluar igualdad de caracteres.
•	PROPÓSITO: Clasificar los Nutriscore restantes discriminando si corresponden a la categoría intermedia "c" (Moderado) o en su defecto a las categorías bajas "d" o "e".
•	INPUT: Flujo remanente no calificado como saludable óptimo.
•	OUTPUT:
o	ÉXITO (True): Redirecciona al asignador de Nutriscore Moderado.
o	FAILURE (False): Redirecciona al asignador de Nutriscore No Saludable.
________________________________________
## NODO: NSCORE "SALUDABLE" / "MODERADO" / "NO SALUDABLE" (Nodos Set)
•	ACCIÓN: Construir cadenas literales parametrizadas.
•	PROPÓSITO: Declarar la etiqueta cualitativa del Nutriscore acompañada de su correspondiente explicación textual descriptiva.
•	INPUT: Salidas específicas de los respectivos nodos de decisión previos.
•	OUTPUT: JSON formateado con el atributo NUTRISCORE evaluado.
________________________________________
## NODO: CLASIFICACIÓN NUTRIENTES ¿SON SALUDABLES?
•	ACCIÓN: Evaluar múltiples comparaciones numéricas simultáneas (AND).
•	PROPÓSITO: Validar si el 100% de los nutrientes analizados (azúcares, grasas y sal) se encuentran por debajo del umbral preventivo saludable.
•	INPUT: Variables numéricas de azúcares, grasas y sal extraídas de los datos generales.
•	OUTPUT:
o	ÉXITO (True): Deriva al asignador de Nutrientes Saludables.
o	FAILURE (False): Envía al clasificador de nutrientes excedidos.
________________________________________
## NODO: CLASIF NUTRIENTES NO SALUDABLE
•	ACCIÓN: Evaluar condiciones numéricas de exceso (AND).
•	PROPÓSITO: Identificar si todos los valores de azúcares, grasas y sal superan o igualan los umbrales de alerta de manera simultánea para catalogarlos como críticos.
•	INPUT: Nutrientes que fallaron la prueba de saludabilidad total.
•	OUTPUT:
o	ÉXITO (True): Deriva a la asignación de Nutrientes No Saludables.
o	FAILURE (False): Asume que solo algunos superaron las marcas y deriva a la asignación de Nutrientes Moderados.
________________________________________
## NODO: NUTRIENTES "SALUDABLE" / "MODERADO" / "NO SALUDABLE" (Nodos Set)
•	ACCIÓN: Redactar resúmenes descriptivos estructurados.
•	PROPÓSITO: Generar el texto explicativo detallando los miligramos/gramos encontrados de cada componente crítico basándose en el estado lógico asignado.
•	INPUT: Activaciones específicas de las compuertas condicionales de nutrientes.
•	OUTPUT: Objeto JSON con el texto final formateado bajo la propiedad de resultado correspondiente.
________________________________________
## NODO: MERGE NSCORE / MERGE NUTRIENTES
•	ACCIÓN: Fusionar ramas alternativas (Multiplexación).
•	PROPÓSITO: Recoger los flujos salientes paralelos e independientes de las evaluaciones y unificarlos nuevamente en una sola línea de datos limpia para entregar a la Inteligencia Artificial.
•	INPUT: Cualquiera de las 3 entradas activadas de los bloques de seteo cualitativo.
•	OUTPUT: Un único stream unificado con la información consolidada de la evaluación.
________________________________________
## NODO: IA RECOLECCIÓN Y VEREDICTO
•	ACCIÓN: Ejecutar Prompt Engineering a través de un Modelo de Lenguaje Grande (LLM).
•	PROPÓSITO: Consolidar toda la información analizada y estructurar el veredicto final interactivo del producto utilizando un tono amigable, emotivo y preventivo, evitando interrupciones erróneas anteriores.
•	INPUT: Los JSON consolidados de los nodos Merge previos junto con las instrucciones del sistema en el prompt con las siguientes instrucciones:
      Eres un asistente nutricional. Con los datos del producto, retorna una valoración siguiendo EXACTAMENTE estas reglas.

      Input de producto:
        Producto: {{ $('RECOLECCIÓN INFORMACION NUTRICIONAL').item.json.Producto }}
        Nutriscore: {{ $('RECOLECCIÓN INFORMACION NUTRICIONAL').item.json.Nutriscore }}
        Azúcares por 100g: {{ $('RECOLECCIÓN INFORMACION NUTRICIONAL').item.json['Azúcares por 100g'] }}
        Grasas por 100g: {{ $('RECOLECCIÓN INFORMACION NUTRICIONAL').item.json['Grasas por 100g'] }}
        Sal por 100g: {{ $('RECOLECCIÓN INFORMACION NUTRICIONAL').item.json['Sal por 100g'] }}

    Evalúa el Nutriscore:  
        a. A y B: Saludable       🟢
        b. C: Moderado            🟡
        c. D y E: No Saludable    🔴

    Evalúa los nutrientes:
        umbrales por 100g: azúcares ALTO si >= 5, grasas saturadas ALTO si >= 17.5, sal ALTO si >= 1.5
        Cuando azúcares, grasas y sal son BAJOS: Saludable
        Cuando solo uno de los nutrientes es ALTO: Moderado
        Cuando 2 o más nutrientes son ALTOS: No Saludable


    Lógica interna (veredicto final combinando Nutriscore y nutrientes):   
        Cuando nutrientes y Nutriscore son Saludable: Saludable   
        Cuando uno de ellos es Moderado: Moderado   
        Cuando uno de ellos es No Saludable: No Saludable   
        Si nutrientes o nutriscore no existe, retornar el mensaje de "información insuficiente" de una manera amigable e invitando a que busque otro producto en nuestra app. 

    Mensaje de salida:
        Incluye un encabezado que destaque "INFORMACIÓN NUTRICIONAL de {{ $('RECOLECCIÓN INFORMACION NUTRICIONAL').item.json.Producto }}" con emoticonos relevantes.
        Evaluar el nutriscore con cualquiera de los 3 emoticonos semáforo (🟢 verde, 🟡 amarillo, 🔴 rojo)  y su descripción (Saludable, Moderado o No Saludable
        Valora los nutrientes con el mismo criterio (3 nutrientes bajos 🟢; 1 nutriente alto 🟡 y 2 o 3 nutrientes altos).
        Nivel de preocupación con el mismo criterio (0 🟢, 1 🟡 y 2 o 3 🔴)
        Tono amigable que aliente el consumo cuando sea Saludable o Moderado.   
        Tono amigable pero preocupado cuando sea No Saludable.   
        En caso de tratarse de mensajes de error por encontrar un código de barras no válido o de que el producto no se encuentre en la base de datos, devolver el mensaje de error código 400 y especificar el error.  Dar un mensaje similar al output de los nodos set de error.

        Ejemplo de salida:

            INFORMACIÓN NUTRICIONAL para Nutella Biscuits croquants au coeur onctueux de Nutella®

                🔴 NUTRISCORE: E    No Saludable
                🔴 NUTRIENTES: No Saludable
                🔴 NIVEL PREOCUPACIÓN:  Alto

            Este delicioso biscuit tiene un sabor que encanta, pero su contenido de azúcares y grasas es bastante alto y su Nutriscore indica que no es la opción más saludable. Si lo vas a disfrutar, hazlo con moderación y compáralo con alternativas más equilibradas que puedas encontrar en nuestra app. ¡Tu bienestar es lo más importante!  ¿Te gustaría buscar otro producto con mejor perfil nutricional? ¡Explora la sección de opciones saludables y encuentra la alternativa perfecta para ti! 

•	OUTPUT: Cadena de texto enriquecida (con emojis de semáforos 🟢🟡🔴) lista para ser mostrada al usuario final con la recomendación nutricional.
________________________________________
## NODO: Groq Chat Model
•	ACCIÓN: Proveer el backend de computación cognitiva (Inferencia).
•	PROPÓSITO: Suministrar el modelo subyacente openai/gpt-oss-120b a través del proveedor Groq para procesar la cadena del LLM de manera óptima.
•	INPUT: Credenciales e instrucciones de llamada de la API de Groq.
•	OUTPUT: Tokenizador y motor de inferencia conectado nativamente al nodo Basic LLM Chain.
