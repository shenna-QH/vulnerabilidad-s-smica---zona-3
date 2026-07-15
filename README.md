"""
eda_cismid.py
=============

Módulo de Análisis Exploratorio de Datos (EDA) para registros acelerográficos
del CISMID — Zona Sísmica 3 (Perú).

Cubre la sección 1 del informe T2 (Grupo 5, UPC — Aplicaciones de IA en
Estructuras): adquisición/carga de registros y metadatos, depuración
(nulos, duplicados, inconsistencias de formato), caracterización
estadística y análisis de correlación entre variables sísmicas.

Metodología: Veridical Data Science (VDS) — Yu & Barter (2024).
Cada función es determinista, documentada y trazable (sin modelos de ML).

Autor: Grupo 5 — UPC
"""

from __future__ import annotations

import logging
from dataclasses import dataclass, field
from pathlib import Path

import numpy as np
import pandas as pd

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
    datefmt="%H:%M:%S",
)
logger = logging.getLogger("eda_cismid")


# ---------------------------------------------------------------------------
# 1. Configuración y esquema de datos esperado
# ---------------------------------------------------------------------------

# Columnas mínimas requeridas en el catálogo de metadatos CISMID.
COLUMNAS_REQUERIDAS = [
    "id_registro",
    "fecha_evento",
    "magnitud_mw",
    "profundidad_km",
    "latitud",
    "longitud",
    "estacion",
    "intervalo_muestreo_s",
    "duracion_s",
    "pga_cm_s2",
]

# Rangos físicamente plausibles usados como criterio de calidad (Sección 3.5
# del informe: "Calidad de la selección y gestión de registros").
RANGOS_VALIDOS = {
    "magnitud_mw": (3.0, 9.5),
    "profundidad_km": (0.0, 700.0),
    "intervalo_muestreo_s": (0.001, 0.05),
    "duracion_s": (1.0, 600.0),
    "pga_cm_s2": (0.0, 3000.0),
}


@dataclass
class ReporteCalidad:
    """Resumen de la etapa de depuración, para trazabilidad (principio VDS)."""

    n_inicial: int = 0
    n_final: int = 0
    nulos_eliminados: int = 0
    duplicados_eliminados: int = 0
    fuera_de_rango_eliminados: int = 0
    formato_invalido_eliminados: int = 0
    detalle: list[str] = field(default_factory=list)

    @property
    def porcentaje_retenido(self) -> float:
        if self.n_inicial == 0:
            return 0.0
        return round(100 * self.n_final / self.n_inicial, 2)

    def resumen(self) -> str:
        return (
            f"Registros iniciales: {self.n_inicial}\n"
            f"  - Eliminados por nulos: {self.nulos_eliminados}\n"
            f"  - Eliminados por duplicados: {self.duplicados_eliminados}\n"
            f"  - Eliminados por formato inválido: {self.formato_invalido_eliminados}\n"
            f"  - Eliminados por rango físico inválido: {self.fuera_de_rango_eliminados}\n"
            f"Registros finales (aptos): {self.n_final} "
            f"({self.porcentaje_retenido}% retenido)"
        )


# ---------------------------------------------------------------------------
# 2. Carga de datos (Sección 2.1 — Adquisición y organización)
# ---------------------------------------------------------------------------

def cargar_catalogo(ruta: str | Path) -> pd.DataFrame:
    """
    Carga el catálogo de metadatos de registros CISMID desde CSV o Excel.

    Parameters
    ----------
    ruta : str | Path
        Ruta al archivo .csv, .xlsx o .txt (delimitado) con el catálogo.

    Returns
    -------
    pd.DataFrame
        Catálogo crudo, sin depurar.
    """
    ruta = Path(ruta)
    if not ruta.exists():
        raise FileNotFoundError(f"No se encontró el archivo: {ruta}")

    if ruta.suffix.lower() == ".csv":
        df = pd.read_csv(ruta)
    elif ruta.suffix.lower() in (".xlsx", ".xls"):
        df = pd.read_excel(ruta)
    elif ruta.suffix.lower() == ".txt":
        df = pd.read_csv(ruta, sep=None, engine="python")
    else:
        raise ValueError(f"Formato no soportado: {ruta.suffix}")

    logger.info("Catálogo cargado: %d registros, %d columnas", *df.shape)
    return df


def cargar_acelerograma(ruta: str | Path) -> pd.DataFrame:
    """
    Carga un archivo individual de registro acelerográfico CISMID
    (formato .cor / .txt: columnas de tiempo y aceleración, con o sin
    cabecera de metadatos al inicio).

    Se asume la convención habitual del REDACIS: primeras filas de
    cabecera (metadatos) seguidas de dos columnas numéricas
    (tiempo [s], aceleración [cm/s²] o [gal]).

    Returns
    -------
    pd.DataFrame con columnas ['tiempo_s', 'aceleracion_cm_s2']
    """
    ruta = Path(ruta)
    if not ruta.exists():
        raise FileNotFoundError(f"No se encontró el archivo: {ruta}")

    # Se descartan líneas de cabecera no numéricas automáticamente.
    filas_validas = []
    with open(ruta, "r", encoding="latin-1") as f:
        for linea in f:
            partes = linea.split()
            if len(partes) >= 2:
                try:
                    t, a = float(partes[0]), float(partes[1])
                    filas_validas.append((t, a))
                except ValueError:
                    continue  # línea de cabecera / texto, se ignora

    if not filas_validas:
        raise ValueError(f"No se pudieron extraer datos numéricos de {ruta}")

    df = pd.DataFrame(filas_validas, columns=["tiempo_s", "aceleracion_cm_s2"])
    logger.info("Acelerograma cargado (%s): %d muestras", ruta.name, len(df))
    return df


# ---------------------------------------------------------------------------
# 3. Depuración y validación (Sección 2.2 — Limpieza y validación de datos)
# ---------------------------------------------------------------------------

def depurar_catalogo(
    df: pd.DataFrame,
    columnas_requeridas: list[str] = COLUMNAS_REQUERIDAS,
    rangos_validos: dict[str, tuple[float, float]] = RANGOS_VALIDOS,
) -> tuple[pd.DataFrame, ReporteCalidad]:
    """
    Aplica la etapa de depuración descrita en la Sección 2.2 del informe:
    valores nulos, duplicados, inconsistencias de formato y valores fuera
    de rango físico plausible.

    Returns
    -------
    (df_limpio, reporte) : DataFrame depurado y reporte de trazabilidad.
    """
    reporte = ReporteCalidad(n_inicial=len(df))
    df = df.copy()

    # 3.1 — Verificar columnas mínimas requeridas.
    faltantes = [c for c in columnas_requeridas if c not in df.columns]
    if faltantes:
        raise ValueError(f"Faltan columnas requeridas en el catálogo: {faltantes}")

    # 3.2 — Eliminar filas con nulos en columnas clave.
    n_antes = len(df)
    df = df.dropna(subset=columnas_requeridas)
    reporte.nulos_eliminados = n_antes - len(df)

    # 3.3 — Eliminar duplicados por id_registro.
    n_antes = len(df)
    df = df.drop_duplicates(subset="id_registro", keep="first")
    reporte.duplicados_eliminados = n_antes - len(df)

    # 3.4 — Forzar tipos numéricos; lo no convertible se marca como formato inválido.
    columnas_numericas = [
        "magnitud_mw", "profundidad_km", "latitud", "longitud",
        "intervalo_muestreo_s", "duracion_s", "pga_cm_s2",
    ]
    n_antes = len(df)
    for col in columnas_numericas:
        df[col] = pd.to_numeric(df[col], errors="coerce")
    df = df.dropna(subset=columnas_numericas)
    reporte.formato_invalido_eliminados = n_antes - len(df)

    # 3.5 — Filtrar por rangos físicamente plausibles.
    n_antes = len(df)
    mascara = pd.Series(True, index=df.index)
    for col, (lo, hi) in rangos_validos.items():
        mascara &= df[col].between(lo, hi)
        n_fuera = (~df[col].between(lo, hi)).sum()
        if n_fuera:
            reporte.detalle.append(f"{col}: {n_fuera} valores fuera de [{lo}, {hi}]")
    df = df[mascara]
    reporte.fuera_de_rango_eliminados = n_antes - len(df)

    reporte.n_final = len(df)
    logger.info("Depuración completada.\n%s", reporte.resumen())
    return df.reset_index(drop=True), reporte


def clasificar_calidad(df: pd.DataFrame) -> pd.DataFrame:
    """
    Añade la columna 'apto_procesamiento' (Sección 1.1 — Indicadores
    automáticos de calidad) clasificando cada registro como apto o no
    apto según los rangos definidos en RANGOS_VALIDOS.
    """
    df = df.copy()
    apto = pd.Series(True, index=df.index)
    for col, (lo, hi) in RANGOS_VALIDOS.items():
        if col in df.columns:
            apto &= df[col].between(lo, hi)
    df["apto_procesamiento"] = np.where(apto, "apto", "no apto")
    return df


# ---------------------------------------------------------------------------
# 4. Caracterización estadística (Sección 1, párrafo 2)
# ---------------------------------------------------------------------------

def estadisticas_descriptivas(
    df: pd.DataFrame,
    variables: list[str] | None = None,
) -> pd.DataFrame:
    """
    Calcula estadísticos descriptivos (n, media, desv. estándar, min, cuartiles,
    max, asimetría y curtosis) para las variables sísmicas de interés.
    """
    if variables is None:
        variables = ["magnitud_mw", "profundidad_km", "pga_cm_s2",
                     "duracion_s", "intervalo_muestreo_s"]
    variables = [v for v in variables if v in df.columns]

    resumen = df[variables].describe().T
    resumen["asimetria"] = df[variables].skew()
    resumen["curtosis"] = df[variables].kurt()
    return resumen.round(4)


def detectar_atipicos(df: pd.DataFrame, columna: str, factor_iqr: float = 1.5) -> pd.DataFrame:
    """
    Identifica valores atípicos en una variable mediante el criterio de
    rango intercuartílico (IQR), útil para depurar antes del procesamiento
    de señales (corrección de línea base, filtrado).

    Returns
    -------
    Subconjunto de df con los registros atípicos en `columna`.
    """
    q1, q3 = df[columna].quantile([0.25, 0.75])
    iqr = q3 - q1
    lim_inf, lim_sup = q1 - factor_iqr * iqr, q3 + factor_iqr * iqr
    atipicos = df[(df[columna] < lim_inf) | (df[columna] > lim_sup)]
    logger.info(
        "Variable '%s': %d valores atípicos fuera de [%.3f, %.3f]",
        columna, len(atipicos), lim_inf, lim_sup,
    )
    return atipicos


# ---------------------------------------------------------------------------
# 5. Análisis de correlación (Sección 1, párrafo 2)
# ---------------------------------------------------------------------------

def matriz_correlacion(
    df: pd.DataFrame,
    variables: list[str] | None = None,
    metodo: str = "pearson",
) -> pd.DataFrame:
    """
    Calcula la matriz de correlación entre variables sísmicas (magnitud,
    profundidad, distancia epicentral, PGA, etc.).

    Parameters
    ----------
    metodo : {'pearson', 'spearman', 'kendall'}
    """
    if variables is None:
        variables = ["magnitud_mw", "profundidad_km", "pga_cm_s2", "duracion_s"]
    variables = [v for v in variables if v in df.columns]
    return df[variables].corr(method=metodo).round(3)


def pares_mas_correlacionados(
    matriz_corr: pd.DataFrame, umbral: float = 0.7
) -> pd.DataFrame:
    """
    Extrae, en formato tabular, los pares de variables con correlación
    absoluta mayor o igual al umbral indicado (por defecto 0.7).
    """
    pares = []
    columnas = matriz_corr.columns
    for i, var_a in enumerate(columnas):
        for var_b in columnas[i + 1:]:
            r = matriz_corr.loc[var_a, var_b]
            if abs(r) >= umbral:
                pares.append({"variable_1": var_a, "variable_2": var_b, "r": r})
    columnas_salida = ["variable_1", "variable_2", "r"]
    if not pares:
        return pd.DataFrame(columns=columnas_salida)
    return (
        pd.DataFrame(pares, columns=columnas_salida)
        .sort_values("r", key=abs, ascending=False)
        .reset_index(drop=True)
    )


# ---------------------------------------------------------------------------
# 6. Flujo completo (orquestador)
# ---------------------------------------------------------------------------

def ejecutar_eda(ruta_catalogo: str | Path) -> dict:
    """
    Ejecuta el flujo completo de EDA: carga → depuración → estadísticas →
    correlación → clasificación de calidad.

    Returns
    -------
    dict con las claves: 'df_limpio', 'reporte_calidad', 'estadisticas',
    'correlacion', 'pares_correlacionados'.
    """
    df_crudo = cargar_catalogo(ruta_catalogo)
    df_limpio, reporte = depurar_catalogo(df_crudo)
    df_limpio = clasificar_calidad(df_limpio)

    stats = estadisticas_descriptivas(df_limpio)
    corr = matriz_correlacion(df_limpio)
    pares = pares_mas_correlacionados(corr)

    return {
        "df_limpio": df_limpio,
        "reporte_calidad": reporte,
        "estadisticas": stats,
        "correlacion": corr,
        "pares_correlacionados": pares,
    }


# ---------------------------------------------------------------------------
# 7. Ejemplo de uso
# ---------------------------------------------------------------------------

if __name__ == "__main__":
    # Genera un catálogo sintético de ejemplo si no se dispone aún de datos
    # reales del CISMID, solo para validar que el módulo corre correctamente.
    import tempfile

    rng = np.random.default_rng(42)
    n = 50
    catalogo_demo = pd.DataFrame({
        "id_registro": [f"REG_{i:03d}" for i in range(n)],
        "fecha_evento": pd.date_range("2015-01-01", periods=n, freq="30D"),
        "magnitud_mw": rng.uniform(4.0, 7.5, n),
        "profundidad_km": rng.uniform(10, 120, n),
        "latitud": rng.uniform(-14, -9, n),
        "longitud": rng.uniform(-78, -74, n),
        "estacion": rng.choice(["ICA1", "LIMA2", "NAZCA1"], n),
        "intervalo_muestreo_s": rng.choice([0.005, 0.01, 0.02], n),
        "duracion_s": rng.uniform(20, 180, n),
        "pga_cm_s2": rng.uniform(20, 600, n),
    })

    with tempfile.TemporaryDirectory() as tmp:
        ruta_demo = Path(tmp) / "catalogo_demo.csv"
        catalogo_demo.to_csv(ruta_demo, index=False)

        resultados = ejecutar_eda(ruta_demo)

        print("\n=== REPORTE DE CALIDAD ===")
        print(resultados["reporte_calidad"].resumen())

        print("\n=== ESTADÍSTICAS DESCRIPTIVAS ===")
        print(resultados["estadisticas"])

        print("\n=== MATRIZ DE CORRELACIÓN (Pearson) ===")
        print(resultados["correlacion"])

        print("\n=== PARES MÁS CORRELACIONADOS (|r| >= 0.7) ===")
        print(resultados["pares_correlacionados"])
