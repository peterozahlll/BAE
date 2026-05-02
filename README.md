# Garantía de consistencia en Redis
Descripción

En Redis, la consistencia se garantiza mediante:

Transacciones con MULTI, EXEC, DISCARD
Manejo de errores limitado (no hay EXCEPTION, pero sí validaciones previas)
Automatización mediante scripts Lua (equivalente a triggers)
Operaciones atómicas
1. Transacciones básicas

En Redis, las transacciones se manejan así:

MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC

- Explicación
MULTI: inicia la transacción
DECRBY / INCRBY: operaciones en cola
EXEC: ejecuta todo de forma atómica

Si Redis se cae antes de EXEC, no se aplica nada.

2. Manejo de errores

Redis no tiene ROLLBACK automático, pero sí:

Validación previa
Cancelación manual con DISCARD
Ejemplo:
WATCH cuenta:1

GET cuenta:1
# Supongamos saldo = 50

MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC
Caso de error

Si el saldo es insuficiente, debes evitar ejecutar:

DISCARD
Alternativa (recomendada): Script Lua
EVAL "
local saldo = redis.call('GET', KEYS[1])
if tonumber(saldo) < tonumber(ARGV[1]) then
    return 'Error: saldo insuficiente'
end
redis.call('DECRBY', KEYS[1], ARGV[1])
redis.call('INCRBY', KEYS[2], ARGV[1])
return 'Transferencia exitosa'
" 2 cuenta:1 cuenta:2 100

Esto actúa como:
Validación
Transacción
Control de errores

3. Agrupación lógica de transacciones
Caso real: Transferencia bancaria
MULTI
DECRBY cuenta:1 200
INCRBY cuenta:2 200
EXEC
Caso más seguro con Lua:
EVAL "
if tonumber(redis.call('GET', KEYS[1])) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    redis.call('INCRBY', KEYS[2], ARGV[1])
    return 'OK'
else
    return 'Fondos insuficientes'
end
" 2 cuenta:1 cuenta:2 200
4. ⚙️ Automatización (equivalente a triggers)

Redis no tiene triggers nativos, pero puedes simularlos con:

• Opción 1: Keyspace Notifications

Activar:

CONFIG SET notify-keyspace-events KEA

Escuchar eventos:

PSUBSCRIBE __keyevent@0__:set

Esto detecta cuando se insertan o modifican datos.

• Opción 2: Script Lua (más potente)

Ejemplo tipo trigger en inserción:

EVAL "
redis.call('SET', KEYS[1], ARGV[1])
redis.call('LPUSH', 'log', 'Nuevo registro insertado')
return 'Insertado con auditoría'
" 1 cliente:1 "Juan"

5. Casos de prueba
• Caso 1: Todo correcto
SET cuenta:1 500
SET cuenta:2 100

MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC

Resultado:

cuenta:1 → 400
cuenta:2 → 200

Transacción exitosa

• Caso 2: Error lógico (sin validación)
SET cuenta:1 50

MULTI
DECRBY cuenta:1 100
INCRBY cuenta:2 100
EXEC

Resultado:

cuenta:1 → -50 

Redis NO evita inconsistencias por sí solo

• Caso 3: Error controlado con Lua
-- saldo insuficiente

Resultado:

"Error: saldo insuficiente"

No se ejecuta la operación

• Caso 4: Simulación de trigger
SET cliente:1 "Ana"

Resultado (evento capturado):

"Se modificó una clave"

- Conclusión técnica
Concepto SQL	Redis equivalente
BEGIN / COMMIT	MULTI / EXEC
ROLLBACK	DISCARD (limitado)
EXCEPTION	Scripts Lua + validaciones
TRIGGERS	Keyspace Notifications / Lua
Transacciones ACID	Parcial (atomicidad básica)

- Reflexión

Redis prioriza rendimiento y simplicidad, no integridad compleja como una base relacional.

- Para garantizar consistencia real:

Usa scripts Lua
Evita lógica crítica en múltiples comandos separados
Implementa validaciones antes de ejecutar operaciones
