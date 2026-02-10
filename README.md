# RECONCILIATION GURU - Google Colab Demo
# ==============================

# 1️⃣ Install dependencies
!pip install streamlit pandas pdfplumber openai pyngrok --quiet

# 2️⃣ Import libraries
import streamlit as st
import pandas as pd
import pdfplumber
import openai
import os
from pyngrok import ngrok

# 3️⃣ Set your OpenAI API key
# Replace 'YOUR_OPENAI_API_KEY' with your actual key
os.environ['OPENAI_API_KEY'] = 'YOUR_OPENAI_API_KEY'
openai.api_key = os.getenv("OPENAI_API_KEY")

# 4️⃣ Save Streamlit demo code
streamlit_code = """
import streamlit as st
import pandas as pd
import pdfplumber
import openai
import os

st.set_page_config(page_title="RECONCILIATION GURU Demo", layout="wide")
st.title("RECONCILIATION GURU - AI Accounting & Reconciliation Dashboard")

# Upload multiple files
st.sidebar.header("Upload Files")
uploaded_files = st.sidebar.file_uploader(
    "Upload Excel, CSV, or PDF (multiple allowed)", 
    type=["xlsx", "csv", "pdf"], 
    accept_multiple_files=True
)

all_data = pd.DataFrame()

for uploaded_file in uploaded_files:
    file_type = uploaded_file.name.split(".")[-1]
    if file_type in ["xlsx", "csv"]:
        if file_type == "xlsx":
            df = pd.read_excel(uploaded_file)
        else:
            df = pd.read_csv(uploaded_file)
        df['SourceFile'] = uploaded_file.name
        all_data = pd.concat([all_data, df], ignore_index=True)
    elif file_type == "pdf":
        st.sidebar.info(f"Parsing {uploaded_file.name}")
        text = ""
        with pdfplumber.open(uploaded_file) as pdf:
            for page in pdf.pages:
                text += page.extract_text() + "\\n"
        prompt = f\"\"\"Extract financial transaction data from the text below.
        Return a CSV table with columns: Date, Vendor, Amount, Description.
        Text:
        {text}\"\"\"
        try:
            response = openai.ChatCompletion.create(
                model="gpt-4",
                messages=[{"role": "user", "content": prompt}],
                max_tokens=1000
            )
            output_text = response['choices'][0]['message']['content']
            from io import StringIO
            try:
                df = pd.read_csv(StringIO(output_text))
                df['SourceFile'] = uploaded_file.name
                all_data = pd.concat([all_data, df], ignore_index=True)
            except:
                st.sidebar.warning(f"AI could not parse {uploaded_file.name}. Showing raw text.")
                st.sidebar.text(output_text)
        except Exception as e:
            st.sidebar.error(f"AI parsing failed for {uploaded_file.name}: {e}")

if not all_data.empty:
    st.subheader("All Transactions")
    if 'Amount' in all_data.columns:
        all_data['Matched'] = all_data['Amount'].apply(lambda x: True if pd.notna(x) and x % 2 == 0 else False)
    def highlight_rows(row):
        if 'Matched' in row and row['Matched']:
            return ['background-color: #d4edda']*len(row)
        elif 'Matched' in row:
            return ['background-color: #f8d7da']*len(row)
        else:
            return ['']*len(row)
    st.dataframe(all_data.style.apply(highlight_rows, axis=1))
    # Filters
    st.sidebar.header("Filters")
    if 'Vendor' in all_data.columns:
        vendor_filter = st.sidebar.multiselect("Filter by Vendor", options=all_data['Vendor'].unique())
        if vendor_filter:
            all_data = all_data[all_data['Vendor'].isin(vendor_filter)]
    if 'Matched' in all_data.columns:
        match_filter = st.sidebar.radio("Show:", ["All", "Matched Only", "Unmatched Only"])
        if match_filter == "Matched Only":
            all_data = all_data[all_data['Matched'] == True]
        elif match_filter == "Unmatched Only":
            all_data = all_data[all_data['Matched'] == False]
    # Dashboard Metrics
    st.subheader("Dashboard Metrics")
    col1, col2, col3 = st.columns(3)
    col1.metric("Total Transactions", len(all_data))
    if 'Amount' in all_data.columns:
        col2.metric("Total Amount", f"R{all_data['Amount'].sum():,.2f}")
        col3.metric("Matched Transactions", all_data['Matched'].sum())
    # Export
    csv = all_data.to_csv(index=False).encode('utf-8')
    st.download_button(
        label="Export Dashboard Data to CSV",
        data=csv,
        file_name='reconciled_dashboard.csv',
        mime='text/csv'
    )
else:
    st.info("Upload files to see AI-extracted transactions and reconciliation dashboard.")
"""

with open("demo_reconciliation_guru.py", "w") as f:
    f.write(streamlit_code)

# 5️⃣ Expose Streamlit app via ngrok
public_url = ngrok.connect(port='8501')
print(f"Your RECONCILIATION GURU demo is live at: {public_url}")

# 6️⃣ Run Streamlit in background
get_ipython().system_raw("streamlit run demo_reconciliation_guru.py &")
