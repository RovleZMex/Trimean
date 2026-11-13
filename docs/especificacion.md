# Especificación — Trimean

Fecha: 2026-11-13 · Estado: borrador para revisión

## 1. Por qué existe

El primer vistazo a un dataset casi siempre es igual: abrir el archivo, revisar los tipos de columna, contar los faltantes, ver las distribuciones y limpiar un poco. Con pandas o polars son las mismas diez líneas en cada proyecto. Esta app hace ese primer vistazo en una interfaz nativa de Mac y al final te entrega el código para seguir en Python.

**No reemplaza a Python.** Es el paso anterior al notebook.

## 2. Principios

1. **Primero los datos.** Al abrir un archivo solo aparece la tabla. No hay paneles, gráficas ni resúmenes encima.
2. **Nada de cajas negras.** Cada número se puede rastrear hasta sus filas. Si una columna tiene 10 faltantes, das clic y ves esas 10 filas.
3. **Sin IA ni sugerencias.** La app no opina ni recomienda. Da herramientas y el científico de datos decide.
4. **Lo que ves es lo que copias.** El snippet de pandas o polars debe producir exactamente el mismo resultado que muestra la app (ver §7).
5. **Estética Apple.** Sigue las Human Interface Guidelines y usa los controles del sistema (Liquid Glass, toolbar, inspector, sidebar). Nada de controles hechos a mano cuando existe uno nativo.
6. **Modular.** Agregar una operación nueva debe ser agregar un archivo, sin tocar el resto de la app (ver §6).

## 3. Alcance

### Primera versión
- Abrir **CSV y Parquet**.
- Tabla virtualizada que soporta archivos más grandes que la RAM.
- Análisis por columna en el inspector.
- Lista de pasos con 7 operaciones (§5).
- Exportar el dataset limpio a CSV o Parquet.
- Snippets en pandas y polars, por paso y como script completo.
- Guardar el proyecto y retomarlo.

### Fuera de la primera versión
- Análisis entre columnas: correlaciones, scatter, tablas cruzadas.
- Escalado, encoding, outliers y columnas calculadas.
- Excel, JSON y bases de datos.
- Cualquier función con IA.

### Anti-objetivos (si pasa esto, fallamos aunque funcione)
- El snippet da un resultado distinto al de la app.
- La app modifica el archivo original.
- Al abrir, la pantalla está llena de widgets antes de que el usuario pida algo.
- Una estadística sale de una muestra sin que el usuario lo sepa.

## 4. Experiencia

```
┌──────────────────────────────────────────────────────────────┐
│ ●●●  ventas_2025.csv                        [⊞ Pasos] [ⓘ]    │
├──────────┬───────────────────────────────────┬───────────────┤
│ Pasos    │  id   fecha        monto   región │ monto         │
│ 1 Dedup  │  1    2025-01-03   120.5   Norte  │ Double        │
│ 2 Imputar│  2    2025-01-03   —       Sur    │ 10 faltantes →│
│ ☐ 3 Drop │  3    2025-01-04   88.0    Norte  │ ▁▂▅▇▅▂▁       │
│          │  …                                │ media  104.2  │
│ [+]      │                                   │ [Imputar…]    │
└──────────┴───────────────────────────────────┴───────────────┘
```

- **Al abrir:** solo la tabla. La barra lateral y el inspector empiezan ocultos.
- **Tabla:** carga solo las filas visibles mientras haces scroll. Los encabezados muestran nombre y tipo.
- **Inspector (derecha):** se abre al seleccionar una columna o con su botón. Muestra:
  - Tipo detectado y número de valores no convertibles, si los hay.
  - Conteo y porcentaje de faltantes. Al dar clic, la tabla se filtra a esas filas.
  - Valores distintos.
  - En columnas numéricas: histograma, mínimo, máximo, media, mediana, desviación estándar y cuartiles.
  - En texto y categóricas: los valores más frecuentes con su conteo. Al dar clic en uno, se filtra.
  - En fechas: rango y distribución en el tiempo.
  - Acciones rápidas que abren una operación ya prellenada para esa columna (p. ej. "Imputar…").
- **Antes y después:** al seleccionar un paso, el inspector compara la distribución de la columna afectada antes y después de ese paso.
- **Barra lateral de pasos (izquierda):** lista ordenada que se puede reordenar arrastrando, desactivar con un checkbox, editar y borrar. ⌘Z deshace cualquier cambio en la lista.
- **Snippets:** un panel o hoja con dos pestañas (pandas y polars). Puedes copiar el script completo o el código de un paso.
- **Estadísticas exactas:** se calculan sobre el archivo completo, en segundo plano y con progreso visible. No hay muestreo. Si algún día se agrega muestreo para archivos enormes, se tiene que indicar en la interfaz.

## 5. Operaciones de la primera versión

| Operación | Variantes |
|---|---|
| Quitar duplicados | Todas las columnas o un subconjunto; conservar el primero o el último |
| Imputar faltantes | Media, mediana, moda, constante, forward fill, backward fill |
| Quitar filas con faltantes | En cualquier columna o en un subconjunto |
| Quitar columna | — |
| Renombrar columna | — |
| Cambiar tipo | Texto, entero, decimal, fecha, booleano. Muestra cuántos valores no se pudieron convertir y permite ver cuáles |
| Filtrar filas | Condiciones simples: comparación, igualdad, nulo o no nulo, combinadas con Y/O |

## 6. Arquitectura

### Plataforma
- **SwiftUI**, macOS 26 como mínimo y Swift 6 con concurrencia estricta.
- **DuckDB** como motor, usando el paquete oficial de Swift (`duckdb/duckdb-swift`). Lee CSV y Parquet, ejecuta consultas fuera de memoria y exporta.
  - **Versión fijada:** commit `2bc1adcf863f83994c27a909d475f8185fbabfa3` (DuckDB `v1.6.0-dev11145`, julio 2026). Es un snapshot de desarrollo porque la rama `main` de `duckdb-swift` no compila (sep 2026) y la última etiqueta estable es la 1.1.3, de 2024. Como el motor es el único módulo que conoce DuckDB, cambiar de versión después solo toca ese módulo.
  - **Importación al abrir:** el archivo se importa una vez a una base DuckDB temporal en disco, con un índice de fila `__trimean_row` asignado al leer. Leer el CSV directamente en cada consulta tarda 5.8 s por página a mitad de un archivo de 2.4 GB; ya importado, tarda 2 ms (medido el 2026-11-13, 30 M filas). Importar ese archivo tarda unos 16 s y la base ocupa cerca del 35 % del CSV.
  - La base temporal vive en el directorio temporal de la app, se borra al cerrar la ventana y al iniciar la app se limpian restos. Nunca va dentro del documento `.trimean`.
  - Cada dataset abierto usa dos conexiones: una para la tabla y otra para estadísticas, así un cálculo largo no congela el scroll. El wrapper de Swift no permite interrumpir una consulta en curso, así que una estadística que ya no se necesita termina y se descarta.
- **App basada en documentos** (`DocumentGroup`). Así el proyecto se abre con doble clic y tiene autoguardado y versiones del sistema.
- **App Sandbox** activado, con acceso a archivos elegidos por el usuario y security-scoped bookmarks hacia el dataset. Esto deja abierta la opción de publicar en la App Store.
- **Licencia MIT.**

### Módulos (paquetes de Swift Package Manager)

```
App (SwiftUI, target de Xcode)
 └─ UI ─────────────► Pipeline ──► Engine (DuckDB)
                        ▲
                     Operations ──► Codegen (pandas / polars)
```

- **Engine:** envuelve DuckDB. Abre archivos, ejecuta consultas, pagina filas, calcula perfiles de columna y exporta. Es lo único que conoce DuckDB.
- **Pipeline:** lista ordenada de pasos (`[Step]`), cada uno con su operación, parámetros y estado activo. Compila los pasos activos a una cadena de vistas SQL (`paso_n = SELECT … FROM paso_{n-1}`) que DuckDB evalúa de forma perezosa. Es serializable para el documento.
- **Operations:** una operación por archivo, todas conformes al mismo protocolo.
- **Codegen:** junta los fragmentos de cada operación en un script completo de pandas o de polars (imports, lectura con tipos explícitos, pasos en orden).
- **UI:** tabla, inspector, barra lateral y snippets. Para el inspector y los formularios solo depende de los protocolos, nunca de operaciones concretas.

### Contrato de una operación

Cada operación es un tipo que se describe a sí mismo. Agregar una operación nueva es:

1. Crear un archivo que implemente el protocolo.
2. Registrarla en el catálogo, en una línea.
3. Agregar sus fixtures de paridad (§7).

Esquema del protocolo (indicativo, no definitivo):

```swift
protocol Operation: Codable, Sendable {
    static var id: String { get }                 // identificador estable, va en el documento
    static var displayName: LocalizedStringResource { get }
    var affectedColumns: [String] { get }
    func validate(against schema: Schema) -> [ValidationIssue]
    func sql(input: String, schema: Schema) -> String   // SELECT … FROM input
    func outputSchema(from schema: Schema) -> Schema
    func pandas(df: String, schema: Schema) -> CodeFragment
    func polars(df: String, schema: Schema) -> CodeFragment
}
// El formulario de parámetros (SwiftUI) vive junto a la operación.
```

El `id` estable y los parámetros `Codable` permiten que un documento guardado se siga abriendo cuando se agregan operaciones nuevas.

## 7. Paridad entre la app y los snippets

Este es el riesgo principal del proyecto. DuckDB, pandas y polars se comportan distinto en varios casos. Reglas:

- **Tipos:** la app usa los tipos que infiere DuckDB, y el snippet lee el archivo con esos tipos explícitos (`dtype=` / `schema=`), así nadie vuelve a inferirlos.
- **Orden de filas:** al leer se agrega un índice de fila interno. "Conservar el primero o el último" y forward/backward fill dependen de ese orden.
- **Nulos:** se definen por operación (p. ej. en duplicados, dos nulos cuentan como iguales, como en pandas). Cada caso queda documentado en el archivo de la operación.
- **Mediana:** mediana interpolada (`quantile_cont` ↔ `median()`).
- **Moda con empate:** se toma el valor menor, de forma determinista, en los tres motores.
- **Cambiar tipo:** las conversiones nativas no coinciden entre motores. Con `'1.5'` a entero, DuckDB redondea a 2, polars da nulo y pandas falla. DuckDB acepta `'yes'` y `'T'` como booleano, pandas falla y polars no tiene esa conversión. Por eso Trimean define sus propias reglas y las escribe igual en los tres motores, sin usar el cast nativo:
  - **Entero:** solo texto que cumple `^[+-]?[0-9]+$`, o decimales sin parte fraccionaria. El resto queda nulo y se cuenta como no convertible.
  - **Booleano:** solo `true` y `false`, sin distinguir mayúsculas.
  - **Fecha:** con un formato explícito (por defecto `%Y-%m-%d`).
- **Fechas al leer:** el sniffer de DuckDB puede detectar formatos como `%d/%m/%Y`. La app guarda el formato detectado y el snippet lo usa (`pd.to_datetime(..., format=…)`, `str.to_date(…)`). Ni `dtype=` ni `schema=` bastan.
- **Decimal a texto:** DuckDB, pandas y polars dan el mismo texto para dobles (verificado: `0.30000000000000004`, `1e+20`).
- **Pruebas de paridad:** un arnés corre cada operación sobre fixtures pequeños con casos difíciles (nulos, empates, tipos mezclados, filas duplicadas). El lado DuckDB corre **dentro del motor de Swift de la app**, no con el paquete `duckdb` de Python, porque sus versiones no coinciden (PyPI va en 2.0.0). El motor exporta su resultado a Parquet, Python ejecuta los snippets de pandas y polars y compara. Una operación sin pruebas de paridad no se acepta.

## 8. Documento de proyecto

- Es un paquete con extensión `.trimean`, con un `project.json` dentro: versión del formato, referencia al dataset (bookmark + ruta legible), lista de pasos y estado de la interfaz (columnas visibles, paneles abiertos).
- Si el dataset se movió o cambió, la app lo avisa y permite volver a vincularlo.
- Nunca guarda una copia de los datos dentro del documento. La única copia es la base temporal de §6, que se borra al cerrar.

## 9. Decisiones tomadas en la conversación

| Tema | Decisión | De quién |
|---|---|---|
| Motor | DuckDB | Usuario |
| UI | 100% nativa, SwiftUI, guías de Apple | Usuario |
| Pantalla inicial | Solo la tabla | Usuario |
| Análisis | Por columna, en un inspector a la derecha | Propuesta aceptada |
| Pasos | Lista editable + deshacer | Usuario |
| Exportar | Sí, CSV y Parquet | Usuario |
| Snippets | pandas y polars; por paso y script completo | Usuario |
| Proyectos | Se guardan y se retoman | Usuario |
| Formatos | CSV y Parquet | Usuario |
| Tamaño | Archivos más grandes que la RAM; estadísticas exactas | Propuesta aceptada |
| Distribución | Open source en GitHub, MIT | Usuario / propuesta |
| macOS mínimo | 26 | Propuesta aceptada |
| Modularidad | Una operación = un archivo + registro | Usuario |
| Nombre | Trimean (la trimedia de Tukey); documento `.trimean` | Usuario, a partir de propuestas investigadas |

## 10. Pendiente

- Revisar marcas registradas (USPTO, IMPI) antes de publicar en la App Store.
- Ícono e identidad visual.
- Diseño detallado de cada vista (se hará con mockups antes de implementar la UI).
- Versiones de pandas y polars con las que se garantiza la paridad: propuesta pandas 3.1 y polars 2.1 (las actuales a nov 2026).
