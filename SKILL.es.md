name: cross-generator
version: 1.2.0
description: Generador de análisis impulsado por IA que transforma datos sin procesar en información estructurada, informes y visualizaciones
author: SMOUJBOT
tags:
  - analysis
  - ai
  - automation
  - reporting
  - data-processing
maintainer: ops@openclaw.ai
last_updated: 2026-03-02
category: generators
dependencies:
  - openclaw-core >= 2.1.0
  - analysis-engine >= 1.4.0
  - report-renderer >= 2.0.0
  - python >= 3.9
  - pandas >= 1.5.0
  - matplotlib >= 3.6.0
  - numpy >= 1.24.0
required_env_vars:
  - CROSS_GEN_API_KEY (optional, for enhanced AI models)
  - ANALYSIS_OUTPUT_DIR (default: ./analysis-output)
  - CROSS_GEN_LOG_LEVEL (default: INFO)
compatible_platforms:
  - linux
  - darwin
  - windows
conflicts_with:
  - legacy-report-generator
  - manual-analyzer
```

# Habilidad Cross Generator

## Propósito

Cross Generator es un motor de análisis impulsado por IA que automatiza la conversión de datos sin procesar en información procesable, informes completos y visualizaciones. Elimina el trabajo manual de datos al detectar patrones inteligentemente, generar resúmenes estadísticos y producir resultados listos para publicación en múltiples formatos.

**Casos de Uso Reales:**
- Convertir registros de métricas del sistema (CPU, memoria, red) en informes de tendencias de rendimiento con detección de anomalías
- Transformar conjuntos de datos CSV/JSON en resúmenes ejecutivos con métricas clave y recomendaciones
- Generar informes de auditoría de seguridad a partir de registros de firewall y patrones de acceso
- Crear informes de análisis de dependencias a partir de cambios en package.json/requirements.txt
- Automatizar informes de estado operativo semanales a partir de datos de monitorización con información predictiva

## Alcance

Cross Generator opera dentro de un espacio de trabajo confinado y produce artefactos de análisis sin modificar los datos de origen. Los comandos se ejecutan a través del contenedor CLI de OpenClaw.

**Formatos de Entrada Soportados:**
- CSV, TSV, JSON, YAML
- Archivos de log (texto plano, syslog, JSON logs)
- JSON Lines (NDJSON)
- Bases de datos SQLite (solo lectura)
- Estructuras de directorio con archivos con patrones

**Formatos de Salida:**
- Informes Markdown con gráficos incrustados
- Dashboards HTML con gráficos Plotly interactivos
- PDF (vía LaTeX o wkhtmltopdf)
- Resultados de análisis estructurados JSON
- Activos de visualización PNG/SVG
- Excel/Workbook (.xlsx) con múltiples hojas

**Comandos:**

### `cross-gen analyze`
Comando de análisis principal con interpretación impulsada por IA.

```
cross-gen analyze --input <path> --output <dir> [options]
```

**Flags:**
- `--input`, `-i`: Archivo o directorio fuente (requerido)
- `--output`, `-o`: Directorio de salida (predeterminado: actual)
- `--type`, `-t`: Tipo de análisis: `metrics`, `security`, `dependencies`, `custom`
- `--ai-model`, `-m`: Modelo de IA: `gpt-4`, `claude-3`, `local` (predeterminado: auto)
- `--format`, `-f`: Formato de salida: `markdown`, `html`, `pdf`, `json` (predeterminado: markdown)
- `--template`, `-T`: Archivo de plantilla personalizado (opcional)
- `--threshold`, `-th`: Umbral de detección de anomalías (0.01-0.99, predeterminado: 0.95)
- `--include-charts`, `-c`: Generar visualizaciones (predeterminado: true)
- `--summary-only`, `-s`: Omitir secciones detalladas (más rápido)
- `--context`, `-ctx`: Cadena de contexto adicional para interpretación de IA

### `cross-gen validate`
Validación previa al vuelo de los datos de entrada.

```
cross-gen validate --input <path>
```

### `cross-gen list-reports`
Listar informes generados previamente en el directorio de salida.

```
cross-gen list-reports --dir <path>
```

### `cross-gen compare`
Comparar dos ejecuciones de análisis para detectar cambios.

```
cross-gen compare --baseline <file1> --current <file2> --output <dir>
```

## Proceso de Trabajo

1. **Inicialización (1-2 segundos)**
   - Cargar configuración desde `~/.openclaw/config/cross-gen.yaml` si existe
   - Verificar dependencias: pandas, matplotlib, plantillas Jinja2
   - Verificar credenciales de API si se usan modelos de IA remotos
   - Crear directorio de salida con marca de tiempo: `output/YYYY-MM-DD_HH-MM-SS/`

2. **Ingestión de Datos (depende del tamaño)**
   - Detectar formato de entrada por extensión de archivo o bytes mágicos
   - Transferir archivos grandes (>100MB) para evitar presión de memoria
   - Analizar en DataFrame de pandas estándar o estructura JSON
   - Validar esquema: columnas requeridas, tipos de datos, conteos de nulos
   - Registrar métricas de ingesta: rows=1234, cols=12, memory=45MB

3. **Fundación Estadística (1-5 segundos)**
   - Calcular estadísticas descriptivas: media, mediana, desviación estándar, cuartiles
   - Detectar valores atípicos usando IQR o z-score (configurable)
   - Matriz de correlación para columnas numéricas
   - Análisis de datos faltantes y estrategia de imputación
   - Detección de tendencias: regresión lineal, promedios móviles

4. **Interpretación de IA (3-15 segundos)**
   - Construir prompt con:
     - Resumen de datos (forma, columnas, filas de muestra)
     - Aspectos estadísticos destacados
     - Contexto desde flag `--context`
     - Instrucciones de tipo de análisis
   - Llamar al modelo de IA (OpenAI, Anthropic o local via Ollama)
   - Extraer información estructurada: key_findings, anomalies, recommendations
   - Caché de resultados en `.cross-gen-cache/` para evitar re-ejecutar en datos idénticos

5. **Generación de Informes (2-10 segundos)**
   - Cargar plantilla Jinja2 apropiada según formato/tipo
   - Inyectar información de IA, estadísticas y metadatos
   - Renderizar gráficos si está habilitado:
     - Series temporales: gráficos de línea con bandas de confianza
     - Distribuciones: histogramas + KDE
     - Correlaciones: heatmap o matriz de dispersión
     - Categóricos: gráficos de barras con etiquetas de porcentaje
   - Guardar en directorio de salida con prefijo de marca de tiempo

6. **Validación y Empaquetado**
   - Verificar que los archivos generados existan y no estén vacíos
   - Calcular checksums SHA256 para reproducibilidad
   - Escribir manifest.json con lista de archivos y metadatos
   - Opcional: subir a S3 si `CROSS_GEN_S3_BUCKET` está configurado

## Reglas de Oro

1. **Nunca modificar datos de origen** - Cross Generator opera estrictamente en solo lectura sobre entradas. Todas las salidas van a directorios dedicados con marcas de tiempo.

2. **Caché agresivo** - El directorio `.cross-gen-cache/` debe conservarse entre ejecuciones. Si `INPUT_SHA256` coincide con datos en caché, omitir llamada a IA y reutilizar información previa.

3. **Respetar límites de memoria** - Para archivos >500MB, forzar modo streaming. Fallar gracefulmente con "memory limit exceeded" en lugar de kills por OOM.

4. **Anonimizar datos sensibles** - Si nombres de columna coinciden con patrones como `*password*`, `*token*`, `*secret*`, enmascarar automáticamente valores de logs e informes. Registrar evento de enmascaramiento en nivel WARN.

5. **Umbrales deben ser configurables** - Nunca hardcodear umbral de detección de anomalías. Siempre leer desde `--threshold` o archivo de configuración.

6. **Fallar rápido en desajuste de esquema** - Si faltan columnas requeridas, abortar antes de llamada a IA. Retornar código de salida 2 con mensaje claro: "Missing required column: 'timestamp'".

7. **Gráficos deben ser reproducibles** - Establecer semilla aleatoria `np.random.seed(42)` para todas las visualizaciones. Incluir fuente de datos y marca de tiempo de generación en pie de gráfico.

8. **Códigos de salida son contratos**:
   - 0 = éxito
   - 1 = error general (con mensaje stderr)
   - 2 = fallo de validación (deps faltantes, entrada incorrecta)
   - 3 = servicio de IA no disponible
   - 4 = cuota excedida

## Ejemplos

**Ejemplo 1: Generar auditoría de seguridad desde logs de autenticación**

Entrada: `logs/auth.jsonl` (formato JSON Lines)
Comando:
```bash
cross-gen analyze \
  --input logs/auth.jsonl \
  --output reports/security-$(date +%Y%m%d) \
  --type security \
  --format html \
  --threshold 0.98 \
  --context \"Weekly security review for PCI compliance\"
```

Estructura de salida:
```
reports/security-20260302/
├── index.html              # Dashboard principal
├── charts/
│   ├── login_attempts_by_ip.png
│   ├── failed_auth_trends.svg
│   └── geographic_distribution.html  # Plotly interactivo
├── insights.json           # Información generada por IA
├── statistics.csv          # Números crudos para auditoría
└── manifest.json           # Checksums y metadatos
```

Fragmento de información generada por IA (de `insights.json`):
```json
{
  \"key_findings\": [
    \"Unusual spike in failed logins from IP 192.168.1.105: 1,247 attempts (99th percentile)\",
    \"Geographic spread expanded to 12 new countries vs last week\",
    \"Average session duration decreased by 23%, potential session hijacking\"
  ],
  \"anomalies_detected\": 7,
  \"recommendations\": [
    \"Block IP 192.168.1.105 at firewall immediately\",
    \"Enable MFA requirement for admin panel\",
    \"Review geolocation allowlist for non-business regions\"
  ]
}
```

**Ejemplo 2: Comparar cambios de dependencias**

Comando:
```bash
# Generar baseline (semana pasada)
cross-gen analyze -i package.json -o deps-baseline -t dependencies

# Generar actual
cross-gen analyze -i package.json -o deps-current -t dependencies

# Comparar
cross-gen compare --baseline deps-baseline/insights.json \
                 --current deps-current/insights.json \
                 --output deps-delta
```

Salida `deps-delta/delta.md`:
```markdown
# Dependency Change Analysis

## Critical Updates
- **lodash** upgraded from 4.17.21 to 4.17.25 (CVE-2023-XXXX patched)
- **express** downgraded from 4.18.2 to 4.17.3 (introduced regression)

## New Dependencies Added
1. `winston` (3.8.2) - logging infrastructure
2. `helmet` (7.0.0) - security headers (PASS)

## Removed Dependencies
- `debug` (no longer needed after Winston integration)

## Risk Score: MEDIUM (6/10)
Recommended: Lock `express` at 4.18.2 via `npm install express@4.18.2`
```

**Ejemplo 3: Streaming de métricas en tiempo real**

Entrada: Directorio de archivos de métricas recolectados cada 5 minutos
Comando:
```bash
cross-gen analyze \
  --input metrics/20260302/ \
  --type metrics \
  --output reports/realtime-$(date +%H%M) \
  --format markdown \
  --include-charts \
  --summary-only \
  --context \"Real-time Monolith P99 latency alert triage\"
```

**Ejemplo 4: Análisis personalizado con anulación manual**

Cuando las plantillas predeterminadas son insuficientes:
```bash
cross-gen analyze \
  --input data/campaign_performance.csv \
  --output reports/q1-campaign \
  --format pdf \
  --template templates/custom_marketing_report.html \
  --context \"Q1 2026 product launch campaign; target ROAS 3.5\"
```

Variables de plantilla personalizada disponibles:
- `{{ ai_insights }}` - objeto JSON completo
- `{{ stats_table }}` - tabla HTML
- `{{ chart_paths }}` - lista de rutas de imágenes
- `{{ metadata }}` - marca de tiempo de ejecución, checksum de entrada, configuración usada

## Comandos de Rollback

Cross Generator genera directorios de salida inmutables con marcas de tiempo. Rollback significa restaurar un directorio de salida anterior a estado activo o eliminar una ejecución fallida.

**Eliminar una ejecución de análisis fallida/mala:**
```bash
# Eliminar directorio de salida completo de forma segura (después de verificación)
rm -rf reports/security-20260302/
# O mover a cuarentena
mv reports/security-20260302/ /tmp/quarantine/
```

**Restaurar informe anterior como actual:**
```bash
# Symlink ejecución anterior exitosa como 'current'
ln -sfn reports/security-20260301 reports/current
# Actualizar symlink desplegado en servidor web
sudo ln -sfn /opt/openclaw/reports/security-20260301 /var/www/reports/latest
```

**Rollback caché de IA (forzar recomputación):**
```bash
# Limpiar caché para archivo de entrada específico
rm .cross-gen-cache/8a3f2c1d.json

# O limpiar caché completa (costoso - re-ejecuta todos los análisis)
rm -rf .cross-gen-cache/
```

**Rollback de configuración:**
```bash
# Restaurar configuración predeterminada
cp ~/.openclaw/config/cross-gen.default.yaml ~/.openclaw/config/cross-gen.yaml

# O revertir a configuración versionada
git checkout HEAD -- config/cross-gen.yaml
```

**Deshacer modificación accidental de datos (defensivo):**
Cross Generator nunca modifica datos de origen. Si se detecta corrupción de datos de origen:
```bash
# Restaurar desde backup (asume que existen snapshots)
rsync -av /backup/raw-data/20260301/ logs/
```

**Purgar todos los informes generados (limpieza de emergencia):**
```bash
# Listar antes de eliminar
find reports/ -maxdepth 1 -type d -name \"20*\" | sort
# Eliminar más antiguos de 30 días
find reports/ -maxdepth 1 -type d -name \"20*\" -mtime +30 -exec rm -rf {} \;
```

## Dependencias y Requisitos

**Paquetes de Python (auto-instalados vía `cross-gen setup`):**
```
pandas>=1.5.0
numpy>=1.24.0
matplotlib>=3.6.0
seaborn>=0.12.0
plotly>=5.14.0
jinja2>=3.1.0
openpyxl>=3.0.0  # para salida Excel
pyyaml>=6.0
requests>=2.28.0
```

**Dependencias del sistema:**
- `wkhtmltopdf` (opcional, para generación PDF)
- `pdflatex` (opcional, salida PDF LaTeX)
- `ffmpeg` (opcional, para gráficos animados)
- 512MB RAM mínimo, 2GB recomendado
- 500MB espacio en disco para caché + salida

**Configuración inicial:**
```bash
cross-gen setup
# Crea:
# - ~/.openclaw/config/cross-gen.yaml
# - ~/.openclaw/templates/cross-gen/
# - directorio .cross-gen-cache/
```

## Pasos de Verificación

**Verificación post-instalación:**
```bash
# 1. Verificar CLI accesible
cross-gen --version
# Esperado: cross-gen 1.2.0

# 2. Verificar dependencias
cross-gen doctor
# Salida debe incluir: [OK] pandas 1.5.3, [OK] matplotlib 3.6.2

# 3. Probar con datos de muestra
cross-gen analyze -i samples/sample_metrics.csv -o /tmp/test-output
# Verificar: /tmp/test-output/index.html existe, tamaño no cero

# 4. Validar conectividad IA (si se usa remoto)
cross-gen ping-ai
# Esperado: \"AI service reachable: gpt-4 (latency 2.3s)\"
```

**En cada ejecución, Cross Generator escribe a su log en `~/.openclaw/logs/cross-gen.log`. Buscar:**
```
[INFO] Analysis started: input=..., output=...
[INFO] Ingested 15,423 rows, 8 columns in 2.1s
[INFO] AI interpretation completed: 3 key_findings, 2 anomalies
[INFO] Report generated: /path/to/output/index.html (1.2MB)
[INFO] SHA256 manifest: abc123... verified
```

**Verificar integridad de salida:**
```bash
# Verificar checksums de manifest
cd reports/latest/
sha256sum -c manifest.json.sha256  # debe mostrar "OK"

# Validar estructura JSON
python -c \"import json; json.load(open('insights.json'))\"
# No debe levantar excepción

# Probar renderizado HTML (verificación rápida de sintaxis)
grep -q \"</html>\" index.html && echo \"HTML structure valid\"
```

## Solución de Problemas

**\"MemoryError: Unable to allocate\"**
- Causa: Archivo de entrada demasiado grande para memoria
- Solución: Usar modo streaming (automático para >100MB), o dividir entrada:
  ```bash
  split -l 100000 large.csv chunk_
  for f in chunk_*; do cross-gen analyze -i $f -o chunks/$(basename $f .csv); done
  ```

**\"AI service unavailable: rate limit exceeded\"**
- Causa: Demasiadas solicitudes en periodo corto
- Solución: Aumentar caché `--context` (automático), o cambiar a modelo local:
  ```bash
  cross-gen analyze -i data.csv -o out --ai-model local
  ```
  Requiere Ollama ejecutándose localmente con modelo `llama2`.

**\"Validation failed: Missing required column 'timestamp'\"**
- Causa: Desajuste de esquema de entrada
- Solución: Renombrar columna o especificar mapeo personalizado:
  ```bash
  cross-gen analyze -i data.csv -o out --column-map '{\"time\":\"timestamp\"}'
  ```

**\"PDF generation failed: wkhtmltopdf not found\"**
- Causa: Dependencia del sistema faltante
- Solución: Instalar wkhtmltopdf o usar salida HTML:
  ```bash
  sudo apt-get install wkhtmltopdf  # Ubuntu
  brew install wkhtmltopdf          # macOS
  ```
  Alternativa: `cross-gen analyze --format html`

**Gráficos aparecen en blanco o faltantes**
- Causa: Problema de backend de matplotlib en entorno sin cabeza
- Solución: Establecer backend explícitamente en config:
  ```yaml
  ~/.openclaw/config/cross-gen.yaml:
  matplotlib_backend: Agg
  ```

**\"Permission denied: .cross-gen-cache/\"**
- Causa: Directorio de caché propiedad de usuario diferente (uso de sudo)
- Solución: Restablecer propiedad:
  ```bash
  sudo chown -R $(whoami) ~/.openclaw/.cross-gen-cache/
  ```

**Rendimiento lento en conjuntos de datos grandes**
- Habilitar compresión en config:
  ```yaml
  compression: true  # usa formato intermedio parquet
  chunk_size: 50000  # procesar en chunks
  ```
- O reducir resolución de gráficos: `--chart-dpi 72`

**Fallos silenciosos (salida 0 pero salida vacía)**
- Verificar log: `tail -50 ~/.openclaw/logs/cross-gen.log`
- Causa probable: archivo de entrada vacío o cero filas después de filtrado
- Añadir flag `--verbose` para ver pasos de procesamiento

---

**Última validación:** 2026-03-02 con OpenClaw 2.1.0 y Cross Generator 1.2.0
```