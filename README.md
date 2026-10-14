# Multitienda del Sur SpA — Proyecto BI, Entrega 1

**IIC 3752 · Tecnologías para Inteligencia de Negocios · Escuela de Ingeniería UC · 2do semestre 2026 · Grupo 12**

**Pregunta de negocio (README del caso, pregunta 1):** ¿La cadena está reteniendo a sus clientes o los está reponiendo con compradores de una sola vez?

Este repositorio contiene la cadena de datos completa de la Entrega 1: lee las cuatro fuentes del caso, las limpia dejando registro de cada regla, construye el modelo dimensional, calcula la métrica bajo las dos definiciones y genera los gráficos del informe.

---

## Cómo reproducir los resultados

### Opción A — Google Colab (recomendada)

1. Abrir el notebook: [**E1_pipeline.ipynb en Colab**](https://colab.research.google.com/github/matiasbasm/retail_multitienda_grupo_12/blob/master/E1_pipeline.ipynb)
2. Menú **Entorno de ejecución → Ejecutar todo**.
3. La primera celda clona este repositorio dentro de la sesión, así que no hay que subir archivos ni montar Drive.

No requiere instalar nada: Colab ya trae pandas, numpy, matplotlib y requests.

### Opción B — Local

```bash
git clone https://github.com/matiasbasm/retail_multitienda_grupo_12.git
cd retail_multitienda_grupo_12
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute E1_pipeline.ipynb --inplace
```

### Cuánto demora

Menos de 30 segundos de ejecución, más lo que tarde la clonación del repositorio en Colab (unos segundos).

### Qué se espera ver al final

La última celda imprime la tabla de cifras del informe (también queda en `outputs/tablas/cifras_informe.csv`). Las cifras principales son:

| Cifra | Valor esperado |
|---|---|
| Boletas originales | 6.321 |
| Boletas válidas 2025 | 6.173 |
| % de boletas originales excluidas de la métrica | 15,1% |
| Retención de la base activa, W = 90 días (principal) | 35,9% |
| Retención de la base activa, W = 30 días (alternativa) | 13,9% |
| % de la base activa al 31-dic que es nueva (W = 90) | 70,0% |
| Índice de reposición (W = 90 / W = 30) | 1,71 / 1,23 |
| Ventas no capturadas abr–dic por clientes perdidos (W = 90, $ dic-2025) | $30.424.176 |

Además se generan 5 gráficos en `outputs/figuras/`, el diagrama del modelo en `docs/modelo_dimensional.png` y las tablas del modelo en `data/processed/`.

---

## Estructura del repositorio

```
├── E1_pipeline.ipynb        Notebook único: extracción → limpieza → modelo → métrica → gráficos
├── requirements.txt         Librerías para ejecución local
├── data/
│   ├── raw/                 Fuentes originales, nunca se modifican
│   │   ├── origen_pos.sqlite
│   │   ├── origen_maestro_productos.csv
│   │   ├── origen_promociones.json
│   │   └── ipc_2025_cache.json      Respaldo de la API del IPC
│   └── processed/           Dimensiones y tablas de hechos generadas por el notebook
├── outputs/
│   ├── figuras/             G1 a G5, en PNG para el informe
│   └── tablas/              registro_limpieza, cobertura, ficha_metrica, sensibilidad, caso_negocio, cifras_informe
└── docs/
    ├── Acta_4sep.pdf        Acta de la reunión del 4 de septiembre
    └── modelo_dimensional.png
```

## Fuentes

| Fuente | Formato | Uso en la cadena |
|---|---|---|
| `origen_pos.sqlite` | SQLite (boletas, detalle, clientes, tiendas, comunas, regiones, medios de pago) | Esqueleto del modelo: hechos de boleta y snapshot de actividad por cliente |
| `origen_maestro_productos.csv` | CSV `;`, cp1252 | Costo unitario para el margen bruto; SKU para cruzar promociones |
| `origen_promociones.json` | JSON | Marca de boletas con productos en promoción; recompra de clientes captados con promoción |
| `https://mindicador.cl/api/ipc/2025` | API REST (JSON) | Deflactor: todos los montos se expresan en pesos de diciembre 2025 |

**Sobre la API del IPC.** El notebook consulta la API en cada ejecución. Si no responde (sin red, caída del servicio), usa `data/raw/ipc_2025_cache.json`, que contiene la misma serie con los valores oficiales del INE 2025 en el formato de la API. La celda indica qué origen se usó. La API entrega la **variación mensual en %**, no un índice; el notebook encadena las variaciones para construir el índice (dic-2024 = 100). Al encadenar variaciones redondeadas a un decimal, la inflación acumulada resulta 3,44%; el INE informa 3,5% calculado sobre el índice sin redondear.

## Decisiones principales

Todas están argumentadas en el informe y registradas en `outputs/tablas/registro_limpieza.csv`.

- **Cliente activo:** al menos una boleta válida en los últimos W días. **W = 90** (principal: cubre el 94% de los intervalos reales entre compras) frente a **W = 30** (creencia verbalizada por la gerencia).
- **Cliente nuevo:** fecha de alta en el programa dentro del período, no "primera compra observada" (los datos contradicen que ambas fechas coincidan).
- **Grano de `fact_boleta`:** una fila es una boleta válida emitida en 2025.
- **Grano de `fact_cliente_mes`:** una fila es un cliente identificado al cierre de un mes de 2025.
- **Segmento (Clásico/Plus/Black):** se sobrescribe en origen (SCD tipo 1), por lo que no se usa para cortar la retención en el tiempo.
