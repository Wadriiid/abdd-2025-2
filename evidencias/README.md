# 🔄 Documentación de Pruebas - Replicación Heterogénea

## Información del Proyecto

| Campo | Valor |
|-------|-------|
| **Estudiante** | Rolando Jair Delgado Parraga |
| **Cédula** | 1313463208 |
| **Fecha de Ejecución** | 29/01/2026 |
| **Asignatura** | Administración de Bases de Datos Distribuidas |
| **Período** | 2025-2 |

---

## 🎯 Objetivo del Examen

Implementar una arquitectura de **replicación lógica bidireccional** entre dos sistemas de bases de datos heterogéneos:

- **Nodo América**: PostgreSQL 15 (Base de datos origen/destino)
- **Nodo Europa**: MySQL 8.0 (Base de datos origen/destino)
- **Motor de Replicación**: SymmetricDS 3.16

La replicación debe ser **bidireccional**, es decir, los cambios realizados en cualquiera de las dos bases de datos deben propagarse automáticamente a la otra.

---

## 🏗️ Arquitectura Implementada

```
┌─────────────────────────────────────────────────────────────────┐
│                    DOCKER COMPOSE NETWORK                        │
│                    (globalshop-network)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐              ┌──────────────────┐         │
│  │  POSTGRES-AMERICA │              │   MYSQL-EUROPE   │         │
│  │    (PostgreSQL)   │              │     (MySQL)      │         │
│  │    Puerto: 5432   │              │   Puerto: 3306   │         │
│  │    BD: globalshop │              │   BD: globalshop │         │
│  └────────┬─────────┘              └────────┬─────────┘         │
│           │                                  │                   │
│           ▼                                  ▼                   │
│  ┌──────────────────┐              ┌──────────────────┐         │
│  │ SYMMETRICDS      │◄────────────►│ SYMMETRICDS      │         │
│  │ AMERICA          │  Replicación │ EUROPE           │         │
│  │ Puerto: 31415    │ Bidireccional│ Puerto: 31416    │         │
│  │ (Nodo Raíz)      │              │ (Nodo Cliente)   │         │
│  └──────────────────┘              └──────────────────┘         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🧪 Pruebas de Replicación Realizadas

### Prueba 1: INSERT PostgreSQL → MySQL

**Operación realizada en PostgreSQL (América):**
```sql
INSERT INTO products (product_id, product_name, category, base_price, description, is_active, created_at, updated_at)
VALUES ('EVIDENCIA-01', 'Producto de Prueba', 'Testing', 99.99, 'Creado para evidencia', true, NOW(), NOW());
```

**Verificación en MySQL (Europa):**
```sql
SELECT * FROM products WHERE product_id = 'EVIDENCIA-01';
```

**Resultado:** El registro aparece en MySQL automáticamente (~10 segundos).

---

### Prueba 2: INSERT MySQL → PostgreSQL

**Operación realizada en MySQL (Europa):**
```sql
INSERT INTO customers (customer_id, first_name, last_name, email, country, tier, created_at, updated_at)
VALUES ('EVIDENCIA-02', 'Cliente', 'Prueba', 'test@evidencia.com', 'España', 'Premium', NOW(), NOW());
```

**Verificación en PostgreSQL (América):**
```sql
SELECT * FROM customers WHERE customer_id = 'EVIDENCIA-02';
```

**Resultado:** El cliente aparece automáticamente en PostgreSQL, confirmando la replicación bidireccional.

---

### Prueba 3: UPDATE Bidireccional

**Actualización en PostgreSQL:**
```sql
UPDATE products SET base_price = 149.99, description = 'Precio actualizado' 
WHERE product_id = 'EVIDENCIA-01';
```

**Verificación en MySQL:**
```sql
SELECT product_id, base_price, description FROM products WHERE product_id = 'EVIDENCIA-01';
```

**Resultado:** Los cambios se propagan correctamente a MySQL.

---

### Prueba 4: DELETE Bidireccional

**Eliminación en MySQL:**
```sql
DELETE FROM customers WHERE customer_id = 'EVIDENCIA-02';
```

**Verificación en PostgreSQL:**
```sql
SELECT * FROM customers WHERE customer_id = 'EVIDENCIA-02';
-- Resultado: 0 filas
```

**Resultado:** La eliminación se replica correctamente a PostgreSQL.

---

## 🔧 Configuración Implementada

### Tablas Replicadas
| Tabla | Canal | Descripción |
|-------|-------|-------------|
| `products` | products_channel | Catálogo de productos |
| `inventory` | inventory_channel | Control de stock |
| `customers` | customers_channel | Base de clientes |
| `regional_pricing` | promotions_channel | Precios por región |

### Nodos Configurados
| Nodo | group.id | external.id | Base de Datos |
|------|----------|-------------|---------------|
| América | america-store | 001 | PostgreSQL 15 |
| Europa | europe-store | 002 | MySQL 8.0 |

### Enlaces de Replicación
```
america-store ──── W (Wait/Write) ────► europe-store
europe-store  ──── W (Wait/Write) ────► america-store
```

---

## ✅ Checklist de Verificación

- [x] Los 4 contenedores inician correctamente
- [x] PostgreSQL acepta conexiones en puerto 5432
- [x] MySQL acepta conexiones en puerto 3306
- [x] SymmetricDS América escucha en puerto 31415
- [x] SymmetricDS Europa escucha en puerto 31416
- [x] Tablas sym_* creadas en ambas bases de datos
- [x] Nodo Europa registrado en América
- [x] INSERT de PostgreSQL → MySQL funciona
- [x] INSERT de MySQL → PostgreSQL funciona
- [x] UPDATE bidireccional funciona
- [x] DELETE bidireccional funciona

---

## 📊 Métricas de Rendimiento

| Operación | Tiempo Promedio |
|-----------|-----------------|
| INSERT propagación | ~5-10 segundos |
| UPDATE propagación | ~5-10 segundos |
| DELETE propagación | ~5-10 segundos |
| Registro inicial de nodo | ~30-60 segundos |

*Nota: Los tiempos pueden variar según la carga del sistema y la configuración de jobs en SymmetricDS.*

---

## 🎓 Conclusiones

1. **Replicación Exitosa:** Se logró implementar correctamente la replicación bidireccional entre PostgreSQL y MySQL usando SymmetricDS como motor de sincronización.

2. **Heterogeneidad:** El sistema maneja automáticamente las diferencias entre los dos motores de base de datos (tipos de datos, sintaxis SQL, etc.).

3. **Consistencia:** Los datos se mantienen sincronizados en ambas direcciones, garantizando la consistencia eventual del sistema distribuido.

4. **Docker:** La arquitectura containerizada facilita el despliegue y la portabilidad de la solución.

5. **Escalabilidad:** La configuración implementada permite agregar más nodos en el futuro si fuera necesario.

---

## 📝 Notas Adicionales

- La configuración de SymmetricDS se realiza principalmente en el nodo raíz (América)
- El nodo cliente (Europa) hereda la configuración automáticamente al registrarse
- Los conflictos de replicación se resuelven con la política "newer_wins" (gana el más reciente)
- Se recomienda esperar al menos 2 minutos después de `docker-compose up` antes de hacer pruebas

---

*Documento generado como parte del Examen Práctico de ABDD 2025-2*
