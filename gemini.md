# Protocolo de Carga y Cómputo de Regatas con Sailwave

Este documento establece el flujo operativo para la captura, transcripción, validación e inyección de resultados de regatas en archivos de **Sailwave (`.blw`)** a partir de fotografías de las planillas de llegada de la lancha de comisión.

---

## 1. Contexto y Objetivos

* **Campeonato:** XLV Gran Premio Internacional de Vela "Luis Alberto Cerrato" 2026 - Yacht Club Olivos.
* **Desafío:** Flotas numerosas con mayor propensión a errores de manuscrito (números de vela confusos, tachados, dígitos faltantes o barcos no registrados).
* **Meta:** Procesamiento ágil, seguro y trazable, manteniendo la integridad de los archivos `.blw` (codificación ANSI Windows-1252) y documentando cada carga con un reporte de novedades.

---

## 2. Flujo Operativo Paso a Paso

```
[Foto de Planilla de Llegadas]
             │
             ▼
[1. Transcripción Visual y OCR Asistido]
             │
             ▼
[2. Validación Cruzada con Docs/<Clase>.csv y <Clase>.blw]
    ├── Barco encontrado ───────────► Asignar compID y posición
    ├── Número ambiguo / error ─────► Resolución por patrón / similitud en flota
    └── Barco no inscripto ─────────► Alerta en reporte (marcar para revisión)
             │
             ▼
[3. Asignación a Flota Restante (DNC / DNF / OCS / RET)]
             │
             ▼
[4. Inyección en Archivo .blw (Windows-1252)]
             │
             ▼
[5. Generación del Reporte Markdown de Novedades]
             │
             ▼
[6. Scoring en Sailwave (F7) por el Oficial de Regatas]
```

---

## 3. Reglas de Validación y Manejo de Errores

Para agilizar la carga en flotas masivas, se aplican las siguientes reglas automáticas:

1. **Cruce preliminar de Clase:**
   * Verificar siempre la clase escrita en el manuscrito contra el archivo `.blw` destino. Si hay discrepancia (ej. membrete impreso distinto de la clase anotada a mano), se prioriza la anotación manuscrita y se notifica al usuario.

2. **Detección de ambigüedades numéricas (dígitos confusos):**
   * Números comunes de confusión en manuscrito:
     * `0` vs `6` vs `8` (ej. `208` vs `268`).
     * `1` vs `7` (ej. `214` vs `274`).
     * `3` vs `5` vs `8`.
   * **Acción:** Se cruza con el universo de barcos inscriptos en `Docs/<Clase>.csv` y en el `.blw`. Si sólo una variante existe en la lista oficial de inscriptos, se asume esa variante y se explicita en el reporte.

3. **Velas con sufijo o país:**
   * En clases internacionales (ej. Snipe o ILCA) pueden existir prefijos de país o letras agregadas. Se normaliza contra la columna `compsailno`.

4. **Barcos ausentes en la planilla (No llegados):**
   * Todos los competidores inscriptos en el `.blw` que no figuren en la planilla de llegada recibirán por defecto el código **`DNC`** (tipo `rrestyp = 3`, código `DNC`, puntaje = `N_inscriptos + 1`), a menos que la planilla explicite otro código (`DNF`, `DNS`, `OCS`, `UFD`, `BFD`, `RET`).

5. **Códigos de penalización o retiro:**
   * Anotaciones marginales en la planilla (ej. "OCS 1234", "DNF 5678", "UFD 9999") se ingresan con su respectivo `rcod` y `rrestyp = 2` o `3`.

6. **Barco no encontrado (Vela fantasma):**
   * Si un número de llegada no figura bajo ninguna variante en la lista de inscriptos:
     * Se emite una alerta destacada en el reporte.
     * No se interrumpe la carga del resto de los competidores válidos.
     * Se solicita confirmación al usuario (puede ser un cambio de vela no declarado o un timonel de último momento).

---

## 4. Estructura Interna del Archivo `.blw` para Inyecciones

* **Codificación obligatoria:** `Windows-1252 (ANSI)`. Nunca guardar como UTF-8 directo con BOM.
* **Nueva Regata:**
  * Incrementar el correlativo visible `racerank` (ej. `6`).
  * Asignar un nuevo `raceID` único (no colisionar con IDs existentes).
  * Cabecera de regata:
    ```csv
    "racerank","<NroRegata>","","<raceID>"
    "racesailed","1","","<raceID>"
    "racestart","||Place|Start 1|||0||0|0||||1","","<raceID>"
    ```
* **Registros de Llegada Estándar por Competidor (`compID`):**
  ```csv
  "rpts","<pos>","<compID>","<raceID>"
  "rpos","<pos>","<compID>","<raceID>"
  "rdisc","0","<compID>","<raceID>"
  "rrecpos","<pos>","<compID>","<raceID>"
  "rrestyp","1","<compID>","<raceID>"
  "srat","0","<compID>","<raceID>"
  "rrset","0","<compID>","<raceID>"
  ```
* **Registros con Código (ej. DNC):**
  ```csv
  "rcod","DNC","<compID>","<raceID>"
  "rpts","<TotalInscriptos + 1>","<compID>","<raceID>"
  "rpos","<TotalInscriptos>","<compID>","<raceID>"
  "rdisc","0","<compID>","<raceID>"
  "rrestyp","3","<compID>","<raceID>"
  "srat","0","<compID>","<raceID>"
  "rrset","1","<compID>","<raceID>"
  ```

---

## 5. Formato del Reporte Post-Carga (`Docs/reportes/<Clase>_R<N>_report.md`)

Tras procesar cada foto y actualizar el archivo `.blw`, se generará automáticamente un reporte Markdown guardado en la carpeta `Docs/` con la siguiente estructura estándar:

```markdown
# Reporte de Carga - [Clase] - Regata [N]
**Fecha y Hora de Carga:** YYYY-MM-DD HH:MM
**Imagen Procesada:** [Ruta / Nombre de Foto]
**Archivo Actualizado:** [Ruta del archivo .blw]

## 1. Novedades y Discrepancias Detectadas
* **Lecturas dudosas resueltas:**
  - *Ejemplo:* Vela manuscrita leída como `208`, resuelta como `268` (Rainbow - Gerónimo Lutteral) tras verificar padrón de inscriptos.
* **Barcos no inscriptos / Nros no encontrados:**
  - *(Ninguno / o detalle de velas no encontradas para revisión manual)*.
* **Códigos especiales aplicados:**
  - *Ejemplo:* Vela 89 asignada con `DNC` (no figura en la planilla de llegada).

## 2. Orden de Llegada Cargado
| Puesto | N° Vela | Barco / Timonel | compID | Tipo / Código | Puntos |
| :---: | :---: | :--- | :---: | :---: | :---: |
| 1° | ... | ... | ... | Puesto | 1 |
| ... | ... | ... | ... | ... | ... |
| DNC | ... | ... | ... | DNC | ... |

## 3. Instrucciones de Cierre
1. Abrir `<Clase>.blw` en Sailwave.
2. Presionar **Score Series** (`F7`).
3. Exportar resultados en PDF / HTML según corresponda.
```

---

## 6. Checklist Rápido para el Fin de Semana

* [ ] Tener abierto el chat de Antigravity.
* [ ] Arrastrar / pegar la foto de la planilla de la regata.
* [ ] Indicar la clase y número de regata deseado si no fuera evidente en la foto.
* [ ] Revisar el resumen visual de llegadas y novedades en la respuesta del agente.
* [ ] Abrir Sailwave y presionar `F7` para confirmar resultados acumulados.
