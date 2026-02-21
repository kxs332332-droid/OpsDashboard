import streamlit as st
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import io

st.set_page_config(page_title="Mortgage Ops Dashboard", layout="wide", initial_sidebar_state="expanded")

@st.cache_data
def load_data(file, sheet_name=0):
    if file.name.endswith('.xlsx'):
        df = pd.read_excel(file, sheet_name=sheet_name, engine='openpyxl')
    else:
        df = pd.read_csv(file)
    return df

def parse_summary_data(df):
    # Filter key columns from Operational Summary format [file:1]
    key_metrics = ['Billable Headcount', 'Buffer HC', 'Quality ', 'Turn Around Time', 'Capability', 
                   'Monthly Volume', 'Capacity Utilization', 'Cross Utilization', 'Attrition - Monthly', 'Comments']
    df = df[df['Category'].isin(key_metrics)]
    df_pivot = df.pivot_table(index=['Process', 'LOB'], columns=['TargetActual'], values='VALUE', aggfunc='first').fillna('')
    return df_pivot.reset_index()

def parse_program_data(df):
    # Program Details: LOB, Process, monthly columns for Quality, Volume, Util, etc. [file:1]
    months = ['Jun25', 'Jul25', 'Aug25', 'Sep25', 'Oct25', 'Nov25', 'Dec 25', 'Jan 26']
    df_melted = pd.melt(df, id_vars=['Process', 'LOB'], value_vars=months, var_name='Month', value_name='Value')
    return df_melted

st.title("🏢 Mortgage Operations KPI Dashboard")
st.markdown("Upload Excel/CSV for a month (Operational Summary sheet) or Program Details for trends.")

# Sidebar for uploads and selection
st.sidebar.header("Data Input")
uploaded_files = st.sidebar.file_uploader("Choose files (XLSX/CSV)", accept_multiple_files=True)
month = st.sidebar.selectbox("Select Month", ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'])
year = st.sidebar.number_input("Year", value=2025, min_value=2020, max_value=2030)
label = f"{month} {year}"

if uploaded_files:
    data = {}
    for f in uploaded_files:
        try:
            # Try Operational Summary first
            df_sum = load_data(f, sheet_name='Operational Summary Dec25Co' if 'Dec25' in f.name else 0)
            data['summary'] = parse_summary_data(df_sum)
            st.sidebar.success(f"Loaded Summary from {f.name}")
        except:
            # Try Program Details
            df_prog = load_data(f, sheet_name='Program Details')
            data['program'] = parse_program_data(df_prog)
            st.sidebar.success(f"Loaded Trends from {f.name}")

    if 'summary' in data:
        st.session_state.summary = data['summary']
        st.session_state.label = label
    if 'program' in data:
        st.session_state.program = data['program']

# Multi-page tabs
tab1, tab2, tab3 = st.tabs(["📊 Summary Table", "📈 Trends", "🎨 Charts"])

with tab1:
    if 'summary' in st.session_state:
        st.subheader(f"Actual vs Target - {st.session_state.label}")
        df_show = st.session_state.summary
        st.dataframe(df_show, use_container_width=True, hide_index=False)
    else:
        st.info("Upload Operational Summary sheet for table.")

with tab2:
    st.subheader("MoM Trends (Program Details)")
    if 'program' in st.session_state:
        fig_trend = px.line(st.session_state.program, x='Month', y='Value', color='Process', facet_row='LOB',
                            title="Quality/Volume/Util Trends", markers=True)
        st.plotly_chart(fig_trend, use_container_width=True)
        st.dataframe(st.session_state.program)
    else:
        st.info("Upload Program Details for trends.")

with tab3:
    st.subheader("Visual Comparisons")
    if 'summary' in st.session_state:
        df_viz = st.session_state.summary.melt(ignore_index=False, id_vars=['Process', 'LOB'])
        fig_bar = px.bar(df_viz, x='Process', y='value', color='TargetActual', facet_col='LOB',
                         title=f"{st.session_state.label} Actual vs Target", barmode='group')
        st.plotly_chart(fig_bar, use_container_width=True)
        
        # Heatmap for Quality %
        qual_df = st.session_state.summary.loc[st.session_state.summary.index.get_level_values('Category') == 'Quality %'] if 'Category' in st.session_state.summary.index.names else st.session_state.summary
        fig_heat = px.imshow(qual_df.T, title="Quality % Heatmap", aspect="auto")
        st.plotly_chart(fig_heat)
    else:
        st.info("Upload data for charts.")

st.sidebar.markdown("---")
st.sidebar.info("💡 Deploy: GitHub → Streamlit Cloud. Add auth via secrets.toml for teams.")
# OpsDashboard
Operation metrics
