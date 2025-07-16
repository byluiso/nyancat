# Análisis del Problema: Consulta SQL sqlGetExistePNT

## Consulta SQL Actual
```sql
SELECT *
FROM ttask t
WHERE Capture01 = CAST(@Recipe as nvarchar(max))
AND TaskDefinitionID = (select ID from tTaskDefinition where [key] = 'ASOCIACION.PNT')
AND (SELECT value FROM tTaskItem tti WHERE tti.TaskID=t.id and tti.TaskFormItemID = @TFICodigo) = @CodigoPNT
AND CompletedDateTime is null
```

## **PROBLEMA IDENTIFICADO** ❌

La consulta busca PNTs donde `Capture01 = @Recipe`, pero según el código analizado:

### En `ButtonCargarPlantillaPNTs_onClick` (Plantilla):
```csharp
_NewTaskData.Capture01 = rowPNT["proceso"].ToString();     // ❌ NO es Recipe ID
_NewTaskData.Capture02 = _recipe.ID.ToString();           // ✅ Recipe ID está aquí
_NewTaskData.Capture03 = rowPNT["general"].ToString();
```

### En `ButtonAsociarPNT_OnClick` (Individual):
```csharp
// Para iGeneral == 0 (General)
_NewTaskData.Capture01 = _recipe.ID.ToString();           // ✅ Recipe ID está aquí
_NewTaskData.Capture02 = _recipe.ID.ToString();
_NewTaskData.Capture03 = iGeneral.ToString();

// Para iGeneral == 1 (Procesos específicos)
_NewTaskData.Capture01 = id;                              // ❌ NO es Recipe ID
_NewTaskData.Capture02 = _recipe.ID.ToString();           // ✅ Recipe ID está aquí
_NewTaskData.Capture03 = iGeneral.ToString();
```

## **CAUSA DEL PROBLEMA**

La consulta SQL solo encuentra PNTs individuales asociados como "General" (`iGeneral == 0`), pero NO encuentra:
1. ❌ PNTs de plantilla (donde `Capture01 = proceso`)
2. ❌ PNTs individuales asociados a procesos específicos (donde `Capture01 = procesoID`)

## **SOLUCIÓN PROPUESTA**

### Opción 1: Consulta SQL Corregida (Recomendada)
```sql
SELECT *
FROM ttask t
WHERE (
    -- Buscar por Recipe ID en Capture01 (PNTs individuales generales)
    Capture01 = CAST(@Recipe as nvarchar(max))
    OR 
    -- Buscar por Recipe ID en Capture02 (PNTs de plantilla y procesos específicos)
    Capture02 = CAST(@Recipe as nvarchar(max))
)
AND TaskDefinitionID = (select ID from tTaskDefinition where [key] = 'ASOCIACION.PNT')
AND (SELECT value FROM tTaskItem tti WHERE tti.TaskID=t.id and tti.TaskFormItemID = @TFICodigo) = @CodigoPNT
AND CompletedDateTime is null
```

### Opción 2: Consulta SQL Más Específica
```sql
SELECT *
FROM ttask t
WHERE TaskDefinitionID = (select ID from tTaskDefinition where [key] = 'ASOCIACION.PNT')
AND (SELECT value FROM tTaskItem tti WHERE tti.TaskID=t.id and tti.TaskFormItemID = @TFICodigo) = @CodigoPNT
AND CompletedDateTime is null
AND (
    -- Caso 1: PNT individual general (Capture01 = Recipe, Capture02 = Recipe)
    (Capture01 = CAST(@Recipe as nvarchar(max)) AND Capture02 = CAST(@Recipe as nvarchar(max)))
    OR
    -- Caso 2: PNT de plantilla o proceso específico (Capture02 = Recipe, Capture01 ≠ Recipe)
    (Capture02 = CAST(@Recipe as nvarchar(max)) AND Capture01 != CAST(@Recipe as nvarchar(max)))
)
```

### Opción 3: Consulta SQL Simplificada (Más Robusta)
```sql
SELECT *
FROM ttask t
WHERE Capture02 = CAST(@Recipe as nvarchar(max))  -- Recipe ID siempre está en Capture02
AND TaskDefinitionID = (select ID from tTaskDefinition where [key] = 'ASOCIACION.PNT')
AND (SELECT value FROM tTaskItem tti WHERE tti.TaskID=t.id and tti.TaskFormItemID = @TFICodigo) = @CodigoPNT
AND CompletedDateTime is null
```

## **RECOMENDACIÓN DE IMPLEMENTACIÓN**

### Paso 1: Usar la Opción 3 (Más Simple y Robusta)
```csharp
// Reemplazar la consulta actual por:
string sqlGetExistePNT = @"
SELECT *
FROM ttask t
WHERE Capture02 = CAST({0} as nvarchar(max))
AND TaskDefinitionID = (select ID from tTaskDefinition where [key] = 'ASOCIACION.PNT')
AND (SELECT value FROM tTaskItem tti WHERE tti.TaskID=t.id and tti.TaskFormItemID = @TFICodigo) = '{1}'
AND CompletedDateTime is null";
```

### Paso 2: Agregar Logging para Verificar
```csharp
Ets.Api.Util.Log.WriteInformation($"Ejecutando sqlGetExistePNT con Recipe: {_recipe.ID}, PNT: {this.Ets.Values["PntCode"].ToString().Trim()}", "PNT_DEBUG");
```

### Paso 3: Testear la Corrección
1. Cargar una plantilla con PNTs
2. Intentar asociar un PNT individual con el mismo código
3. Verificar que ahora SÍ detecte el duplicado

## **IMPACTO DE LA SOLUCIÓN**

✅ **Detectará correctamente duplicados de:**
- PNTs de plantilla
- PNTs individuales generales
- PNTs individuales de procesos específicos

✅ **Mantendrá la funcionalidad existente**

✅ **Resolverá el problema reportado**

Esta corrección debería resolver completamente el problema de detección de PNTs duplicados entre plantillas y asociaciones individuales.