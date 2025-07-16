# Análisis de la Consulta SQL sqlGetExistePNT

## Información Proporcionada
```sql
DECLARE @Recipe int = 7759
```

## Información Necesaria para el Análisis

Para poder analizar correctamente por qué la consulta `sqlGetExistePNT` no detecta los PNTs duplicados cuando se carga una plantilla, necesito ver:

### 1. **Consulta SQL Completa**
- El SELECT statement completo
- Las tablas que se están consultando (JOINs)
- Las condiciones WHERE
- Los parámetros que se están usando

### 2. **Estructura de Datos**
- Cómo se almacenan los PNTs de plantilla en la base de datos
- Cómo se almacenan los PNTs individuales
- Qué campos se usan para identificar duplicados

### 3. **Contexto de Uso**
- Cómo se llama la consulta desde el código C#
- Qué parámetros se pasan (FormatWith)
- En qué momento se ejecuta

## Preguntas Específicas

1. **¿La consulta busca en los campos correctos?**
   - ¿Busca en TaskItems donde el Key sea el código del PNT?
   - ¿Considera el estado del PNT (CompletedDateTime)?

2. **¿Los parámetros son correctos?**
   - ¿El primer parámetro es el Recipe ID?
   - ¿El segundo parámetro es el código del PNT?

3. **¿La consulta considera ambos tipos de PNT?**
   - PNTs de plantilla (con estructura diferente en Capture01, Capture02, Capture03)
   - PNTs individuales

## Solicitud
Por favor, proporciona el código completo de la consulta SQL `sqlGetExistePNT` para poder hacer un análisis detallado y proponer la solución correcta al problema de detección de duplicados.