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
