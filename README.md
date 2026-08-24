# minidb - Motor SQL Educativo

Motor SQL simple en Python para practicar el flujo básico de una consulta y la organización de metadatos.

## Cómo levantar el proyecto

1. Activar el entorno virtual:

```powershell
.\.venv\Scripts\Activate.ps1
```

2. Ejecutar el test básico:

```powershell
python .\test_minidb.py
```

3. Ejecutar el ejemplo de uso:

```powershell
python .\example.py
```

4. Abrir la REPL interactiva:

```powershell
python .\main.py repl
```

## Qué hace el proyecto

- procesa SQL con tokenizer, parser y validator,
- ejecuta `SELECT`, `INSERT`, `CREATE TABLE`, `CREATE INDEX` y `ANALYSE`,
- permite elegir optimizadores de join (`basic`, `selinger`, `bayes`),
- guarda metadatos en CSV,
- mantiene índices B-tree persistidos,
- permite probar todo desde consola.

## Documentación

La documentación técnica detallada está en `docs/DOCUMENTACION_ARCHIVOS.md`.
