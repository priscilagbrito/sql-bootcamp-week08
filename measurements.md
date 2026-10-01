# sql-bootcamp-week08
Optimizar un sistema lento

Fase 2 — Las 6 queries lentas 
Query 1: búsqueda de texto: 0.109 sec / 0.000 sec
Query 2: JOIN con filtro por categoría: 0.000 sec / 0.000 sec
Query 3: filtro por fecha (con función en WHERE — antipatrón clásico): 0.063 sec / 0.000 sec
Query 4: lookup por email: 0.047 sec / 0.000 sec
Query 5: filtro compuesto: 0.016 sec / 0.000 sec
Query 6: top clientes con subconsultas (antipatrón): Muestra el estado Running... durante varios segundos (o incluso minutos) o se interrumpe si excede el timeout del cliente.

Fase 3 — Diagnóstico con EXPLAIN
Query           Problema Metrica(EXPLAIN)                                               Solución Metrica(EXPLAIN) 
Q1          Type: ALL, Key: Null, Row: ~150,000                                  Type: fulltext, Key: ft_products_name, Row: ~1 - 10
Q2    Type: ALL (en ambas tablas), Key: Null, Row: ~150,000 $\times$ 1,000     Type: feq_ref / ref, Key: idx_products_category_id, Row:                                                                                  ~1 por fila
Q3    Type: ALL, Key: NULL (por usar DATE(sale_date)), Row: ~200,000         Type: const, Key: idx_sales_date, Row: Solo el rango filtrado
Q4       Type: ALL, Key: Null, Row: ~20,000                                       Type: const, Key: idx_customers_email, Row: 1
Q5   Type: ALL, Key: Null, Row: ~150,000                        Type: ref, Key: idx_products_stock_active, Row: Solo filas que coinciden
Q6 Type: DEPENDENT SUBQUERY, Key: ($2 \times 20,000$ subconsultas,       Type: ref (en sales) / ALL (1 sola vez en customers), Key:       Row: NULL    ~200,000 por cada cliente                                    idx_sales_customer, Row: ~10 - 15 por cliente

Resumen de las transformaciones clave en EXPLAIN
1. type (Tipo de acceso): Evolucionó desde el peor escenario (ALL = escaneo completo de la tabla) hacia los niveles más eficientes del motor SQL (const, eq_ref, ref, range y fulltext).

2. key (Índice utilizado): Pasó de ser NULL (sin uso de índices) a utilizar las claves B-Tree y FULLTEXT recién creadas.

3. rows (Filas examinadas): Se redujo dramáticamente, pasando de evaluar cientos de miles de filas por consulta a examinar únicamente las filas necesarias (en muchos casos, solo 1 fila).

Query	Problema
Q1	LIKE '%xxx%' con % al inicio: ningún índice B-tree puede ayudar. Necesita FULLTEXT INDEX.
Q2	categories.name y products.category_id sin índice → full scan + nested loop
Q3	DATE(sale_date) aplica función a la columna → MySQL no puede usar índice aunque exista
Q4	email sin índice → 20,000 comparaciones para encontrar 1 fila
Q5	stock y is_active sin índice; combinación común que merece índice compuesto
Q6	Dos subconsultas correlacionadas que se ejecutan una vez por cliente → 20,000 ejecuciones de cada subconsulta

-- Fase 5 — Reescribir queries malas
Q1: 0.000 sec / 0.000 sec
Q3: 0.000 sec / 0.015 sec
Q6: 1.609 sec / 0.000 sec

Fase 6 — Medir mejoras
Q2: 0.000 sec / 0.000 sec
Q4: 0.000 sec / 0.000 sec
Q5: 0.000 sec / 0.000 sec

Consulta,      Descripción / Cambio Aplicado,       Tiempo Original (Sin Índice / Antipatrón),     Nuevo Tiempo (Optimizado)
Q1,             Búsqueda FULLTEXT en products.name,                  ~0.109 sec,                               0.001 sec
Q2,             JOIN de categorías con índice category_id,           ~0.000 sec,                               0.000 sec
Q3,             Búsqueda por fecha con índice sale_date,             ~0.063 sec,                               0.001 sec
Q4,             Búsqueda por email con índice UNIQUE,                ~0.047 sec,                               0.000 sec
Q5,             Filtro compuesto stock = 0 AND is_active = TRUE,     ~0.016 sec,                               0.001 sec
Q6,             Reescritura a LEFT JOIN + GROUP BY e índices FK,     > 30.0 sec (Error 2013 / Running),        1.609 sec
