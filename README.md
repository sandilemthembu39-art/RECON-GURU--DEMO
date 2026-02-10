%%writefile app.py
import streamlit as st
import pandas as pd
import pdfplumber
import openai
import os

st.set_page_config(page_title="RECONCILIATION GURU", layout="wide")
st.title("RECONCILIATION GURU – AI Reconciliation Dashboard")

openai.api_key = os.getenv("OPENAI_API_KEY")

st.sidebar.header("Upload Files")
uploaded_files = st.sidebar.file_uploader(
    "Upload Excel, CSV, or PDF",
    type=["xlsx", "csv", "pdf"],
    accept_multiple_files=True
)

all_data = pd.DataFrame()

for uploaded_file in uploaded_files:
    ext = uploaded_file.name.split(".")[-1]
    if ext in ["xlsx", "csv"]:
        df = pd.read_excel(uploaded_file) if ext == "xlsx" else pd.read_csv(uploaded_file)
        df["SourceFile"] = uploaded_file.name
        all_data = pd.concat([all_data, df])
    elif ext == "pdf":
        text = ""
        with pdfplumber.open(uploaded_file) as pdf:
            for page in pdf.pages:
                text += page.extract_text() or ""

        prompt = f"""
        Extract transactions and return CSV with:
        Date, Vendor, Amount, Description
        Text:
        {text}
        """

        try:
            response = openai.ChatCompletion.create(
                model="gpt-4",
                messages=[{"role":"user","content":prompt}],
                max_tokens=800
            )
            from io import StringIO
            df = pd.read_csv(StringIO(response.choices[0].message.content))
            df["SourceFile"] = uploaded_file.name
            all_data = pd.concat([all_data, df])
        except Exception as e:
            st.error(e)

if not all_data.empty:
    if "Amount" in all_data.columns:
        all_data["Matched"] = all_data["Amount"] % 2 == 0

    st.dataframe(all_data)

    csv = all_data.to_csv(index=False).encode("utf-8")
    st.download_button("Download CSV", csv, "reconciliation_guru.csv")
else:
    st.info("Upload files to begin")
