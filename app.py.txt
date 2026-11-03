import streamlit as st
import pandas as pd
import numpy as np
import plotly.express as px
import plotly.graph_objects as go

# 1. Page Configuration
st.set_page_config(
    page_title="Life Insurance Sales Performance Dashboard",
    page_icon="📊",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Custom CSS for executive look
st.markdown("""
    <style>
    .main-metric { font-size: 24px; font-weight: bold; color: #1E3A8A; }
    .sub-text { font-size: 14px; color: #6B7280; }
    </style>
""", unsafe_allow_html=True)

# 2. Synthetic Data Generator (Life Insurance Specific)
@st.cache_data
def generate_life_insurance_data():
    np.random.seed(42)
    n_records = 1000
    
    products = ['Term Life', 'Whole Life', 'Endowment', 'Unit-Linked (ULIP)']
    channels = ['Bancassurance', 'Agency', 'Direct Digital', 'Brokers']
    regions = ['North', 'South', 'East', 'West']
    statuses = ['In-Force', 'Lapsed', 'Surrendered', 'Paid-Up']
    
    dates = pd.date_range(start="2025-01-01", end="2026-10-31", periods=n_records)
    
    data = pd.DataFrame({
        'Policy_Date': dates,
        'Product_Type': np.random.choice(products, n_records, p=[0.4, 0.25, 0.15, 0.2]),
        'Distribution_Channel': np.random.choice(channels, n_records, p=[0.35, 0.4, 0.15, 0.1]),
        'Region': np.random.choice(regions, n_records),
        'Policy_Status': np.random.choice(statuses, n_records, p=[0.82, 0.10, 0.05, 0.03]),
        'Single_Premium': np.random.choice([0, 5000, 10000, 25000], n_records, p=[0.85, 0.08, 0.05, 0.02]),
        'Regular_Premium': np.random.randint(1200, 18000, n_records),
        'Sum_Assured': np.random.randint(50000, 1000000, n_records),
        'Agent_ID': np.random.randint(100, 130, n_records)
    })
    
    # Life Insurance Key Metrics Derivations
    # APE = Regular Premium + 10% of Single Premium
    data['APE'] = data['Regular_Premium'] + (data['Single_Premium'] * 0.10)
    # NBWP calculation variant
    data['NBWP'] = data['Regular_Premium'] + data['Single_Premium']
    
    return data

df = generate_life_insurance_data()

# 3. Sidebar Filters
st.sidebar.header("📋 Global Dashboard Filters")
selected_region = st.sidebar.multiselect("Select Region", options=df['Region'].unique(), default=df['Region'].unique())
selected_channel = st.sidebar.multiselect("Distribution Channel", options=df['Distribution_Channel'].unique(), default=df['Distribution_Channel'].unique())
selected_product = st.sidebar.multiselect("Product Portfolio", options=df['Product_Type'].unique(), default=df['Product_Type'].unique())

# Filter data dynamically
filtered_df = df[
    (df['Region'].isin(selected_region)) &
    (df['Distribution_Channel'].isin(selected_channel)) &
    (df['Product_Type'].isin(selected_product))
]

# 4. Executive Header & Core KPIs
st.title("📊 Life Insurance Executive Sales Dashboard")
st.markdown("### Strategic Performance & Underwriting Revenue Overview")

total_ape = filtered_df['APE'].sum()
total_nbwp = filtered_df['NBWP'].sum()
total_policies = len(filtered_df)
avg_sum_assured = filtered_df['Sum_Assured'].mean()
persistency_13m = (filtered_df['Policy_Status'] == 'In-Force').sum() / total_policies * 100 if total_policies > 0 else 0

col1, col2, col3, col4 = st.columns(4)
with col1:
    st.metric("Total APE Written", f"${total_ape:,.2f}", delta="+8.4% YoY vs 2025")
    st.caption("Regular Premium + 10% Single Premium")
with col2:
    st.metric("New Business Weighted Premium", f"${total_nbwp:,.2f}")
    st.caption("Total New Premium Inflow")
with col3:
    st.metric("Active Policies Issued", f"{total_policies:,}")
    st.caption("Volume of underwritten risks")
with col4:
    st.metric("13th-Month Persistency Rate", f"{persistency_13m:.1f}%", delta="0.6% vs Target")
    st.caption("Critical retention health index")

st.markdown("---")

# 5. Advanced Analytics Section
left_col, right_col = st.columns(2)

with left_col:
    st.subheader("📈 Premium Velocity Trends (Monthly APE)")
    filtered_df['YearMonth'] = filtered_df['Policy_Date'].dt.to_period('M').astype(str)
    trend_df = filtered_df.groupby('YearMonth')['APE'].sum().reset_index()
    fig_trend = px.line(trend_df, x='YearMonth', y='APE', markers=True, 
                        labels={'APE': 'APE ($)', 'YearMonth': 'Production Month'},
                        color_discrete_sequence=['#1E3A8A'])
    fig_trend.update_layout(margin=dict(l=20, r=20, t=10, b=20))
    st.plotly_chart(fig_trend, use_container_width=True)

with right_col:
    st.subheader("🤝 Mix Analysis: Channel vs Product Segment")
    mix_df = filtered_df.groupby(['Distribution_Channel', 'Product_Type'])['APE'].sum().reset_index()
    fig_mix = px.bar(mix_df, x='Distribution_Channel', y='APE', color='Product_Type',
                     barmode='stack', color_discrete_sequence=px.colors.qualitative.Prism)
    fig_mix.update_layout(margin=dict(l=20, r=20, t=10, b=20))
    st.plotly_chart(fig_mix, use_container_width=True)

# 6. Deep-Dive Metrics Table
st.markdown("### 🗃️ Performance Matrix by Distribution Channel")
summary_table = filtered_df.groupby('Distribution_Channel').agg(
    Policies_Issued=('Policy_Status', 'count'),
    Total_APE=('APE', 'sum'),
    Average_Sum_Assured=('Sum_Assured', 'mean')
).reset_index()

summary_table['Total_APE'] = summary_table['Total_APE'].map('${:,.2f}'.format)
summary_table['Average_Sum_Assured'] = summary_table['Average_Sum_Assured'].map('${:,.2f}'.format)

st.dataframe(summary_table, use_container_width=True)
