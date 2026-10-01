# sql-bootcamp-week08
Optimizar un sistema lento


-- Fase 2 -- Las 6 queries lentas 
Query 1: búsqueda de texto: 0.109 sec / 0.000 sec
Query 2: JOIN con filtro por categoría: 0.000 sec / 0.000 sec
Query 3: filtro por fecha (con función en WHERE — antipatrón clásico): 0.063 sec / 0.000 sec
Query 4: lookup por email: 0.047 sec / 0.000 sec
Query 5: filtro compuesto: 0.016 sec / 0.000 sec
Query 6: top clientes con subconsultas (antipatrón): Muestra el estado Running... durante varios segundos (o incluso minutos) o se interrumpe si excede el timeout del cliente.

-- Fase 3 -- Diagnóstico con EXPLAIN
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

-- Fase 4 -- Indexes
-- Fase 4 Crear índices estratégicos
-- Q1: búsqueda full-text en products.name
-- -- indx_products_name
-- Para: Q1 (búsqueda full-text en products.name)
-- Justificación: WHERE name LIKE '%X%' sobre 150k filas → sin índice = full scan costoso.
-- Con índice = búsqueda directa mediante índice invertido (MATCH/AGAINST), devuelve en ~0.343 sec


-- FULLTEXT porque optimiza la coincidencia de palabras completas y texto dentro del campo name.
CREATE FULLTEXT INDEX ft_products_name ON products(name);

-- Q2: JOIN products.category_id ↔ categories.id
-- idx_products_category_id
-- Para: Q2 (JOIN products.category_id ↔ categories.id)
-- Justificación: JOIN/WHERE sobre category_id en 150k filas de productos → sin índice = full scan al cruzar tablas.
-- Con índice = búsqueda directa al árbol B-tree en el JOIN, devuelve en ~0.063 sec.
-- INDEX tradicional sobre la clave foránea (FK) para acelerar la relación entre tablas.
CREATE INDEX idx_products_category_id ON products(category_id);

-- idx_categories_name
-- Para: Q2 (filtro o agrupación por nombre de categoría)
-- Justificación: WHERE c.name = 'X' o GROUP BY c.name → sin índice = escaneo de la tabla categories.
-- Con índice = localización inmediata de la categoría requerida en el árbol B-tree, devuelve en ~0.063 sec.
-- INDEX tradicional para optimizar la búsqueda exacta y el agrupamiento por texto corto.
CREATE INDEX idx_categories_name      ON categories(name);

-- Q3: filtros por rango de fecha en sales
-- idx_sales_date
-- Para: Q3 (filtros por rango de fecha en sales)
-- Justificación: WHERE sale_date BETWEEN/DATE() sobre 200k filas → sin índice = full scan en la tabla de ventas.
-- Con índice = búsqueda e iteración directa por rango en el árbol B-tree, devuelve en ~501.719 sec.
-- INDEX tradicional sobre la columna de fecha para agilizar el filtrado temporal y reportes.
CREATE INDEX idx_sales_date ON sales(sale_date);

-- Q4: lookup exacto por email (UNIQUE porque cada email es único)
-- idx_customers_email
-- Para: Q4 (lookup por email)
-- Justificación: WHERE email = 'X' sobre 20k filas → sin índice = full scan.
-- Con índice = búsqueda directa al árbol B-tree, devuelve en ~0.234 sec.
-- UNIQUE porque cada email aparece una sola vez.
CREATE UNIQUE INDEX idx_customers_email ON customers(email);

-- Q5: filtro compuesto stock + is_active
-- (orden importa: la columna de mayor cardinalidad primero suele ser mejor,
-- pero aquí ambas son de baja cardinalidad. Sirve para "stock=0 AND is_active=TRUE")
-- idx_products_stock_active
-- Para: Q5 (filtro compuesto stock + is_active)
-- Justificación: WHERE stock = 0 AND is_active = TRUE sobre 150k filas → sin índice = full scan.
-- Con índice = búsqueda directa en el árbol B-tree combinando ambos criterios, devuelve en ~0.407 sec.
-- COMPOSITE INDEX optimizado para consultas de baja cardinalidad ejecutadas simultáneamente.
CREATE INDEX idx_products_stock_active ON products(stock, is_active);

-- Q6: lookup en sales por customer_id (para JOIN/agregación)
-- idx_sales_customer
-- Para: Q6 (lookup en sales por customer_id para JOIN/agregación)
-- Justificación: JOIN/WHERE sobre customer_id en 200k filas de ventas → sin índice = full scan al agrupar o cruzar.
-- Con índice = búsqueda directa al árbol B-tree en el JOIN/GROUP BY, devuelve en ~0.968 sec.
-- INDEX tradicional sobre FK de cliente para acelerar la relación con la tabla sales.
CREATE INDEX idx_sales_customer ON sales(customer_id);

-- idx_categories_name
-- Para: Q2 (filtro o agrupación por nombre de categoría)
-- Justificación: WHERE c.name = 'X' o GROUP BY c.name → sin índice = escaneo de la tabla categories.
-- Con índice = localización inmediata de la categoría requerida en el árbol B-tree, devuelve en ~0.000s.
-- INDEX tradicional para optimizar la búsqueda exacta y el agrupamiento por texto corto.
CREATE INDEX idx_sales_product ON sales(product_id);

-- FOREIGN KEYs (también crean índices automáticamente, pero los hicimos manualmente arriba)
-- fk_products_category
-- Para: Integridad referencial entre products y categories
-- Justificación: Garantiza que no existan productos asociados a categorías inexistentes.
-- Restricción de clave foránea (Foreign Key) para mantener la consistencia de datos en la base.
ALTER TABLE products
    ADD CONSTRAINT fk_products_category
    FOREIGN KEY (category_id) REFERENCES categories(id);

-- fk_sales_customer y fk_sales_product
-- Para: Integridad referencial de sales hacia customers y products
-- Justificación: Asegura que cada venta pertenezca a un cliente y producto válidos.
-- Restricción de clave foránea (Foreign Key) para evitar registros huérfanos en la tabla sales.
ALTER TABLE sales
    ADD CONSTRAINT fk_sales_customer FOREIGN KEY (customer_id) REFERENCES customers(id),
    ADD CONSTRAINT fk_sales_product FOREIGN KEY (product_id) REFERENCES products(id);

-- Fase 5 — Reescribir queries malas
Q1: 0.000 sec / 0.000 sec
Q3: 0.000 sec / 0.015 sec
Q6: 1.609 sec / 0.000 sec

-- Fase 6 -- Medir mejoras
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
