# Garantía de consistencia en Redis
 
> Redis es una base de datos orientada a rendimiento, no a integridad compleja. En lugar de SQL, utiliza mecanismos propios para gestionar la consistencia de datos.
 
---
 
## Tabla de contenidos
 
- [Descripción](#descripción)
- [1. Transacciones básicas](#1-transacciones-básicas)
- [2. Control de errores](#2-control-de-errores)
- [3. Agrupación lógica de transacciones](#3-agrupación-lógica-de-transacciones)
- [4. Automatización (simulación de triggers)](#4-automatización-simulación-de-triggers)
- [5. Casos de prueba](#5-casos-de-prueba)
- [Conclusión](#conclusión)
---
 
## Descripción
 
En Redis, la consistencia de datos se gestiona mediante mecanismos distintos a los de bases de datos relacionales:
 
- Transacciones (`MULTI`, `EXEC`, `DISCARD`)
- Control de concurrencia (`WATCH`)
- Scripts Lua (para validación y atomicidad)
- Notificaciones de eventos (simulación de triggers)
---
 
## 1. Transacciones básicas
 
```redis
MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC
```
 
### Explicación técnica
 
Las transacciones en Redis permiten agrupar múltiples comandos para que se ejecuten de forma atómica.
 
| Comando | Descripción |
|---------|-------------|
| `MULTI` | Inicia la transacción y pone los comandos en cola |
| `DECRBY` / `INCRBY` | Operaciones que no se ejecutan inmediatamente |
| `EXEC` | Ejecuta todos los comandos juntos |
 
### Objetivo
 
Garantizar que todas las operaciones se ejecuten como un bloque único.
 
> **Limitación:** Redis no valida reglas de negocio (por ejemplo, evitar saldo negativo). Solo asegura atomicidad, no consistencia lógica.
 
---
 
## 2. Control de errores
 
### Con WATCH y DISCARD
 
```redis
WATCH cuenta:1
 
GET cuenta:1
# Validación previa en la aplicación
 
MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC
```
 
Si ocurre un problema, cancelar con:
 
```redis
DISCARD
```
 
### Recomendado: Script Lua
 
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
 
### Explicación técnica
 
Redis no dispone de `ROLLBACK` automático ni bloques `EXCEPTION`. Para manejar errores se utilizan:
 
| Mecanismo | Descripción |
|-----------|-------------|
| `WATCH` | Detecta cambios en claves (control de concurrencia) |
| `DISCARD` | Cancela la transacción antes de ejecutarse |
| Scripts Lua | Validan condiciones y ejecutan lógica completa de forma atómica |
 
### Objetivo
 
Evitar inconsistencias como saldo negativo u operaciones incompletas.
 
---
 
## 3. Agrupación lógica de transacciones
 
### Transferencia simple
 
```redis
MULTI
DECRBY cuenta:1 200
INCRBY cuenta:2 200
EXEC
```
 
### Transferencia segura con Lua
 
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
 
### Explicación técnica
 
Este apartado simula un caso real: una transferencia bancaria. La operación implica restar dinero de una cuenta y sumarlo a otra. Ambas deben ejecutarse juntas para mantener coherencia.
 
### Objetivo
 
Garantizar que operaciones relacionadas no queden a medias y mantengan consistencia.
 
> **Recomendacion:** Lua es la mejor opción porque valida y ejecuta todo en un solo paso atómico.
 
---
 
## 4. Automatización (simulación de triggers)
 
### Opción 1: Keyspace Notifications
 
Activar notificaciones:
 
```redis
CONFIG SET notify-keyspace-events KEA
```
 
Suscribirse a eventos:
 
```redis
PSUBSCRIBE __keyevent@0__:set
```
 
### Opción 2: Script Lua
 
```lua
EVAL "
redis.call('SET', KEYS[1], ARGV[1])
redis.call('LPUSH', 'log', 'Nuevo registro insertado')
return 'Insertado con auditoria'
" 1 cliente:1 "Juan"
```
 
### Explicación técnica
 
Redis no tiene triggers como SQL, pero se pueden simular mediante:
 
| Mecanismo | Descripción |
|-----------|-------------|
| Keyspace Notifications | Detectan eventos en claves |
| Scripts Lua | Ejecutan lógica adicional automáticamente |
 
### Objetivo
 
Automatizar tareas como auditoría, registro de eventos y validaciones automáticas.
 
---
 
## 5. Casos de prueba
 
### Caso 1: ejecución correcta
 
```redis
SET cuenta:1 500
SET cuenta:2 100
 
MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC
```
 
**Resultado:**
 
```
cuenta:1 → 400
cuenta:2 → 200
```
 
Transacción exitosa.
 
---
 
### Caso 2: error sin control
 
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
 
> **Advertencia:** Redis no evita inconsistencias por si solo.
 
---
 
### Caso 3: error controlado con Lua
 
Con saldo insuficiente, el script retorna:
 
```
"Error: saldo insuficiente"
```
 
La operación no se ejecuta y los datos permanecen consistentes.
 
---
 
### Caso 4: simulación de trigger
 
```redis
SET cliente:1 "Ana"
```
 
**Resultado esperado:**
 
```
"Nuevo registro insertado"
```
 
### Explicación técnica
 
Los casos de prueba permiten verificar el funcionamiento correcto, el comportamiento ante errores, la efectividad de las validaciones y la reacción de las automatizaciones.
 
### Objetivo
 
Demostrar cómo Redis maneja (o no) la consistencia, y por qué requiere lógica adicional para garantizar integridad.
 
---
 
## Conclusión
 
Redis prioriza rendimiento sobre integridad compleja. Para garantizar consistencia real, seguir estas buenas practicas:
 
1. Usar **scripts Lua** para operaciones críticas
2. **Validar datos** antes de ejecutar
3. Evitar dividir lógica en **múltiples comandos separados**
 
