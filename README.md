# Agente Conversacional de Cobranzas

## 1. Descripción del proyecto

Este proyecto implementa un agente conversacional de cobranzas para orientar a clientes morosos en consultas frecuentes de deuda, medios de pago, alternativas referenciales de pago, reclamos y derivación con asesor humano.

La solución está construida en un notebook de Python usando LangChain 1.x, un agente ReAct creado con `create_agent`, herramientas personalizadas, memoria conversacional persistente en PostgreSQL mediante `PostgresSaver` y trazabilidad opcional con LangSmith.

El objetivo no es reemplazar todas las funciones de cobranza del banco, sino construir una primera solución generativa funcional, controlada y justificable, que atienda consultas de cobranza de forma consistente, respetuosa y sin asumir decisiones de negocio que deben ser validadas por canales oficiales.

---

## 2. Problema o dolor de negocio

Los bancos suelen administrar carteras de clientes con pagos vencidos. El proceso tradicional de cobranza presenta tres problemas principales:

1. **Sesgo operativo del asesor humano**  
   Los asesores pueden priorizar clientes con deuda menor o de cobranza más sencilla para mejorar sus indicadores individuales. Esto puede dejar sin atención a clientes con deuda alta, mora avanzada o posibilidad real de negociación.

2. **Costo y baja escalabilidad**  
   La atención por call center requiere asesores humanos, turnos, supervisión y control de calidad. Un asesor puede atender un número limitado de clientes por día, mientras que un agente conversacional puede atender múltiples conversaciones en paralelo.

3. **Experiencia negativa del cliente**  
   Las llamadas de cobranza pueden percibirse como incómodas, insistentes o agresivas. Esto reduce la disposición del cliente a conversar, explicar su situación o buscar una alternativa de pago.

La solución propuesta busca mejorar este proceso mediante un agente conversacional que atiende al cliente de manera clara, empática y no agresiva, usando información estructurada desde archivos CSV y memoria conversacional para mantener contexto entre turnos.

---

## 3. Usuario objetivo

El usuario objetivo principal es el **cliente moroso** que necesita orientación sobre su situación de deuda y posibles alternativas de pago.

También existe un usuario indirecto: el **equipo de cobranzas del banco**, que puede usar el agente para reducir carga operativa y derivar a asesores humanos solo los casos que requieren revisión especializada.

---

## 4. Alcance funcional

El agente puede atender cinco tipos de consultas:

1. **Consultar deuda**  
   Informa deuda referencial, producto asociado, días de atraso y estado de mora.

2. **Ver medios de pago**  
   Explica canales disponibles y reglas de seguridad.

3. **Negociar un plan referencial**  
   Muestra o simula alternativas de pago según las opciones cargadas en el CSV.  
   El cliente no digita el porcentaje de descuento ni el número de cuotas; esos datos salen de la base.

4. **Presentar reclamo o deuda no reconocida**  
   Si el cliente indica que ya pagó, no reconoce la deuda o existe un pago no aplicado, el agente deriva el caso a revisión humana.

5. **Hablar con un asesor**  
   Cuando el caso excede el alcance del agente, se orienta al cliente hacia un asesor humano.

---

## 5. Fuera de alcance

El agente no realiza acciones transaccionales ni toma decisiones finales de negocio.

En particular:

- No aprueba descuentos reales.
- No registra acuerdos definitivos.
- No registra promesas de pago.
- No crea tickets reales.
- No solicita contraseñas, CVV, tokens ni datos sensibles.
- No inventa información si el DNI no existe en el CSV.
- No atiende temas fuera del dominio de cobranzas.
- No reemplaza la validación formal por canales oficiales del banco.

---

## 6. Análisis previo del caso

### 6.1 Entradas del sistema

El sistema recibe mensajes de texto del cliente. Algunos ejemplos:

- `Hola`
- `Mi DNI es 12345678`
- `Quiero saber cuánto debo`
- `Simula cuotas para mi caso`
- `¿Puedo acceder a un descuento?`
- `Ya pagué y me siguen cobrando`
- `Quiero hablar con un asesor`

### 6.2 Información que necesita consultar

El agente consulta dos fuentes estructuradas en CSV:

| Archivo | Propósito |
|---|---|
| `clientes_cobranza.csv` | Contiene DNI, nombre, producto, deuda total, días de atraso, estado de mora, segmento, capacidad de pago y alternativas de pago cargadas. |
| `politicas_cobranza.csv` | Contiene reglas de seguridad, políticas de descuento, reglas de reclamo, promesa de pago y trato al cliente. |

### 6.3 Decisiones que debe tomar el agente

El agente decide dinámicamente:

- Si necesita pedir DNI.
- Si el DNI informado existe en la base.
- Si debe consultar deuda.
- Si debe mostrar alternativas de pago.
- Si debe simular una opción cargada en el CSV.
- Si debe responder con medios de pago.
- Si debe derivar a un asesor humano.
- Si debe rechazar una consulta por estar fuera del dominio.

### 6.4 Tareas automatizadas

El proyecto automatiza:

- Validación básica de DNI.
- Consulta de deuda referencial.
- Presentación de alternativas de pago desde CSV.
- Simulación de cuotas usando opciones predefinidas.
- Consulta de políticas de cobranza.
- Identificación de reclamos o pagos no aplicados.
- Respuesta de fallback para temas fuera de dominio.
- Resumen de cierre recuperando el estado persistido por `thread_id`.

### 6.5 Partes predecibles del proceso

Son predecibles:

- Validar si el DNI tiene 8 dígitos.
- Verificar si el DNI existe en el CSV.
- Calcular monto final y cuota referencial desde opciones cargadas.
- Mostrar medios de pago.
- Aplicar reglas de seguridad.
- Rechazar temas fuera de cobranza.

### 6.6 Partes que requieren decisión dinámica

Requieren decisión dinámica:

- Interpretar la intención del cliente.
- Decidir qué herramienta usar en cada turno.
- Mantener continuidad conversacional.
- Cambiar de consulta de deuda a simulación o reclamo dentro de la misma conversación.
- Responder de forma natural sin depender de un flujo rígido de preguntas y respuestas.

---

## 7. Arquitectura seleccionada

### 7.1 Tipo de arquitectura

La arquitectura seleccionada es una **arquitectura basada en agente**.

Flujo general:

```text
Usuario
  ↓
Procesamiento de entrada y validación mínima
  ↓
Agente ReAct con create_agent
  ↓
Herramientas personalizadas
  ↓
Memoria conversacional con PostgreSQL / PostgresSaver
  ↓
Respuesta final al cliente
```

### 7.2 Justificación

Se eligió una arquitectura basada en agente porque el proceso de cobranza conversacional no sigue siempre una secuencia fija.

Un cliente puede iniciar saludando, luego consultar deuda, después pedir cuotas, luego indicar que ya pagó y finalmente solicitar un asesor. En ese escenario, un workflow rígido obligaría a clasificar manualmente cada ruta o construir demasiadas ramas condicionales.

El agente permite decidir dinámicamente qué herramienta ejecutar según la intención del mensaje, usando memoria para conservar contexto por `thread_id`.

### 7.3 Por qué no se eligió un workflow puro

No se eligió un workflow puro porque el flujo conversacional no es completamente predecible. El cliente puede cambiar de intención en cualquier momento.

Un workflow sería útil si el proceso fuera siempre:

```text
Identificar cliente → Consultar deuda → Elegir opción → Confirmar
```

Pero el caso real permite reclamos, dudas, solicitudes de asesor, preguntas de seguridad, medios de pago y cambios de intención.

### 7.4 Por qué no se usó arquitectura híbrida

No se usó una arquitectura híbrida con dispatcher externo porque agregaría complejidad innecesaria para el alcance del proyecto.

El agente, junto con herramientas bien definidas y validaciones previas, cubre adecuadamente el caso de uso. Esto aplica el principio de mínima complejidad: usar solo los componentes necesarios para resolver el problema.

---

## 8. Principio de mínima complejidad

El diseño evita componentes que no aportan valor directo en esta etapa.

Decisiones aplicadas:

- Se usa un solo agente principal.
- No se usa dispatcher externo.
- No se usa workflow separado.
- Se reducen las herramientas a cinco funciones claras.
- La base de negocio se mantiene en CSV para facilitar la demo.
- PostgreSQL se usa solo para memoria conversacional.
- LangSmith se usa para trazabilidad, no para tomar decisiones de negocio.
- El agente no registra acuerdos reales ni modifica sistemas transaccionales.

Esto permite una solución más fácil de explicar, probar y defender.

---

## 9. Componentes del sistema

### 9.1 Orquestador

El orquestador corresponde principalmente a la función:

```python
procesar_mensaje_cliente(...)
```

Esta función prepara el mensaje, valida casos básicos, gestiona el `thread_id`, adjunta metadata para trazabilidad y llama al agente.

Responsabilidades:

- Limpiar el mensaje del cliente.
- Detectar saludos simples.
- Validar DNI cuando aplica.
- Controlar temas fuera del dominio.
- Invocar al agente con configuración de memoria.
- Enviar metadata a LangSmith.

### 9.2 Agente

El agente se crea con:

```python
agent = create_agent(
    model=modelo,
    tools=tools,
    system_prompt=SYSTEM_PROMPT_COBRANZA,
    checkpointer=checkpointer,
)
```

El agente decide qué herramienta usar según el mensaje del cliente y el contexto conversacional.

### 9.3 Herramientas personalizadas

El proyecto usa cinco herramientas:

| Herramienta | Función |
|---|---|
| `consultar_deuda_cliente` | Consulta deuda, producto, días de atraso y estado del cliente. |
| `generar_alternativas_pago` | Muestra alternativas disponibles en el CSV para el perfil del cliente. |
| `simular_plan_pago` | Simula una opción cargada en el CSV. No solicita porcentaje ni cuotas al cliente. |
| `consultar_medios_y_politicas` | Devuelve medios de pago y reglas de seguridad o políticas de cobranza. |
| `gestionar_derivacion_o_promesa` | Orienta selección de opción, promesa referencial, reclamo o derivación. |

### 9.4 Memoria conversacional

La memoria se implementa con PostgreSQL mediante:

```python
pool = ConnectionPool(...)
checkpointer = PostgresSaver(pool)
```

El agente recibe el `checkpointer` y conserva el estado por `thread_id`.

Esto permite que el agente mantenga contexto entre turnos. Además, el chat interactivo incluye una función de cierre que reconstruye el historial desde la memoria persistente:

```python
reconstruir_historial_desde_checkpointer(thread_id)
```

Luego genera un resumen con:

```python
resumir_sesion_persistente(...)
```

Esto evita depender solo de un historial local del notebook.

### 9.5 LangSmith

LangSmith se usa para trazabilidad y observabilidad.

Permite revisar:

- Mensajes enviados al agente.
- Herramientas utilizadas.
- Errores de ejecución.
- Metadata por canal, `thread_id` y uso de CSV.
- Flujo de interacción durante la demo.

LangSmith no toma decisiones de negocio; solo permite observar y depurar el comportamiento del agente.

### 9.6 Dispatcher y workflow

No se implementan como componentes separados.

Motivo:

- El dispatcher no es necesario porque el agente decide la herramienta adecuada.
- El workflow no es necesario porque el flujo conversacional no es completamente secuencial.
- Las tareas predecibles se encapsulan como herramientas y funciones auxiliares.

---

## 10. Datos utilizados

### 10.1 `clientes_cobranza.csv`

Contiene clientes de ejemplo para la demo.

Campos relevantes:

- `dni`
- `nombre`
- `producto`
- `deuda_total`
- `dias_atraso`
- `estado`
- `capacidad_pago`
- `segmento`
- `ultimo_pago`
- `descuento_max_pct`
- `cuotas_max`
- `opcion_1_descuento_pct`
- `opcion_1_cuotas`
- `opcion_2_descuento_pct`
- `opcion_2_cuotas`
- `opcion_3_descuento_pct`
- `opcion_3_cuotas`
- `flag_reclamo_abierto`
- `fecha_ultimo_contacto`

### 10.2 `politicas_cobranza.csv`

Contiene reglas operativas de la demo:

- Seguridad.
- Descuentos.
- Reclamos.
- Trato al cliente.
- Promesas de pago.
- Derivación.

### 10.3 Creación automática

El notebook crea o actualiza los CSV en:

```text
/content/data_cobranza/
```

Si los archivos no existen, se generan con datos de ejemplo. Si existen, el notebook conserva la base y agrega columnas faltantes cuando corresponde.

---

## 11. Controles mínimos implementados

| Control | Implementación |
|---|---|
| Validación de entrada | Se valida mensaje vacío y DNI de 8 dígitos. |
| Validación contra base | El DNI debe existir en `clientes_cobranza.csv`. |
| Manejo de errores | Las herramientas retornan mensajes controlados si falta información. |
| Falta de información | Si falta DNI, el agente lo solicita explícitamente. |
| Temas fuera de dominio | Se responde con fallback de cobranza. |
| Acciones sensibles | El agente no aprueba descuentos reales ni registra acuerdos. |
| Datos sensibles | No solicita CVV, claves, tokens ni contraseñas. |
| Simulación controlada | Los descuentos y cuotas salen del CSV, no del cliente. |
| Reclamos | Se derivan a revisión humana. |
| Memoria | Se conserva contexto por `thread_id` en PostgreSQL. |

---

## 12. Flujo de funcionamiento

### 12.1 Flujo normal de consulta y simulación

```text
Cliente: Hola
Agente: Solicita DNI

Cliente: 12345678
Sistema: Valida DNI contra CSV

Cliente: Quiero saber cuánto debo
Agente: Usa consultar_deuda_cliente

Cliente: Simula cuotas para mi caso
Agente: Usa generar_alternativas_pago o simular_plan_pago

Cliente: Me interesa la opción 2
Agente: Usa gestionar_derivacion_o_promesa y explica que la simulación es referencial
```

### 12.2 Flujo de reclamo

```text
Cliente: Mi DNI es 44556677. Yo ya pagué y me siguen cobrando.
Sistema: Valida DNI contra CSV
Agente: Identifica reclamo o pago no aplicado
Agente: Deriva a revisión humana
```

### 12.3 Flujo fuera de dominio

```text
Cliente: Cuéntame un chiste
Agente: Indica que solo atiende temas de cobranza
```

### 12.4 Cierre del chat

```text
Cliente: XEND
Sistema: Recupera historial persistido por thread_id
Agente: Genera resumen de sesión
```

---

## 13. Ejecución del proyecto

### 13.1 Requisitos

El notebook instala las librerías principales:

```python
!pip install -q langsmith langchain langchain-openai langgraph langgraph-checkpoint-postgres psycopg[binary,pool]==3.2.6 pydantic pandas
```

### 13.2 Archivos de credenciales esperados en Colab

El notebook espera los siguientes archivos:

```text
/content/api_key.txt          # API key de OpenAI
/content/postgrest_bd.txt     # cadena de conexión PostgreSQL
/content/langgraphapi.txt     # API key de LangSmith
```

También puede leer variables de entorno equivalentes:

```text
OPENAI_API_KEY
DB_URI
LANGSMITH_API_KEY
```

### 13.3 Pasos de ejecución

1. Abrir el notebook en Google Colab o Jupyter.
2. Cargar los archivos de credenciales.
3. Ejecutar instalación de dependencias.
4. Ejecutar configuración de secretos.
5. Ejecutar creación o actualización de CSV.
6. Crear modelo LLM.
7. Crear conexión PostgreSQL y `PostgresSaver`.
8. Crear agente con `create_agent`.
9. Ejecutar pruebas funcionales.
10. Ejecutar el chat interactivo:

```python
iniciar_chat_cobranzas(thread_id="chat_cliente_12345678", canal="whatsapp")
```

Para salir:

```text
XEND
```

---

## 14. Pruebas funcionales incluidas

El notebook incluye pruebas para:

1. Saludo inicial.
2. Validación de DNI.
3. Consulta de deuda.
4. Simulación de cuotas desde CSV.
5. Selección de opción.
6. Solicitud de descuento sin pedir porcentaje al cliente.
7. Reclamo o pago no aplicado.
8. DNI inexistente.
9. Consulta fuera de dominio.
10. Resumen de cierre desde memoria persistente.

---

## 15. Ejemplo de interacción esperada

```text
Cliente: Hola

Agente: Hola, soy el asistente virtual de cobranzas. Puedo ayudarte con consultas de deuda, medios de pago, alternativas de pago, promesas o reclamos. Para revisar tu caso, indícame tu DNI de 8 dígitos.

Cliente: 12345678

Agente: Gracias, Juan Pérez. Ya validé tu DNI. Puedo ayudarte a consultar tu deuda, revisar medios de pago, simular las alternativas disponibles, orientar un reclamo o derivarte con un asesor.

Cliente: Simula cuotas para mi caso.

Agente: Te puedo mostrar las alternativas cargadas para tu perfil. La simulación es referencial y debe confirmarse por canal oficial o con un asesor.
```

Punto importante: el agente no debe pedir al cliente que digite un porcentaje de descuento. Las opciones salen del CSV.

---


## 18. Valor para el banco

El agente aporta valor en tres dimensiones:

1. **Eficiencia operativa**  
   Permite atender múltiples clientes en paralelo y reducir carga del call center.

2. **Consistencia del tratamiento**  
   Aplica reglas homogéneas de comunicación, seguridad y derivación.

3. **Mejor experiencia del cliente**  
   Usa un tono empático y evita presión indebida, lo que puede facilitar que el cliente continúe la conversación y evalúe alternativas de pago.

El agente no sustituye la decisión humana en casos sensibles; la complementa. Los asesores pueden enfocarse en reclamos complejos, excepciones o negociaciones que requieren validación formal.

---

## 21. Archivos principales

```text
Agente_Cobranzas_Ajustado_Caso_Uso.ipynb   # Notebook principal del proyecto
README.md                                  # Documentación del proyecto
/content/data_cobranza/clientes_cobranza.csv
/content/data_cobranza/politicas_cobranza.csv
```


