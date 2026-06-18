# ERMaster Modernization Notes

This file records the local source changes made while porting this ERMaster
fork to a modern Eclipse/PDE runtime.

## Target Environment

- Eclipse IDE: 2026-06
- Java: 21
- PDE execution environment: JavaSE-21
- GEF: GEF Classic 3.28.0
- Main plug-in: `org.insightech.er`

## Porting Changes

### Eclipse/PDE metadata

- Updated plug-in metadata for a modern Eclipse PDE workspace.
- Added/kept `Automatic-Module-Name` headers where Eclipse reports the header
  as required.
- Removed the legacy `DOCTYPE` declaration from `org.insightech.er/plugin.xml`
  because current Eclipse XML validation rejects DOCTYPE when
  `http://apache.org/xml/features/disallow-doctype-decl` is enabled.
- Updated the target/platform setup to run against Eclipse 2026-06 and
  GEF Classic 3.28.0.

### GEF Classic 3.28 compatibility

- `CopyAction.java`
  - Replaced the old `ArrayList<EditPart>(List<Object>)` style conversion with
    safer wildcard/list handling for current Java generics.

- `ZoomAdjustAction.java`
  - Updated the zoom listener import to the current Draw2D/GEF API shape.
  - Keeps `ZoomManager.addZoomListener(...)` and
    `removeZoomListener(...)` working with GEF Classic 3.28.0.

- `TableLayout.java`
  - Fixed generic list handling for child figures and separator figures.
  - Changed separator tracking so layout no longer mutates the Draw2D child
    list while laying out figures.
  - This addresses the UI hang observed when opening `.erm` files.

### Runtime hang fix

- The `.erm` open hang was traced from thread dumps to
  `org.insightech.er.editor.view.figure.layout.TableLayout`.
- The layout code now stores separator rectangles separately instead of adding
  separator `Polyline` figures into the container child list during layout.
- This prevents repeated layout/refresh growth and the resulting UI freeze.

### Validation scheduling fix

- `ERDiagramMultiPageEditor.java`
  - Changed the background "Validate ER diagram" work to use a compatible
    workspace scheduling rule.
  - This addresses:
    `Attempted to beginRule: R/, does not match outer scope rule: L/erd-new/newfile.erm`

## PostgreSQL Type Support

### Static type list

- Updated `org.insightech.er/resources/SqlType.xls`.
- Added PostgreSQL built-in/general-purpose types based on PostgreSQL 18
  documentation.
- Added JSON-related types:
  - `json`
  - `jsonb`
  - `jsonpath`
- Added serial/system/network/search/range types including:
  - `smallserial`
  - `macaddr8`
  - `pg_lsn`
  - `pg_snapshot`
  - `txid_snapshot`
  - `tsquery`
  - `tsvector`
  - `int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`,
    `daterange`
  - `int4multirange`, `int8multirange`, `nummultirange`, `tsmultirange`,
    `tstzmultirange`, `datemultirange`
- Added OID/system identifier types:
  - `regclass`
  - `regcollation`
  - `regconfig`
  - `regdictionary`
  - `regnamespace`
  - `regoper`
  - `regoperator`
  - `regproc`
  - `regprocedure`
  - `regrole`
  - `regtype`
  - `xid`
  - `xid8`
  - `cid`
  - `tid`
  - `name`
  - `aclitem`

### PostgreSQL import normalization

- `PostgresTableImportManager.java`
  - Added PostgreSQL type alias normalization for DB import.
  - Examples:
    - `integer` -> `int4`
    - `smallint` -> `int2`
    - `bigint` -> `int8`
    - `boolean` -> `bool`
    - `character` / `char` -> `bpchar`
    - `character varying` -> `varchar`
    - `bit varying` -> `varbit`
    - `time with time zone` -> `timetz`
    - `timestamp with time zone` -> `timestamptz`
  - Added array type normalization:
    - `integer[]` and PostgreSQL internal array names such as `_int4` are
      converted to ERMaster's existing array representation.

### Small serial handling

- `SqlType.java`
  - Added `SQL_TYPE_ID_SMALL_SERIAL`.
  - Added `SQL_TYPE_ID_SMALL_INT`.
  - Added `isSerialType(SqlType)` helper covering `serial`, `bigserial`,
    and `smallserial`.
  - Made `SqlType.main(String[] args)` null-safe because
    `SqlTypeFactory.main(...)` calls it with `null`.

- Updated serial handling call sites to use `SqlType.isSerialType(...)`:
  - `ImportFromDBManagerBase.java`
  - `PostgresDDLCreator.java`
  - `ColumnDialog.java`
  - `TableSet.java`

- `NormalColumn.java`
  - Foreign-key referenced serial type conversion now maps:
    - `serial` -> `integer`
    - `bigserial` -> `bigint`
    - `smallserial` -> `smallint`

## Verification Performed

- Verified PostgreSQL type aliases and import keys in `SqlType.xls`.
  - Result: no missing aliases for the added PostgreSQL types.
- Compiled the changed Java files with JDK 21 and the Eclipse 2026-06 plug-in
  classpath.
  - Result: compilation succeeded.
  - Remaining warnings are pre-existing `new Integer(int)` removal warnings in
    `ImportFromDBManagerBase.java`.

## Known Boundaries

- PostgreSQL user-defined types are not statically enumerable.
  - This includes enum, domain, composite, and extension-provided types such as
    PostGIS `geometry` / `geography`, `hstore`, `citext`, `ltree`, or `vector`.
  - Supporting those properly should be done by reading `pg_type` /
    `pg_namespace` during DB import and adding dynamic custom-type handling.
- The bundled jar files under `org.insightech.er/lib` should keep their
  third-party license notices if the fork is published.
- The original project license is Apache License 2.0. Keep the existing
  `LICENSE` file and original copyright notices when publishing a fork.

## Recommended Next Checks

- In Eclipse:
  - `Project > Clean`
  - Launch a Runtime Eclipse Application
  - Open existing `.erm` files
  - Create a PostgreSQL table using `json`, `jsonb`, `smallserial`, and range
    types
  - Import from a PostgreSQL database containing scalar and array columns

