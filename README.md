# Extracción documental SECOP – Atenea

Notebook que completa la **tabla contractual ETL SECOP** de ATENEA (NIT 901508361) con información sacada de los documentos contractuales de SECOP II:

- **URLs de los documentos clasificados**: `crp_cdp`, `estudios_previos`, `minutas` y `contrato_ejecucion`.
- **Obligaciones específicas** del contrato, tomadas de la minuta o de los estudios previos: `obligaciones_específicas`.
- **Datos del CRP**: `numero_crp`, `numero_cdp`, `rubro`, `descripción_rubro`, `concepto_gasto`, `fondo`, `valor_crp`, `fecha_crp` y `crp_interno_bogdata`.

La salida es `tabla_contractual_atenea.xlsx`: la tabla contractual tal como viene, una fila por contrato, con esas 14 columnas agregadas. Opcionalmente incluye las hojas `Log obligaciones` y `Log CRP`.

## Contenido del repositorio

| Archivo | Qué es |
| --- | --- |
| `Extracción documental SECOP Atenea.ipynb` | El proceso completo |
| `requirements.txt` | Librerías de Python |
| `.gitignore` | Excluye los documentos descargados, el Excel de salida y el entorno virtual |
| `cache_secop/` | Resultados ya procesados (`resultados_obligaciones.json`, `resultados_crp.json`) |
| `docs/` | Guía de instalación y ejecución, y documentación del proceso (reglas de clasificación y extracción) |

## Flujo general

1. Descarga los documentos de ATENEA de la API de datos.gov.co (conjunto `dmgg-8hin`).
2. Completa el número de contrato desde el proceso, solo cuando el proceso tiene un único contrato, y quita duplicados.
3. Carga la tabla contractual ETL SECOP, que es la tabla principal. Solo se procesan los documentos de sus contratos.
4. Clasifica los documentos por nombre: CRP/CDP, estudios previos, minutas, CDP, primera cuenta de cobro y "En ejecución".
5. Descarga y lee los documentos (PDF, Word, .zip, .rar, .7z, con OCR para los escaneados) y extrae las obligaciones y los datos del CRP.
6. Agrega las columnas a la tabla contractual y exporta el Excel.

El detalle de cada regla está en `docs/`.

## Requisitos

- **Python 3.12** (funciona desde 3.9) y las librerías de `requirements.txt`:
  ```
  python -m venv .venv
  .venv\Scripts\activate          (Windows)   |   source .venv/bin/activate   (Linux/Mac)
  pip install -r requirements.txt
  ```
- **Tesseract OCR con idioma español**, para los PDF escaneados:
  - Windows: instalador de [UB-Mannheim](https://github.com/UB-Mannheim/tesseract/wiki), marcando *Spanish*.
  - Linux: `apt-get install tesseract-ocr tesseract-ocr-spa`.
- **7-Zip**, para los .rar y .7z:
  - Windows: [7-zip.org](https://www.7-zip.org/).
  - Linux: `apt-get install p7zip-full`; para .rar, además `p7zip-rar` (repositorio non-free).

Sin Tesseract o sin 7-Zip el proceso corre igual, pero esos documentos quedan sin leer.

## Configuración

Todo lo configurable está en la primera celda de código del notebook, **Configuración general**:

| Variable | Valor por defecto | Qué es |
| --- | --- | --- |
| `DATOS_GOV_TOKEN` | `""` | App Token de datos.gov.co. Hay que pegarlo entre las comillas; si se deja vacío, se usa la variable de entorno `DATOS_GOV_TOKEN`. Sin token, la API funciona igual pero más lenta |
| `TABLA_CONTRACTUAL_URL` | Dirección del CSV en Oracle Object Storage | Tabla contractual ETL SECOP |
| `CARPETA_CACHE` | `"cache_secop"` | Carpeta del caché, relativa a la carpeta del notebook |
| `CARPETA_SALIDA` | `"."` | Carpeta donde se guarda el Excel (la del notebook) |

No hay que cambiar rutas: el notebook trabaja en la carpeta donde está guardado. En VS Code hay que abrir la carpeta del proyecto con *File > Open Folder*.

## Caché (`cache_secop/`)

Cada documento se descarga y se lee una sola vez, y los resultados se guardan en `cache_secop/`:

- `resultados_obligaciones.json` y `resultados_crp.json` **están en el repositorio**. Guardan lo ya extraído; con ellos la ejecución tarda minutos en vez de horas.
- `archivos/`, con los documentos descargados, **no se sube** porque pesa demasiado; se crea sola. Solo se usa si se cambian las reglas y hay que releer documentos; en ese caso se vuelven a descargar de SECOP.

Sin caché, la primera ejecución descarga y lee todos los documentos y puede tardar varias horas.

## Notas para el despliegue en la nube

- Tesseract y 7-Zip se instalan con los paquetes de Linux de arriba. En Azure Databricks, mediante un *init script* del clúster o una imagen propia. El código ya los busca en las rutas de Linux.
- `cache_secop/` debe vivir en almacenamiento persistente (por ejemplo un Volume de Unity Catalog o ADLS), no en el disco del clúster.
- El token y la dirección de la tabla se pueden pasar a un gestor de secretos (por ejemplo Databricks secrets), y `CARPETA_CACHE` / `CARPETA_SALIDA` a rutas del almacenamiento persistente.
- SECOP bloquea las descargas que no parecen de un navegador o que son demasiado seguidas. El código ya lo maneja: sesión tipo navegador, máximo 2 descargas en paralelo y un freno tras 20 bloqueos. Conviene probar desde la nube con la celda "Prueba rápida" de la sección 6.
