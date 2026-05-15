# /git:commit-es — Analizar diff y sugerir mensajes de commit (Español)

Ejecutar `git diff --staged` (o `git diff HEAD` si no hay nada en stage),
analizar los cambios, y sugerir 3 opciones de mensaje de commit siguiendo
el formato Conventional Commits.

## Instrucciones

1. Ejecutar `git diff --staged` para obtener los cambios en stage.
2. Si el output está vacío, ejecutar `git diff HEAD` para los cambios sin stage.
3. Si ambos están vacíos, ejecutar `git status` e informar qué está pasando.
4. Leer el diff con cuidado — entender QUÉ cambió y POR QUÉ importa.
5. Generar exactamente 3 opciones de mensaje de commit ordenadas por especificidad.
6. Producir el análisis y las opciones en la estructura definida abajo.

## Estructura de output

### Qué cambió
2–3 oraciones describiendo el impacto real del diff — no solo los archivos tocados,
sino qué comportamiento, contrato o estructura cambió. Marcar cualquier cosa que
parezca un breaking change o un efecto secundario no intencional.

### Opciones de mensaje de commit

**Opción 1 — específica** (recomendada)
```
<tipo>(<alcance>): <descripción concisa del cambio real>

[cuerpo opcional: una línea explicando POR QUÉ si no es obvio del título]
```

**Opción 2 — alcance más amplio**
```
<tipo>(<alcance>): <descripción de nivel un poco más alto>
```

**Opción 3 — mínima**
```
<tipo>: <descripción corta sin alcance>
```

### Alertas (si las hay)
- ⚠️ Breaking change detectado → sugerir `!` después del tipo o footer `BREAKING CHANGE:`
- ⚠️ Múltiples responsabilidades en un diff → sugerir dividir en commits separados
- ⚠️ Código de debug encontrado → `console.log`, `System.out.println`, `logger.debug`

## Referencia de tipos Conventional Commits

| Tipo | Cuándo usarlo |
|------|--------------|
| `feat` | Nueva funcionalidad o capacidad |
| `fix` | Corrección de bug |
| `refactor` | Cambio de código sin cambio de comportamiento |
| `perf` | Mejora de performance |
| `test` | Agregar o corregir tests |
| `docs` | Solo documentación |
| `chore` | Build, dependencias, config (sin código de producción) |
| `style` | Formato, espacios en blanco (sin cambio de lógica) |
| `ci` | Cambios en pipelines de CI/CD |

## Ejemplos de buenos mensajes de commit

```
feat(orders): agrega lógica de reintento para pagos fallidos
fix(auth): evita el refresh de sesión en JWT vencido
refactor(users): extrae validación de email a utilidad compartida
perf(catalog): reemplaza N+1 queries con un JOIN en listado de productos
test(payments): agrega test de integración para prevenir cobros duplicados
chore(deps): actualiza Quarkus a 3.8.1
```

## Uso

Ejecutar desde la raíz del repo donde están los cambios en stage o sin stage.

`/git:commit-es` — sin argumentos, lee el diff automáticamente.
