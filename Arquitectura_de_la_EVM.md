# 🚀 Arquitectura de la EVM: Flujos de Datos y Componentes

*Documento completo - Octubre 2024*

## 📋 Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Diagrama de Flujos](#diagrama-de-flujos)
- [Componentes Principales](#componentes-principales)
- [Flujos de Datos](#flujos-de-datos)
- [Ejemplo Práctico](#ejemplo-práctico)
- [Opcodes y Gas](#opcodes-y-gas)
- [Flujo de Ejecución](#flujo-de-ejecución)
- [Mejores Prácticas](#mejores-prácticas)
- [Recursos](#recursos)
- [📎 ANEXO: Transaction Traces](#-anexo-transaction-traces-en-ethereum)

---

## Descripción General

Este documento explica detalladamente la arquitectura de la Ethereum Virtual Machine (EVM), mostrando cómo los datos fluyen entre los diferentes componentes durante la ejecución de un contrato inteligente.

**Enfoque principal:** Entender el **movimiento de datos** entre Stack, Memory, Storage y Calldata durante la ejecución.

---

## 🎯 Diagrama de Flujos de la EVM

```text
EJECUCIÓN EVM EN CONTEXTO - FLUJOS DE DATOS:
┌─────────────────┐    ┌─────────────────────────────────────────────────────┐
│   TRANSACCIÓN   │───▶│                   EVM                               │
│ • data: "Alice" │    │ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │
│ • value: 1 ETH  │    │ │   STACK     │ │   MEMORY    │ │  STORAGE    │    │
│ • gas: 21000    │    │ │ • LIFO      │ │ • Volátil   │ │ • Persist.  │    │
│ • to: 0x1234..  │    │ │ • 1024 elem │ │ • Expandible│ │ • key/value │    │
│ • from: 0xabcd  │    │ │ • 256 bits  │ │ • ~3 gas/wp │ │ • ~20k gas  │    │
└─────────────────┘    │ └─────────────┘ └─────────────┘ └─────────────┘    │
          │            │    │        │        │              │              │
          ├────────────┼────┼────────┼────────┼──────────────┼──────────────┤
          │            │    │        │        │              │              │
          ▼            │    ▼        ▼        ▼              ▼              │
┌─────────────────┐   │  ┌──────┐ ┌──────┐ ┌──────┐      ┌─────────────┐   │
│     BLOQUE      │──▶│  │CALL- │ │BLOCK │ │GAS   │      │  VARIABLES  │   │
│ • number: N     │   │  │DATA  │ │NUMBER│ │LEFT  │      │  DE ESTADO  │   │
│ • timestamp: T  │   │  │"Alice"│ │  N   │ │21000 │      │ • balances  │   │
│ • hash: ...     │   │  └──────┘ └──────┘ └──────┘      │ • owners    │   │
│ • coinbase: ..  │   │     │          │        │         │ • totals    │   │
│ • difficulty: . │   │     │          │        │         └─────────────┘   │
└─────────────────┘   │     ▼          ▼        ▼                 │         │
          │            │  ┌─────────────────────────────────────┐  │         │
          ▼            │  │            STACK                    │  │         │
┌─────────────────┐   │  │ ┌─────────────────────────────────┐ │  │         │
│    CUENTAS      │──▶│  │ │ • msg.sender (0xabcd)           │ │  │         │
│ • balances      │   │  │ │ • msg.value (1 ETH)             │ │  │         │
│ • nonces        │   │  │ │ • block.number (N)              │ │  │         │
│ • code hash     │   │  │ │ • tx.origin (0xabcd)            │ │  │         │
└─────────────────┘   │  │ │ • calldata[0:] ("Alice")        │ │  │         │
          │            │  │ │ • temp results                  │ │  │         │
          ▼            │  │ └─────────────────────────────────┘ │  │         │
┌─────────────────┐   │  │                                     │  │         │
│  CONTRATO ACT.  │──▶│  └─────────────────────────────────────┘  │         │
│ • bytecode      │   │          │                                │         │
│ • storage root  │   │          ▼                                │         │
└─────────────────┘   │  ┌─────────────┐                         │         │
                      │  │   MEMORY    │                         │         │
                      │  │ • Arrays    │←────────────────────────┘         │
                      │  │ • Strings   │                                   │
                      │  │ • Structs   │                                   │
                      │  │ • Temp data │                                   │
                      │  └─────────────┘                                   │
                      │          │                                         │
                      │          ▼                                         │
                      │  ┌─────────────┐    ┌──────────────────────────┐  │
                      │  │  OPCODES    │    │       BYTECODE           │  │
                      │  │ • PUSH      │───▶│ • Contract instructions  │  │
                      │  │ • MSTORE    │    │ • Deployed code          │  │
                      │  │ • SLOAD     │    └──────────────────────────┘  │
                      │  │ • SSTORE    │               │                  │
                      │  │ • CALL      │               ▼                  │
                      │  └─────────────┘    ┌─────────────┐              │
                      │                     │   RETURN    │              │
                      │                     │ • Output    │              │
                      │                     │ • 0x...     │              │
                      │                     └─────────────┘              │
                      └──────────────────────────────────────────────────┘
                             │
                             ▼
                       ┌─────────────┐
                       │   ESTADO    │
                       │  MUNDIAL    │
                       │ • Balances  │←─────────── value: 1 ETH
                       │ • Storage   │←─────────── variables persistidas
                       │ • Nonces    │←─────────── nonce incrementado
                       └─────────────┘
```

---

## 🏗️ Componentes Principales de la EVM

### Stack (Pila)

**Características:**
- **Tipo:** LIFO (Last In, First Out)
- **Capacidad:** 1024 elementos de 256 bits cada uno
- **Costo:** ~3 gas por operación básica
- **Uso:** Operaciones aritméticas y lógicas temporales

**Analogía:** Como una pila de platos, solo puedes agregar/quitar desde arriba.

**Ejemplo:**
```text
┌─────────────┐
│   0xABCD    │ ← Top (último en entrar, primero en salir)
├─────────────┤
│   0x1234    │
├─────────────┤
│   0x5678    │
└─────────────┘
```

---

### Memory (Memoria Volátil)

**Características:**
- **Tipo:** Array de bytes expandible
- **Duración:** Se limpia entre llamadas a funciones
- **Costo:** ~3 gas por palabra (32 bytes)
- **Uso:** Datos temporales durante ejecución

**Analogía:** Como la RAM de tu computadora, rápida pero temporal.

**Cuándo usar:**
- Arrays dinámicos que necesitas modificar
- Strings que vas a transformar
- Datos temporales complejos
- Parámetros para llamadas internas

---

### Storage (Almacenamiento Persistente)

**Características:**
- **Tipo:** Key-Value store (256 bits → 256 bits)
- **Duración:** Permanente en la blockchain
- **Costo:** ~20,000 gas para escribir (0 → non-zero)
- **Uso:** Estado del contrato (balances, mappings, etc.)

**Analogía:** Como el disco duro, permanente pero caro.

**Cuándo usar:**
- Variables de estado del contrato
- Mappings
- Datos que deben persistir entre transacciones
- Información crítica

---

### Calldata

**Características:**
- **Tipo:** Datos de entrada de la transacción (Read-only)
- **Acceso:** Solo lectura mediante puntero
- **Costo:** Más barato que memory (~16 gas por byte non-zero)
- **Uso:** Parámetros de funciones externas
- **⚠️ INMUTABLE:** No se puede modificar, solo leer o copiar

**Analogía:** Como un documento PDF, puedes leerlo pero no editarlo.

**Cuándo usar:**
- Parámetros de funciones `external`
- Cuando NO necesitas modificar los datos
- Para ahorrar gas en lecturas

**⚠️ Concepto clave:**
```solidity
function procesar(string calldata texto) external {
    // 'texto' está en CALLDATA (inmutable)
    
    // ✅ BIEN: Leer directamente
    if (bytes(texto).length > 0) { }
    
    // ✅ BIEN: Copiar a memory para modificar
    string memory textoCopia = texto; // Copia a MEMORY
    // Ahora puedo modificar 'textoCopia'
    
    // ❌ IMPOSIBLE: No puedo modificar 'texto' directamente
    // texto[0] = 'X'; // ← ERROR: calldata es read-only
}
```

---

## 📊 Flujos de Datos - ¿Qué va dónde?

### Hacia el STACK (Operaciones Temporales)

```text
Transacción/cuenta → msg.sender (0xabcd)
Transacción        → msg.value (1 ETH)
Bloque             → block.number (N)
Transacción        → tx.origin (0xabcd)
Transacción/CALLDATA → calldata[0:] ("Alice")
Cálculos           → temp results (resultados intermedios)
```

**Regla:** Variables simples y cálculos temporales van al stack.

---

### Hacia MEMORY (Datos Temporales Grandes)

```text
Arrays/Strings/Structs → MEMORY (cuando necesitas modificar una COPIA)
Calldata complejo      → MEMORY (COPIAS para procesar)
Funciones internas     → MEMORY (parámetros entre funciones)
Construcción de datos  → MEMORY (buffers temporales)
```

**Regla:** Datos complejos que necesitas modificar temporalmente van a memory.

**⚠️ IMPORTANTE:** Cuando copias desde CALLDATA a MEMORY:
- El dato en CALLDATA permanece **inmutable**
- Solo la **copia** en MEMORY puede ser modificada
- CALLDATA nunca se modifica directamente

---

### Hacia STORAGE (Persistente)

```text
Actualizaciones de saldos → balances[user]
Asociaciones clave-valor  → mappings
Datos de entidades        → structs completos
Variables de contrato     → state variables
```

**Regla:** Solo datos que DEBEN persistir entre transacciones van a storage.

---

### Transformaciones entre Componentes

```text
STACK → MEMORY     (cuando guardas resultados temporales)
MEMORY → STORAGE   (cuando persistes datos procesados)
CALLDATA → MEMORY  (cuando COPIAS inputs para modificar la copia)
STORAGE → STACK    (cuando lees estado para cálculos)
CALLDATA → STACK   (lectura directa de parámetros simples)
```

**Diagrama de flujo corregido:**
```text
┌──────────┐
│ CALLDATA │ (read-only, INMUTABLE)
└────┬─────┘
     │
     ├──▶ STACK ──────────────────┐
     │    (lectura directa)        │
     │                             ▼
     │                         Cálculos
     │                             │
     └──▶ MEMORY ──────────────────┤
          (COPIA para modificar)   │
          │                        │
          ├──▶ STACK ──────────────┤
          │    (resultados)        │
          │                        ▼
          └──▶ STORAGE ◀────── STACK
               (persistir)     (escritura)

IMPORTANTE: CALLDATA nunca se modifica.
            Solo se LEE o se COPIA a MEMORY.
```

**Ejemplos de cada flujo:**

```solidity
// Flujo 1: CALLDATA → STACK (lectura directa)
function ejemplo1(uint256 valor) external {
    uint256 x = valor;  // CALLDATA → STACK
    // 'valor' se lee directamente, no se copia
}

// Flujo 2: CALLDATA → MEMORY (copia para modificar)
function ejemplo2(string calldata texto) external {
    string memory textoModificado = string(bytes(texto)); // COPIA
    // Ahora 'textoModificado' está en MEMORY y puedo cambiarlo
    // 'texto' original en CALLDATA sigue intacto
}

// Flujo 3: MEMORY → STORAGE
function ejemplo3(string calldata texto) external {
    string memory temp = string(bytes(texto)); // CALLDATA → MEMORY
    miVariable = temp;  // MEMORY → STORAGE
}

// Flujo 4: STORAGE → STACK
function ejemplo4() external view returns (uint256) {
    return contador;  // STORAGE → STACK
}
```

---

## 💻 Ejemplo Práctico en Solidity

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract EjemploFlujos {
    // ═══════════════════════════════════════════════════════
    // → STORAGE: Variables persistentes (CARAS)
    // ═══════════════════════════════════════════════════════
    uint256 public total;
    mapping(address => uint256) public balances;
    
    event Procesado(string nombre, uint256 cantidad);
    
    // ═══════════════════════════════════════════════════════
    // FUNCIÓN PRINCIPAL - Muestra todos los flujos
    // ═══════════════════════════════════════════════════════
    function procesar(string calldata nombre, uint256 cantidad) external payable {
        // ─────────────────────────────────────────────────
        // → STACK: Parámetros y contexto
        // ─────────────────────────────────────────────────
        // - msg.sender, msg.value, block.timestamp
        // - nombre (como referencia calldata), cantidad
        
        require(cantidad > 0, "Cantidad inválida");
        
        // ─────────────────────────────────────────────────
        // CALLDATA → MEMORY: Copiamos para modificar
        // ─────────────────────────────────────────────────
        string memory nombreUpper = toUpper(nombre);
        
        // ─────────────────────────────────────────────────
        // STACK → STORAGE: Persistimos datos (CARO)
        // ─────────────────────────────────────────────────
        balances[msg.sender] += cantidad;  // ← ~20k gas si nuevo
        total += cantidad;                 // ← ~5k gas (ya existe)
        
        // ─────────────────────────────────────────────────
        // MEMORY → EVENT: Emitimos log
        // ─────────────────────────────────────────────────
        emit Procesado(nombreUpper, cantidad);
    }
    
    // ═══════════════════════════════════════════════════════
    // FUNCIÓN VIEW - Solo lectura (BARATA)
    // ═══════════════════════════════════════════════════════
    function consultarBalance(address usuario) external view returns (uint256) {
        // ─────────────────────────────────────────────────
        // STORAGE → STACK: Leemos estado persistente
        // ─────────────────────────────────────────────────
        return balances[usuario];  // ← ~200 gas (SLOAD)
    }
    
    // ═══════════════════════════════════════════════════════
    // FUNCIÓN INTERNA - Procesamiento de datos
    // ═══════════════════════════════════════════════════════
    function toUpper(string calldata str) internal pure returns (string memory) {
        // ─────────────────────────────────────────────────
        // CALLDATA → MEMORY: COPIAMOS para poder modificar
        // IMPORTANTE: 'str' en CALLDATA permanece intacto
        //             Solo modificamos la COPIA en MEMORY
        // ─────────────────────────────────────────────────
        bytes memory strBytes = bytes(str);  // ← Copia a MEMORY
        
        // ─────────────────────────────────────────────────
        // MEMORY: Modificamos cada carácter de LA COPIA
        // ─────────────────────────────────────────────────
        for (uint256 i = 0; i < strBytes.length; i++) {
            if (strBytes[i] >= 0x61 && strBytes[i] <= 0x7A) { // a-z
                strBytes[i] = bytes1(uint8(strBytes[i]) - 32); // A-Z
            }
        }
        
        return string(strBytes);  // ← Retorna la copia modificada desde MEMORY
        // 'str' original en CALLDATA nunca cambió
    }
    
    // ═══════════════════════════════════════════════════════
    // EJEMPLO: Optimización con calldata
    // ═══════════════════════════════════════════════════════
    function procesarOptimizado(
        string calldata nombre,  // ← CALLDATA (barato)
        uint256 cantidad
    ) external payable {
        // Solo copiamos a memory si REALMENTE lo necesitamos
        if (cantidad > 1000) {
            // Solo en este caso copiamos
            string memory nombreProcesado = toUpper(nombre);
            emit Procesado(nombreProcesado, cantidad);
        } else {
            // Usamos directamente desde calldata
            emit Procesado(nombre, cantidad);
        }
    }
}
```

### 📈 Análisis de Costos del Ejemplo

```text
┌─────────────────────────────┬────────────┬─────────────────┐
│ Operación                   │ Gas (aprox)│ Componente      │
├─────────────────────────────┼────────────┼─────────────────┤
│ Leer msg.sender             │ 2          │ STACK           │
│ Leer calldata               │ 4-16/byte  │ CALLDATA→STACK  │
│ Copiar string a memory      │ ~100+      │ CALLDATA→MEMORY │
│ Modificar en memory         │ ~3/palabra │ MEMORY          │
│ Escribir nuevo en storage   │ ~20,000    │ STACK→STORAGE   │
│ Actualizar storage          │ ~5,000     │ STACK→STORAGE   │
│ Leer storage (SLOAD)        │ ~200       │ STORAGE→STACK   │
│ Emitir event                │ ~1,000+    │ MEMORY→LOG      │
└─────────────────────────────┴────────────┴─────────────────┘

Total función procesar(): ~26,000 - 30,000 gas
Total función consultarBalance(): ~250 gas (solo lectura)
```

---

## ⚡ Opcodes Principales y sus Flujos

### Stack Operations

```text
PUSH1-32  →  STACK              ← Añade valores al stack
POP       →  STACK              ← Remueve valores del stack
DUP1-16   →  STACK              ← Duplica valores en el stack
SWAP1-16  →  STACK              ← Intercambia posiciones en stack
```

**Costo:** 3 gas cada uno

---

### Memory Operations

```text
MLOAD     →  MEMORY → STACK     ← Lee de memoria al stack
MSTORE    →  STACK → MEMORY     ← Escribe del stack a memoria
MSTORE8   →  STACK → MEMORY     ← Escribe 1 byte del stack a memoria
```

**Costo:** 3 gas + expansión de memoria si crece

---

### Storage Operations

```text
SLOAD     →  STORAGE → STACK    ← Lee de storage al stack (200 gas)
SSTORE    →  STACK → STORAGE    ← Escribe del stack a storage (20k gas)
```

**Costos detallados:**
- SLOAD: 200 gas (cold) / 100 gas (warm)
- SSTORE (0 → non-zero): 20,000 gas
- SSTORE (non-zero → non-zero): 5,000 gas
- SSTORE (non-zero → 0): 5,000 gas + 15,000 refund

---

### Calldata Operations

```text
CALLDATALOAD    →  CALLDATA → STACK    ← Lee calldata al stack
CALLDATACOPY    →  CALLDATA → MEMORY   ← Copia calldata a memoria
CALLDATASIZE    →  STACK               ← Tamaño de calldata
```

**Costo:** 3-4 gas por palabra

---

### Tabla Resumen de Flujos

```text
┌──────────────┬───────────┬────────────┬──────────┬─────────────┐
│ Opcode       │ De        │ Hacia      │ Gas      │ Uso común   │
├──────────────┼───────────┼────────────┼──────────┼─────────────┤
│ PUSH         │ Bytecode  │ STACK      │ 3        │ Constantes  │
│ MLOAD        │ MEMORY    │ STACK      │ 3+       │ Leer arrays │
│ MSTORE       │ STACK     │ MEMORY     │ 3+       │ Temp data   │
│ SLOAD        │ STORAGE   │ STACK      │ 200      │ Leer estado │
│ SSTORE       │ STACK     │ STORAGE    │ 5k-20k   │ Guardar     │
│ CALLDATALOAD │ CALLDATA  │ STACK      │ 3        │ Leer params │
│ CALLDATACOPY │ CALLDATA  │ MEMORY     │ 3+       │ Copiar data │
│ ADD          │ STACK     │ STACK      │ 3        │ Aritmética  │
│ CALL         │ Multi     │ Multi      │ Variable │ Ext call    │
│ RETURN       │ MEMORY    │ Output     │ 0        │ Resultado   │
└──────────────┴───────────┴────────────┴──────────┴─────────────┘
```

---

## 🔄 Flujo de Ejecución Completo

### Fase 1: Inicialización

1. **Transacción llega al nodo**
   - Se recibe y valida formato
   - Se verifica firma digital (ECDSA)

2. **Verificación de suficiencia de gas**
   - Gas limit suficiente?
   - Balance suficiente para (gas × gas price)?

3. **Se cargan datos en memoria del nodo**
   - Calldata
   - Context (sender, value, block info)

---

### Fase 2: De Transacción a Bytecode

**Este es el paso que faltaba explicar:**

#### 2.1. Anatomía de una Transacción

```javascript
{
  from: "0xUser",
  to: "0xContract",
  value: "0",
  data: "0xa9059cbb000000000000000000000000recipient00000000000000000000000000000064",
  gas: 100000,
  gasPrice: "50000000000"
}
```

**Desglose del campo `data` (calldata):**
```text
0xa9059cbb  →  Function selector (primeros 4 bytes)
               = keccak256("transfer(address,uint256)")[0:4]

0x000...recipient  →  Parámetro 1: address (32 bytes)
0x000...0064      →  Parámetro 2: uint256 = 100 (32 bytes)
```

#### 2.2. El Contrato Destino tiene Bytecode

Cuando desplegaste el contrato, su bytecode quedó almacenado:

```text
┌─────────────────────────────────────┐
│  Cuenta del Contrato (0xContract)  │
├─────────────────────────────────────┤
│ balance: 10 ETH                     │
│ nonce: 1                            │
│ code: 0x608060405234801561001057... │ ← Bytecode
│ storage: {...}                      │
└─────────────────────────────────────┘
```

**El bytecode contiene:**
```text
• Function dispatcher (lee selector, salta a función correcta)
• Código de cada función (como opcodes)
• Constructor (solo en deployment)
• Metadata
```

#### 2.3. Cómo se Conectan

```text
TRANSACCIÓN                       CONTRATO
────────────                      ─────────
• from: 0xUser                    
• to: 0xContract     ─────────▶   • code: bytecode
• data: 0xa9059cbb...             
  ↓                                  ↓
  Function selector              [Bytecode decodifica]
  = "transfer(...)"              
                                    ↓
                                 [Salta a función transfer]
                                    ↓
                                 [Ejecuta opcodes de transfer]
```

---

### Fase 3: De Bytecode a Opcodes

#### 3.1. Estructura del Bytecode

**Bytecode real de ejemplo:**
```text
608060405234801561001057600080fd5b506004361061004c5760003560e01c8063...

Desglosado:
60 80  →  PUSH1 0x80     (opcode 0x60, valor 0x80)
60 40  →  PUSH1 0x40     (opcode 0x60, valor 0x40)
52     →  MSTORE         (opcode 0x52)
34     →  CALLVALUE      (opcode 0x34)
80     →  DUP1           (opcode 0x80)
...
```

#### 3.2. Program Counter (PC)

La EVM usa un **Program Counter** que apunta al byte actual:

```text
Bytecode: [60][80][60][40][52][34][80][15]...
           ↑
          PC = 0

Ciclo de ejecución:
1. PC = 0  →  Lee 0x60 (PUSH1)  →  Lee siguiente byte (0x80) → PUSH 0x80 al stack
2. PC = 2  →  Lee 0x60 (PUSH1)  →  Lee siguiente byte (0x40) → PUSH 0x40 al stack
3. PC = 4  →  Lee 0x52 (MSTORE) →  Ejecuta MSTORE
4. PC = 5  →  Lee 0x34 (CALLVALUE) → Ejecuta CALLVALUE
...
```

#### 3.3. Function Dispatcher

**Todo contrato Solidity empieza con un dispatcher:**

```text
Bytecode al inicio:
─────────────────────────────────────────────────────
1. CALLDATA → Load function selector (primeros 4 bytes)
2. Comparar selector con cada función:
   
   ¿selector == 0xa9059cbb (transfer)?     → JUMP a offset 0x0234
   ¿selector == 0x70a08231 (balanceOf)?    → JUMP a offset 0x0456
   ¿selector == 0x095ea7b3 (approve)?      → JUMP a offset 0x0678
   ...
   
3. Si no coincide → REVERT
```

**Ejemplo en opcodes:**

```text
PUSH1 0x00          ; Push 0 al stack
CALLDATALOAD        ; Carga primeros 32 bytes de calldata
PUSH1 0xe0          ; Push 224 (para shift)
SHR                 ; Shift right para obtener solo 4 bytes
DUP1                ; Duplica selector
PUSH4 0xa9059cbb    ; Push selector de transfer()
EQ                  ; ¿Son iguales?
PUSH2 0x0234        ; Push offset de transfer
JUMPI               ; Si iguales, salta a 0x0234
...                 ; Continua verificando otros selectores
```

#### 3.4. Ejecución de la Función

Una vez que salta a la función correcta, ejecuta sus opcodes:

**Ejemplo: función `transfer(address to, uint256 amount)`**

```solidity
function transfer(address to, uint256 amount) external returns (bool) {
    balances[msg.sender] -= amount;
    balances[to] += amount;
    return true;
}
```

**Se compila a (simplificado):**

```text
; Cargar balances[msg.sender]
CALLER              ; msg.sender → stack
PUSH1 0x00          ; slot 0 (balances mapping)
SHA3                ; keccak256(sender, 0) = storage key
SLOAD               ; Cargar balance actual → stack

; Restar amount
PUSH1 0x24          ; Offset de 'amount' en calldata
CALLDATALOAD        ; Cargar amount → stack
SWAP1               ; Intercambiar orden
SUB                 ; balance - amount → stack

; Guardar nuevo balance[msg.sender]
DUP1                ; Duplicar resultado
CALLER
PUSH1 0x00
SHA3                ; storage key
SSTORE              ; Guardar nuevo balance

; Cargar balances[to]
PUSH1 0x04          ; Offset de 'to' en calldata
CALLDATALOAD        ; Cargar to → stack
PUSH1 0x00
SHA3                ; keccak256(to, 0)
SLOAD               ; Cargar balance actual

; Sumar amount
PUSH1 0x24
CALLDATALOAD        ; Cargar amount
ADD                 ; balance + amount

; Guardar nuevo balances[to]
PUSH1 0x04
CALLDATALOAD
PUSH1 0x00
SHA3
SSTORE              ; Guardar nuevo balance

; Return true
PUSH1 0x01
PUSH1 0x00
MSTORE
PUSH1 0x20
PUSH1 0x00
RETURN              ; Retornar 32 bytes (true)
```

---

### Fase 4: Diagrama Completo del Flujo

```text
┌────────────────────────────────────────────────────────────────┐
│                    FLUJO COMPLETO                              │
└────────────────────────────────────────────────────────────────┘

1. TRANSACCIÓN LLEGA
   ┌─────────────────────┐
   │ TX: {               │
   │  to: 0xContract,    │
   │  data: 0xa9059cbb...│  ← Function selector + params
   │  gas: 100000        │
   │ }                   │
   └──────────┬──────────┘
              │
              ▼
2. EVM CARGA BYTECODE DEL CONTRATO
   ┌─────────────────────┐
   │ Contract.code       │
   │ = 0x608060405234... │  ← Bytecode almacenado
   └──────────┬──────────┘
              │
              ▼
3. PC = 0 (Program Counter inicia en 0)
   ┌─────────────────────┐
   │ Bytecode[PC]        │
   │ = 0x60 (PUSH1)      │  ← Lee primer byte
   └──────────┬──────────┘
              │
              ▼
4. EJECUTA OPCODES UNO POR UNO
   ┌─────────────────────┐
   │ PUSH1 0x80          │ → Stack: [0x80]
   │ PUSH1 0x40          │ → Stack: [0x40, 0x80]
   │ MSTORE              │ → Memory[0x40] = 0x80
   │ CALLVALUE           │ → Stack: [msg.value]
   │ ...                 │
   │ CALLDATALOAD        │ → Lee selector (0xa9059cbb)
   │ ...                 │
   │ JUMPI 0x0234        │ → Salta a función transfer
   └──────────┬──────────┘
              │
              ▼
5. EJECUTA FUNCIÓN (más opcodes)
   ┌─────────────────────┐
   │ CALLER              │ → Stack: [msg.sender]
   │ SLOAD               │ → Lee balance[sender] de storage
   │ CALLDATALOAD        │ → Lee amount de calldata
   │ SUB                 │ → balance - amount
   │ SSTORE              │ → Guarda nuevo balance
   │ ...                 │
   │ RETURN              │ → Retorna resultado
   └──────────┬──────────┘
              │
              ▼
6. ESTADO ACTUALIZADO
   ┌─────────────────────┐
   │ Storage modificado: │
   │ balances[sender]↓   │
   │ balances[to]↑       │
   └─────────────────────┘
```

---

### 🔍 Visualización Detallada

#### Antes de ejecutar:
```text
┌──────────────────────┐
│  Transaction         │
├──────────────────────┤
│ data: 0xa9059cbb...  │ ← "transfer(0xTo, 100)"
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Contract            │
├──────────────────────┤
│ code: [bytecode]     │ ← Secuencia de bytes
│ storage: {           │
│   slot0: {           │
│     sender: 500,     │
│     to: 200          │
│   }                  │
│ }                    │
└──────────────────────┘
```

#### Durante ejecución:
```text
┌──────────────────────┐
│  EVM                 │
├──────────────────────┤
│ PC: 0x0234           │ ← Apunta a código de transfer()
│ Stack: [100, 0xTo..]│ ← Datos temporales
│ Memory: [...]        │
│ Gas: 95000           │ ← Consumiendo gas
└──────────────────────┘
         │
         ▼ (leyendo bytecode byte a byte)
┌──────────────────────┐
│ Bytecode[0x0234]:    │
│ 0x33 → CALLER        │ ← Ejecutando este opcode
│ 0x60 → PUSH1         │
│ 0x00 → ...           │
│ 0x20 → SHA3          │
│ 0x54 → SLOAD         │
│ ...                  │
└──────────────────────┘
```

#### Después de ejecutar:
```text
┌──────────────────────┐
│  Contract            │
├──────────────────────┤
│ storage: {           │
│   slot0: {           │
│     sender: 400, ← -100
│     to: 300      ← +100
│   }                  │
│ }                    │
└──────────────────────┘
```

---

### 💡 Puntos Clave

**1. Transacción NO es opcodes**
```text
Transacción = Instrucción de alto nivel
"Llamar función transfer con estos parámetros"

Bytecode = Instrucciones de bajo nivel
PUSH, SLOAD, ADD, SSTORE, etc.
```

**2. El puente: Function Selector**
```text
data: 0xa9059cbb...
      └─┬──┘
        │
Function selector (4 bytes)
= primeros 4 bytes de keccak256("transfer(address,uint256)")

El dispatcher lee esto y sabe dónde saltar en el bytecode
```

**3. Ejecución secuencial**
```text
PC (Program Counter) avanza byte a byte:
PC=0 → Lee opcode → Ejecuta → PC++ → Lee siguiente opcode → ...
```

**4. De Solidity a Opcodes**
```solidity
balances[msg.sender] -= amount;

    ↓ (Compilador)

CALLER           // msg.sender
PUSH1 0x00       // slot de mapping
SHA3             // calcular storage key
SLOAD            // cargar balance
PUSH1 0x24       // offset de amount
CALLDATALOAD     // cargar amount
SUB              // restar
... SHA3
SSTORE           // guardar
```

---

### 🛠️ Herramientas para Ver Esta Transformación

#### 1. Ver bytecode de tu contrato

```bash
# Con Hardhat
npx hardhat compile
# Bytecode está en artifacts/

# Ver en Etherscan
# Ve a contrato → "Contract" tab → "Code" → "Bytecode"
```

#### 2. Decodificar calldata

```javascript
const ethers = require('ethers');

const iface = new ethers.Interface([
  "function transfer(address to, uint256 amount)"
]);

const calldata = "0xa9059cbb000000000000000000000000recipient00000000000000000000000000000064";

const decoded = iface.parseTransaction({ data: calldata });
console.log(decoded.name);      // "transfer"
console.log(decoded.args);      // [recipient, 100]
```

#### 3. Ver opcodes ejecutándose

**Remix Debugger:**
```text
1. Despliega contrato en Remix
2. Ejecuta función
3. Click en "Debug"
4. Ve opcodes paso a paso con PC
```

**Tenderly:**
```text
1. Pega hash de transacción
2. Ve opcodes + Stack + Memory + Storage
3. Paso a paso con visualización
```

---

### Fase 5: Ejecución (continuación del flujo original)

**Ahora que entendemos cómo llegamos a los opcodes:**

```text
┌──────────────────────────────────────────┐
│  LOOP DE EJECUCIÓN                       │
├──────────────────────────────────────────┤
│  1. Leer opcode en Bytecode[PC]          │
│  2. Verificar gas suficiente             │
│  3. Ejecutar operación:                  │
│     • Leer de STACK/MEMORY/STORAGE       │
│     • Realizar cálculo                   │
│     • Escribir a STACK/MEMORY/STORAGE    │
│  4. Consumir gas                         │
│  5. Incrementar PC                       │
│  6. ¿RETURN o REVERT? → Salir            │
│  7. ¿Out of gas? → REVERT                │
│  8. Repetir desde paso 1                 │
└──────────────────────────────────────────┘
```

**Diagrama de flujo:**
```text
Inicio
  │
  ▼
[Leer Bytecode[PC]] ──▶ [¿Gas suficiente?] ──NO──▶ [REVERT]
  │                           │
  │                          SÍ
  │                           │
  ▼                           ▼
[Decodificar Opcode] ──▶ [Ejecutar Operación]
  │                           │
  │                           ▼
  │                    [Actualizar Stack/Memory/Storage]
  │                           │
  │                           ▼
  │                    [PC += opcode_length]
  │                           │
  │                           ▼
  │                    [¿RETURN/REVERT?] ──SÍ──▶ [Fin]
  │                           │
  │                          NO
  │                           │
  └───────────────────────────┘
```

---

### Fase 4: Finalización

**Si gas se agota:**
```text
• Revertir TODOS los cambios en storage
• Memory y stack se descartan
• Consumir todo el gas
• Retornar error
```

**Si ejecución exitosa:**
```text
• Actualizar estado mundial:
  - Balances modificados
  - Storage actualizado
  - Nonces incrementados
• Emitir logs (events)
• Retornar datos (si los hay)
• Consumir gas usado
• Refund de gas no usado
```

---

## 🛠️ Mejores Prácticas

### ✅ Para Ahorrar Gas

#### 1. Usar `calldata` en lugar de `memory` para parámetros externos

```solidity
// ✅ BIEN: Usar calldata para parámetros de solo lectura
function procesar(string calldata nombre) external {
    // nombre se lee directamente desde calldata
    // Ahorro: ~100+ gas por cada lectura
}

// ❌ MAL: Usar memory innecesariamente  
function procesar(string memory nombre) external {
    // Se copia todo el string a memory (más caro)
    // Costo adicional: ~100+ gas
}
```

**Ahorro aproximado:** 100-500 gas dependiendo del tamaño

---

#### 2. Usar `unchecked` para cálculos seguros

```solidity
// ✅ BIEN: Si sabes que no habrá overflow
function sumar(uint256 a, uint256 b) internal pure returns (uint256) {
    unchecked {
        return a + b;  // Ahorra ~20 gas
    }
}

// ❌ MAL: Checks innecesarios en Solidity 0.8+
function sumar(uint256 a, uint256 b) internal pure returns (uint256) {
    return a + b;  // +20 gas por verificaciones
}
```

**Ahorro:** ~20 gas por operación

---

#### 3. Agrupar variables en storage

```solidity
// ✅ BIEN: Variables agrupadas en un slot
contract Optimizado {
    uint128 a;  // ┐
    uint128 b;  // ├─ 1 slot (32 bytes)
                // ┘
}

// ❌ MAL: Cada variable en un slot diferente
contract NoOptimizado {
    uint128 a;  // Slot 1
    uint256 x;  // Slot 2 (rompe el empaquetado)
    uint128 b;  // Slot 3
}
```

**Ahorro:** ~20,000 gas por slot ahorrado en escritura

---

### 💡 Para Manejo Eficiente de Datos

#### Reglas de oro:

```solidity
// ✅ Usar STACK para cálculos temporales
uint256 resultado = a + b;  // → Stack (barato)

// ✅ Usar MEMORY para datos complejos temporales
string memory temp = "hola"; // → Memory (medio)

// ✅ Usar STORAGE solo para datos persistentes
uint256 public contador;     // → Storage (caro)

// ✅ Usar CALLDATA para leer sin modificar
function procesar(string calldata data) external {
    // Leer directamente desde calldata
}
```

---

#### Tabla de decisión:

```text
┌──────────────────────┬─────────────┬────────────────────┐
│ Necesidad            │ Usar        │ Por qué            │
├──────────────────────┼─────────────┼────────────────────┤
│ Cálculo simple       │ STACK       │ Más barato         │
│ Variable temporal    │ MEMORY      │ Se borra después   │
│ Modificar array      │ MEMORY      │ Necesitas mutar    │
│ Leer sin modificar   │ CALLDATA    │ Más barato que mem │
│ Persistir entre txs  │ STORAGE     │ Solo opción        │
│ Pasar a otra función │ MEMORY      │ Necesitas copiar   │
└──────────────────────┴─────────────┴────────────────────┘
```

---

### 🎯 Patrones Comunes

#### Patrón 1: Leer y modificar storage

```solidity
// ✅ BIEN: Leer una vez, modificar en memory
function actualizarDatos(uint256 newValue) external {
    uint256 valor = miVariable;  // STORAGE → STACK (200 gas)
    valor = valor * 2 + newValue; // STACK (6 gas)
    miVariable = valor;          // STACK → STORAGE (5k gas)
}

// ❌ MAL: Múltiples accesos a storage
function actualizarDatos(uint256 newValue) external {
    miVariable = miVariable * 2;     // SLOAD + SSTORE
    miVariable = miVariable + newValue; // SLOAD + SSTORE (2x costo)
}
```

---

#### Patrón 2: Trabajar con arrays

```solidity
// ✅ BIEN: Copiar a memory, modificar, escribir una vez
function procesarArray(uint256[] storage arr) internal {
    uint256[] memory temp = arr;  // STORAGE → MEMORY
    
    for (uint i = 0; i < temp.length; i++) {
        temp[i] *= 2;  // Modificar en MEMORY (barato)
    }
    
    // Escribir de vuelta si es necesario
    for (uint i = 0; i < temp.length; i++) {
        arr[i] = temp[i];  // MEMORY → STORAGE
    }
}

// ❌ MAL: Modificar directamente en storage
function procesarArray(uint256[] storage arr) internal {
    for (uint i = 0; i < arr.length; i++) {
        arr[i] *= 2;  // SLOAD + SSTORE cada vez (muy caro)
    }
}
```

---

#### Patrón 3: Optimizar bucles

```solidity
// ✅ BIEN: Cachear variable de storage
function procesarLote() external {
    uint256 total_ = total;  // STORAGE → STACK (1 SLOAD)
    
    for (uint i = 0; i < 10; i++) {
        total_ += calcular(i);  // STACK (barato)
    }
    
    total = total_;  // STACK → STORAGE (1 SSTORE)
}

// ❌ MAL: Acceder a storage en cada iteración
function procesarLote() external {
    for (uint i = 0; i < 10; i++) {
        total += calcular(i);  // SLOAD + SSTORE × 10
    }
}
```

**Ahorro:** ~1,950 gas (10 SLOAD ahorrados)

---

## 📚 Recursos

### Documentación Oficial

- **Ethereum Yellow Paper:** https://ethereum.github.io/yellowpaper/paper.pdf
  - Especificación técnica completa de la EVM

- **Solidity Docs:** https://docs.soliditylang.org/
  - Documentación del lenguaje más usado

- **EVM Opcodes:** https://www.evm.codes/
  - Referencia interactiva de todos los opcodes

### Herramientas de Análisis

- **Gas Reporter (Hardhat):** 
  ```bash
  npm install --save-dev hardhat-gas-reporter
  ```
  Analiza costos de gas de tus contratos

- **Remix IDE:** https://remix.ethereum.org
  - Debugger visual con stack/memory/storage

- **Tenderly:** https://tenderly.co
  - Debugging avanzado con visualización de estado

### Tutoriales y Guías

- **"EVM Deep Dives"** - noxx.substack.com
- **"Understanding the EVM"** - ethdocs.org
- **"Solidity Gas Optimization"** - Alchemy University

### Optimización de Gas

- **"Solidity Gas Optimization Tips"** - GitHub gist
- **"EVM Storage Layout"** - Solidity docs
- **"Gas Costs Cheatsheet"** - Notion/Obsidian templates

---

## 📝 Resumen Ejecutivo

### Puntos Clave

1. **Stack** = Cálculos rápidos y temporales (3 gas)
2. **Memory** = Datos temporales complejos (~3 gas/palabra)
3. **Storage** = Persistencia permanente (20k gas escritura)
4. **Calldata** = Lectura eficiente de inputs (4-16 gas/byte)

### Reglas de Optimización

```text
✅ Usa calldata para parámetros externos
✅ Cachea variables de storage en stack/memory
✅ Agrupa variables para ahorrar slots
✅ Evita escrituras innecesarias en storage
✅ Usa unchecked cuando sea seguro
```

### Flujo Típico

```text
CALLDATA → STACK → Cálculos → MEMORY → Procesamiento → STORAGE
   ↓                                                        ↓
Inputs                                               Estado Final
```

---

**¡Este documento es tu referencia completa sobre flujos de datos en la EVM!** 🚀

---

## 📎 ANEXO: Transaction Traces en Ethereum

### 📖 ¿Qué son los Transaction Traces?

Un **transaction trace** es un registro detallado de **cada operación** que se ejecutó durante una transacción. Es como una "grabación" completa de todo lo que hizo la EVM.

**Analogía:** Como el historial de cambios en un documento de Google Docs
- Cada edición queda registrada
- Puedes ver quién hizo qué y cuándo
- Puedes "reproducir" toda la secuencia

---

### 🎯 ¿Para qué sirven los traces?

```text
✅ Debugging de contratos
✅ Análisis de gas consumption
✅ Auditoría de seguridad
✅ Entender flujos complejos (DeFi)
✅ Detectar comportamientos inesperados
✅ Optimización de contratos
```

---

### 🔍 Anatomía de un Trace

#### Estructura básica:

```json
{
  "type": "CALL",
  "from": "0xsender",
  "to": "0xcontract",
  "gas": 100000,
  "gasUsed": 21000,
  "input": "0x...",
  "output": "0x...",
  "value": "1000000000000000000",
  "calls": [
    {
      "type": "DELEGATECALL",
      "from": "0xcontract",
      "to": "0xlogic",
      // ... subcalls
    }
  ]
}
```

#### Elementos clave:

| Campo | Descripción | Ejemplo |
|-------|-------------|---------|
| `type` | Tipo de llamada | CALL, DELEGATECALL, STATICCALL, CREATE |
| `from` | Quien hace la llamada | 0xabcd... |
| `to` | Contrato destino | 0x1234... |
| `gas` | Gas disponible | 100000 |
| `gasUsed` | Gas consumido | 21000 |
| `input` | Calldata enviada | 0xa9059cbb... |
| `output` | Datos retornados | 0x0000...0001 |
| `value` | ETH transferido | 1 ETH (en wei) |
| `calls` | Sub-llamadas | Array de traces |
| `error` | Si hubo error | "out of gas", "revert" |

---

### 📊 Tipos de Traces

#### 1. **CALL** - Llamada normal

```text
┌─────────────┐         CALL          ┌─────────────┐
│   EOA/      │ ────────────────────▶ │  Contrato   │
│  Contrato A │                        │      B      │
└─────────────┘ ◀──────────────────── └─────────────┘
                    return data
                    
• Nuevo contexto de ejecución
• msg.sender cambia a caller
• Puede transferir ETH
```

**Ejemplo práctico:**
```solidity
// Contrato A llama a Contrato B
ContractB.transfer(recipient, amount);
```

**Trace resultante:**
```json
{
  "type": "CALL",
  "from": "0xContractA",
  "to": "0xContractB",
  "input": "0xa9059cbb...", // transfer(address,uint256)
  "gas": 50000,
  "gasUsed": 25000
}
```

---

#### 2. **DELEGATECALL** - Llamada con contexto preservado

```text
┌─────────────┐     DELEGATECALL     ┌─────────────┐
│  Contrato   │ ───────────────────▶ │   Logic     │
│   (Proxy)   │                       │  Contract   │
└─────────────┘ ◀─────────────────── └─────────────┘
                     
• Ejecuta código de Logic en contexto de Proxy
• msg.sender NO cambia
• Storage de Proxy se modifica
• Usado en proxies/upgradeable contracts
```

**Ejemplo práctico:**
```solidity
// Proxy usa lógica de implementación
(bool success, ) = implementation.delegatecall(msg.data);
```

**Trace resultante:**
```json
{
  "type": "DELEGATECALL",
  "from": "0xProxy",
  "to": "0xImplementation",
  "input": "0x...",
  "gas": 100000
}
```

**⚠️ Importante:** En el trace, `from` es el Proxy pero el código ejecutado es del Implementation.

---

#### 3. **STATICCALL** - Llamada de solo lectura

```text
┌─────────────┐     STATICCALL      ┌─────────────┐
│  Contrato   │ ──────────────────▶ │  Contrato   │
│      A      │                      │      B      │
└─────────────┘ ◀────────────────── └─────────────┘
                    read-only
                    
• NO puede modificar estado
• NO puede transferir ETH
• Si intenta escribir → revierte
• Usado en view/pure functions
```

**Ejemplo práctico:**
```solidity
// Leer balance de otro contrato
uint256 balance = token.balanceOf(user);
```

**Trace resultante:**
```json
{
  "type": "STATICCALL",
  "from": "0xContractA",
  "to": "0xToken",
  "input": "0x70a08231...", // balanceOf(address)
  "output": "0x00...0064"   // 100 tokens
}
```

---

#### 4. **CREATE** - Despliegue de contrato

```text
┌─────────────┐         CREATE        ┌─────────────┐
│  Deployer   │ ────────────────────▶ │    NEW      │
│             │                        │  Contract   │
└─────────────┘ ◀──────────────────── └─────────────┘
                   contract address
                   
• Crea un nuevo contrato
• Retorna la dirección del nuevo contrato
• Consume mucho gas
```

**Ejemplo práctico:**
```solidity
// Factory crea nuevo token
Token newToken = new Token("MyToken", "MTK");
```

**Trace resultante:**
```json
{
  "type": "CREATE",
  "from": "0xFactory",
  "to": "0x0000...0000",
  "input": "0x608060...", // bytecode
  "output": "0xNewContract",
  "gasUsed": 500000
}
```

---

#### 5. **CREATE2** - Despliegue determinístico

```text
Similar a CREATE pero:
• Dirección predecible
• Usa salt para determinismo
• address = hash(deployer, salt, bytecode)
```

---

#### 6. **SELFDESTRUCT** - Destrucción de contrato

```text
┌─────────────┐    SELFDESTRUCT     ┌─────────────┐
│  Contrato   │ ──────────────────▶ │  Recipient  │
│  (a morir)  │    transfer ETH     │             │
└─────────────┘                     └─────────────┘
                                    
• Destruye el contrato
• Envía ETH restante a recipient
• Código se mantiene hasta fin de tx
```

---

### 🛠️ Herramientas para Ver Traces

#### 1. **Etherscan**

```text
Pasos:
1. Ve a etherscan.io
2. Busca tu transacción
3. Click en "More Details"
4. Click en "State" o "Trace"
```

**Ventajas:**
- ✅ Interfaz visual
- ✅ Expande/colapsa calls
- ✅ Muestra valor transferido
- ✅ Links a contratos

**Desventajas:**
- ❌ Solo mainnet/testnets públicas
- ❌ No muestra opcodes detallados

---

#### 2. **Tenderly**

```text
Website: tenderly.co

Características:
• Debugger visual potente
• Stack/Memory/Storage en cada paso
• Gas profiler
• Simulación de transacciones
• Alertas y monitoring
```

**Ejemplo de uso:**
```bash
1. Ve a tenderly.co
2. Pega el hash de tu transacción
3. Click "Debugger"
4. Navega step-by-step
```

**Vista de Tenderly:**
```text
┌────────────────────────────────────────────────┐
│ Transaction Debugger                           │
├────────────────────────────────────────────────┤
│ Opcodes    │ Stack    │ Memory    │ Storage   │
├────────────┼──────────┼───────────┼───────────┤
│ > PUSH1    │ 0x00     │           │           │
│   ADD      │ 0x01     │           │           │
│   SSTORE   │ 0x05     │           │ slot0: 5  │
└────────────┴──────────┴───────────┴───────────┘
```

---

#### 3. **Remix Debugger**

```text
IDE: remix.ethereum.org

Uso:
1. Despliega contrato en Remix
2. Ejecuta transacción
3. Click en "Debug" en la consola
4. Step through ejecución
```

**Ventajas:**
- ✅ Integrado con desarrollo
- ✅ Gratis
- ✅ Stack/Memory/Storage visible
- ✅ Breakpoints

**Desventajas:**
- ❌ Solo para contratos en Remix
- ❌ No sirve para mainnet (salvo via fork)

---

#### 4. **Hardhat + tenderly**

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  networks: {
    hardhat: {
      forking: {
        url: "https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY"
      }
    }
  }
};

// Test con trace
it("should trace transaction", async function() {
  const tx = await contract.transfer(recipient, amount);
  const trace = await network.provider.send("debug_traceTransaction", [
    tx.hash
  ]);
  console.log(trace);
});
```

---

#### 5. **Curl + Node RPC**

```bash
# Obtener trace de una transacción
curl -X POST \
  -H "Content-Type: application/json" \
  --data '{
    "jsonrpc": "2.0",
    "method": "debug_traceTransaction",
    "params": ["0xtx_hash", {"tracer": "callTracer"}],
    "id": 1
  }' \
  https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
```

**Tracers disponibles:**
- `callTracer` - Muestra llamadas (más legible)
- `prestateTracer` - Estado antes/después
- `4byteTracer` - Muestra function signatures
- `opcodeTracer` - Todos los opcodes (muy detallado)

---

### 📈 Ejemplo Completo: Trace de Swap en Uniswap

#### Transacción original:
```text
Usuario swapea 1 ETH por USDC en Uniswap V2
```

#### Trace simplificado:

```json
{
  "type": "CALL",
  "from": "0xUser",
  "to": "0xUniswapRouter",
  "value": "1000000000000000000",  // 1 ETH
  "input": "0x7ff36ab5...",  // swapExactETHForTokens
  "gasUsed": 150000,
  "calls": [
    {
      "type": "CALL",
      "from": "0xUniswapRouter",
      "to": "0xWETH",
      "value": "1000000000000000000",
      "input": "0xd0e30db0",  // deposit()
      "gasUsed": 25000,
      "calls": []
    },
    {
      "type": "CALL",
      "from": "0xUniswapRouter",
      "to": "0xWETH",
      "input": "0x23b872dd...",  // transferFrom(router, pair, 1 ETH)
      "gasUsed": 35000,
      "calls": []
    },
    {
      "type": "CALL",
      "from": "0xUniswapRouter",
      "to": "0xUniswapPair",
      "input": "0x022c0d9f...",  // swap(0, amount, user, 0x)
      "gasUsed": 70000,
      "calls": [
        {
          "type": "CALL",
          "from": "0xUniswapPair",
          "to": "0xUSDC",
          "input": "0xa9059cbb...",  // transfer(user, amountOut)
          "gasUsed": 25000,
          "calls": []
        }
      ]
    }
  ]
}
```

#### Diagrama de flujo:

```text
Usuario
  │
  ├─▶ CALL UniswapRouter.swapExactETHForTokens(1 ETH)
       │
       ├─▶ CALL WETH.deposit() payable 1 ETH
       │    └─ Convierte ETH → WETH
       │
       ├─▶ CALL WETH.transferFrom(router, pair, 1 WETH)
       │    └─ Mueve WETH a pair
       │
       └─▶ CALL UniswapPair.swap(amountOut)
            │
            └─▶ CALL USDC.transfer(user, amountOut)
                 └─ Usuario recibe USDC
```

---

### 🔎 Interpretando Traces: Casos Comunes

#### Caso 1: Transaction Reverted

```json
{
  "type": "CALL",
  "from": "0xUser",
  "to": "0xContract",
  "error": "revert",
  "gasUsed": 50000,
  "output": "0x08c379a0..."  // Mensaje de error encoded
}
```

**¿Cómo leerlo?**
```javascript
// Decodificar el error
const errorData = "0x08c379a0...";
const decoded = ethers.utils.defaultAbiCoder.decode(
  ['string'],
  '0x' + errorData.slice(10)
);
console.log(decoded[0]); // "Insufficient balance"
```

**En Tenderly:**
```text
❌ REVERT at opcode 234
   Reason: "Insufficient balance"
   Gas used: 50,000 / 100,000
```

---

#### Caso 2: Out of Gas

```json
{
  "type": "CALL",
  "from": "0xUser",
  "to": "0xContract",
  "error": "out of gas",
  "gasUsed": 100000,  // = gas enviado
  "gas": 100000
}
```

**Señales:**
- `gasUsed` = `gas` (consumió todo)
- `error: "out of gas"`
- Transacción revirtió

**Solución:**
```javascript
// Aumentar gas limit
await contract.functionName({
  gasLimit: 200000  // ← Aumentar
});
```

---

#### Caso 3: Proxy Pattern

```json
{
  "type": "CALL",
  "from": "0xUser",
  "to": "0xProxy",
  "input": "0xa9059cbb...",
  "calls": [
    {
      "type": "DELEGATECALL",
      "from": "0xProxy",
      "to": "0xImplementation",
      "input": "0xa9059cbb...",  // mismo input
      "calls": []
    }
  ]
}
```

**Interpretación:**
1. Usuario llama al Proxy
2. Proxy hace DELEGATECALL a Implementation
3. Código de Implementation se ejecuta en contexto de Proxy
4. Storage del Proxy se modifica

---

#### Caso 4: Reentrancy Attack

```json
{
  "type": "CALL",
  "from": "0xUser",
  "to": "0xVulnerable",
  "input": "0x...", // withdraw()
  "calls": [
    {
      "type": "CALL",
      "from": "0xVulnerable",
      "to": "0xAttacker",
      "value": "1000000000000000000",
      "calls": [
        {
          "type": "CALL",
          "from": "0xAttacker",
          "to": "0xVulnerable",
          "input": "0x...", // withdraw() OTRA VEZ!
          "calls": [
            {
              "type": "CALL",
              "from": "0xVulnerable",
              "to": "0xAttacker",
              "value": "1000000000000000000",
              // ...ciclo continúa
            }
          ]
        }
      ]
    }
  ]
}
```

**🚨 Señal de reentrancy:**
- Llamadas recursivas al mismo contrato
- Antes de actualizar balances
- Patrón circular en el trace

---

### 📊 Análisis de Gas con Traces

#### Gas Profiling

```json
{
  "type": "CALL",
  "from": "0xUser",
  "to": "0xContract",
  "gas": 200000,
  "gasUsed": 150000,
  "calls": [
    {
      "to": "0xToken1",
      "gasUsed": 50000,  // 33% del total
    },
    {
      "to": "0xToken2",
      "gasUsed": 80000,  // 53% del total
    },
    {
      "to": "0xOracle",
      "gasUsed": 20000,  // 13% del total
    }
  ]
}
```

**Análisis:**
```text
┌─────────────────┬──────────┬──────────┐
│ Llamada         │ Gas      │ % Total  │
├─────────────────┼──────────┼──────────┤
│ Token1.transfer │ 50,000   │ 33%      │
│ Token2.transfer │ 80,000   │ 53% ← 🎯 │
│ Oracle.getPrice │ 20,000   │ 13%      │
└─────────────────┴──────────┴──────────┘

Optimización: Enfocarse en Token2
```

---

### 🧰 Tips Prácticos para Debugging

#### 1. Empezar con callTracer

```bash
# Más legible que opcode tracer
curl -X POST ... --data '{
  "params": ["0xtx_hash", {"tracer": "callTracer"}]
}'
```

#### 2. Buscar patrones de error

```javascript
function analyzeTrace(trace) {
  // Verificar si hay reverts
  if (trace.error) {
    console.log("❌ Error:", trace.error);
  }
  
  // Verificar gas usado vs disponible
  const gasEfficiency = trace.gasUsed / trace.gas;
  if (gasEfficiency > 0.95) {
    console.log("⚠️ Casi sin gas!");
  }
  
  // Buscar llamadas recursivas (reentrancy?)
  const addresses = [];
  function checkRecursion(t) {
    if (addresses.includes(t.to)) {
      console.log("🚨 Posible reentrancy:", t.to);
    }
    addresses.push(t.to);
    t.calls?.forEach(checkRecursion);
  }
  checkRecursion(trace);
}
```

#### 3. Comparar traces

```javascript
// Comparar transacción exitosa vs fallida
const successTrace = await getTrace("0xsuccess");
const failTrace = await getTrace("0xfail");

// ¿Dónde divergen?
function findDivergence(t1, t2, path = "") {
  if (t1.error !== t2.error) {
    console.log(`Divergencia en ${path}:`, {
      success: t1.error,
      fail: t2.error
    });
  }
  // ... comparar más campos
}
```

#### 4. Verificar storage changes

```bash
# Usar prestateTracer para ver cambios de storage
curl -X POST ... --data '{
  "params": ["0xtx_hash", {"tracer": "prestateTracer"}]
}'
```

**Output:**
```json
{
  "pre": {
    "0xContract": {
      "balance": "1000000000000000000",
      "storage": {
        "0x00": "0x00000064"  // 100
      }
    }
  },
  "post": {
    "0xContract": {
      "balance": "2000000000000000000",
      "storage": {
        "0x00": "0x000000c8"  // 200 ← cambió!
      }
    }
  }
}
```

---

### 📝 Checklist de Debugging con Traces

```text
□ Obtener trace de la transacción
□ Verificar tipo de llamadas (CALL, DELEGATECALL, etc.)
□ Revisar si hay errores ("revert", "out of gas")
□ Analizar consumo de gas por llamada
□ Buscar patrones sospechosos (recursión, loops)
□ Verificar valores transferidos (ETH)
□ Revisar inputs/outputs de cada call
□ Comparar con trace de transacción exitosa
□ Verificar storage changes (prestateTracer)
□ Documentar hallazgos
```

---

### 🎓 Ejercicio Práctico

#### Paso 1: Elige una transacción

```text
Ve a Etherscan y busca una transacción DeFi:
Ejemplo: Swap en Uniswap, Borrow en Aave, etc.
```

#### Paso 2: Obtén el trace

```text
1. En Etherscan, busca la tx
2. Ve a "More Details" → "Trace"
3. O usa Tenderly para mejor visualización
```

#### Paso 3: Analiza

```text
Responde:
□ ¿Cuántas llamadas internas hubo?
□ ¿Cuál consumió más gas?
□ ¿Hay DELEGATECALLS? (indica proxies)
□ ¿Se transfirió ETH? ¿Cuánto?
□ ¿Qué contratos interactuaron?
```

#### Paso 4: Reconstruye el flujo

```text
Dibuja un diagrama como:

Usuario
 └─▶ RouterA
      ├─▶ Token1.transfer()
      ├─▶ Pair.swap()
      └─▶ Token2.transfer()
```

---

### 🔗 Recursos Adicionales

**Herramientas:**
- Etherscan: https://etherscan.io
- Tenderly: https://tenderly.co
- Remix: https://remix.ethereum.org
- Alchemy Debug API: https://docs.alchemy.com/reference/debug-api

**Documentación:**
- Geth debug_traceTransaction: https://geth.ethereum.org/docs/interacting-with-geth/rpc/ns-debug
- Ethereum JSON-RPC: https://ethereum.org/en/developers/docs/apis/json-rpc/

**Tutoriales:**
- "Understanding Transaction Traces" - Alchemy University
- "Debugging Smart Contracts" - Hardhat docs
- "Transaction Analysis" - Tenderly blog

---

### 💡 Resumen del Anexo

**Transaction Traces te permiten:**
- ✅ Ver CADA operación en una transacción
- ✅ Debuggear errores y reverts
- ✅ Analizar consumo de gas
- ✅ Detectar vulnerabilidades
- ✅ Entender flujos complejos (DeFi)
- ✅ Auditar contratos

**Herramientas principales:**
- Etherscan (básico, visual)
- Tenderly (avanzado, debugger)
- Remix (desarrollo)
- RPC APIs (programático)

**Tipos de traces:**
- CALL, DELEGATECALL, STATICCALL
- CREATE, CREATE2
- SELFDESTRUCT

**Usa traces cuando:**
- Tu transacción revierte
- Quieres optimizar gas
- Auditas contratos
- Aprendes DeFi
- Investigas hacks

---

*Fin del Anexo - Transaction Traces*

*Última actualización: Octubre 2024*
