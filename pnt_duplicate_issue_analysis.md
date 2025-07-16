# Análisis del Problema: Detección de PNTs Duplicados

## Problema Identificado
Cuando se carga una plantilla de PNTs y posteriormente se intenta asociar un PNT individual con el mismo código, la consulta SQL `sqlGetExistePNT` no detecta que ya existe un PNT con ese código en la plantilla cargada.

## Análisis del Código

### En `ButtonCargarPlantillaPNTs_onClick`:
```csharp
// Se crean nuevos PNTs desde la plantilla
_NewTaskData.Capture01 = rowPNT["proceso"].ToString();
_NewTaskData.Capture02 = _recipe.ID.ToString();
_NewTaskData.Capture03 = rowPNT["general"].ToString();
```

### En `ButtonAsociarPNT_OnClick`:
```csharp
// Validación de PNT existente
DataTable sPNT = api.Util.Db.GetDataTable(sqlGetExistePNT.FormatWith(_recipe.ID, this.Ets.Values["PntCode"].ToString().Trim())).Return;
if (sPNT != null)
{
  if (sPNT.Rows.Count > 0)
  {
    this.Ets.Form.AddValidationError(string.Format(ETS.Core.Scripting.Globales.Metodos.GetTranslation("pnt", "pntAsociado")));
    return ;
  }
}
```

## Posibles Causas del Problema

### 1. **Diferencias en los Parámetros de Captura**
- **Plantilla**: `Capture01 = proceso`, `Capture02 = _recipe.ID`, `Capture03 = general`
- **Individual**: `Capture01 = procesoID/recipeID`, `Capture02 = _recipe.ID`, `Capture03 = iGeneral`

### 2. **Consulta SQL Incorrecta**
La consulta `sqlGetExistePNT` probablemente está buscando en campos que no coinciden entre PNTs de plantilla y PNTs individuales.

### 3. **Estados de PNT**
Los PNTs de plantilla pueden tener un estado diferente que no es considerado en la validación.

## Soluciones Propuestas

### Solución 1: Verificar Consulta SQL
```sql
-- La consulta debería buscar en todos los campos relevantes
SELECT * FROM TaskData td
INNER JOIN TaskItems ti ON td.ID = ti.TaskDataID
INNER JOIN TaskFormItems tfi ON ti.TaskFormItemID = tfi.ID
WHERE tfi.Key = 'KeyCodigo' 
  AND ti.Value = @PntCode
  AND td.TaskDefinitionID = @TaskDefinitionID
  AND (td.CompletedDateTime IS NULL OR td.CompletedDateTime > @CurrentDate)
```

### Solución 2: Validación Mejorada en ButtonAsociarPNT_OnClick
```csharp
// Verificar tanto en la tabla dtPNT como en la base de datos
bool pntExisteEnTabla = false;
foreach (DataRow row in dtPNT.Rows)
{
    if (row["Codigo"].ToString().Trim() == this.Ets.Values["PntCode"].ToString().Trim() 
        && row["Estado"].ToString() != "Desasociado")
    {
        pntExisteEnTabla = true;
        break;
    }
}

if (pntExisteEnTabla)
{
    this.Ets.Form.AddValidationError(string.Format(ETS.Core.Scripting.Globales.Metodos.GetTranslation("pnt", "pntAsociado")));
    return;
}

// Continuar con la validación en base de datos
DataTable sPNT = api.Util.Db.GetDataTable(sqlGetExistePNT.FormatWith(_recipe.ID, this.Ets.Values["PntCode"].ToString().Trim())).Return;
```

### Solución 3: Unificar Estructura de Datos
```csharp
// En ButtonCargarPlantillaPNTs_onClick, usar la misma estructura que en asociación individual
_NewTaskData.Capture01 = rowPNT["proceso"].ToString();
_NewTaskData.Capture02 = _recipe.ID.ToString();
_NewTaskData.Capture03 = "0"; // Usar el mismo formato que iGeneral
```

### Solución 4: Consulta SQL Mejorada
```csharp
// Crear una consulta que busque por código de PNT independientemente del tipo
string sqlGetExistePNTMejorada = @"
SELECT td.ID, ti.Value as Codigo, 
       CASE WHEN td.CompletedDateTime IS NULL THEN 'Asociado' ELSE 'Desasociado' END as Estado
FROM TaskData td
INNER JOIN TaskItems ti ON td.ID = ti.TaskDataID
INNER JOIN TaskFormItems tfi ON ti.TaskFormItemID = tfi.ID
WHERE tfi.Key = '{0}' 
  AND ti.Value = '{1}'
  AND td.TaskDefinitionID = (SELECT ID FROM TaskDefinitions WHERE Key = '{2}')
  AND (td.CompletedDateTime IS NULL)";

DataTable sPNT = api.Util.Db.GetDataTable(
    sqlGetExistePNTMejorada.FormatWith(
        ConstantesPNT.KeyCodigo, 
        this.Ets.Values["PntCode"].ToString().Trim(),
        ConstantesPNT.KeyTaskDefPNT
    )
).Return;
```

## Recomendación de Implementación

### Paso 1: Verificar la Consulta SQL Actual
Revisar qué campos está consultando `sqlGetExistePNT` y asegurarse de que coincidan con los campos donde se almacenan los códigos de PNT tanto para plantillas como para asociaciones individuales.

### Paso 2: Implementar Validación Dual
```csharp
private bool ValidarPNTDuplicado(string pntCode, string recipeId)
{
    // Verificar en la tabla en memoria
    foreach (DataRow row in dtPNT.Rows)
    {
        if (row["Codigo"].ToString().Trim().Equals(pntCode.Trim(), StringComparison.OrdinalIgnoreCase) 
            && row["Estado"].ToString() != "Desasociado")
        {
            return true; // PNT ya existe
        }
    }
    
    // Verificar en la base de datos
    DataTable sPNT = api.Util.Db.GetDataTable(sqlGetExistePNT.FormatWith(recipeId, pntCode.Trim())).Return;
    return sPNT != null && sPNT.Rows.Count > 0;
}
```

### Paso 3: Usar la Validación en ButtonAsociarPNT_OnClick
```csharp
if (ValidarPNTDuplicado(this.Ets.Values["PntCode"].ToString(), _recipe.ID.ToString()))
{
    this.Ets.Form.AddValidationError(string.Format(ETS.Core.Scripting.Globales.Metodos.GetTranslation("pnt", "pntAsociado")));
    return;
}
```

## Debugging Recomendado

1. **Agregar logging** para ver qué valores se están comparando:
```csharp
Ets.Api.Util.Log.WriteInformation($"Buscando PNT: {this.Ets.Values["PntCode"].ToString().Trim()}", "PNT_DEBUG");
Ets.Api.Util.Log.WriteInformation($"En Recipe: {_recipe.ID}", "PNT_DEBUG");
```

2. **Verificar el contenido de dtPNT** después de cargar la plantilla
3. **Revisar la consulta SQL** ejecutada y sus resultados

Esta solución debería resolver el problema de detección de PNTs duplicados entre plantillas y asociaciones individuales.