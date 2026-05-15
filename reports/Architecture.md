# Software Architecture Report

## 1. Introduction 

## 2. First level: Context level

## 3. Second level: Container level

## 4. Third level: Component level

## 5. Architectural characteristics 

Since DuckDB is a library, the developers chose to design the database with a monolithic structure. In particular, examining the code, they adopted a modular-like architecture, distinguished by a clear separation of the source code based on functionality. This approach is excellent for maintaining a simple structure while still allowing for some degree of modularity. The developers decided against implementing a microkernel architecture because they anticipated frequent updates to the components and a high level of interdependence among them. It is also evident that the code has been organized around components.

By analyzing the coupling matrix, extracted from the project using Python, the communication between modules and their distinctions become very clear. We can observe that the project follows the typical architecture of embedded OLAP systems, but with some differences: the designers opted for a simplified module structure, grouping together those without a clear separation in the characteristic architecture. The result is illustrated in the previous C4 diagrams; here, we will limit ourselves to commenting on the strongly connected structures, which are:

- Catlog &rarr; Parser;<br>
- Parser &rarr; Planner;<br>
- Planner &rarr; Optimazier (very strong coupling);<br>
- Planner &rarr; Execution.

There are also weakly connected elements for the sake of development simplicity.

<img src="graphyc/coupling.png" alt="coupling metrics" width="75%" height="75%">
<br><br><br>

To reinforce the idea of grouping certain modules, we can look at the cohesion table, also extracted from the project using Python, which highlights the most cohesive modules. It emerges that the parser is the most cohesive module of all: it contains all the submodules required to translate SQL queries, and within it, there are other tightly linked submodules. This helps developers simplify maintainability, reuse, and module testing. Following that, we find other modules that tend to resemble the canonical structure more closely, such as the optimizer, which contains the files for optimizing queries and tends to have a clearer separation.

| Module    | Internal Edges | External Edges | Cohesion |
|:---------:|:--------------:|:--------------:|:--------:|
| parser    | 29070          | 1084           | 0.9641   |
| execution | 40602          | 24929          | 0.6196   |
| storage   | 14577          | 9206           | 0.6129   |
| planner   | 36290          | 26973          | 0.5736   |
| optimizer | 13340          | 28151          | 0.3215   |
| catalog   | 1560           | 6902           | 0.1844   |

In conclusion, thanks to its modular monolithic structure, the code is easy to develop while also facilitating the addition of new components. Moreover, it streamlines the code, improving performance by avoiding unnecessary calls to other modules due to the grouping, and it also simplifies testing. However, being a monolith, it has the drawback of needing to be fully recompiled, and changes to central files can propagate to other modules because of the strong coupling between some of them.
