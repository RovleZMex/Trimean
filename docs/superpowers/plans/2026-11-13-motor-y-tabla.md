# Trimean — Plan 1: motor, tabla e inspector de solo lectura

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Una app de Mac que abre un CSV o Parquet de cualquier tamaño, muestra la tabla con scroll fluido y, al seleccionar una columna, presenta su perfil exacto en un inspector. Al hacer clic en un número, la tabla se filtra a las filas que lo producen.

**Architecture:** Un paquete SwiftPM (`TrimeanKit`) en la raíz del repo con tres bibliotecas:

- `TrimeanCore`: tipos puros, sin DuckDB.
- `TrimeanEngine`: lo único que importa DuckDB. Importa el archivo a una base temporal en disco y expone filas paginadas y perfiles de columna.
- `TrimeanUI`: vistas SwiftUI/AppKit y sus modelos.

La app (`App/`, generada con XcodeGen) es una capa delgada sobre `TrimeanUI`. Toda la lógica se prueba sin interfaz gráfica con `swift test`.

**Tech Stack:** Swift 6 (tools 6.2, toolchain 6.4), SwiftUI + AppKit (`NSTableView`), Swift Charts, DuckDB vía `duckdb-swift`, XcodeGen 2.x, macOS 26+.

**Spec:** `docs/especificacion.md`

**Fuera de este plan (Plan 2 y siguientes):** lista de pasos, las 7 operaciones, generación de snippets, arnés de paridad, documento `.trimean` (guardar y retomar), exportar y comparación antes/después.

## Global Constraints

- macOS 26 mínimo: `platforms: [.macOS(.v26)]`, `MACOSX_DEPLOYMENT_TARGET = 26.0`.
- `// swift-tools-version: 6.2`, Swift 6 language mode con concurrencia estricta, sin `@unchecked Sendable` salvo en el envoltorio de conexión de DuckDB (documentado ahí).
- DuckDB: `.package(url: "https://github.com/duckdb/duckdb-swift", revision: "2bc1adcf863f83994c27a909d475f8185fbabfa3")`. Solo `TrimeanEngine` puede hacer `import DuckDB`.
- Columna interna de orden: `__trimean_row` (BIGINT, desde 0, en el orden del archivo). Nunca se muestra en la UI ni en perfiles.
- Las estadísticas son exactas sobre todas las filas. No hay muestreo.
- El archivo original nunca se modifica. La base temporal vive en `FileManager.default.temporaryDirectory/Trimean/<UUID>/` y se borra al cerrar.
- App Sandbox activado: `com.apple.security.app-sandbox`, `com.apple.security.files.user-selected.read-only`, `com.apple.security.files.bookmarks.app-scope`.
- Bundle id `com.rovlezmex.Trimean`. Nombre visible "Trimean".
- Al abrir un dataset solo se ve la tabla. El inspector empieza oculto.
- Solo controles del sistema: `NSTableView`, `.inspector`, `.toolbar`, Swift Charts, `ProgressView`. Nada dibujado a mano cuando existe un control nativo.
- Sin atribuciones de IA en commits ni archivos.
- Textos de UI en español.

## Review Focus

1. **CSV cuyo tipo cambia después de la muestra del sniffer** (p. ej. 30 000 enteros y luego `"abc"`): se abre igual, con la columna como texto, sin error. Prueba en Task 2.
2. **Nombres de columna difíciles** (espacios, comillas dobles y simples, acentos, emoji, nombres duplicados, encabezado vacío): se abren, se muestran tal cual y se pueden perfilar. Prueba en Task 2 y Task 4.
3. **Archivo vacío o solo con encabezado:** abre con 0 filas. El perfil muestra conteos en 0 y estadísticas `nil`, sin crash. Prueba en Task 2 y Task 4.
4. **Cambiar rápido de columna:** el inspector muestra siempre el perfil de la última columna seleccionada, nunca uno viejo que llegó tarde. Prueba en Task 8.
5. **Cerrar la ventana a mitad de la importación o salir de la app:** no quedan bases temporales huérfanas. Las que sobrevivan a un crash se borran al siguiente inicio. Prueba en Task 2.

---

## Estructura de archivos

```
Package.swift
Sources/TrimeanCore/
  ColumnType.swift        # enum de tipos + mapeo desde nombres de tipo DuckDB
  Schema.swift            # Column, Schema
  SQL.swift               # quoting de identificadores y literales
  SourceDescriptor.swift  # formato, URL, dialecto CSV detectado, esquema, filas
  RowFilter.swift         # filtros de "ver estas filas"
  RowPage.swift
  ColumnProfile.swift     # NumericSummary, TemporalSummary, HistogramBin, ValueCount
Sources/TrimeanEngine/
  DuckLane.swift          # actor que serializa una Connection
  Workspace.swift         # directorio temporal + limpieza
  Ingest.swift            # sniff + DESCRIBE + CREATE TABLE src
  Dataset.swift           # API pública: open, rows, count, profile, close
  Profiler.swift          # SQL de perfiles
Sources/TrimeanUI/
  RowCache.swift          # caché de páginas para la tabla
  DatasetModel.swift      # estado de una ventana (@Observable, @MainActor)
  DataTableView.swift     # NSViewRepresentable sobre NSTableView
  InspectorView.swift     # perfil de columna (solo lectura)
  HistogramChart.swift
  DatasetWindow.swift     # carga, error, tabla + inspector + toolbar
  DatasetRef.swift        # referencia con bookmark para WindowGroup(for:)
Tests/TrimeanCoreTests/…
Tests/TrimeanEngineTests/… + Fixtures/
Tests/TrimeanUITests/…
App/project.yml
App/Trimean/TrimeanApp.swift
App/Trimean/Info.plist
App/Trimean/Trimean.entitlements
scripts/bootstrap.sh
```

Decisiones de esta estructura:

- `NSTableView` en lugar de SwiftUI `Table`, porque `Table` necesita la colección completa de identificadores en memoria, lo que no escala a decenas de millones de filas. `NSTableView` pide filas por índice, es el control nativo de Mac para esto y se ve como el sistema.
- La ventana usa `WindowGroup(for: DatasetRef.self)` y no `DocumentGroup`. El documento `.trimean` llega en el Plan 2, y ahí se decide la arquitectura de documentos.

---

### Task 1: Paquete y tipos base (`TrimeanCore`)

**Files:**
- Create: `Package.swift`, `Sources/TrimeanCore/{ColumnType,Schema,SQL,SourceDescriptor,RowFilter,RowPage,ColumnProfile}.swift`
- Create: `Tests/TrimeanCoreTests/{ColumnTypeTests,SQLTests}.swift`
- Modify: `.gitignore` (agregar `App/*.xcodeproj`)

**Interfaces:**
- Produces:
  - `public enum ColumnType: Hashable, Sendable, Codable { case boolean, integer, double, string, date, timestamp, other(String) }`
  - `static func fromDuckDB(_ typeName: String) -> ColumnType` y `var isNumeric: Bool`, `var isTemporal: Bool`.
  - `public struct Column: Hashable, Sendable, Codable { name: String; type: ColumnType; duckDBType: String }`.
  - `public struct Schema: Hashable, Sendable, Codable { columns: [Column]; func column(named:) -> Column? }`.
  - `public enum SQL { static func identifier(_ s: String) -> String; static func literal(_ s: String) -> String }`.
  - `public enum SourceFormat: String, Codable, Sendable { case csv, parquet }`.
  - `public struct CSVDialect: Hashable, Sendable, Codable { delimiter, quote, escape: String; hasHeader: Bool; dateFormat, timestampFormat: String? }`.
  - `public struct SourceDescriptor: Sendable, Codable { url: URL; format: SourceFormat; dialect: CSVDialect?; schema: Schema; rowCount: Int; usedFullScan: Bool }`.
  - `public enum RowFilter: Hashable, Sendable { case isNull(column: String); case equals(column: String, display: String) }`.
  - `public struct RowPage: Sendable, Equatable { offset: Int; rowIDs: [Int64]; cells: [[String?]] }` (`cells[fila][columna]`, en el orden del `Schema`).
  - `public struct HistogramBin: Sendable, Equatable { lower: Double; upper: Double; count: Int }`.
  - `public struct NumericSummary: Sendable, Equatable { min, max, mean, median, q1, q3: Double; std: Double?; histogram: [HistogramBin] }`.
  - `public struct TemporalSummary: Sendable, Equatable { min, max: String; histogram: [HistogramBin] }` (los bordes de bin en segundos desde epoch).
  - `public struct ValueCount: Sendable, Equatable { display: String; count: Int }`.
  - `public struct ColumnProfile: Sendable, Equatable { column: Column; rowCount: Int; nullCount: Int; distinctCount: Int; numeric: NumericSummary?; temporal: TemporalSummary?; topValues: [ValueCount] }`.

- [ ] **Step 1: Escribir `Package.swift`** con productos `TrimeanCore`, `TrimeanEngine` y `TrimeanUI`, sus targets de prueba y la dependencia DuckDB fijada (Global Constraints). `TrimeanEngine` depende de `TrimeanCore` y `DuckDB`. `TrimeanUI` depende de `TrimeanCore` y `TrimeanEngine`. `TrimeanEngineTests` tiene `resources: [.copy("Fixtures")]`.

- [ ] **Step 2: Escribir las pruebas que fallan**

```swift
@Test func mapsDuckDBTypes() {
    #expect(ColumnType.fromDuckDB("BIGINT") == .integer)
    #expect(ColumnType.fromDuckDB("INTEGER") == .integer)
    #expect(ColumnType.fromDuckDB("UTINYINT") == .integer)
    #expect(ColumnType.fromDuckDB("UBIGINT") == .other("UBIGINT"))
    #expect(ColumnType.fromDuckDB("HUGEINT") == .other("HUGEINT"))
    #expect(ColumnType.fromDuckDB("DOUBLE") == .double)
    #expect(ColumnType.fromDuckDB("FLOAT") == .double)
    #expect(ColumnType.fromDuckDB("DECIMAL(18,3)") == .double)
    #expect(ColumnType.fromDuckDB("VARCHAR") == .string)
    #expect(ColumnType.fromDuckDB("BOOLEAN") == .boolean)
    #expect(ColumnType.fromDuckDB("DATE") == .date)
    #expect(ColumnType.fromDuckDB("TIMESTAMP") == .timestamp)
    #expect(ColumnType.fromDuckDB("TIMESTAMP_NS") == .timestamp)
    #expect(ColumnType.fromDuckDB("TIMESTAMP WITH TIME ZONE") == .other("TIMESTAMP WITH TIME ZONE"))
    #expect(ColumnType.fromDuckDB("INTEGER[]") == .other("INTEGER[]"))
}
@Test func quotesIdentifiersAndLiterals() {
    #expect(SQL.identifier(#"monto "neto""#) == #""monto ""neto""""#)
    #expect(SQL.identifier("año 🚀") == #""año 🚀""#)
    #expect(SQL.literal("O'Brien.csv") == "'O''Brien.csv'")
}
```

- [ ] **Step 3: Correr y ver que fallan.** `swift test --filter TrimeanCoreTests`. Esperado: errores de compilación por tipos no definidos. La primera vez también compila DuckDB, unos 2-3 min.

- [ ] **Step 4: Implementar los tipos.** `fromDuckDB` normaliza a mayúsculas y aplica las reglas de las pruebas: familia entera con signo y `U{TINY,SMALL}INT`/`UINTEGER` → `.integer`; `FLOAT`, `DOUBLE` y `DECIMAL(...)` → `.double`; `ENUM(...)` → `.string`; `TIMESTAMP`, `TIMESTAMP_S/MS/NS` → `.timestamp`; el resto → `.other(nombre)`.

- [ ] **Step 5: Correr las pruebas.** Esperado: PASS.

- [ ] **Step 6: Commit** `git commit -m "TrimeanKit: paquete y tipos base"`

---

### Task 2: Abrir e importar (`Workspace`, `DuckLane`, `Ingest`, `Dataset.open`)

**Files:**
- Create: `Sources/TrimeanEngine/{DuckLane,Workspace,Ingest,Dataset}.swift`
- Create: `Tests/TrimeanEngineTests/{OpenTests,WorkspaceTests}.swift`, `Tests/TrimeanEngineTests/Fixtures/{basic.csv, semicolon.csv, dmy_dates.csv, weird_names.csv, empty.csv, header_only.csv, basic.parquet}`
- Create: `Tests/TrimeanEngineTests/Support/FixtureBuilder.swift` (genera en tiempo de prueba los CSV grandes: `late_type_change.csv` con 30 000 filas enteras y la 30 001 `"abc"`, y `ordered.csv` con 200 000 filas `id` secuencial)

**Interfaces:**
- Consumes: todo `TrimeanCore` (Task 1).
- Produces:
  - `actor DuckLane { init(_ connection: Connection); func query(_ sql: String) throws -> ResultSet; func execute(_ sql: String) throws }` (internal).
  - `public enum Workspace { static var root: URL; static func make() throws -> URL; static func removeAll() }`. `root` es `temporaryDirectory/Trimean`.
  - `public final class Dataset: Sendable`:
    - `public static func open(_ url: URL) async throws -> Dataset`
    - `public let descriptor: SourceDescriptor`
    - `public func close()` (idempotente; borra su directorio)
  - `public enum DatasetError: Error, Equatable { case unsupportedFormat(String); case unreadable(String); case closed }`.

Reglas de importación (las fijan las pruebas):

1. **Formato por extensión.** `.csv`, `.tsv` y `.txt` son CSV; `.parquet` es Parquet. Cualquier otra → `unsupportedFormat`.
2. **Dialecto CSV.** Se lee de `SELECT Delimiter, Quote, Escape, HasHeader, DateFormat, TimestampFormat FROM sniff_csv(<literal>)`. Los valores `"(empty)"` se guardan como `""`.
3. **Esquema.** Se obtiene con `DESCRIBE SELECT * FROM read_csv(<literal>)` (o `read_parquet`). Cada columna se normaliza según su `ColumnType`: `.integer` → `CAST(c AS BIGINT)`, `.double` → `CAST(c AS DOUBLE)`, `ENUM` → `CAST(c AS VARCHAR)`, el resto sin cambio. El esquema guardado usa el tipo normalizado.
4. **Importación.**

   ```sql
   CREATE TABLE src AS
   SELECT row_number() OVER () - 1 AS __trimean_row, <lista normalizada>
   FROM <lector>
   ```

   Si falla con un error de conversión, se repite una sola vez con `read_csv(<literal>, sample_size=-1)` y `usedFullScan = true`. Cualquier otro error → `unreadable(mensaje de DuckDB)`.
5. **Base temporal.** `Database(store: .file(at: dir/data.duckdb))`, con `SET temp_directory = '<dir>/spill'` y `SET preserve_insertion_order = true`. Se abren dos conexiones, cada una en su `DuckLane`: `rowsLane` y `profileLane`.
6. **Limpieza.** `close()` suelta las conexiones y la base, y luego borra `dir`. `deinit` llama `close()`. Si `open` falla, también borra `dir`.

- [ ] **Step 1: Escribir las pruebas que fallan**

```swift
@Test func opensBasicCSV() async throws {
    let ds = try await Dataset.open(fixture("basic.csv"))   // id,monto,region,fecha ; 5 filas, 1 monto vacío
    #expect(ds.descriptor.format == .csv)
    #expect(ds.descriptor.rowCount == 5)
    #expect(ds.descriptor.schema.columns.map(\.name) == ["id", "monto", "region", "fecha"])
    #expect(ds.descriptor.schema.columns.map(\.type) == [.integer, .double, .string, .date])
    #expect(ds.descriptor.dialect?.delimiter == ",")
    #expect(ds.descriptor.usedFullScan == false)
}
@Test func detectsSemicolonAndDayMonthDates() async throws {
    #expect(try await Dataset.open(fixture("semicolon.csv")).descriptor.dialect?.delimiter == ";")
    let d = try await Dataset.open(fixture("dmy_dates.csv"))    // 03/01/2025, 15/01/2025
    #expect(d.descriptor.dialect?.dateFormat == "%d/%m/%Y")
    #expect(d.descriptor.schema.columns[0].type == .date)
}
@Test func lateTypeChangeFallsBackToFullScan() async throws {
    let ds = try await Dataset.open(FixtureBuilder.lateTypeChange())
    #expect(ds.descriptor.schema.columns[0].type == .string)
    #expect(ds.descriptor.rowCount == 30_001)
    #expect(ds.descriptor.usedFullScan == true)
}
@Test func weirdColumnNames() async throws {
    // encabezado: "monto ""neto""",año 🚀,a,a,  (incluye duplicado y vacío)
    let names = try await Dataset.open(fixture("weird_names.csv")).descriptor.schema.columns.map(\.name)
    #expect(names.count == 5)
    #expect(names[0] == #"monto "neto""#)
    #expect(names[1] == "año 🚀")
    #expect(Set(names).count == 5)    // DuckDB desambigua duplicados y vacíos
}
@Test func emptyAndHeaderOnly() async throws {
    #expect(try await Dataset.open(fixture("header_only.csv")).descriptor.rowCount == 0)
    await #expect(throws: DatasetError.self) { try await Dataset.open(fixture("empty.csv")) }
}
@Test func opensParquet() async throws {
    let ds = try await Dataset.open(fixture("basic.parquet"))   // mismo contenido que basic.csv
    #expect(ds.descriptor.format == .parquet && ds.descriptor.rowCount == 5 && ds.descriptor.dialect == nil)
}
@Test func pathWithApostrophe() async throws {
    let copy = try FixtureBuilder.copy(fixture("basic.csv"), named: "O'Brien data.csv")
    #expect(try await Dataset.open(copy).descriptor.rowCount == 5)
}
@Test func rejectsUnknownExtension() async {
    await #expect(throws: DatasetError.unsupportedFormat("xlsx")) { try await Dataset.open(URL(filePath: "/tmp/x.xlsx")) }
}
@Test func closeRemovesWorkspace() async throws {
    let ds = try await Dataset.open(fixture("basic.csv"))
    let dirs = try FileManager.default.contentsOfDirectory(atPath: Workspace.root.path)
    ds.close(); ds.close()
    let after = try FileManager.default.contentsOfDirectory(atPath: Workspace.root.path)
    #expect(after.count == dirs.count - 1)
}
@Test func removeAllClearsOrphans() throws {
    _ = try Workspace.make(); Workspace.removeAll()
    #expect(!FileManager.default.fileExists(atPath: Workspace.root.path))
}
@Test func originalFileUntouched() async throws {
    let url = fixture("basic.csv"); let before = try Data(contentsOf: url)
    try await Dataset.open(url).close()
    #expect(try Data(contentsOf: url) == before)
}
```

`fixture(_:)` resuelve `Bundle.module.url(forResource:withExtension:subdirectory: "Fixtures")`. Los tests de engine corren en serie (`@Suite(.serialized)`) porque comparten `Workspace.root`. Genera `basic.parquet` una vez desde `basic.csv` con DuckDB (`COPY … (FORMAT parquet)`) y súbelo como fixture.

- [ ] **Step 2: Correr y ver que fallan.** `swift test --filter TrimeanEngineTests.OpenTests`. Esperado: no compila (`Dataset` no existe).
- [ ] **Step 3: Implementar `DuckLane` y `Workspace`.** `DuckLane` es un `actor`, así que serializa el uso de su `Connection`; la `Connection` de `duckdb-swift` ya es `Sendable`.
- [ ] **Step 4: Implementar `Ingest` y `Dataset.open/close`** con las reglas de arriba. `open` corre la importación en el `rowsLane`.
- [ ] **Step 5: Correr las pruebas.** Esperado: PASS (11 pruebas).
- [ ] **Step 6: Commit** `git commit -m "Engine: abrir e importar CSV y Parquet a base temporal"`

---

### Task 3: Filas paginadas y filtros (`Dataset.rows`, `Dataset.count`)

**Files:**
- Modify: `Sources/TrimeanEngine/Dataset.swift`
- Create: `Tests/TrimeanEngineTests/RowsTests.swift`

**Interfaces:**
- Consumes: `Dataset`, `RowFilter`, `RowPage` (Tasks 1-2).
- Produces:
  - `public func rows(offset: Int, limit: Int, filter: RowFilter?) async throws -> RowPage`
  - `public func count(filter: RowFilter?) async throws -> Int`

SQL:

```sql
SELECT __trimean_row, CAST(<c> AS VARCHAR), …
FROM src
[WHERE <pred>]
ORDER BY __trimean_row
LIMIT <limit> OFFSET <offset>
```

- `.isNull(c)` → `<c> IS NULL`.
- `.equals(c, d)` → `CAST(<c> AS VARCHAR) = <literal d>`.
- Si la columna del filtro no existe en el esquema → `DatasetError.unreadable`.
- Después de `close()` → `DatasetError.closed`.
- Corre en `rowsLane`.

- [ ] **Step 1: Escribir las pruebas que fallan**

```swift
@Test func pagesInFileOrder() async throws {
    let ds = try await Dataset.open(FixtureBuilder.ordered())          // 200 000 filas, id = 0…199 999
    let p = try await ds.rows(offset: 150_000, limit: 3, filter: nil)
    #expect(p.rowIDs == [150_000, 150_001, 150_002])
    #expect(p.cells.map { $0[0] } == ["150000", "150001", "150002"])
}
@Test func filtersNullsAndValues() async throws {
    let ds = try await Dataset.open(fixture("basic.csv"))
    #expect(try await ds.count(filter: .isNull(column: "monto")) == 1)
    let nulls = try await ds.rows(offset: 0, limit: 10, filter: .isNull(column: "monto"))
    #expect(nulls.rowIDs == [1] && nulls.cells[0][1] == nil)
    #expect(try await ds.count(filter: .equals(column: "region", display: "Norte")) == 2)
    #expect(try await ds.count(filter: nil) == 5)
}
@Test func pageBeyondEndIsEmpty() async throws {
    let ds = try await Dataset.open(fixture("basic.csv"))
    #expect(try await ds.rows(offset: 10, limit: 5, filter: nil).rowIDs.isEmpty)
}
@Test func rowsAfterCloseThrows() async throws {
    let ds = try await Dataset.open(fixture("basic.csv")); ds.close()
    await #expect(throws: DatasetError.closed) { try await ds.rows(offset: 0, limit: 1, filter: nil) }
}
@Test func weirdNamesQueryable() async throws {
    let ds = try await Dataset.open(fixture("weird_names.csv"))
    _ = try await ds.count(filter: .isNull(column: #"monto "neto""#))
}
```

- [ ] **Step 2: Correr y ver que fallan.** `swift test --filter TrimeanEngineTests.RowsTests`. Esperado: no compila.
- [ ] **Step 3: Implementar `rows` y `count`.**
- [ ] **Step 4: Correr las pruebas.** Esperado: PASS.
- [ ] **Step 5: Commit** `git commit -m "Engine: filas paginadas y filtros"`

---

### Task 4: Perfiles de columna (`Profiler`, `Dataset.profile`)

**Files:**
- Create: `Sources/TrimeanEngine/Profiler.swift`
- Modify: `Sources/TrimeanEngine/Dataset.swift`
- Create: `Tests/TrimeanEngineTests/ProfileTests.swift`, `Tests/TrimeanEngineTests/Fixtures/profile.csv`

**Interfaces:**
- Consumes: `Dataset`, `ColumnProfile` y sus tipos (Task 1).
- Produces: `public func profile(column: String) async throws -> ColumnProfile` (corre en `profileLane`; antes de cada consulta llama `try Task.checkCancellation()`).

Reglas (las fijan las pruebas):

- **Todas las columnas.**
  - `rowCount = count(*)`.
  - `nullCount = count(*) - count(c)`.
  - `distinctCount = count(DISTINCT c)`, sin contar el nulo.
- **Numéricas** (`.integer`, `.double`).
  - `min`, `max`, `avg`, `quantile_cont(c, [0.25, 0.5, 0.75])`.
  - `std = stddev_samp`, que es `nil` con menos de 2 valores.
  - Histograma de 30 bins de igual ancho entre `min` y `max`. El bin `i` cuenta `least(floor((c-min)/ancho), 29) = i`, así que el último bin incluye `max`. Si `min == max`, hay un solo bin `[min, max]` con todos los valores.
  - Si no hay valores no nulos, `numeric = nil`.
- **Temporales** (`.date`, `.timestamp`).
  - `min` y `max` como `CAST(... AS VARCHAR)`.
  - Histograma igual que el numérico, sobre `epoch(c)` en segundos.
- **Texto, booleano y `.other`.**
  - `topValues` son los 10 valores más frecuentes: `ORDER BY count DESC, CAST(c AS VARCHAR) ASC`, sin incluir el nulo.
  - En numéricas y temporales, `topValues = []`.

- [ ] **Step 1: Escribir las pruebas que fallan.** `profile.csv` tiene las columnas `n` = 1, 2, 3, 4, vacío; `s` = b, a, b, vacío, a; `c` = 7, 7, 7, 7, 7; `z` = todo vacío; `d` = 2025-01-01, 2025-01-03, vacío, 2025-01-02, 2025-01-01.

```swift
@Test func numericProfile() async throws {
    let p = try await open("profile.csv").profile(column: "n")
    #expect(p.rowCount == 5 && p.nullCount == 1 && p.distinctCount == 4)
    let s = try #require(p.numeric)
    #expect(s.min == 1 && s.max == 4 && s.mean == 2.5 && s.median == 2.5 && s.q1 == 1.75 && s.q3 == 3.25)
    #expect(abs(try #require(s.std) - 1.2909944487358056) < 1e-12)
    #expect(s.histogram.count == 30 && s.histogram.map(\.count).reduce(0, +) == 4)
    #expect(s.histogram.last?.count == 1)          // el máximo cae en el último bin
}
@Test func constantColumnHasSingleBin() async throws {
    let s = try #require(try await open("profile.csv").profile(column: "c").numeric)
    #expect(s.histogram == [HistogramBin(lower: 7, upper: 7, count: 5)])
}
@Test func allNullColumn() async throws {
    let p = try await open("profile.csv").profile(column: "z")
    #expect(p.nullCount == 5 && p.distinctCount == 0 && p.numeric == nil && p.topValues.isEmpty)
}
@Test func topValuesTieBreakAlphabetical() async throws {
    let p = try await open("profile.csv").profile(column: "s")
    #expect(p.topValues == [ValueCount(display: "a", count: 2), ValueCount(display: "b", count: 2)])
}
@Test func temporalProfile() async throws {
    let t = try #require(try await open("profile.csv").profile(column: "d").temporal)
    #expect(t.min == "2025-01-01" && t.max == "2025-01-03")
    #expect(t.histogram.map(\.count).reduce(0, +) == 4)
}
@Test func headerOnlyProfile() async throws {
    let p = try await open("header_only.csv").profile(column: "id")
    #expect(p.rowCount == 0 && p.nullCount == 0 && p.numeric == nil)
}
@Test func cancelledProfileThrows() async throws {
    let ds = try await open("profile.csv")
    let task = Task { try await ds.profile(column: "n") }
    task.cancel()
    await #expect(throws: CancellationError.self) { try await task.value }
}
```

- [ ] **Step 2: Correr y ver que fallan.** `swift test --filter TrimeanEngineTests.ProfileTests`. Esperado: no compila.
- [ ] **Step 3: Implementar `Profiler`.** Usa una consulta para los conteos, una para el resumen y una para el histograma o los valores más frecuentes. Revisa la cancelación entre consultas.
- [ ] **Step 4: Correr las pruebas.** Esperado: PASS.
- [ ] **Step 5: Commit** `git commit -m "Engine: perfiles de columna exactos"`

---

### Task 5: Benchmark de rendimiento (opcional, fuera de CI)

**Files:**
- Create: `Tests/TrimeanEngineTests/BenchmarkTests.swift`

- [ ] **Step 1: Escribir el benchmark.** Se activa con la variable `TRIMEAN_BENCH=1` (`.enabled(if:)`).
  1. Genera con DuckDB un CSV de 30 M filas, el mismo del spike: `id`, `monto` DOUBLE, `edad` con 5 % nulos, `region`, `fecha` y `hash`. Lo escribe en `Workspace.root/bench.csv` si no existe.
  2. Mide y exige:
     - `open` < 40 s.
     - `rows(offset: 15_000_000, limit: 100)` < 0.1 s.
     - `profile("monto")` < 3 s.
     - `count(.isNull("edad"))` < 0.5 s.
     - Mientras corre `profile("hash")`, `rows(offset: 20_000_000, limit: 100)` responde en < 0.5 s. Esto prueba que las dos conexiones no se bloquean.
- [ ] **Step 2: Correr.** `TRIMEAN_BENCH=1 swift test -c release --filter BenchmarkTests`. Esperado: PASS y tiempos impresos.
- [ ] **Step 3: Commit** `git commit -m "Engine: benchmark opcional de 30M filas"`

---

### Task 6: Caché de filas y modelo de ventana (`TrimeanUI`, sin vistas)

**Files:**
- Create: `Sources/TrimeanUI/{RowCache,DatasetModel,DatasetRef}.swift`
- Create: `Tests/TrimeanUITests/{RowCacheTests,DatasetModelTests}.swift`

**Interfaces:**
- Consumes: `Dataset` (Tasks 2-4).
- Produces:
  - `@MainActor public final class RowCache`
    - `init(pageSize: Int = 500, maxPages: Int = 40, fetch: @escaping @Sendable (_ offset: Int, _ limit: Int) async throws -> RowPage)`
    - `func cell(row: Int, column: Int) -> CellState`, donde `enum CellState: Equatable { case loading, value(String?) }`.
    - `var onPageLoaded: ((Range<Int>) -> Void)?`
    - `func invalidate()`.
    - Pide cada página una sola vez aunque se consulten muchas celdas. Desaloja la página menos usada recientemente (LRU) al superar `maxPages`.
  - `public struct DatasetRef: Hashable, Codable, Sendable { bookmark: Data; displayName: String; static func make(from url: URL) throws -> DatasetRef; func resolve() throws -> URL }`. Usa bookmark de seguridad con alcance de app (`.withSecurityScope`); `resolve()` llama `startAccessingSecurityScopedResource()`.
  - `@MainActor @Observable public final class DatasetModel`:
    - `enum Phase { case loading(fileName: String, bytes: Int64), ready, failed(String) }`
    - `var phase: Phase`, `var descriptor: SourceDescriptor?`, `var visibleRowCount: Int`
    - `var filter: RowFilter?` (al cambiar, recalcula `visibleRowCount` e invalida la caché)
    - `var selectedColumn: String?`, `var profile: ColumnProfile?`, `var isProfiling: Bool`
    - `var isInspectorPresented: Bool` (empieza en `false`)
    - `let rows: RowCache`
    - `func load(url: URL) async`, `func select(column: String?)`, `func close()`

Reglas de `select`: cancela la tarea de perfil anterior, pone `profile = nil` y `isProfiling = true`, y lanza una nueva tarea. Al terminar, solo asigna el resultado si `selectedColumn` sigue siendo esa columna (Review Focus 4). Seleccionar una columna también abre el inspector.

- [ ] **Step 1: Escribir las pruebas que fallan**

```swift
@MainActor @Test func cacheFetchesEachPageOnce() async throws {
    let counter = Counter()
    let cache = RowCache(pageSize: 10) { off, lim in await counter.inc(); return .stub(offset: off, count: lim) }
    #expect(cache.cell(row: 3, column: 0) == .loading)
    _ = cache.cell(row: 4, column: 0)
    try await waitUntil { cache.cell(row: 3, column: 0) != .loading }
    #expect(await counter.value == 1)
    #expect(cache.cell(row: 3, column: 0) == .value("r3c0"))
}
@MainActor @Test func cacheEvictsLeastRecentlyUsed() async throws { /* maxPages 2, pedir páginas 0, 1, 2 → la 0 vuelve a .loading */ }
@MainActor @Test func lastSelectedColumnWins() async throws {
    let m = DatasetModel(); await m.load(url: fixture("profile.csv"))
    m.select(column: "n"); m.select(column: "s")
    try await waitUntil { !m.isProfiling }
    #expect(m.profile?.column.name == "s")
    #expect(m.isInspectorPresented)
}
@MainActor @Test func filterUpdatesVisibleCount() async throws {
    let m = DatasetModel(); await m.load(url: fixture("basic.csv"))
    #expect(m.visibleRowCount == 5)
    m.filter = .isNull(column: "monto"); try await waitUntil { m.visibleRowCount == 1 }
    m.filter = nil; try await waitUntil { m.visibleRowCount == 5 }
}
@MainActor @Test func failedLoadShowsMessage() async {
    let m = DatasetModel(); await m.load(url: URL(filePath: "/tmp/nope.xlsx"))
    guard case .failed = m.phase else { Issue.record("expected failed"); return }
}
```

`TrimeanUITests` usa los fixtures de engine copiándolos con `resources: [.copy("../TrimeanEngineTests/Fixtures")]`. Si SwiftPM no acepta esa ruta, duplica `basic.csv` y `profile.csv` en `Tests/TrimeanUITests/Fixtures`.

- [ ] **Step 2: Correr y ver que fallan.** `swift test --filter TrimeanUITests`.
- [ ] **Step 3: Implementar `RowCache`, `DatasetRef` y `DatasetModel`.**
- [ ] **Step 4: Correr las pruebas.** Esperado: PASS.
- [ ] **Step 5: Commit** `git commit -m "UI: caché de filas y modelo de ventana"`

---

### Task 7: App, ventana y tabla

**Files:**
- Create: `App/project.yml`, `App/Trimean/{TrimeanApp.swift, Info.plist, Trimean.entitlements}`, `scripts/bootstrap.sh`
- Create: `Sources/TrimeanUI/{DataTableView,DatasetWindow}.swift`

**Interfaces:**
- Consumes: `DatasetModel`, `RowCache`, `DatasetRef` (Task 6).
- Produces:
  - `public struct DatasetWindow: View { public init(ref: DatasetRef) }`
  - `struct DataTableView: NSViewRepresentable` (internal), que lee `model.descriptor.schema`, `model.visibleRowCount` y `model.rows`, y escribe `model.select(column:)`.

**Comportamiento:**

- **`TrimeanApp`**
  - `WindowGroup(for: DatasetRef.self) { $ref in if let ref { DatasetWindow(ref: ref) } }`.
  - Comando "Abrir…" (⌘O) con `NSOpenPanel`, que permite `.commaSeparatedText`, `.tabSeparatedText`, `.plainText` y `UTType("org.apache.parquet")`. Por cada archivo elegido hace `openWindow(value: DatasetRef.make(from:))`.
  - `init()` llama `Workspace.removeAll()`.
  - Sin ventana abierta, la app no muestra una ventana vacía: el menú Archivo › Abrir es el punto de entrada. Usa `.defaultLaunchBehavior(.suppressed)` en el `WindowGroup`.
- **`Info.plist`**
  - `UTImportedTypeDeclarations` para `org.apache.parquet`: extensión `parquet`, conforma a `public.data`.
  - `CFBundleDocumentTypes` (rol Viewer) para CSV, TSV y Parquet, para que funcione "Abrir con" desde Finder. `application(_:open:)` se maneja con `.handlesExternalEvents` u `onOpenURL`, abriendo una ventana por URL.
- **`DatasetWindow`**
  - En `.loading` muestra `ProgressView` indeterminado con "Importando <nombre>… (<tamaño con ByteCountFormatter>)" y debajo, en texto secundario, "Se lee una vez; después todo es instantáneo."
  - En `.failed` muestra `ContentUnavailableView` con el mensaje.
  - En `.ready` muestra solo la tabla, con título de ventana = nombre del archivo y subtítulo = "<filas formateadas> filas · <columnas> columnas".
  - Al cerrar la ventana llama `model.close()`.
- **`DataTableView`**
  - `NSScrollView` + `NSTableView` con estilo `.inset`, `usesAlternatingRowBackgroundColors = true`, `allowsColumnSelection = true`, `allowsMultipleSelection = false` y una columna por `Column`.
  - El título del encabezado es el nombre. El tipo aparece en el tooltip del encabezado y en el inspector, porque el encabezado nativo de una sola línea es lo más limpio.
  - `numberOfRows = visibleRowCount`.
  - Las celdas son `NSTableCellView` con `NSTextField` en fuente monoespaciada de dígitos (`.monospacedDigitSystemFont`).
  - `.loading` se pinta vacío. El nulo se muestra como "—" en `secondaryLabelColor`. Los números van alineados a la derecha.
  - `onPageLoaded` → `reloadData(forRowIndexes:columnIndexes:)`.
  - Al cambiar `schema` o `filter`, recarga todo.
  - Seleccionar una columna (clic en encabezado o `tableViewSelectionDidChange` con `selectedColumn`) llama `model.select(column:)`.
- **`project.yml`**
  - Target app `Trimean`, macOS 26.0, dependencia al paquete local `..` producto `TrimeanUI`.
  - `CODE_SIGN_ENTITLEMENTS`, `PRODUCT_BUNDLE_IDENTIFIER = com.rovlezmex.Trimean`, `CODE_SIGN_IDENTITY = "-"` para builds locales.
- **`bootstrap.sh`** ejecuta `cd App && xcodegen generate`.

- [ ] **Step 1: Generar y compilar.** `scripts/bootstrap.sh && xcodebuild -project App/Trimean.xcodeproj -scheme Trimean -configuration Debug -destination 'platform=macOS' build`. Esperado: `** BUILD SUCCEEDED **`.
- [ ] **Step 2: Verificar entitlements.** `codesign -d --entitlements - <ruta del .app en DerivedData>`. Esperado: incluye `app-sandbox`, `files.user-selected.read-only` y `files.bookmarks.app-scope`.
- [ ] **Step 3: Volver a correr la suite.** `swift test`. Esperado: PASS (no cambió el comportamiento del modelo).
- [ ] **Step 4: Commit** `git commit -m "App: ventana por dataset con tabla virtualizada"`

---

### Task 8: Inspector de solo lectura

**Files:**
- Create: `Sources/TrimeanUI/{InspectorView,HistogramChart}.swift`
- Modify: `Sources/TrimeanUI/DatasetWindow.swift`, `App/Trimean/TrimeanApp.swift`

**Interfaces:**
- Consumes: `DatasetModel.{selectedColumn, profile, isProfiling, filter, isInspectorPresented}` (Task 6).
- Produces: `struct InspectorView: View` (internal) y `struct HistogramChart: View { init(bins: [HistogramBin], highlight: Bool = false) }`.

**Comportamiento:**

- **Contenedor.** `.inspector(isPresented: $model.isInspectorPresented)` con `.inspectorColumnWidth(min: 260, ideal: 300, max: 420)`.
  - En la toolbar va un botón con `systemImage: "sidebar.right"` que alterna el inspector.
  - En la app se agrega `InspectorCommands()`.
- **Contenido** (`Form` con `.formStyle(.grouped)`, en este orden):
  1. Nombre de la columna como título y su tipo legible: Entero, Decimal, Texto, Booleano, Fecha, Fecha y hora, o el nombre DuckDB para `.other`.
  2. **Faltantes:** "<n> (<porcentaje con 1 decimal>)". Si n > 0, es un botón que pone `model.filter = .isNull(column:)`.
  3. **Distintos:** el número.
  4. **Numéricas:** `HistogramChart` y la grilla de mínimo, Q1, mediana, media, Q3, máximo y desviación estándar. Usa `formatted(.number.precision(.significantDigits(...)))`, con máximo 6 cifras significativas.
  5. **Temporales:** `HistogramChart` y el rango "min – max".
  6. **Texto, booleano y otros:** lista de los valores más frecuentes con su conteo y una barra de proporción (`Gauge` estilo `.accessoryLinear`). Al hacer clic en uno, pone `model.filter = .equals(column:display:)`.
- **Estados.**
  - Mientras `isProfiling`, muestra `ProgressView` con "Calculando sobre <filas> filas…".
  - Sin columna seleccionada, muestra `ContentUnavailableView("Selecciona una columna", systemImage: "tablecells")`.
- **Filtro activo.** Aparece en un `ToolbarItem(placement: .principal)` como una cápsula "Faltantes en monto ✕" o "region = Norte ✕", que al pulsarse quita el filtro. El subtítulo de la ventana pasa a "<visibles> de <total> filas".
- **`HistogramChart`.** Swift Charts `BarMark` por bin, sin ejes Y y con etiquetas X solo en min y max. Usa el color de acento del sistema y una altura de 120 pt.

- [ ] **Step 1: Agregar la prueba del modelo para el texto del filtro.** `@Test func filterLabel()` sobre una propiedad nueva `DatasetModel.filterLabel: String?`. Esperado: `"Faltantes en monto"` para `.isNull` y `"region = Norte"` para `.equals`.
- [ ] **Step 2: Correr y ver que falla.** `swift test --filter TrimeanUITests`.
- [ ] **Step 3: Implementar `filterLabel`, `InspectorView`, `HistogramChart` y la cápsula del filtro.**
- [ ] **Step 4: Correr las pruebas y compilar la app.** `swift test` y el `xcodebuild … build` de Task 7. Esperado: PASS y `BUILD SUCCEEDED`.
- [ ] **Step 5: Revisión visual manual** (la hace el usuario en su Mac, no bloquea el commit):
  1. Abrir `bench.csv` de 2.4 GB (Task 5) o cualquier CSV real.
  2. Al abrir solo se ve la tabla.
  3. El scroll es fluido hasta el final.
  4. Al seleccionar una columna, aparece el inspector.
  5. Al hacer clic en los faltantes, la tabla se filtra.
  6. La cápsula ✕ quita el filtro.
  7. Funciona en modo claro y oscuro.
- [ ] **Step 6: Commit** `git commit -m "UI: inspector de columna de solo lectura"`

---

### Task 9: README de desarrollo

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Agregar la sección "Desarrollo"** con:
  - Requisitos: Xcode 26+ y XcodeGen.
  - `scripts/bootstrap.sh`.
  - `swift test`. La primera vez compila DuckDB, unos 3 min.
  - `TRIMEAN_BENCH=1 swift test -c release --filter BenchmarkTests`.
  - Una línea que explique la versión fijada de DuckDB y por qué.
- [ ] **Step 2: Commit y push** `git commit -m "README: instrucciones de desarrollo" && git push`
