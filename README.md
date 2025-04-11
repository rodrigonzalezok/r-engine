# R-Engine: Documentación del Sistema

---

## 1. Guía para Usuarios

### 1.1. Introducción

Este sistema es un **motor de reglas genérico**. Su propósito es tomar un conjunto de datos de entrada sobre un caso específico (por ejemplo, datos de un cliente, atributos crediticios) y aplicar un conjunto de reglas de negocio definidas externamente para llegar a una decisión y generar datos de salida relevantes.

Es "genérico" porque el mismo código Python puede ejecutar diferentes lógicas de negocio (diferentes "modelos") simplemente cambiando el archivo o estructura de configuración de reglas que se le proporciona.

Existen dos modos principales de operación, indicados por un parámetro en la entrada (`fullGCP`):

* **Modo Migrado (`fullGCP: false`):** Genera una salida con una estructura "plana", manteniendo compatibilidad con sistemas anteriores.
* **Modo FullGCP (`fullGCP: true`):** Genera una salida con una estructura diferente, agrupando la información del consumidor principal en `primaryConsumer`.

### 1.2. Cómo Usar el Sistema

Para usar el motor, necesitas proporcionarle un único objeto JSON que contenga dos claves principales: `"data"` y `"rules"`.

```json
{
  "data": {
    // ... Objeto con los datos del caso ...
  },
  "rules": {
    // ... Objeto con la configuración de las reglas a aplicar ...
  }
}
```

### 1.3. El Input data
El objeto "data" contiene la información específica de la ejecución que quieres procesar. Se divide en:

**Atributos**: Diccionario con variables, métricas, scores y datos relevantes del caso.

**Parametros**: Diccionario con umbrales y configuraciones específicas para esta ejecución. Permite ajustar el comportamiento sin cambiar las reglas base. Clave importante aquí:

**fullGCP (Booleano true/false)**: Indica el formato de salida deseado (FullGCP o Migrado). Default: false.
VariablesDeEntrada: Diccionario con los identificadores básicos del caso (ej. documento, nombre).

### 1.4. La Configuración rules (¡La Clave!)
El objeto "rules" define toda la lógica que el motor ejecutará. Es la "receta". Sus partes principales son:

**config_id, description**: Nombres para identificar esta configuración.

**decision_keys_config**: Define la estructura del objeto Decision final.

**keys**: Un diccionario que lista las claves que tendrá la decisión y su valor inicial/tipo base (ej. "resolucion": "", "motivos": []). ¡Puedes definir las claves que necesites!

**accumulate_keys**: Una lista con los nombres de las claves (de las definidas arriba) que deben acumular valores en una lista si múltiples reglas coinciden en un grupo exhaustive (ej. ["motivos"]). Otras claves tomarán el valor de la última regla coincidente.

**formulas**: (Uso técnico) Define cálculos intermedios usando expresiones Python (eval). Deben ser creadas y validadas cuidadosamente por el equipo técnico debido a implicaciones de seguridad.

**rule_groups**: Lista de grupos lógicos de reglas de decisión. Cada grupo tiene:

**group_id**: Nombre del grupo.

**strategy**: "exclusive" (se detiene en la primera regla del grupo que se cumple y finaliza la evaluación) o "exhaustive" (revisa todas las reglas del grupo).

**rules**: Lista de reglas del grupo. Cada regla tiene:

**condition**: La lógica a verificar. Puede ser simple (field, operator, value/value_field, cast_to opcional) o compuesta (operator: "AND"/"OR", clauses: [...]). El cast_to ("int", "float", "str", "bool") fuerza la conversión de tipos antes de comparar para evitar errores con datos de entrada que vienen como texto.

**action**: Un diccionario que define qué valores asignar a las claves de Decision configuradas si la condición es verdadera (ej. "resolucion": "RECHAZAR", "motivos": "Score bajo").

**output_assignments**: Define cómo se construyen los campos finales en VariablesDeSalida y otros. Usa bloques condicionales (basados en la Decision final o Parametros.fullGCP) y diferentes tipos de asignación (direct, static, conditional_source, formatted_value, clear_list).

**output_configuration**: Configuraciones avanzadas (ej. limpieza condicional de datos).

**default_decision**: Qué objeto Decision (con la estructura configurada) se asigna si ninguna regla en ningún grupo coincide.

## Ejemplo Completo de Configuración rules
Este ejemplo muestra muchas de las funcionalidades disponibles. Puedes usarlo como base y modificarlo según tus necesidades. Nota: Este ejemplo es ilustrativo y debe adaptarse a la lógica de negocio específica.

```json

{
  "config_id": "EjemploCompleto_v1",
  "description": "Ejemplo demostrativo con diversas funcionalidades",

  "decision_keys_config": {
    "keys": {
      "estado_final": "PENDIENTE",
      "codigo_resultado": null,
      "motivos_rechazo": [],
      "alertas": [],
      "requiere_revision": false
    },
    "accumulate_keys": ["motivos_rechazo", "alertas"]
  },

  "formulas": [
    {
      "id": "ratio_deuda_ingreso",
      "output_field": "_calculated.ratio_di",
      "expression": "(Atributos.get('deuda_total', 0) / Atributos.get('ingreso_declarado', 1)) if Atributos.get('ingreso_declarado', 0) > 0 else 999",
      "default": 999
    },
    {
       "id": "edad_cliente",
       "output_field": "_calculated.edad",
       "expression": "Atributos.get('edad', 0)",
       "default": 0
    }
  ],

  "rule_groups": [
    {
      "group_id": "VALIDACIONES_ENTRADA",
      "strategy": "exclusive",
      "rules": [
        {
          "rule_id": "DOC_INVALIDO",
          "condition": {
            "field": "Atributos.codigo_validacion_documento",
            "operator": "<",
            "value": 8,
            "cast_to": "int"
          },
          "action": {
            "estado_final": "RECHAZO_ID",
            "codigo_resultado": "E01_DOC",
            "motivos_rechazo": "Validación de documento insuficiente (<8)."
          }
        },
        {
          "rule_id": "EDAD_MINIMA",
          "condition": {
            "operator": "OR",
            "clauses": [
              {"field": "_calculated.edad", "operator": "<", "value_field": "Parametros.edad_minima"},
              {"field": "_calculated.edad", "operator": "not exists"}
            ]
          },
          "action": { "estado_final": "RECHAZO_EDAD", "codigo_resultado": "E02_EDAD", "motivos_rechazo": "Cliente no cumple edad mínima."}
        }
      ]
    },
    {
      "group_id": "RECHAZOS_NEGOCIO",
      "strategy": "exhaustive",
      "rules": [
        {
          "rule_id": "SCORE_BAJO",
          "condition": {"field": "Atributos.score_riesgo", "operator": "<", "value_field": "Parametros.score_minimo", "cast_to": "float"},
          "action": {"estado_final": "RECHAZO", "codigo_resultado": "R01_SCORE", "motivos_rechazo": "Score inferior al mínimo."}
        },
        {
          "rule_id": "RATIO_DI_ALTO",
          "condition": {"field": "_calculated.ratio_di", "operator": ">", "value": 0.40, "cast_to": "float"},
          "action": {"estado_final": "RECHAZO", "codigo_resultado": "R02_RATIO", "motivos_rechazo": "Ratio Deuda/Ingreso alto."}
        },
         {
          "rule_id": "LISTA_NEGRA",
          "condition": {"field": "Atributos.status_lista_negra", "operator": "==", "value": "ACTIVO"},
          "action": {"estado_final": "RECHAZO", "codigo_resultado": "R03_LISTA", "motivos_rechazo": "Cliente figura en lista interna."}
        }
      ]
    },
    {
      "group_id": "ALERTAS_REVISION",
      "strategy": "exhaustive",
      "rules": [
          {
              "rule_id": "ALERTA_ZONA",
              "condition": {"field": "Atributos.zona_postal", "operator": "in", "value_field": "Parametros.zonas_revision"},
              "action": { "requiere_revision": true, "alertas": "Zona postal de revisión." }
          },
          {
              "rule_id": "ALERTA_INGRESO_ALTO",
              "condition": {"field": "Atributos.ingreso_declarado", "operator": ">=", "value": 5000000, "cast_to": "float"},
              "action": { "requiere_revision": true, "alertas": "Ingreso elevado, requiere verificación." }
          }
      ]
    },
    {
      "group_id": "APROBACION_FINAL",
      "strategy": "exclusive",
      "rules": [
        {
          "rule_id": "APROBAR",
          // Esta condición se evalúa DESPUÉS de los grupos anteriores
          "condition": { "field": "Decision.estado_final", "operator": "not in", "value": ["RECHAZO_ID", "RECHAZO_EDAD", "RECHAZO"] },
          "action": { "estado_final": "APROBADO", "codigo_resultado": "A000", "motivos_rechazo": [] } // Limpia motivos si aprueba
        }
      ]
    }
  ],

  "output_assignments": {
    "assignment_blocks": [
      { // Asignaciones Comunes (siempre al final)
        "block_id": "COMUNES_FINALES",
        "condition": {},
        "assignments": [
           {"target": "VariablesDeSalida.decision_motor", "source": "Decision.estado_final"},
           {"target": "VariablesDeSalida.codigo_motor", "source": "Decision.codigo_resultado", "default": "S/C"},
           {"target": "VariablesDeSalida.motivos_decision", "source": "Decision.motivos_rechazo"},
           {"target": "VariablesDeSalida.edad_calculada", "source": "_calculated.edad", "default": 0},
           {"target": "VariablesDeSalida.necesita_revision", "source": "Decision.requiere_revision", "default": false}
        ]
      },
      { // Asignaciones específicas si APROBADO y FULL_GCP
        "block_id": "APROBADO_GCP",
        "condition": { "operator": "AND", "clauses": [ {"field": "Parametros.fullGCP", "operator": "==", "value": True}, {"field": "Decision.estado_final", "operator": "==", "value": "APROBADO"} ]},
        "assignments": [
            {"target": "VariablesDeSalida.limite_credito", "source": "Atributos.limite_preaprobado", "default": 0},
            {"target": "VariablesDeSalida.tasa_interes", "type": "static", "source": 0.25},
            { "target": "VariablesDeSalida.codigo_especial", "type": "formatted_value", "source": "Atributos.cvar_36", "default": "*",
              "formatting_rules": [ {"condition": {"operator": "==", "value": "-"}, "result": "*"}, {"replace": {"find": "R", "with": "", "ignore_case": True}} ] }
        ]
      },
      { // Asignaciones si NO APROBADO
        "block_id": "NO_APROBADO",
        "condition": {"field": "Decision.estado_final", "operator": "!=", "value": "APROBADO"},
        "assignments": [
             {"target": "VariablesDeSalida.limite_credito", "type": "static", "source": 0},
             {"target": "VariablesDeSalida.tasa_interes", "type": "static", "source": null}
             // Podrías añadir más defaults para rechazos aquí
        ]
      }
    ]
  },

  "output_configuration": {
     "conditional_logic": [
         {
             "trigger_field": "Decision.estado_final",
             "trigger_values_for_clearing": ["RECHAZO_ID", "RECHAZO_EDAD", "RECHAZO"],
             "clear_sensitive_data": True,
             "fields_to_clear_ref": "campos_sensibles_rechazo"
         }
     ],
     "field_lists": {
         "campos_sensibles_rechazo": [
             "VariablesDeSalida.limite_credito",
             "VariablesDeSalida.tasa_interes",
             "Atributos.score_riesgo"
         ]
     }
  },

  "default_decision": { // Usa la estructura definida en decision_keys_config
      "estado_final": "INDETERMINADO",
      "codigo_resultado": "NO_MATCH",
      "motivos_rechazo": ["Ninguna regla aplicó al caso."],
      "alertas": [],
      "requiere_revision": true
  }
}
```

### 1.5. Entendiendo la Salida (Actualizado)
El motor devuelve una lista JSON [ { ... } ]. La estructura del objeto interno depende de Parametros.fullGCP:

**Si fullGCP: false** (Modo Migrado): Estructura "plana" con Atributos, Parametros, VariablesDeEntrada, Decision (con las claves que definiste en decision_keys_config), VariablesDeSalida, Formulas.

**Si fullGCP: true** (Modo FullGCP): Estructura con primaryConsumer (que contiene variablesDeEntrada, variablesDeSalida, decision procesadas) y en la raíz atributos, variablesDeEntrada (originales), parametros.

El objeto Decision dentro de la salida contendrá las claves y valores resultantes de la ejecución de las reglas, según lo configurado en decision_keys_config.

### 1.6. Manejo de Errores (Actualizado)
**Errores Internos**: Si el motor falla (error en eval, configuración inválida grave), la salida estándar NO será el JSON de resultado. En su lugar, se imprimirán mensajes de log técnicos en la salida de error (stderr) indicando el problema.

**Errores de Casting (Tipos de Datos)**: Si una regla especifica "cast_to" y la conversión falla, se registrará un ERROR en los logs (visibles en stderr si falla la ejecución total, u opcionalmente en la clave "Logs" de la salida exitosa), indicando la regla y el campo. La condición fallará, y el motor continuará.

## 2. Documentación Técnica
### 2.1. Visión General
El script python_rules.py implementa un motor de reglas genérico configurable vía JSON (main(datos)). 

**Principios clave**: separación lógica/ejecución, configuración externa total (rules), flexibilidad (dos formatos de salida, estructura de Decision configurable vía rules['decision_keys_config']), manejo explícito de tipos (cast_to), trazabilidad vía logs.

### 2.2. Flujo Principal (main())
La función main() sigue estos pasos orquestados:

**Logging y Preparación**: Configura logging temporal, valida input, deepcopy de data.

**Configuración de Decisión**: Lee rules['decision_keys_config'] (keys, accumulate_keys) y rules['default_decision'].

**Flag fullGCP**: Determina modo desde Parametros.fullGCP.

**Inicialización**: Prepara datos_en_proceso, inicializando _calculated, VariablesDeSalida, VariablesDeEntrada, y Decision (basado en decision_keys_config y default_decision).

**Fórmulas (_ejecutar_formulas)**: Llama a _calcular_formula_eval (usa eval()) para rules['formulas']. Advertencia Seguridad eval().

**Evaluación de Grupos de Reglas**: Itera rules['rule_groups'].

Mantiene match_occurred_overall y decision_final_tomada.

Inicializa decision_actual con default_decision.

Salta el último grupo si es default y ya hubo match.

Para cada regla, evalúa condition con _evaluar_condicion (que maneja cast_to).

Si coincide:
* Resetea decision_actual en el primer match global usando decision_keys_config.
  
* Marca match_occurred_overall = True.
  
* Aplica rule['action']: itera key: value. Si key está en accumulate_keys, añade value a decision_actual[key]; si no, decision_actual[key] = value.
  
* Si strategy es exclusive, marca decision_final_tomada = True y rompe bucles.
  
* Al final de grupo exhaustive, aplica valores acumulados.
  
* Actualiza datos_en_proceso['Decision'].
  
* La decision_final_obj es la que queda en datos_en_proceso['Decision'] al final (o default_decision).
  
* Asignaciones (_aplicar_asignaciones_salida): Modifica datos_en_proceso según rules['output_assignments'].
  
* Construcción Respuesta Final: Crea resultado_final_dict (Migrado o FullGCP) usando datos finales de datos_en_proceso. El objeto Decision tendrá la estructura configurada.
  
* Limpieza (_aplicar_limpieza_condicional): Modifica resultado_final_dict.
  
* Errores y Logs (try...except...finally): Captura excepciones, recupera logs, limpia handler.
  
* **Retorno:** Devuelve [resultado_final_dict] (éxito) o [dict_error] (fallo, logs en stderr).

### 2.3. Funciones Auxiliares
**_obtener_valor_anidado(diccionario_datos, ruta_campo, default=None)**:

```Python

"""
Obtiene un valor de un diccionario posiblemente anidado usando una ruta ("clave1.clave2.clave3").
Devuelve un valor por defecto si la ruta no existe o es inválida, garantizando un acceso seguro.
"""
Elaboración: Lee datos de Atributos, Parametros, etc., referenciados en condiciones y asignaciones, sin generar KeyError.*
```

**_establecer_valor_anidado(diccionario_datos, ruta_campo, valor)**:

```Python

"""
Establece (o crea) un valor en un diccionario anidado usando una ruta ("clave1.clave2.clave3").
Crea diccionarios intermedios si no existen para asegurar que la asignación sea posible.
"""
Elaboración: Permite a asignaciones y fórmulas escribir resultados en estructuras como VariablesDeSalida o _calculated.*
```

**_intentar_casting(valor, tipo_objetivo, contexto_error)**:

```Python

"""
Intenta convertir un valor al tipo_objetivo ('int', 'float', 'str', 'bool') de forma segura.
Registra un error detallado si la conversión falla y devuelve un indicador especial (_CASTING_FAILED).
Es usado por _evaluar_condicion para asegurar comparaciones correctas.
"""
Elaboración: Provee robustez ante tipos inconsistentes. El log de error incluye regla y campo para depuración.*
```

**_evaluar_operador(valor_actual, operador, valor_objetivo, tipo_esperado=None)**:

```Python

"""
Realiza la comparación lógica (ej: '==', '<', 'in', 'exists') entre dos valores (posiblemente ya casteados).
Contiene la lógica específica para cada operador soportado por el motor.
Devuelve True o False basado en el resultado de la comparación.
"""
Elaboración: Desacopla la lógica de operadores. Asume tipos mayormente correctos tras _intentar_casting.*
```

**_evaluar_condicion(datos_entrada, condicion, regla_id='N/A')**:

Python

"""
Evalúa una estructura de condición (simple o compuesta con AND/OR) definida en las reglas.
Maneja el casting de tipos opcional (usando _intentar_casting) antes de comparar (con _evaluar_operador).
Es recursiva para manejar condiciones anidadas y determina si una regla aplica.
"""
Elaboración: Interpreta la estructura condition, orquesta casting y comparación, y maneja lógica AND/OR recursiva.*

**_calcular_formula_eval(id_formula, expresion, datos_entrada, default=None)**:

```Python

"""
Calcula una fórmula ejecutando una expresión Python (string) mediante `eval()`. (¡RIESGO SEGURIDAD!)
Construye un contexto limitado para `eval` con acceso a datos como Atributos, Parametros, etc.
Devuelve el resultado del cálculo o un valor por defecto en caso de error.
"""
Elaboración: Permite lógica compleja externa vía eval, pero requiere validación estricta de la fuente de expresion.*
```

**_ejecutar_formulas(datos_entrada, definiciones_formulas)**:

```Python

"""
Orquesta el cálculo de todas las fórmulas definidas en la configuración (`rules['formulas']`).
Llama a _calcular_formula_eval para cada una y guarda los resultados en `datos_entrada['_calculated']`.
"""
Elaboración: Ejecuta pre-cálculos antes de la evaluación de reglas.*
```

**_formatear_valor_asignacion(valor_original, reglas_formato, datos_completos)**:

```Python

"""
Aplica una serie de reglas de formato/transformación definidas a un valor dado.
Permite aplicar lógica condicional, reemplazos de texto, o usar un valor por defecto de otra fuente.
Usado en asignaciones de salida para formatear valores antes de incluirlos en la respuesta.
"""
Elaboración: Provee transformaciones declarativas y seguras controladas por formatting_rules.*
```

**_aplicar_asignaciones_salida(datos_en_proceso, config_asignaciones)**:

```Python

"""
Modifica los datos (`datos_en_proceso`) aplicando las asignaciones definidas en `rules['output_assignments']`.
Evalúa condiciones de bloques y ejecuta asignaciones (directas, estáticas, condicionales, formateadas, etc.).
Construye principalmente `VariablesDeSalida`, pero puede modificar cualquier parte de `datos_en_proceso`.
"""
Elaboración: Interpreta output_assignments, evaluando condiciones y aplicando lógica de asignación según el type.*
```

**_aplicar_limpieza_condicional(respuesta_final, config_reglas, decision_final)**:

```Python

"""
Limpia (pone a default: "", 0, None) campos específicos en la `respuesta_final`.
La limpieza se activa según la `decision_final` y la configuración en `rules['output_configuration']`.
Útil para remover datos sensibles en casos de rechazo, por ejemplo.
"""
Elaboración: Lee output_configuration, verifica triggers y aplica limpieza a campos listados en field_lists.*
```
