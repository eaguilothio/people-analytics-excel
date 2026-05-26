# People Analytics — Hospital Universitari Nexeus
¿Existe realmente un problema de rotación en Urgencias?

---

## ¿Cuál era el problema de negocio?

Quise plantear una problemática habitual en el sector sanitario: la rotación de personal. Para contextualizar el proyecto, situé el análisis en un hospital ficticio — el Hospital Universitari Nexeus — asumiendo una problemática típica de recursos humanos ( la rotación de personal) y desde el rol de analista de Recursos Humanos.

La problematica era la siguiente: existía la percepción de que el departamento de Urgencias estaba experimentando una mayor pérdida de profesionales que el resto del hospital. La sospecha parecía razonable, ya que se trata de un servicio caracterizado por una elevada presión asistencial, trabajo a turnos y una mayor exposición al desgaste profesional.

Sin embargo, una percepción no siempre refleja la realidad. Por ello, se solicitó un análisis que permita responder dos cuestiones fundamentales:

Determinar si la rotación del hospital se encuentra realmente en niveles preocupantes.
Verificar si el problema está concentrado en Urgencias o si afecta a otras áreas de la organización.

---

## ¿Qué datos usé y de dónde salieron?

Dataset simulado con IA (Claude) con información típica de empleados de un hospital.
Las variables cubren diferentes dimensiones: identificación y características del empleado (como edad o género), condiciones del puesto (como departamento o turno), trayectoria (como la antigüedad), y señales de riesgo de rotación (como las horas extra).

La idea era tener suficiente información para poder hacerme hipótesis.

**300 filas (300 empleados únicos que forman la plantilla media, la foto fija de la plantilla) · Columnas originales (23):**
`id_empleado` `departamento` `manager_id` `es_jefe` `tipo_contrato` `modalidad` `turno`
`fecha_ingreso` `fecha_salida` `estado` `antiguedad_meses` `edad` `genero` `nivel_educativo`
`categoria_profesional` `seniority` `banda_salarial` `puntuacion_desempeno`
`horas_extra_mensuales` `absentismo_dias_ano` `training_completado_pct`
`satisfaccion_onboarding` `engagement_score` `motivo_salida`

**Temporalidad: 2018-2026**

---

## ¿Qué herramientas usé y por qué?

- **Excel** 

Es una de las herramientas de análisis de datos más utilizadas en el entorno empresarial, por lo que resulta relevante trabajar con ella.
Permite realizar un análisis de datos de forma eficiente sin necesidad de programación avanzada.
El objetivo principal del proyecto era desarrollar un buen análisis de datos y extraer conclusiones útiles, más que centrarse en la creación de visualizaciones.

---

## ¿Cómo organicé el Excel?

- `datos_crudos` — CSV original. Nunca se modifica.
- `hoja_trabajo` — Copia operativa donde se limpia, transforma y analiza.
- `diccionario_datos` — Definición de cada variable: nombre, tipo, valores posibles y descripción.

> **Colorear encabezados por tipo de dato** — facilita la lectura de un vistazo:
> - Texto/ID: azul
> - Categórico: azul claro
> - Fecha: verde
> - Numérico: naranja
> - Calculado: gris

---

## El proceso analítico de principio a fin

### BLOQUE 1 — Exploración y limpieza

---

#### Paso 1 — Importación y tipado inicial

¿Por qué importar algunas columnas como texto?

- **IDs:** Excel los interpreta como números y puede hacer sumas o eliminar ceros iniciales. Un ID no es un número — es una etiqueta única.
- **Fechas:** Excel las interpreta según la configuración regional del sistema y puede convertir fechas incorrectamente sin avisar. Importar como texto y convertir manualmente garantiza que ves exactamente lo que hay en el fichero.

---

#### Paso 2 — Convertir a tabla y exploración inicial

1. Convertir a tabla para que los filtros sean estables y no se descuadren al añadir o eliminar filas.
2. Revisión columna por columna para entender qué contiene cada variable y detectar dónde puede haber problemas antes de tocar nada.

#### Paso 3 — Duplicados

Cada fila representa un empleado. Si un mismo empleado aparece dos veces, cualquier recuento estará inflado. 

Se encontraron 4 duplicados (EMP-015, EMP-042, EMP-078, EMP-130). Tras eliminarlos: registros válidos.

---

#### Paso 4 — Nulos

Un nulo puede significar dos cosas muy distintas: un error de registro o información válida. Eliminarlos sin entender su significado destruye datos reales.

- `fecha_salida` — nulo significa que el empleado sigue activo. Se rellena con la fecha actual para calcular la antigüedad con la misma fórmula que las bajas.
- `satisfaccion_onboarding` — encuesta no completada. Se rellena con `encuesta_no_completada` para distinguir "el dato no existe" de "la encuesta no se hizo".
- `motivo_salida` — sin entrevista de salida. Se rellena con `sin_motivo_baja` para distinguir empleados activos de bajas sin registrar.
- `manager_id` — el empleado es su propio manager. Se rellena con `propio_manager` para evitar que las tablas dinámicas ignoren esa celda vacía.
- `antiguedad_meses` — baja sin dato calculado. Se deja vacío — no se inventa un dato, se calcula en el paso siguiente.

---

#### Paso 5 — Errores tipográficos en variables categóricas

Excel trata `"Urgencias"`, `"urgencias"` y `"URGENCIAS"` como tres departamentos distintos. Cualquier agrupación posterior estaría contaminada.

Formato estándar elegido: **minúsculas con guion bajo** — consistencia y compatibilidad con cualquier herramienta posterior.

---

#### Paso 6 — Outliers en variables numéricas

Un outlier puede ser un error tipográfico o un valor real extremo. La decisión nunca es automática — hay que contrastar con la fuente.

- `edad` · 17 — menor de edad, imposible. Se elimina la fila.
- `edad` · 91 — fuera de rango laboral real. Se elimina la fila.
- `horas_extra_mensuales` · 180 — error tipográfico evidente. Se corrige a 18h.

Los tres casos se confirmaron con RRHH antes de actuar.

---

#### Paso 7 — Variables calculadas

- `antiguedad_meses` — el dataset original no calculaba la duración de los empleados que causaron baja. Sin esta variable no se puede responder cuánto tiempo aguanta la gente antes de irse. Se divide la diferencia de fechas de salida y fechas de ingreso entre 30,44 — media real de días por mes: 30.44 = 365 / 12 —. Redondeamos a enteros, ya que trabajamos con meses completos porque es la unidad que tiene sentido para RRHH.
- `tramo_antiguedad` — trabajar con meses no tiene significado en RRHH. Así que establecemos Cuatro tramos: `0-12m` (riesgo early attrition), `1-3a` (consolidación), `3-6a` (empleado consolidado), `+6a` (empleado senior).
- `abandono_temprano` — Marca en respuesta binaria ( sí o no) si un empleado se fue antes de cumplir el primer año. 
- `es_salida_voluntaria` — separa fuga de talento (el empleado elige irse) de rotación gestionada (despido, fin de contrato, jubilación). Sin esta distinción cualquier estrategia estaría mal dirigida.

---

### BLOQUE 2 — Análisis por hipótesis en cascada 

---

#### Hipótesis A — ¿Existe un problema real?

Se crea la variable año_fecha_salida para analizar la evolución de las bajas a lo largo del tiempo.

Como indicador principal, se utiliza el Turnover Rate (tasa de rotación), estableciendo como referencia un benchmark del 15 % para el último año que tenemos datos completos: 2025.

Los resultados muestran que la tasa de rotación en 2025 es del 5,33 %, un valor significativamente inferior al benchmark. 
Este porcentaje se obtiene a partir de 16 bajas registradas en 2025 sobre una plantilla media estimada de 300 empleados. 
En los años anteriores también se observan niveles de rotación reducidos.

No obstante, al analizar los motivos de salida, se observa que la baja voluntaria es la causa predominante, con 40 de las 88 bajas totales registradas, superando al resto de motivos en prácticamente todos los años analizados.

Por otro lado, 2024 parece haber sido el año más problemático en términos de salidas, mientras que en 2025 no se aprecia una situación preocupante según los indicadores disponibles.

Conclusión provisional: no existen evidencias de un problema general de rotación en la organización, ya que la tasa de turnover se mantiene claramente por debajo del umbral de referencia. Sin embargo, el peso de las bajas voluntarias justifica seguir profundizando en el análisis. 												

---

#### Hipótesis B — ¿Es Urgencias donde se pierde el talento?

La percepción inicial apuntaba a que el departamento de Urgencias podía presentar mayores índices de rotación debido a las características propias del servicio: presión asistencial elevada, trabajo a turnos y una mayor exposición al desgaste profesional.

Para contrastar esta idea, los resultados muestran que no se registró ninguna baja voluntaria en Urgencias durante 2025. Por tanto, la hipótesis inicial no queda respaldada por los datos disponibles.

#### Conclusiones

La percepción inicial apuntaba a Urgencias como foco de pérdida de profesionales, dado el nivel de presión asistencial del servicio. Los datos no respaldan esa percepción: no se registró ninguna baja voluntaria en Urgencias durante 2025.

Tampoco se identifican otros departamentos problemáticos. El máximo de bajas voluntarias en cualquier área es de 2, un volumen insuficiente para establecer patrones o señalar focos de riesgo.

Cabe matizar que los porcentajes pueden resultar engañosos: aunque las bajas voluntarias representan el 31,25% de las salidas en 2025, el conteo absoluto es de apenas 5 casos sobre una plantilla de 300 empleados. Un porcentaje alto sobre una base pequeña no equivale a un problema de escala.

Conclusión: no existe evidencia de un problema de rotación voluntaria en la organización. La tasa de turnover se mantiene muy por debajo del benchmark sectorial y las salidas voluntarias están dispersas, sin concentración significativa en ningún área.

