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

Knowledge dependencies were measured from Git co-change relations reported in the linked [dataset](https://docs.google.com/spreadsheets/d/1pa1tWlTltAW4g8DvtOI5R4BQnfUC8SB2dUq-Mi9qVB4/edit?usp=sharing). Each row is a pair of files that changed together in the same commits. The `degree` column is the percentage of revisions in which the pair co-changed, while `average-revs` is the average number of revisions of the two files in the pair. To keep the comparison focused on source code, only `.hpp` and `.cpp` files were considered.

After this filter, the co-change dataset contains `2512` pairs. These pairs were compared with the static include dataset by checking whether either file directly includes the other. `781` co-change pairs also have a direct include dependency, while `1731` do not. So only `31.1%` of the knowledge dependencies are also direct static dependencies.

For the comparison, the most useful examples are pairs connected to files already identified in the static analysis. Some of them are consistent with static dependencies:

| File Pair                                   | Degree | Avg. File Revs | Interpretation                 |
|---------------------------------------------|-------:|----------:|--------------------------------|
| `enum_util.cpp` / `enum_util.hpp`           |     50 |       420 | generated enum conversion code |
| `client_context.hpp` / `client_context.cpp` |     37 |       379 | broad session interface        |
| `binder.hpp` / `binder.cpp`                 |     40 |       290 | binding infrastructure         |
| `data_table.hpp` / `data_table.cpp`         |     47 |       473 | broad storage class            |

These are implementation/header relations, but they also support the static dependency findings. `client_context`, `binder`, and `data_table` were already identified as central or broad files, and their implementations co-change with their declarations. `enum_util` again needs to be interpreted separately, because it is generated code.

The percentages are not always very high because these files are large interfaces. A commit can change only `client_context.cpp` or only `client_context.hpp`, for example, without changing both. This is why a moderate percentage with many revisions can be more meaningful than a `100%` pair with only a few revisions.

The more interesting cases are co-change pairs that do not have a direct include relation:

| File Pair                                       | Degree | Avg. File Revs | Interpretation                              |
|-------------------------------------------------|-------:|----------:|---------------------------------------------|
| `data_table.cpp` / `local_storage.cpp`          |     30 |       498 | table storage and transaction-local changes |
| `join_hashtable.cpp` / `physical_hash_join.cpp` |     34 |       473 | hash join implementation split              |
| `row_group.cpp` / `row_group_collection.cpp`    |     33 |       375 | table storage structures                    |
| `config.hpp` / `settings.hpp`                   |     36 |       358 | configuration/settings interface            |
| `config.cpp` / `settings.cpp`                   |     32 |       324 | configuration/settings implementation       |

These are the main co-change relations that are not represented by direct include edges. `data_table.cpp` was already discussed as a high outgoing-dependency implementation file. Its co-change with `local_storage.cpp` shows that table storage and transaction-local changes are maintained together even when the relation is not a direct include edge. The same applies to `row_group.cpp` and `row_group_collection.cpp`, which sit below `DataTable` in the storage layer.

`physical_hash_join.cpp` was also mentioned in the static dependency section. Its co-change with `join_hashtable.cpp` shows the same pattern: one file implements the physical operator and pipeline behavior, while the other implements the hash table mechanics. The configuration pairs are similar. `config.hpp` and `config.cpp` have direct static dependencies, but the co-change data also connects them to `settings.hpp` and `settings.cpp`, because settings declarations, lookup helpers, and configuration registration evolve together.

Overall, static dependencies are better for compile-time coupling and central headers. Co-change dependencies add a different view: they show maintenance coupling around the same areas found in the static analysis, especially storage, hash joins, configuration, and session/binding infrastructure.

## 2. Patterns
Four patterns are identified below. Each is verified against the source code. Class inheritance diagrams are generated by Doxygen using the build configuration already set up by the DuckDB authors.

### 2.1 Facade

**Roles.** [`ClientContext`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/main/client_context.hpp) acts as the unified Facade. Some of the subsystems behind the Facade are [`Parser`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/parser/parser.hpp), [`StatementPreprocessor`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/planner/statement_preprocessor.hpp), and [`QueryProfiler`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/main/query_profiler.hpp).

**Problem Solved:** It isolates external clients and API bindings from the intricate lifecycle of query execution, thereby suppressing the high architectural coupling identified in the preceding dependency analysis.

![ClientContext inheritance diagram](./img/DesignPatterns/classduckdb_1_1_client_context__inherit__graph.png)

*Figure 12: `ClientContext` inheritance diagram.*

The implementation of the Facade pattern enforces a strictly linear and unidirectional execution pipeline. While a *Mediator* configuration could theoretically be posited to coordinate inter-subsystem communication, such an alternative would introduce superfluous bidirectional coupling. In high-performance database engines, core compilation components such as the `Parser` must operate as pure, stateless transformations that map input representations to deterministic outputs. The Facade pattern ensures that upstream clients are completely decoupled from individual pipeline stages, thereby shielding public APIs from internal refactoring within the SQL compilation layers.

---

### 2.2 Composite

**Roles.** [`BaseExpression`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/parser/base_expression.hpp) defines the Component interface. Some nodes contain sub-expressions for example, [`FunctionExpression`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/parser/expression/function_expression.hpp) holds its arguments and [`ComparisonExpression`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/parser/expression/comparison_expression.hpp) holds a left and right side. These are Composites. Nodes with no children, such as [`ConstantExpression`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/parser/expression/constant_expression.hpp) and [`ColumnRefExpression`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/parser/expression/columnref_expression.hpp), are Leaves. [`Binder`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/planner/binder.hpp), [`Optimizer`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/optimizer/optimizer.hpp), and [`PhysicalPlanGenerator`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/execution/physical_plan_generator.hpp) are Clients traversing the tree uniformly.

**Problem Solved:** It provides a uniform interface to manipulate SQL expressions and logical/physical plan nodes of arbitrary nesting depth, eliminating the need for conditional type dispatch during tree traversal.

![BaseExpression inheritance diagram](./img/DesignPatterns/classduckdb_1_1_base_expression__inherit__graph.png)

*Figure 13: `BaseExpression` inheritance diagram.*

The Composite pattern constitutes the foundational abstraction for the representation of hierarchical syntax trees. The *Interpreter* pattern is sometimes cited as an alternative in this context, but the relationship between the two is complementary rather than competitive — Interpreter's grammar rules depend on a Composite topology to represent their recursive structure, so replacing Composite with Interpreter is not a valid substitution; it would simply layer Interpreter on top of an implicit Composite.

---

### 2.3 Visitor

**Roles.** [`LogicalOperatorVisitor`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/planner/logical_operator_visitor.hpp) defines the Visitor interface. Concrete Visitors include, for example, [`ExpressionRewriter`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/optimizer/expression_rewriter.hpp) and [`ColumnBindingResolver`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/execution/column_binding_resolver.hpp). The logical plan and bound expression trees are the visited Elements. The optimizer/planner workflow is the Client, sequencing Visitors as passes.

**Problem solved.** New traversal operations must be added without modifying tree node classes. All passes — expression rewriting, column binding resolution, lifetime analysis, unused column removal, join elimination — share one traversal contract, keeping node code stable.

![LogicalOperatorVisitor inheritance diagram](./img/DesignPatterns/classduckdb_1_1_logical_operator_visitor__inherit__graph.png)

*Figure 14: `LogicalOperatorVisitor` inheritance diagram.*

![ExpressionRewriter inheritance diagram](./img/DesignPatterns/classduckdb_1_1_expression_rewriter__inherit__graph.png)

*Figure 15: `ExpressionRewriter` inheritance diagram.*

[`ExpressionIterator`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/planner/expression_iterator.hpp) and [`ParsedExpressionIterator`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/parser/parsed_expression_iterator.hpp) already exist in the codebase and use `std::function` callbacks instead of a class hierarchy, making them a concrete *Iterator*-based alternative already in production for expression-level traversal. Extending this model to operator traversal would replace Visitor subclasses with lambdas. Even though this eliminates class boilerplate and makes passes composable, any state a lambda needs to track during traversal is hidden inside the closure and harder to inspect in a debugger than a named class field, and there is no interface for the compiler to enforce correctness against.

---

### 2.4 Strategy

**Roles.** [`Rule`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/optimizer/rule.hpp) is the Strategy interface. Concrete Strategies include, for example, [`ConstantFoldingRule`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/optimizer/rule/constant_folding.hpp), [`ArithmeticSimplificationRule`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/optimizer/rule/arithmetic_simplification.hpp), and [`CaseSimplificationRule`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/optimizer/rule/case_simplification.hpp), among [others](https://github.com/duckdb/duckdb/tree/main/src/optimizer/rule) under `src/optimizer/rule/`. [`ExpressionRewriter`](https://github.com/duckdb/duckdb/blob/main/src/include/duckdb/optimizer/expression_rewriter.hpp) is the Context iterating and applying them.

**Problem Solved:** It decouples expression-level optimization algorithms from the traversal framework, making rewriting rules entirely modular and extensible.

![Rule inheritance diagram](./img/DesignPatterns/classduckdb_1_1_rule__inherit__graph.png)

*Figure 16: `Rule` inheritance diagram.*

 The [`ApplyRules`](https://github.com/duckdb/duckdb/blob/main/src/optimizer/expression_rewriter.cpp) loop in `ExpressionRewriter` already tries rules in sequence and applies the first match — semantics close to a chain. A formal *Chain of Responsibility* would give each `Rule` a `next` pointer, making ordering an explicit structural contract rather than a vector-position convention. Even though this keeps the handle-or-pass decision within the rule itself and makes ordering explicit, linked chains are harder to debug than a flat vector and adding `next` to the `Rule` interface complicates stateless rule reuse.

## 3. Summary
Two dependency views were measured: static `#include` relations and knowledge dependencies from Git co-change data.

The static graph reflects DuckDB's layered architecture directly. include/common is the dominant dependency sink — shared infrastructure has no natural scope boundary and changes there propagate everywhere. Above that, coupling concentrates at the boundaries where layers meet: session state (`client_context.cpp`), binding (`binder.cpp`), optimizer orchestration (`optimizer.cpp`), configuration (`config.hpp`), exceptions (`exception.hpp`), and schema catalog management (`bind_create.cpp`, `catalog.cpp`). High outgoing degree in implementation files consistently marks coordination boundaries rather than isolated algorithms.

The key finding from the co-change data is that only 31.1% of co-changing pairs have a direct include edge. Pairs like `join_hashtable.cpp/physical_hash_join.cpp` and `config.hpp/settings.hpp` evolve together due to shared domain responsibility, invisible to the static graph. The two analyses converge on the same hotspots — storage, hash joins, configuration, session state, and binding — but co-change reveals the maintenance cost that compile-time coupling alone cannot show.

 All four patterns address the same underlying problem: the dependency analysis shows that many subsystems converge on a small number of central classes, creating high coupling. The *Facade* does not remove this coupling — it manages it by giving external callers a single, stable entry point through `ClientContext`, hiding the complexity of the query pipeline behind it. The *Composite*, *Visitor*, and *Strategy* patterns work together on the optimizer and planner side: *Composite* gives the expression tree a uniform interface so it can be traversed without knowing the type of each node, *Visitor* separates each traversal operation into its own class so new passes can be added without touching tree node code, and *Strategy* makes each rewrite rule a self-contained unit that can be added or swapped independently. This is especially important because so many subsystems depend on planner structures — any instability there would ripple widely. The remaining concern is that the *Facade* concentrates risk: `ClientContext` is both the most depended-on class in the system and the single point through which all query execution flows.