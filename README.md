# Multitienda del Sur SpA - Proyecto BI, Entrega 1

**IIC 3752 · Tecnologías para Inteligencia de Negocios · Escuela de Ingeniería UC · 2do semestre 2026 · Grupo 12**

**Pregunta de negocio (README del caso, pregunta 1):** ¿La cadena está reteniendo a sus clientes o los está reponiendo con compradores de una sola vez?

Este repositorio contiene la cadena de datos de la Entrega 1: lee las cuatro fuentes del caso, detecta los problemas de calidad, aplica decisiones de limpieza documentadas, construye el modelo dimensional, calcula la métrica bajo las dos definiciones y genera los gráficos del informe.

---

## Cómo reproducir los resultados

### Opción A — Google Colab (recomendada)

1. Abrir el notebook: [**E1_pipeline.ipynb en Colab**](https://colab.research.google.com/github/matiasbasm/retail_multitienda_grupo_12/blob/master/E1_pipeline.ipynb)
2. Menú **Entorno de ejecución → Ejecutar todo**.
3. La primera celda clona este repositorio dentro de la sesión, así que no hay que subir archivos ni montar Drive. Colab ya trae todas las librerías necesarias.

### Opción B — Local

```bash
git clone https://github.com/matiasbasm/retail_multitienda_grupo_12.git
cd retail_multitienda_grupo_12
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute E1_pipeline.ipynb --inplace
```

### Cuánto demora

Menos de 30 segundos de ejecución, más unos segundos de clonación en Colab.

### Qué se espera ver al final

La última celda imprime la tabla de cifras del informe, que también queda en `outputs/tablas/cifras_informe.csv`. Con los datos actuales, las principales son:

| Cifra | Valor esperado |
|---|---|
| Boletas originales / válidas del período | 6.321 / 6.222 |
| Problemas de calidad detectados | 27 |
| % de boletas originales excluidas de la métrica | 14,3% |
| Retención de la base activa, W = 90 días (principal) | 36,2% |
| Retención de la base activa, W = 30 días (alternativa) | 14,3% |
| % de la base activa al cierre que es nueva (W = 90) | 69,8% |
| Índice de reposición (W = 90 / W = 30) | 1,71 / 1,25 |

---

## Qué está en el repositorio y qué no

El repositorio guarda **solo el código y los datos crudos**. Todo lo que genera el notebook se recrea en cada ejecución y está excluido en `.gitignore`.

```
├── E1_pipeline.ipynb        Notebook único (se versiona sin salidas)
├── requirements.txt
├── data/
│   ├── raw/                 Fuentes originales: nunca se modifican a mano
│   │   ├── origen_pos.sqlite
│   │   ├── origen_maestro_productos.csv
│   │   ├── origen_promociones.json
│   │   ├── ipc_2025_respaldo_manual.json   Valores INE copiados a mano en formato de la API
│   │   └── ipc_2025_respaldo_api.json      Respuesta real de la API (se crea al correr con conexión)
│   └── processed/           (no se sube) dimensiones y tablas de hechos
├── outputs/                 (no se sube)
│   ├── calidad/             un CSV por problema detectado, con las filas afectadas, y 00_resumen_calidad.csv
│   ├── tablas/              registro de limpieza, ficha de la métrica, sensibilidades, cobertura, cifras del informe
│   └── figuras/             G1 a G5 y el diagrama del modelo
└── docs/
    └── Acta_4sep.pdf
```

Al guardar el notebook desde Colab en GitHub (*Archivo → Guardar una copia en GitHub*) solo se sube el notebook. Los archivos que crea la sesión de Colab (`data/processed/`, `outputs/`) se pierden al cerrarla; para revisarlos, descárguenlos desde el panel de archivos de Colab.

## Respaldo de la API del IPC

El notebook consulta `https://mindicador.cl/api/ipc/2025` en cada ejecución.

- **Si la API responde**, valida la serie (12 meses, valores numéricos) y guarda la respuesta original, sin modificar, en `data/raw/ipc_2025_respaldo_api.json` junto con la fecha y hora de la consulta. Solo reescribe el archivo si la serie cambió.
- **Si la API no responde**, usa ese respaldo. Si todavía no existe, usa `ipc_2025_respaldo_manual.json`.
- **Para dejar el respaldo en el repositorio** (hay que hacerlo una vez): en Colab, cambiar `DESCARGAR_RESPALDO_IPC = True` en la sección 1.4 y ejecutar. El navegador descarga el archivo; súbanlo a `data/raw/` en GitHub con *Add file → Upload files*.

La API entrega la **variación mensual en %**, no el índice. El notebook encadena las variaciones para construir el índice.

## Cómo está construida la limpieza

El notebook separa **detectar** de **decidir**:

- **Sección 2 — Detección.** Chequeos genéricos que no dependen de conocer los datos de antemano: perfil de formatos por columna, filas que no siguen el patrón de su clave, duplicados exactos y claves repetidas, referencias huérfanas, fechas (con prueba del orden día-mes y del rango), números escritos como texto (con prueba de qué significa el punto), reglas de negocio del detalle, cantidades negativas, columnas redundantes que se contradicen, RUT con dígito verificador, fechas de alta, variantes de escritura en categorías, campañas e IPC. Cada problema queda en un CSV en `outputs/calidad/`.
- **Sección 3 — Decisiones.** Un diccionario `DECISIONES` con una entrada por problema. Las que se pueden decidir con evidencia de los datos usan `"auto"` y se imprime el criterio; el grupo puede fijar otra opción. Las que requieren conocimiento externo (por ejemplo, la región de una comuna) están en `CORRECCIONES_MANUALES`. Cada decisión aplicada queda en `outputs/tablas/registro_limpieza.csv`, la base de la bitácora de supuestos.

## Decisiones principales del análisis

- **Cliente activo:** al menos una boleta válida en los últimos W días. **W = 90** (principal: cubre el 94% de los intervalos reales entre compras) frente a **W = 30** (creencia verbalizada por la gerencia).
- **Cliente nuevo:** según el inicio de la relación, la fecha más temprana entre el alta en el programa y la primera compra, porque hay clientes con compras identificadas antes de su fecha de alta.
- **Grano de `fact_boleta`:** una fila es una boleta válida emitida en el período.
- **Grano de `fact_cliente_mes`:** una fila es un cliente identificado al cierre de un mes del período.
- **Segmento (Clásico/Plus/Black):** se sobrescribe en el origen (SCD tipo 1), así que no se usa para cortar la retención en el tiempo.
