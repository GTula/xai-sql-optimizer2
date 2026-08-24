# motor_SQL — motor SQL explicable

`motor_SQL` es un prototipo de base de datos relacional escrito en Python. El
proyecto permite recorrer de punta a punta el procesamiento de una consulta:
tokenización, construcción del AST, validación semántica, optimización del orden
de los `JOIN`, ejecución y persistencia.

Además del motor básico, incluye una línea experimental de investigación sobre
estimación de cardinalidad y explicabilidad de optimizadores. Compara un orden de
ejecución básico, un optimizador inspirado en Selinger y una integración de
BayesCard; la capa XAI analiza por qué se eligió un plan y contrasta sus
estimaciones con métricas observadas durante la ejecución.

## Funcionalidades

- Parser y validador SQL propios.
- Ejecución de `SELECT`, `INSERT`, `CREATE TABLE`, `CREATE INDEX` y `ANALYSE`.
- Consultas con filtros, expresiones booleanas y múltiples `JOIN`.
- Tipos `INTEGER`, `VARCHAR`, `FLOAT`, `BOOLEAN` y `DATE`.
- Datos, catálogo, estadísticas e índices persistidos en archivos.
- Tres estrategias para ordenar joins:
  - `basic`: conserva el orden escrito en la consulta;
  - `selinger`: usa programación dinámica, cardinalidades y cantidad de valores
    distintos (NDV);
  - `bayes`: usa estimaciones de cardinalidad basadas en BayesCard.
- Explicaciones locales de planes mediante SHAP, con resumen en lenguaje natural,
  factores favorables y desfavorables y comparación con el segundo mejor plan.
- Ejecución instrumentada para comparar features estimadas con features reales.
- Análisis batch de consultas y persistencia de resultados en CSV y JSON.

## Flujo de una consulta

```text
SQL
 └─> Tokenizer ─> Parser/AST ─> Validator ─> Optimizer ─> ExecutionEngine
                           catálogo/estadísticas ↑       └─> tablas CSV
                                      XAI ────────────────> explicación y trazas
```

La clase `Database` es la fachada que conecta estos componentes. Al ejecutar una
sentencia, `ExecutionEngine` la parsea y valida; en los `SELECT`, el optimizador
puede reordenar los joins antes de que el motor los ejecute mediante nested loops.

## Requisitos e instalación

- Python 3.9 o posterior.
- Se recomienda utilizar un entorno virtual.

En PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

El núcleo del motor usa principalmente la biblioteca estándar. Las dependencias
de `requirements.txt` se necesitan para BayesCard y los experimentos de XAI. La
demo que genera gráficos de importancia global también requiere `matplotlib`:

```powershell
python -m pip install matplotlib
```

## Inicio rápido

Ejecutar la prueba del flujo principal:

```powershell
python .\test_minidb.py
```

Ejecutar el ejemplo programático:

```powershell
python .\main.py example
```

Abrir la consola interactiva:

```powershell
python .\main.py repl
```

También se puede indicar el nombre lógico de la base:

```powershell
python .\main.py repl mi_base.db
```

Dentro de la REPL, cada instrucción debe terminar en `;`. `HELP;` muestra la
ayuda y `EXIT;` cierra la sesión.

```sql
CREATE TABLE instructor (
    id INTEGER,
    name VARCHAR(50),
    dept VARCHAR(30),
    salary FLOAT
);

INSERT INTO instructor VALUES
    (1, 'Einstein', 'Physics', 95000),
    (2, 'Mozart', 'Music', 40000),
    (3, 'Darwin', 'Biology', 60000);

ANALYSE instructor;

SELECT name, salary
FROM instructor
WHERE salary > 50000;
```

## Uso desde Python

```python
from database import Database

with Database("investigacion.db", optimizer_type="selinger") as db:
    db.execute("CREATE TABLE department (id INTEGER, name VARCHAR(50))")
    db.execute("INSERT INTO department VALUES (1, 'Physics')")
    rows = db.execute("SELECT id, name FROM department")
    print(rows)
```

`optimizer_type` acepta `basic`, `selinger`, `bayes` o `bayescard`. Ejecutar
`ANALYSE [tabla]` antes de experimentar con los optimizadores actualiza las
cardinalidades y los NDV utilizados por sus modelos de costo.

## Explicabilidad y experimentos

La carpeta `xai/` trata al optimizador como el objeto a explicar, sin modificar
su decisión. Para cada consulta con joins puede enumerar planes candidatos,
extraer features estructurales y de costo, calcular contribuciones SHAP y crear
una explicación contrastiva entre el plan elegido y su alternativa más cercana.

Los principales experimentos son:

```powershell
# Importancia global de features sobre el conjunto de consultas
python .\xai\demo_explainer.py

# Comparación de estimaciones Bayes con trabajo observado en runtime
python .\xai\demo_bayes_runtime_validation.py

# Procesamiento batch de docs/CONSULTAS_100.sql
python .\xai\demo_batch_analysis.py
```

El análisis batch escribe sus resultados en `databases/feature_analysis/`. Entre
ellos se encuentran features estimadas por plan, trazas reales, un resumen de las
consultas y metadatos del experimento. La fidelidad se evalúa comparando el
trabajo estimado y el real, incluyendo q-error y filas intermedias producidas por
cada join.

## Persistencia

Al abrir `Database("mi_base.db")` se genera esta estructura junto al archivo
indicado:

```text
databases/
├── catalog/
│   ├── databases.csv
│   ├── tables.csv
│   ├── columns.csv
│   ├── indexes.csv
│   ├── statistics.csv
│   └── column_stats.csv
└── mi_base/
    ├── <tabla>.csv
    └── <índice>.idx
```

El argumento terminado en `.db` identifica la base, pero los datos se almacenan
actualmente en esta jerarquía de CSV e índices, no dentro de un único archivo de
base de datos.

## Estructura del repositorio

| Ruta | Responsabilidad |
| --- | --- |
| `database.py` | Fachada pública del motor. |
| `sql/` | Tokenizer, parser, AST, validación y optimizadores. |
| `catalog/` | Metadatos y estadísticas persistidos en CSV. |
| `executor/` | Ejecución del AST y operadores físicos. |
| `index/` | Implementación persistente de B-tree. |
| `bayescard/` | Adaptación del estimador y código de BayesCard. |
| `xai/` | Features, SHAP, explicaciones, trazas y análisis batch. |
| `demo_bn_db/` | Dataset pequeño para las demostraciones. |
| `docs/` | Inventario técnico, notas de explicabilidad y consultas de prueba. |

Para un inventario archivo por archivo, consultar
[`docs/DOCUMENTACION_ARCHIVOS.md`](docs/DOCUMENTACION_ARCHIVOS.md).

## Alcance y limitaciones actuales

Este es un prototipo educativo y de investigación, no un DBMS para producción.

- Las tablas se leen desde CSV y los joins se ejecutan en memoria mediante nested
  loops; no hay almacenamiento por páginas ni buffer pool.
- `CREATE INDEX` registra el índice y crea su archivo B-tree, pero todavía no lo
  carga con las filas existentes ni lo usa como camino de acceso en `SELECT`.
- El parser contiene estructuras para `UPDATE` y `DELETE`, pero el motor de
  ejecución aún no implementa esas sentencias.
- No se soportan actualmente `ORDER BY`, `GROUP BY`, subconsultas ni
  transacciones.
- La validación experimental muestra que el modelo Bayes puede asumir filtros
  aplicados antes que los joins, mientras que el executor actual evalúa `WHERE`
  después de ellos. Las trazas de runtime existen precisamente para medir esta
  diferencia.

## Base académica

La integración probabilística toma como referencia **BayesCard: Revitilizing
Bayesian Frameworks for Cardinality Estimation**, de Ziniu Wu, Amir Shaikhha, Rong
Zhu, Kai Zeng, Yuxing Han y Jingren Zhou. El artículo propone usar redes
bayesianas para obtener estimaciones de cardinalidad precisas, interpretables y
eficientes para optimización de consultas.

- [Artículo en arXiv](https://arxiv.org/abs/2012.14743)
- [Implementación original de BayesCard](https://github.com/wuziniu/BayesCard)
