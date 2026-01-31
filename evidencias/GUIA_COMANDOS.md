# Guía Rápida de Comandos - SymmetricDS Replication

## Gestión de Contenedores

### Iniciar todos los servicios
```powershell
cd "c:\Users\wadri\Desktop\abdd-2025-2-main"
docker compose up -d
```

### Detener todos los servicios
```powershell
docker compose down
```

### Ver estado de contenedores
```powershell
docker compose ps
```

### Reiniciar servicios específicos
```powershell
docker compose restart symmetricds-america
docker compose restart symmetricds-europe
```

### Ver logs en tiempo real
```powershell
# Logs de América
docker compose logs -f symmetricds-america

# Logs de Europa
docker compose logs -f symmetricds-europe

# Logs de todas las bases de datos
docker compose logs -f postgres-america mysql-europe
```

---

## Verificación de Estado

### Verificar conectividad de bases de datos

**PostgreSQL (América)**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "SELECT version();"
```

**MySQL (Europa)**:
```powershell
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "SELECT VERSION();"
```

### Verificar registro de nodos

**En PostgreSQL**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "SELECT node_id, node_group_id, external_id, sync_enabled FROM sym_node;"
```

**En MySQL**:
```powershell
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "SELECT node_id, node_group_id, external_id, sync_enabled FROM sym_node;"
```

### Verificar estado de canales

**PostgreSQL**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "SELECT channel_id, processing_order, enabled FROM sym_channel ORDER BY processing_order;"
```

**MySQL**:
```powershell
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "SELECT channel_id, processing_order, enabled FROM sym_channel ORDER BY processing_order;"
```

---

## Monitoreo de Replicación

### Ver batches salientes (Outgoing Batches)

**PostgreSQL → MySQL**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "SELECT batch_id, status, node_id, error_flag, create_time FROM sym_outgoing_batch ORDER BY batch_id DESC LIMIT 10;"
```

### Ver batches entrantes (Incoming Batches)

**MySQL (recibiendo de PostgreSQL)**:
```powershell
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "SELECT batch_id, status, node_id, error_flag, create_time FROM sym_incoming_batch ORDER BY batch_id DESC LIMIT 10;"
```

**PostgreSQL (recibiendo de MySQL)**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "SELECT batch_id, status, node_id, error_flag, create_time FROM sym_incoming_batch ORDER BY batch_id DESC LIMIT 10;"
```

### Ver datos capturados (sym_data)

**PostgreSQL**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "SELECT data_id, table_name, event_type, channel_id, create_time FROM sym_data ORDER BY data_id DESC LIMIT 10;"
```

---

## Consultas de Validación

### Contar registros en todas las tablas

**PostgreSQL**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
SELECT 'products' as table_name, COUNT(*) as count FROM products
UNION ALL SELECT 'inventory', COUNT(*) FROM inventory
UNION ALL SELECT 'customers', COUNT(*) FROM customers
UNION ALL SELECT 'promotions', COUNT(*) FROM promotions;"
```

**MySQL**:
```powershell
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "
SELECT 'products' as table_name, COUNT(*) as count FROM products
UNION ALL SELECT 'inventory', COUNT(*) FROM inventory
UNION ALL SELECT 'customers', COUNT(*) FROM customers
UNION ALL SELECT 'promotions', COUNT(*) FROM promotions;"
```

### Verificar triggers de base de datos

**PostgreSQL**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
SELECT trigger_name, event_object_table, action_timing, event_manipulation 
FROM information_schema.triggers 
WHERE trigger_name LIKE '%sym%' 
ORDER BY event_object_table, event_manipulation;"
```

**MySQL**:
```powershell
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "
SELECT TRIGGER_NAME, EVENT_OBJECT_TABLE, ACTION_TIMING, EVENT_MANIPULATION 
FROM information_schema.triggers 
WHERE TRIGGER_SCHEMA='globalshop' AND TRIGGER_NAME LIKE '%SYM%' 
ORDER BY EVENT_OBJECT_TABLE, EVENT_MANIPULATION;"
```

---

## Pruebas de Replicación

### Test INSERT: PostgreSQL → MySQL
```powershell
# 1. Insertar en PostgreSQL
$TEST_ID = "TEST-$(Get-Date -Format 'yyyyMMddHHmmss')"
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
INSERT INTO products (product_id, product_name, description, base_price, category, is_active, created_at, updated_at) 
VALUES ('$TEST_ID', 'Test Product', 'Testing replication', 99.99, 'Test', true, NOW(), NOW());"

# 2. Esperar 10 segundos
Start-Sleep -Seconds 10

# 3. Verificar en MySQL
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "
SELECT product_id, product_name, base_price FROM products WHERE product_id='$TEST_ID';"
```

### Test INSERT: MySQL → PostgreSQL
```powershell
# 1. Insertar en MySQL
$TEST_ID = "TEST-$(Get-Date -Format 'yyyyMMddHHmmss')"
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "
INSERT INTO products (product_id, product_name, description, base_price, category, is_active, created_at, updated_at) 
VALUES ('$TEST_ID', 'Test Product', 'Testing replication', 99.99, 'Test', 1, NOW(), NOW());"

# 2. Esperar 10 segundos
Start-Sleep -Seconds 10

# 3. Verificar en PostgreSQL
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
SELECT product_id, product_name, base_price FROM products WHERE product_id='$TEST_ID';"
```

### Test UPDATE: Bidireccional
```powershell
# UPDATE en PostgreSQL
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
UPDATE products SET base_price=149.99 WHERE product_id='PROD-EUR-001';"

# Esperar y verificar en MySQL
Start-Sleep -Seconds 8
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "
SELECT product_id, base_price FROM products WHERE product_id='PROD-EUR-001';"
```

### Test DELETE: Bidireccional
```powershell
# Crear producto de prueba
$TEST_ID = "DELETE-TEST-$(Get-Date -Format 'yyyyMMddHHmmss')"
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
INSERT INTO products (product_id, product_name, description, base_price, category, is_active, created_at, updated_at) 
VALUES ('$TEST_ID', 'Delete Test', 'Will be deleted', 1.00, 'Test', true, NOW(), NOW());"

Start-Sleep -Seconds 10

# Eliminar de PostgreSQL
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
DELETE FROM products WHERE product_id='$TEST_ID';"

Start-Sleep -Seconds 8

# Verificar eliminación en MySQL
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "
SELECT COUNT(*) as count FROM products WHERE product_id='$TEST_ID';"
```

---

## Troubleshooting

### Ver errores de batches

**PostgreSQL**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
SELECT batch_id, status, error_flag, sql_state, sql_code, sql_message 
FROM sym_outgoing_batch 
WHERE error_flag = 1 
ORDER BY batch_id DESC 
LIMIT 5;"
```

**MySQL**:
```powershell
docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop -e "
SELECT batch_id, status, error_flag, sql_state, sql_code, sql_message 
FROM sym_incoming_batch 
WHERE error_flag = 1 
ORDER BY batch_id DESC 
LIMIT 5;"
```

### Ver logs recientes con errores

**América**:
```powershell
docker compose logs --tail=100 symmetricds-america | Select-String -Pattern "ERROR|WARN"
```

**Europa**:
```powershell
docker compose logs --tail=100 symmetricds-europe | Select-String -Pattern "ERROR|WARN"
```

### Reiniciar sincronización completa

```powershell
# Detener SymmetricDS
docker compose stop symmetricds-america symmetricds-europe

# Esperar 5 segundos
Start-Sleep -Seconds 5

# Iniciar SymmetricDS
docker compose start symmetricds-america
Start-Sleep -Seconds 10
docker compose start symmetricds-europe

# Verificar logs
docker compose logs --tail=50 symmetricds-america symmetricds-europe
```

### Limpiar batches antiguos

**Ejecutar purge manual en PostgreSQL**:
```powershell
docker exec -i postgres-america psql -U symmetricds -d globalshop -c "
DELETE FROM sym_outgoing_batch WHERE status = 'OK' AND create_time < NOW() - INTERVAL '7 days';"
```

---

## Comandos de Mantenimiento

### Backup de bases de datos

**PostgreSQL**:
```powershell
docker exec postgres-america pg_dump -U symmetricds globalshop > backup_postgres_$(Get-Date -Format 'yyyyMMdd_HHmmss').sql
```

**MySQL**:
```powershell
docker exec mysql-europe mysqldump -u symmetricds -psymmetricds globalshop > backup_mysql_$(Get-Date -Format 'yyyyMMdd_HHmmss').sql
```

### Verificar espacio en disco

```powershell
docker system df
```

### Limpiar recursos Docker no usados

```powershell
docker system prune -a --volumes
```

---

## Accesos Directos a Bases de Datos

### PostgreSQL Shell
```powershell
docker exec -it postgres-america psql -U symmetricds -d globalshop
```

### MySQL Shell
```powershell
docker exec -it mysql-europe mysql -u symmetricds -psymmetricds globalshop
```

---

## Variables de Entorno Útiles

```powershell
# Para scripts de prueba
$PG_CONN = "docker exec -i postgres-america psql -U symmetricds -d globalshop"
$MY_CONN = "docker exec -i mysql-europe mysql -u symmetricds -psymmetricds globalshop"

# Uso:
& $PG_CONN -c "SELECT COUNT(*) FROM products;"
& $MY_CONN -e "SELECT COUNT(*) FROM products;"
```

---

**Nota**: Todos los comandos asumen que estás en el directorio del proyecto:  
`c:\Users\wadri\Desktop\abdd-2025-2-main`
