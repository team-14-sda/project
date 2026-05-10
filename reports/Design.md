# Software Design Report

## 1. Dependencies

### 1.1 Method

The dependency analysis focuses only on DuckDB's `src/` directory and considers static code dependencies and knowledge dependencies based on Git co-change relations.

### 1.2 Code Dependencies

This analysis captures compile-time coupling caused by `#include` directives. CppDepend and SciTools Understand were used to inspect the project, but a custom Python script was used to obtain a consistent file-level dataset. The script extracted include dependencies between files inside `src/`, excluding standard library headers and external dependencies. In the [resulting tabular dataset](https://docs.google.com/spreadsheets/d/1t4Zn6AM0BTieD4ttUnRgLE4Y9ylnJeGL3xuD3J9N6DU/edit?usp=sharing), outgoing dependencies show which internal DuckDB files are directly included by a file, while incoming dependencies show how many other DuckDB files directly include it. The dataset contains `2804` files: `1387` headers and `1417` implementation files.

The distinction between headers and implementation files is important. Headers have lower outgoing dependencies on average (`3.32`) but much higher incoming dependencies (`10.17`), while implementation files have higher outgoing dependencies (`6.71`) but effectively no incoming dependencies. This is expected since `.cpp` files include headers, while direct inclusion of `.cpp` files is generally avoided. For this reason, incoming degree is more useful for identifying central headers, and outgoing degree is more useful for identifying implementation files that coordinate many DuckDB subsystems.

#### 1.2.1 Dependencies in Header Files

The most depended-on files are central interface and utility headers:

![Top 15 header files with the most incoming dependencies](/reports/img/CodeDependencies/Top%2015%20header%20files%20with%20the%20most%20incoming%20dependencies.svg)

*Figure 1: Header files with the most incoming dependencies.*

The top incoming headers fall into two main groups. The first group is shared infrastructure: `exception.hpp` is the most depended-on header because errors can be raised from almost every DuckDB layer, while `string_util.hpp` and `common.hpp` provide common string operations, constants, helper functions, and basic DuckDB types.

![exception_dependencies](/reports/img/CodeDependencies/HeaderFilesIncomingDependencies/exception_dependencies.svg)

*Figure 2: `exception.hpp` incoming dependencies.*

| ![string_util_dependencies](/reports/img/CodeDependencies/HeaderFilesIncomingDependencies/string_util_dependencies.svg) | ![common_dependencies](/reports/img/CodeDependencies/HeaderFilesIncomingDependencies/common_dependencies.svg) |
|-------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| *Figure 3: `string_util.hpp` incoming dependencies.*                                                                    | *Figure 4: `common.hpp` incoming dependencies.*                                                               |

The second group is core query-processing interfaces. `client_context.hpp` defines the session-level context used for query execution, transactions, configuration, interruption, profiling, and result handling; `binder.hpp` defines the infrastructure that connects parsed SQL to catalog objects, table bindings, expressions, and logical operators. These headers are highly depended on because many subsystems need access to session state or bound query structures.

| ![client_context_header_dependencies](/reports/img/CodeDependencies/HeaderFilesIncomingDependencies/client_context_header_dependencies.svg) | ![binder_dependencies](/reports/img/CodeDependencies/HeaderFilesIncomingDependencies/binder_dependencies.svg) |
|---------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| *Figure 5: `client_context.hpp` incoming dependencies.*                                                                                     | *Figure 6: `binder.hpp` incoming dependencies.*                                                               |

The header files with the highest outgoing dependencies are:

| File                                           | Outgoing Dependencies |
|------------------------------------------------|----------------------:|
| `src/include/duckdb/planner/operator/list.hpp` |                    48 |
| `src/include/duckdb/main/config.hpp`           |                    29 |
| `src/include/duckdb/parser/statement/list.hpp` |                    26 |
| `src/include/duckdb/main/client_context.hpp`   |                    22 |
| `src/include/duckdb/planner/binder.hpp`        |                    20 |

These high-outgoing headers are mostly aggregation or coordination points. `planner/operator/list.hpp` and `parser/statement/list.hpp` collect families of logical operator and SQL statement declarations. `config.hpp` is broad because database configuration touches access mode, memory limits, compression, extensions, logging, storage, replacement scans, and optimizer settings. `client_context.hpp` and `binder.hpp` are central session and binding interfaces.

#### 1.2.2 Outgoing Dependencies in Implementation Files

The implementation files with the most outgoing dependencies are:

![Top 15 implementation files with the most outgoing dependencies](/reports/img/CodeDependencies/Top%2015%20implementation%20files%20with%20the%20most%20outgoing%20dependencies.svg)

*Figure 7: Implementation files with the most outgoing dependencies.*

The top outgoing implementation files are mostly subsystem coordinators. Generated/tooling files are special cases: `enum_util.cpp` is generated by `scripts/generate_enum_util.py`, `serialize_nodes.cpp` is generated serialization code, and `symbols.cpp` exists for LLDB symbol instantiation. These should not be interpreted as ordinary handwritten coupling.

Among handwritten files, `client_context.cpp` and `database.cpp` coordinate session/database state with parsing, planning, execution, transactions, extensions, storage, logging, and scheduling.

![client_context_dependencies](/reports/img/CodeDependencies/ImplementationFilesOutgoingDependencies/client_context_dependencies.svg)

*Figure 8: `client_context.cpp` outgoing dependencies.*

`bind_create.cpp`, `catalog.cpp`, and `duck_schema_entry.cpp` coordinate SQL creation and catalog/schema state.

| ![bind_create_dependencies](/reports/img/CodeDependencies/ImplementationFilesOutgoingDependencies/bind_create_dependencies.svg) | ![duck_schema_entry_dependencies](/reports/img/CodeDependencies/ImplementationFilesOutgoingDependencies/duck_schema_entry_dependencies.svg) |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| *Figure 9: `bind_create.cpp` outgoing dependencies.*                                                                            | *Figure 10: `duck_schema_entry.cpp` outgoing dependencies.*                                                                                 |

`optimizer.cpp` wires together optimizer passes, while execution/storage files such as `physical_hash_join.cpp`, `table_scan.cpp`, `checkpoint_manager.cpp`, `wal_replay.cpp`, and `data_table.cpp` connect operators, filters, table storage, indexes, transactions, serialization, checkpointing, and recovery. Overall, high outgoing degree usually marks a boundary or coordination file rather than an isolated algorithm.

![optimizer_dependencies](/reports/img/CodeDependencies/ImplementationFilesOutgoingDependencies/optimizer_dependencies.svg)

*Figure 11: `optimizer.cpp` outgoing dependencies.*

#### 1.2.3 Files with the Least Code Dependencies

The files with the fewest dependencies are mostly tiny operator headers, compatibility stubs, or small catalog implementation units. Examples include `constant_operators.hpp`, `aggregate_operators.hpp`, `extension_util.hpp`, `dependency_dependent_entry.cpp`, `dependency_subject_entry.cpp`, and `index_catalog_entry.cpp`.

#### 1.2.4 Code Dependencies at Module Level

At module level, the strongest dependency flows are:

| From Module | To Module           | Dependency Pairs |
|-------------|---------------------|-----------------:|
| `common`    | `include/common`    |             1031 |
| `planner`   | `include/planner`   |              649 |
| `parser`    | `include/parser`    |              532 |
| `execution` | `include/execution` |              523 |
| `function`  | `include/common`    |              502 |
| `optimizer` | `include/planner`   |              485 |

Several implementation modules depend heavily on their matching header modules, such as `planner -> include/planner`, `parser -> include/parser`, and `execution -> include/execution`. `include/common` is the strongest shared target, receiving `4995` incoming dependencies in the file-level summary, because DuckDB's common layer provides low-level types, containers, exceptions, constants, filesystem abstractions, and utility functions. The `optimizer -> include/planner` relation is also meaningful because optimizer passes transform logical plans and therefore depend on planner data structures.

Overall, the static dependencies match DuckDB's organization as an embedded analytical database engine with a SQL frontend, optimizer, execution layer, storage layer, and shared common infrastructure. The highest coupling appears around session state, binding, optimizer orchestration, configuration, exceptions, and schema catalog management, which represent areas where many subsystems meet.

### 1.3 Knowledge Dependencies

## 2. Patterns

## 3. Summary
