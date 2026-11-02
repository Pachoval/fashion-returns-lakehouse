# Fashion Returns Lakehouse

Proyecto de ingeniería de datos y machine learning en Databricks que responde una pregunta de negocio:

> **¿Qué líneas de pedido tienen más probabilidad de ser devueltas, para ajustar guías de talla, fotos de producto y políticas de devolución?**

Las devoluciones son uno de los costos ocultos más grandes de la moda online. El proyecto recorre el camino completo, desde datos crudos hasta un modelo registrado, para que la respuesta sea reproducible y se pueda auditar.

> **Sobre los datos:** todos los datos son **sintéticos**. Se generan con ruido controlado (nulos, tipos mezclados, duplicados) para que cada capa tenga trabajo real. El valor del proyecto está en la arquitectura y las decisiones que hay detrás, no en las cifras de las métricas del modelo.

## Qué contiene

| Fila de la rúbrica | Notebook | Qué demuestra |
|---|---|---|
| Ingesta y transformación | `00_Setup/00_Configuracion`, `01_Generacion_o_Ingesta`, `01_Exploracion_y_Transformacion` | Widgets, catálogo propio, Bronze con errores a propósito, perfil de calidad y limpieza a Silver |
| Delta Lake | `02_Delta_Lake` | `MERGE INTO` para aplicar un lote tardío, Time Travel (`VERSION AS OF`) y `OPTIMIZE ... ZORDER BY`, cada uno justificado |
| Gold | `02b_Gold_Metricas_y_Features` | Tablas de métricas de negocio y tabla de features sin fuga de datos |
| Gobernanza | `03_Gobernanza` | Catálogo propio con esquemas Medallion, `GRANT` documentado y evidencia de dónde están los datos personales |
| Orquestación | `04_Orquestacion` | Job de tareas dependientes creado con el SDK, en Serverless, con schedule pausado |
| Machine Learning | `05_Machine_Learning` | Dos modelos con tracking en MLflow, métricas guardadas y modelo registrado en Unity Catalog |

## Estructura del repositorio

```
fashion-returns-lakehouse/
  README.md
  00_Setup/
    00_Configuracion.ipynb
  01_Generacion_o_Ingesta.ipynb
  01_Exploracion_y_Transformacion.ipynb
  02_Delta_Lake.ipynb
  02b_Gold_Metricas_y_Features.ipynb
  03_Gobernanza.ipynb
  04_Orquestacion.ipynb
  05_Machine_Learning.ipynb
```

Cada notebook empieza con `%run ./00_Setup/00_Configuracion` para heredar los parámetros. El `b` en `02b` sigue la convención del curso: es un paso que depende del anterior (Gold necesita las devoluciones ya aplicadas con el `MERGE`).

## Arquitectura Medallion: qué responde cada capa

| Capa | Qué contiene | Qué pregunta responde |
|---|---|---|
| **Bronze** | Cuatro tablas crudas: `clientes`, `productos`, `lineas_pedido`, `devoluciones` | ¿Qué llegó y cuándo? No responde negocio: conserva el dato original |
| **Silver** | Tablas limpias y tipadas; `fue_devuelto` se actualiza con `MERGE` | ¿Cuántas líneas son válidas y qué problemas de calidad había? ¿Cuáles se devolvieron? |
| **Gold** | Tasa de devolución por categoría y talla, evolución semanal, resumen por cliente y tabla de features | ¿Qué categorías, tallas y clientes devuelven más? ¿Cómo evoluciona la tasa? |
| **ML** | Modelo de clasificación registrado en Unity Catalog | De las líneas nuevas, ¿cuáles se van a devolver? |

**¿En qué momento hace falta ML?** Gold responde con promedios sobre lo que ya pasó. El modelo entra cuando la decisión se toma **por línea y antes de que ocurra la devolución**, combinando muchas variables a la vez (talla, categoría, descuento, historial del cliente), algo que una tabla agregada no puede hacer.

## Decisiones de diseño

- **La etiqueta llega después de la compra.** Las devoluciones son una tabla aparte y un `MERGE INTO` las aplica sobre Silver. Una línea cuenta como devuelta (`fue_devuelto = 1`) si la devolución llega dentro de **30 días**.
- **`MERGE` y no sobrescribir.** Solo se tocan las filas afectadas y la operación es idempotente, así que un reintento de un Job no duplica datos.
- **Partición por `mes`, no por `fecha`.** Evita cientos de carpetas diminutas; la columna `cliente_id`, de alta cardinalidad, se optimiza con `ZORDER`.
- **Sin fuga de datos.** El historial del cliente (`tasa_devolucion_previa`) usa solo líneas con al menos **35 días** de antigüedad, el plazo tras el cual ya se conocen todas sus devoluciones. La división de entrenamiento y prueba es **temporal**.
- **Métricas para clases desbalanceadas.** Se reportan recall, precision, F1 y PR-AUC; no accuracy, porque un modelo que nunca predice devolución ya acierta cerca del 81 %.
- **Mínimo privilegio.** El equipo de BI lee solo Gold; los datos personales (nombre y correo) existen únicamente en Bronze.
- **Aleatoriedad reproducible con `F.hash`.** El mismo sorteo alimenta dos tablas (líneas y devoluciones), y `F.hash` sobre el identificador garantiza que sean consistentes aunque Spark reordene las filas.

## Cómo ejecutarlo

Requiere una cuenta de Databricks (probado para Free Edition, con compute Serverless).

1. Clona este repositorio en un **Git folder** de Databricks.
2. Ejecuta los notebooks **en este orden**:
   1. `00_Setup/00_Configuracion`
   2. `01_Generacion_o_Ingesta`
   3. `01_Exploracion_y_Transformacion`
   4. `02_Delta_Lake`
   5. `02b_Gold_Metricas_y_Features`
   6. `03_Gobernanza`
   7. `04_Orquestacion`
   8. `05_Machine_Learning`
3. Para reconstruir todo en otro catálogo, cambia el widget `catalog_name` y vuelve a correr desde el principio. No hay nombres de tabla escritos a mano.

## Limitaciones

- Los datos son sintéticos y la señal para el modelo fue diseñada por el autor; los resultados no se pueden extrapolar a un negocio real.
- El modelo se registra, pero no se despliega en un endpoint de Model Serving.
- Free Edition es de un solo usuario y no tiene grupos, por lo que el `GRANT` se aplica sobre el usuario actual como reemplazo de un grupo real.
- Free Edition es para uso no comercial.

## Resultados del modelo

_Completar después de ejecutar `05_Machine_Learning`: pega aquí la tabla de `gold.metricas_modelos` (precision, recall, F1, PR-AUC y ROC-AUC de cada modelo) y compárala contra la tasa base de devolución._
