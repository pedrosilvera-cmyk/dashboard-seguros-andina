import streamlit as st
import pandas as pd
import plotly.express as px

# =====================================================
# CONFIG
# =====================================================

st.set_page_config(
    page_title="Dashboard KPIs Seguros Andina",
    layout="wide"
)

# =====================================================
# FUNCIONES
# =====================================================

def limpiar_columnas(df):
    df.columns = (
        df.columns
        .str.strip()
        .str.replace(" ", "_")
        .str.replace("-", "_")
    )
    return df

def validar_columnas(df, columnas, nombre_hoja):
    faltantes = [c for c in columnas if c not in df.columns]

    if faltantes:
        st.error(
            f"Faltan columnas en la hoja '{nombre_hoja}': {faltantes}"
        )
        st.stop()

# =====================================================
# CARGA DE DATOS
# =====================================================

archivo = "Base_Datos_Dashboard_KPIs_Seguros_Andina.xlsx"

try:

    ventas = pd.read_excel(archivo, sheet_name="Ventas")
    siniestros = pd.read_excel(archivo, sheet_name="Siniestros")
    clientes = pd.read_excel(archivo, sheet_name="Clientes")

except Exception as e:
    st.error(f"Error cargando Excel: {e}")
    st.stop()

# =====================================================
# LIMPIEZA
# =====================================================

ventas = limpiar_columnas(ventas)
siniestros = limpiar_columnas(siniestros)
clientes = limpiar_columnas(clientes)

# =====================================================
# VALIDACION
# =====================================================

validar_columnas(
    ventas,
    ["Prima_USD", "Producto", "Estado", "Canal"],
    "Ventas"
)

validar_columnas(
    siniestros,
    ["Monto_Aprobado", "Tipo_Siniestro"],
    "Siniestros"
)

validar_columnas(
    clientes,
    ["Región"],
    "Clientes"
)

# =====================================================
# NORMALIZACION
# =====================================================

ventas["Estado"] = (
    ventas["Estado"]
    .astype(str)
    .str.upper()
    .str.strip()
)

# =====================================================
# KPIs
# =====================================================

ingresos_totales = ventas["Prima_USD"].sum()

if ingresos_totales > 0:
    ratio_siniestros = (
        siniestros["Monto_Aprobado"].sum()
        / ingresos_totales
    ) * 100
else:
    ratio_siniestros = 0

retencion = (
    len(ventas[ventas["Estado"] == "RENOVADA"])
    / len(ventas)
) * 100

churn = (
    len(ventas[ventas["Estado"].str.contains("CANCEL")])
    / len(ventas)
) * 100

# =====================================================
# SIDEBAR FILTROS
# =====================================================

st.sidebar.header("Filtros")

productos = st.sidebar.multiselect(
    "Producto",
    ventas["Producto"].unique(),
    default=ventas["Producto"].unique()
)

canales = st.sidebar.multiselect(
    "Canal",
    ventas["Canal"].unique(),
    default=ventas["Canal"].unique()
)

ventas_filtradas = ventas[
    (ventas["Producto"].isin(productos)) &
    (ventas["Canal"].isin(canales))
]

# =====================================================
# HEADER
# =====================================================

st.title("📊 Dashboard Ejecutivo - Seguros Andina")

st.markdown("""
Monitoreo estratégico de KPIs corporativos.
""")

# =====================================================
# KPI CARDS
# =====================================================

col1, col2, col3, col4 = st.columns(4)

col1.metric(
    "💰 Ingresos Totales",
    f"USD {ingresos_totales:,.0f}"
)

col2.metric(
    "📉 Ratio Siniestros",
    f"{ratio_siniestros:.1f}%"
)

col3.metric(
    "🔄 Retención",
    f"{retencion:.1f}%"
)

col4.metric(
    "❌ Churn",
    f"{churn:.1f}%"
)

st.divider()

# =====================================================
# INGRESOS POR PRODUCTO
# =====================================================

ventas_producto = (
    ventas_filtradas
    .groupby("Producto")["Prima_USD"]
    .sum()
    .reset_index()
)

fig1 = px.bar(
    ventas_producto,
    x="Producto",
    y="Prima_USD",
    title="Ingresos por Producto",
    text_auto=True
)

st.plotly_chart(fig1, use_container_width=True)

# =====================================================
# POLIZAS POR ESTADO
# =====================================================

estado_polizas = (
    ventas_filtradas["Estado"]
    .value_counts()
    .reset_index()
)

estado_polizas.columns = ["Estado", "Cantidad"]

fig2 = px.pie(
    estado_polizas,
    names="Estado",
    values="Cantidad",
    title="Distribución de Pólizas"
)

st.plotly_chart(fig2, use_container_width=True)

# =====================================================
# SINIESTROS
# =====================================================

siniestros_tipo = (
    siniestros
    .groupby("Tipo_Siniestro")["Monto_Aprobado"]
    .sum()
    .reset_index()
)

fig3 = px.bar(
    siniestros_tipo,
    x="Tipo_Siniestro",
    y="Monto_Aprobado",
    title="Monto de Siniestros por Tipo",
    text_auto=True
)

st.plotly_chart(fig3, use_container_width=True)

# =====================================================
# INGRESOS POR CANAL
# =====================================================

canales_df = (
    ventas_filtradas
    .groupby("Canal")["Prima_USD"]
    .sum()
    .reset_index()
)

fig4 = px.bar(
    canales_df,
    x="Canal",
    y="Prima_USD",
    title="Ingresos por Canal",
    text_auto=True
)

st.plotly_chart(fig4, use_container_width=True)

# =====================================================
# CLIENTES POR REGION
# =====================================================

clientes_region = (
    clientes["Región"]
    .value_counts()
    .reset_index()
)

clientes_region.columns = ["Region", "Clientes"]

fig5 = px.bar(
    clientes_region,
    x="Region",
    y="Clientes",
    title="Clientes por Región",
    text_auto=True
)

st.plotly_chart(fig5, use_container_width=True)

# =====================================================
# STORYTELLING
# =====================================================

st.divider()

st.subheader("📌 Insights Ejecutivos")

if ratio_siniestros > 65:
    st.warning(
        "El ratio de siniestros supera la meta estratégica."
    )

if retencion < 80:
    st.warning(
        "La retención de clientes está por debajo del objetivo."
    )

if churn < 10:
    st.success(
        "El churn se mantiene dentro del rango esperado."
    )

st.markdown("""
### Recomendaciones

- Fortalecer políticas de evaluación de riesgo.
- Mejorar campañas de fidelización.
- Impulsar canales digitales.
- Implementar análisis predictivo de churn.
""")