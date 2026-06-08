The project under consideration is DuckDB, an embedded OLAP database system, meaning it is integrated directly into the developed application and designed for high-performance local data analysis without the need for a separate server.

The main purpose of the application is to provide fast analytical query services on large datasets, directly in-process, simplifying data analysis and reducing infrastructural complexity. This makes it easier to access and manage data without the need to build a traditional database infrastructure.

The software library is open source and distributed under the MIT license, one of the most permissive licenses, which not only allows the software to be used for any purpose, but also permits modification of the source code, integration into proprietary and commercial software, and redistribution, including for paid products.

The main contributors to the creation of DuckDB are two database researchers: Mark Raasveldt, who is also the principal architect of the system, and Hannes Mühleisen, who is responsible for the vision and technical direction of the project. Together, they started the development of the DBMS at the Centrum Wiskunde & Informatica (CWI), a mathematical and computer science research institute in Amsterdam, in 2018. The first public release occurred in 2019. They later founded the DuckDB Foundation to ensure the maintenance and governance of the project in an open-source manner. Today, maintenance and development are carried out by a team of around 20 engineers and researchers.

The key stakeholders, in addition to the developers and maintainers of DuckDB mentioned above, are also its users, the main ones being:

Data scientists and data analysts, who use DuckDB to analyze datasets via SQL;
Data engineers, who use it in ETL/ELT pipelines and data transformation processes;
Software developers, who integrate DuckDB into their applications to provide data analysis functionality.

Other stakeholders include companies and organizations that use applications based on DuckDB, researchers and academic institutions that use it for experimentation and discovery, and the community, which is essential for reporting bugs and contributing code.

As previously mentioned, DuckDB is an embedded analytical Database Management System. This allows the application that integrates it to execute efficient queries without the need for a dedicated server. The system is optimized for OLAP workloads, such as statistical analysis, aggregations, complex joins, and queries on large datasets. It supports various data formats such as CSV and JSON, and can be integrated with different programming languages, including Java, Python, R, and C++. Other key features include support for parallel query execution and integration with data science and machine learning ecosystems. In the database, data is organized in columns rather than rows in order to improve the response time of analytical queries.

The source code, mainly developed in C++, contains about 300,000 lines of code in the core engine and, considering the entire library, it reaches approximately 500,000 lines of code.

The code is spread across tens of thousands of files, around 50,000, organized into several main folders in the Git root repository:

.github, containing GitHub Actions workflows, CI/CD pipelines, issue templates, and pull request templates;
benchmark, which contains benchmarks used to measure database performance;
data, where datasets and files used for testing and benchmarking are stored;
examples, which contains usage examples of DuckDB in C, C++, and Python;
extension, which groups official database extensions;
logo, which contains project logos and graphical assets;
scripts, which is a set of tools useful for system development;
src, the most important folder on which this report is based, containing all DBMS source code;
test, which contains thousands of automated tests and validation suites, written not only in C++ but also in other languages due to the size of the project;
third_party, containing external libraries included in the project;
tools, containing additional development tools and internal utilities.

Focusing on the modules inside the src folder, we find:

catalog, which is used to know what exists in the database;
common, which is the set of basic database utilities (e.g., common data structures and reusable functions);
execution, which is responsible for physically executing the query;
function, which implements all functions available in SQL queries;
include, which exposes the internal system APIs and data structures;
logging, which is responsible for recording internal database events for debugging and monitoring purposes;
main, the entry point of the database;
optimizer, which reduces query execution costs;
parallel, which manages query execution parallelism;
parser, responsible for converting SQL into an AST;
planner, which builds the logical query plan;
storage, which manages physical data reading and writing;
transaction, which manages data consistency and isolation.