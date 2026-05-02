# Garantía de consistencia en Redis
 
> Redis prioriza rendimiento y simplicidad sobre integridad compleja. Esta guía explica cómo garantizar consistencia mediante transacciones, scripts Lua y notificaciones de eventos.
 
---
 
## Tabla de contenidos
 
- [Descripción](#descripción)
- [1. Transacciones básicas](#1-transacciones-básicas)
- [2. Manejo de errores](#2-manejo-de-errores)
- [3. Agrupación lógica de transacciones](#3-agrupación-lógica-de-transacciones)
- [4. Automatización (equivalente a triggers)](#4-automatización-equivalente-a-triggers)
- [5. Casos de prueba](#5-casos-de-prueba)
- [Conclusión técnica](#conclusión-técnica)
---
 
## Descripción
 
En Redis, la consistencia se garantiza mediante:
 
- Transacciones con `MULTI`, `EXEC`, `DISCARD`
- Manejo de errores limitado (no hay `EXCEPTION`, pero sí validaciones previas)
- Automatización mediante scripts Lua (equivalente a triggers)
- Operaciones atómicas
---
 
## 1. Transacciones básicas
 
En Redis, las transacciones se manejan así:
 
```redis
MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC
```
 
### Explicación
 
| Comando | Descripción |
|--------|-------------|
| `MULTI` | Inicia la transacción |
| `DECRBY` / `INCRBY` | Operaciones en cola |
| `EXEC` | Ejecuta todo de forma atómica |
 
> **Nota:** Si Redis se cae antes de `EXEC`, no se aplica ninguna operación.
 
---
 
## 2. Manejo de errores
 
Redis no tiene `ROLLBACK` automático, pero ofrece:
 
- Validación previa con `WATCH`
- Cancelación manual con `DISCARD`
### Ejemplo con WATCH
 
```redis
WATCH cuenta:1
GET cuenta:1
# Supongamos saldo = 50
 
MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC
```
 
Si el saldo es insuficiente, cancela la transacción manualmente:
 
```redis
DISCARD
```
 
### Alternativa recomendada: Script Lua
 
Los scripts Lua se ejecutan de forma atómica en Redis y permiten combinar validación, transacción y control de errores en un solo bloque.
 
```lua
EVAL "
local saldo = redis.call('GET', KEYS[1])
if tonumber(saldo) < tonumber(ARGV[1]) then
    return 'Error: saldo insuficiente'
end
redis.call('DECRBY', KEYS[1], ARGV[1])
redis.call('INCRBY', KEYS[2], ARGV[1])
return 'Transferencia exitosa'
" 2 cuenta:1 cuenta:2 100
```
 
---
 
## 3. Agrupación lógica de transacciones
 
### Caso real: transferencia bancaria
 
```redis
MULTI
DECRBY cuenta:1 200
INCRBY cuenta:2 200
EXEC
```
 
### Caso más seguro con Lua
 
```lua
EVAL "
if tonumber(redis.call('GET', KEYS[1])) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    redis.call('INCRBY', KEYS[2], ARGV[1])
    return 'OK'
else
    return 'Fondos insuficientes'
end
" 2 cuenta:1 cuenta:2 200
```
 
---
 
## 4. Automatización (equivalente a triggers)
 
Redis no tiene triggers nativos, pero pueden simularse con dos enfoques.
 
### Opción 1: Keyspace Notifications
 
Activar notificaciones:
 
```redis
CONFIG SET notify-keyspace-events KEA
```
 
Suscribirse a eventos:
 
```redis
PSUBSCRIBE __keyevent@0__:set
```
 
> Detecta cuando se insertan o modifican claves en la base de datos.
 
### Opción 2: Script Lua
 
Permite ejecutar lógica adicional (como auditoría) junto con la operación principal:
 
```lua
EVAL "
redis.call('SET', KEYS[1], ARGV[1])
redis.call('LPUSH', 'log', 'Nuevo registro insertado')
return 'Insertado con auditoria'
" 1 cliente:1 "Juan"
```
 
---
 
## 5. Casos de prueba
 
### Caso 1: transacción exitosa
 
```redis
SET cuenta:1 500
SET cuenta:2 100
 
MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC
```
 
**Resultado esperado:**
 
```
cuenta:1 → 400
cuenta:2 → 200
```
 
---
 
### Caso 2: error lógico sin validación
 
```redis
SET cuenta:1 50
 
MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC
```
 
**Resultado:**
 
```
cuenta:1 → -50
```
 
> **Advertencia:** Redis no evita inconsistencias por si solo. El saldo queda en negativo.
 
---
 
### Caso 3: error controlado con Lua
 
Con saldo insuficiente, el script Lua retorna:
 
```
"Error: saldo insuficiente"
```
 
La operación no se ejecuta y los datos permanecen consistentes.
 
---
 
### Caso 4: simulación de trigger
 
```redis
SET cliente:1 "Ana"
```
 
Evento capturado por Keyspace Notifications:
 
```
"Se modificó una clave"
```
 
---
 
## Conclusión técnica
 
| Concepto SQL | Equivalente en Redis |
|---|---|
| `BEGIN` / `COMMIT` | `MULTI` / `EXEC` |
| `ROLLBACK` | `DISCARD` (limitado) |
| `EXCEPTION` | Scripts Lua + validaciones |
| `TRIGGERS` | Keyspace Notifications / Lua |
| Transacciones ACID | Parcial (atomicidad básica) |
 
### Recomendaciones
 
Para garantizar consistencia real en Redis:
 
1. Usa **scripts Lua** para operaciones que requieran validación y atomicidad
2. Evita lógica crítica distribuida en múltiples comandos separados
3. Implementa **validaciones previas** antes de ejecutar operaciones sensibles
4. Usa **Keyspace Notifications** para reaccionar a cambios de estado
 
