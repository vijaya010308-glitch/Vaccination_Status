import streamlit as st
import pandas as pd
import plotly.express as px
from pathlib import Path

st.set_page_config(
    page_title="Vaccination Analysis Dashboard",
    page_icon="💉",
    layout="wide"
)

st.title("💉 Vaccination Status Analysis")
st.write("Explore vaccination survey data using interactive charts.")

# Load dataset
file_path = (
    Path(__file__).resolve().parent.parent
    / "dataset"
    / "vaccination_status_clean.xlsx"
)

if file_path.exists():
    df = pd.read_excel(file_path)
else:
    uploaded_file = st.file_uploader(
        "Upload vaccination_status_clean.xlsx",
        type=["xlsx"]
    )
    if uploaded_file is None:
        st.info("Please upload your Excel dataset.")
        st.stop()
    df = pd.read_excel(uploaded_file)

# Check required column
if "vaccination_status" not in df.columns:
    st.error("Column 'vaccination_status' was not found.")
    st.write("Available columns:", df.columns.tolist())
    st.stop()

# Clean data
df = df.drop_duplicates().copy()
df["vaccination_status"] = (
    df["vaccination_status"].astype("string").str.strip()
)
df = df.dropna(subset=["vaccination_status"])
df = df[df["vaccination_status"] != ""]

# Convert date if available
if "submitted_at" in df.columns:
    df["submitted_at"] = pd.to_datetime(
        df["submitted_at"], errors="coerce", utc=True
    )

# Summary metrics
c1, c2, c3 = st.columns(3)

c1.metric("Total Responses", len(df))
c2.metric(
    "Vaccination Categories",
    df["vaccination_status"].nunique()
)
c3.metric(
    "Most Common Status",
    df["vaccination_status"].mode().iloc[0]
    if not df.empty else "N/A"
)

st.divider()

# Vaccination status filter
statuses = sorted(df["vaccination_status"].unique().tolist())
selected = st.multiselect(
    "Filter Vaccination Status",
    statuses,
    default=statuses
)

filtered = df[df["vaccination_status"].isin(selected)]

if filtered.empty:
    st.warning("No data available for the selected statuses.")
    st.stop()

# Status distribution and percentage
counts = filtered["vaccination_status"].value_counts()

left, right = st.columns(2)

with left:
    st.subheader("Vaccination Status Distribution")
    fig = px.bar(
        x=counts.index,
        y=counts.values,
        labels={
            "x": "Vaccination Status",
            "y": "Number of Respondents"
        },
        text=counts.values
    )
    st.plotly_chart(fig, use_container_width=True)

with right:
    st.subheader("Vaccination Status Percentage")
    fig = px.pie(
        values=counts.values,
        names=counts.index,
        hole=0.35
    )
    st.plotly_chart(fig, use_container_width=True)

# Date-based visualizations
if "submitted_at" in filtered.columns:
    dated = filtered.dropna(subset=["submitted_at"]).copy()

    if not dated.empty:
        dated["Date"] = dated["submitted_at"].dt.date
        dated["Month"] = dated["submitted_at"].dt.strftime("%Y-%m")

        st.subheader("Daily Submission Trend")
        daily = dated.groupby("Date").size().reset_index(name="Responses")
        fig = px.line(
            daily, x="Date", y="Responses",
            markers=True
        )
        st.plotly_chart(fig, use_container_width=True)

        st.subheader("Monthly Vaccination Status Analysis")
        monthly = (
            dated.groupby(["Month", "vaccination_status"])
            .size()
            .reset_index(name="Responses")
        )
        fig = px.bar(
            monthly,
            x="Month",
            y="Responses",
            color="vaccination_status",
            barmode="group",
            labels={"vaccination_status": "Vaccination Status"}
        )
        st.plotly_chart(fig, use_container_width=True)
    else:
        st.info("No valid submission dates available.")
else:
    st.info("Date-based charts require the submitted_at column.")

# View dataset
with st.expander("View Dataset"):
    st.dataframe(filtered, use_container_width=True)

st.caption("Vaccination Status Analysis | Python & Streamlit")
