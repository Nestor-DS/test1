```
"""
Inyeccion de evidencias SonarQube en un reporte Word existente.

Flujo:
1. Localiza y enfoca la ventana de Eclipse.
2. Por cada clase configurada, espera ENTER y captura Antes / Despues.
3. Abre el .docx, usa exclusivamente la segunda tabla (indice 1).
4. Reutiliza filas vacias bajo la cabecera; si faltan, clona el XML de la
   ultima fila de datos para conservar bordes, sombreado y anchos.
5. Escribe el ID e inserta las imagenes ajustadas al ancho util de la celda.

Plataforma objetivo: Windows. pygetwindow no enfoca ventanas de forma
nativa fiable en Linux/macOS.
"""

from __future__ import annotations

import logging
import sys
import tempfile
from copy import deepcopy
from pathlib import Path
from typing import Iterable

import pyautogui
import pygetwindow as gw
from docx import Document
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.oxml.ns import qn
from docx.shared import Emu, Pt
from docx.table import Table, _Cell
from PIL import Image

# ---------------------------------------------------------------------------
# Configuracion. Editar solo este bloque.
# ---------------------------------------------------------------------------

ARCHIVOS_A_REVISAR: list[str] = [
    "ClaseUsuarios.java",
    "ControladorLogin.java",
    "ServicioFactura.java",
]

RUTA_DOCUMENTO = Path(r"C:\reportes\revision_sonarqube.docx")
RUTA_SALIDA = Path(r"C:\reportes\revision_sonarqube_evidencias.docx")

# Subcadena del titulo de ventana. Eclipse suele titular
# "NombreProyecto - archivo.java - Eclipse IDE".
TITULO_VENTANA_ECLIPSE = "Eclipse"

# Indice de la tabla de evidencias. 0 = equipo, 1 = pantallas, 2 = firmas.
INDICE_TABLA_EVIDENCIAS = 1
TABLAS_ESPERADAS = 3

# Margen interno restado al ancho de celda para no pisar bordes (EMU).
MARGEN_IMAGEN_EMU = 80_000
ANCHO_IMAGEN_FALLBACK_EMU = 2_400_000  # ~2.6 cm si la celda no declara tcW

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
    datefmt="%H:%M:%S",
)
log = logging.getLogger("evidencias_sonar")


class VentanaEclipseNoEncontrada(RuntimeError):
    """No hay una ventana de Eclipse visible que coincida con el titulo."""


class DocumentoInvalido(RuntimeError):
    """El .docx no tiene la estructura de tablas esperada."""


def localizar_ventana_eclipse(titulo: str):
    """Busca la ventana de Eclipse, la restaura si esta minimizada y la enfoca."""
    coincidencias = [
        ventana
        for ventana in gw.getWindowsWithTitle(titulo)
        if ventana.title and titulo.lower() in ventana.title.lower()
    ]
    if not coincidencias:
        raise VentanaEclipseNoEncontrada(
            f"No se encontro una ventana cuyo titulo contenga '{titulo}'. "
            "Verifique que Eclipse este abierto y no en otra sesion de escritorio."
        )

    ventana = max(coincidencias, key=lambda item: item.width * item.height)
    if ventana.isMinimized:
        ventana.restore()
    ventana.activate()
    pyautogui.sleep(0.4)
    if ventana.left < 0 or ventana.top < 0:
        log.warning(
            "La ventana reporta coordenadas negativas (%s, %s). "
            "Se recortara la region al area visible.",
            ventana.left,
            ventana.top,
        )
    return ventana


def region_visible(ventana) -> tuple[int, int, int, int]:
    """Devuelve (left, top, width, height) recortada a la pantalla principal."""
    pantalla_ancho, pantalla_alto = pyautogui.size()
    left = max(0, int(ventana.left))
    top = max(0, int(ventana.top))
    right = min(pantalla_ancho, int(ventana.left + ventana.width))
    bottom = min(pantalla_alto, int(ventana.top + ventana.height))
    ancho = right - left
    alto = bottom - top
    if ancho < 50 or alto < 50:
        raise VentanaEclipseNoEncontrada(
            "La region visible de Eclipse es demasiado pequena para capturar. "
            "Restaure la ventana y coloquela dentro del monitor principal."
        )
    return left, top, ancho, alto


def capturar_ventana(ventana, destino: Path) -> Path:
    """Captura solo el rectangulo de Eclipse y lo guarda en PNG."""
    region = region_visible(ventana)
    imagen = pyautogui.screenshot(region=region)
    imagen.save(destino, format="PNG")
    log.info("Captura guardada: %s (%sx%s)", destino.name, region[2], region[3])
    return destino


def esperar_enter(mensaje: str) -> None:
    """Bloquea hasta ENTER. Ctrl+C aborta el lote sin corromper el .docx."""
    print(mensaje)
    try:
        input()
    except EOFError as exc:
        raise RuntimeError("Entrada estandar cerrada antes de la captura.") from exc


def capturar_par_evidencia(
    ventana,
    nombre_archivo: str,
    directorio: Path,
    indice: int,
) -> tuple[Path, Path]:
    """Pide Antes y Despues para una clase y devuelve las rutas temporales."""
    base = f"{indice:02d}_{Path(nombre_archivo).stem}"
    ruta_antes = directorio / f"{base}_antes.png"
    ruta_despues = directorio / f"{base}_despues.png"

    esperar_enter(
        f"\n[{indice}] {nombre_archivo}\n"
        "Prepare en Eclipse la pantalla ANTES (hallazgo SonarQube visible) "
        "y presione ENTER para capturar."
    )
    ventana = localizar_ventana_eclipse(TITULO_VENTANA_ECLIPSE)
    capturar_ventana(ventana, ruta_antes)

    esperar_enter(
        f"[{indice}] {nombre_archivo}\n"
        "Prepare en Eclipse la pantalla DESPUES (correccion aplicada) "
        "y presione ENTER para capturar."
    )
    ventana = localizar_ventana_eclipse(TITULO_VENTANA_ECLIPSE)
    capturar_ventana(ventana, ruta_despues)
    return ruta_antes, ruta_despues


def validar_documento(documento: Document) -> Table:
    """Exige al menos 3 tablas y una segunda tabla con 3 columnas."""
    if len(documento.tables) < TABLAS_ESPERADAS:
        raise DocumentoInvalido(
            f"Se esperaban al menos {TABLAS_ESPERADAS} tablas; "
            f"el documento tiene {len(documento.tables)}."
        )
    tabla = documento.tables[INDICE_TABLA_EVIDENCIAS]
    if not tabla.rows:
        raise DocumentoInvalido("La tabla de evidencias no tiene filas.")
    columnas = len(tabla.columns)
    if columnas < 3:
        raise DocumentoInvalido(
            f"La tabla de evidencias tiene {columnas} columnas; se requieren 3 "
            "(ID, Antes, Despues)."
        )
    return tabla


def ancho_celda_emu(celda: _Cell) -> int:
    """Lee tcW en twips (dxa) y lo convierte a EMU. 1 twip = 635 EMU."""
    tc_pr = celda._tc.tcPr
    if tc_pr is None or tc_pr.tcW is None or tc_pr.tcW.w is None:
        return ANCHO_IMAGEN_FALLBACK_EMU
    tipo = tc_pr.tcW.type
    valor = int(tc_pr.tcW.w)
    if tipo == "dxa" and valor > 0:
        return valor * 635
    if tipo == "pct":
        # pct es relativo a 5000 = 100 %. Sin ancho de tabla fiable, fallback.
        return ANCHO_IMAGEN_FALLBACK_EMU
    return ANCHO_IMAGEN_FALLBACK_EMU


def limpiar_celda(celda: _Cell) -> None:
    """Vacia parrafos y dibujos heredados del clon, conservando tcPr."""
    tc = celda._tc
    for hijo in list(tc):
        if hijo.tag == qn("w:p"):
            tc.remove(hijo)
    tc.append(parse_parrafo_vacio())


def parse_parrafo_vacio():
    """Parrafo minimo con el namespace de WordprocessingML."""
    from docx.oxml import OxmlElement

    parrafo = OxmlElement("w:p")
    p_pr = OxmlElement("w:pPr")
    jc = OxmlElement("w:jc")
    jc.set(qn("w:val"), "center")
    p_pr.append(jc)
    parrafo.append(p_pr)
    return parrafo


def clonar_fila(tabla: Table):
    """
    Clona la ultima fila de datos a nivel XML.

    deepcopy del elemento w:tr copia tcPr (anchos, bordes, shading, gridSpan)
    y pPr. Despues se limpian los parrafos para no duplicar texto ni blips
    de imagenes ya insertadas. Anexar el tr al tbl mantiene el orden visual
    y no toca las tablas 0 y 2.
    """
    plantilla = tabla.rows[-1]._tr
    nueva = deepcopy(plantilla)
    tabla._tbl.append(nueva)
    fila = tabla.rows[-1]
    for celda in fila.cells:
        limpiar_celda(celda)
    return fila


def asegurar_filas(tabla: Table, cantidad_archivos: int) -> None:
    """Reutiliza filas existentes tras la cabecera; clona solo el faltante."""
    filas_datos = max(0, len(tabla.rows) - 1)
    faltantes = cantidad_archivos - filas_datos
    if faltantes <= 0:
        log.info(
            "Filas de datos disponibles: %s. No se insertan filas nuevas.",
            filas_datos,
        )
        return
    log.info("Insertando %s fila(s) clonada(s) en la tabla de evidencias.", faltantes)
    for _ in range(faltantes):
        clonar_fila(tabla)


def escribir_id(celda: _Cell, nombre: str) -> None:
    limpiar_celda(celda)
    parrafo = celda.paragraphs[0]
    parrafo.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = parrafo.add_run(nombre)
    run.bold = True
    run.font.size = Pt(10)
    run.font.name = "Calibri"


def insertar_imagen(celda: _Cell, ruta_imagen: Path) -> None:
    """Inserta el PNG escalado al ancho util. Alto proporcional, sin deformar."""
    limpiar_celda(celda)
    ancho_disponible = max(200_000, ancho_celda_emu(celda) - MARGEN_IMAGEN_EMU)
    with Image.open(ruta_imagen) as imagen:
        if imagen.width <= 0 or imagen.height <= 0:
            raise ValueError(f"Imagen invalida: {ruta_imagen}")
    parrafo = celda.paragraphs[0]
    parrafo.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = parrafo.add_run()
    run.add_picture(str(ruta_imagen), width=Emu(ancho_disponible))


def inyectar_evidencias(
    tabla: Table,
    evidencias: Iterable[tuple[str, Path, Path]],
) -> None:
    """Fila 0 es cabecera. Los datos empiezan en el indice 1."""
    for indice, (nombre, antes, despues) in enumerate(evidencias, start=1):
        fila = tabla.rows[indice]
        celdas = fila.cells
        if len(celdas) < 3:
            raise DocumentoInvalido(
                f"La fila {indice} no tiene 3 celdas. Revise celdas combinadas."
            )
        escribir_id(celdas[0], nombre)
        insertar_imagen(celdas[1], antes)
        insertar_imagen(celdas[2], despues)
        log.info("Fila %s actualizada: %s", indice, nombre)


def ejecutar() -> Path:
    if not ARCHIVOS_A_REVISAR:
        raise ValueError("ARCHIVOS_A_REVISAR esta vacia.")
    if not RUTA_DOCUMENTO.is_file():
        raise FileNotFoundError(f"No existe la plantilla: {RUTA_DOCUMENTO}")

    log.info("Archivos a evidenciar: %s", len(ARCHIVOS_A_REVISAR))
    ventana = localizar_ventana_eclipse(TITULO_VENTANA_ECLIPSE)
    log.info("Ventana enfocada: %s", ventana.title)

    capturas: list[tuple[str, Path, Path]] = []
    with tempfile.TemporaryDirectory(prefix="sonar_evidencias_") as temporal:
        directorio = Path(temporal)
        for indice, nombre in enumerate(ARCHIVOS_A_REVISAR, start=1):
            antes, despues = capturar_par_evidencia(
                ventana, nombre, directorio, indice
            )
            capturas.append((nombre, antes, despues))

        try:
            documento = Document(str(RUTA_DOCUMENTO))
        except Exception as exc:
            raise DocumentoInvalido(
                f"No se pudo abrir el documento: {RUTA_DOCUMENTO}"
            ) from exc

        tabla = validar_documento(documento)
        asegurar_filas(tabla, len(capturas))
        inyectar_evidencias(tabla, capturas)

        RUTA_SALIDA.parent.mkdir(parents=True, exist_ok=True)
        try:
            documento.save(str(RUTA_SALIDA))
        except PermissionError as exc:
            raise PermissionError(
                f"Sin permiso para escribir {RUTA_SALIDA}. "
                "Cierre el archivo en Word si esta abierto."
            ) from exc

    log.info("Documento generado: %s", RUTA_SALIDA)
    return RUTA_SALIDA


def main() -> int:
    try:
        ejecutar()
    except VentanaEclipseNoEncontrada as exc:
        log.error("Ventana de Eclipse: %s", exc)
        return 2
    except DocumentoInvalido as exc:
        log.error("Documento: %s", exc)
        return 3
    except PermissionError as exc:
        log.error("Permisos: %s", exc)
        return 4
    except KeyboardInterrupt:
        log.error("Ejecucion cancelada por el usuario. No se guardo el documento.")
        return 130
    except Exception as exc:
        log.exception("Fallo no controlado: %s", exc)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())

```