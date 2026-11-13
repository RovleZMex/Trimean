# csv-eda-mac (nombre de trabajo)

App nativa de Mac para el primer vistazo a un dataset (EDA): abre CSV y Parquet de cualquier tamaño, muestra tipos, faltantes y distribuciones por columna, aplica operaciones de limpieza sencillas como una lista de pasos editable y genera el código equivalente en **pandas** y **polars**.

No reemplaza a Python: es el paso anterior al notebook.

- SwiftUI, macOS 26+
- Motor: [DuckDB](https://duckdb.org)
- Sin IA, sin cajas negras: cada número lleva a las filas que lo producen.

Estado: en diseño. Ver [`docs/especificacion.md`](docs/especificacion.md).

## Licencia

MIT
