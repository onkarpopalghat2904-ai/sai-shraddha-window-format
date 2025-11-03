# sai-shraddha-window-format
Window Soft Ware# window_format_app.py
"""
Sai Shraddha Enterprises - Window Format Generator (Streamlit)

Features:
- Image upload (handwriting OCR attempt with pytesseract)
- Manual data editor to correct OCR results / enter data
- Assign categories (Normal, Louver, Kitchen, Jina, Fix)
- Calculates H2, W2, H3, W3 with your exact formulas
- Converts H3/W3 -> inches using only allowed fractions and rounds UP
- Converts H1/W1 -> feet using your custom mm->ft ranges and computes Sf
- Generates A4 PDF (bold centered shop name, Owner/Contractor/City/Mobile, table, total Sf, footer)
- Provides PDF as download button (works in browser on iPhone)

Notes:
- Expect to correct OCR results when image is handwriting.
- If pytesseract is not available on the host, OCR will be skipped but you can edit data manually.
"""

import streamlit as st
import pandas as pd
import numpy as np
import io
import math
import datetime
from PIL import Image
from reportlab.lib.pagesizes import A4
from reportlab.lib import colors
from reportlab.lib.styles import ParagraphStyle
from reportlab.lib.units import mm
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.pdfbase import pdfmetrics

# Try importing pytesseract (optional). If not available, OCR will be disabled.
try:
    import pytesseract
    PYTESSERACT_AVAILABLE = True
except Exception:
    PYTESSERACT_AVAILABLE = False

st.set_page_config(page_title="Sai Shraddha Enterprises - Window Format", layout="centered")

# ------------------------------
# App header / welcome
# ------------------------------
st.markdown("<h2 style='text-align:center;'>Welcome to <b>Sai Shraddha Enterprises</b></h2>", unsafe_allow_html=True)
st.markdown("<p style='text-align:center;'>Window Format Generator</p>", unsafe_allow_html=True)
st.write("---")

# ------------------------------
# Helper config (user rules)
# ------------------------------
ALLOWED_FRACTIONS = [1/8, 1/4, 3/8, 1/2, 5/8, 3/4, 7/8]  # allowed fractions of an inch
MM_TO_INCH = 25.4

MM_TO_FEET_MAP = [
    (100, 610, 2.0),
    (611, 760, 2.5),
    (761, 915, 3.0),
    (916, 1065, 3.5),
    (1066, 1219, 4.0),
    (1220, 1370, 4.5),
    (1371, 1524, 5.0),
    (1525, 1675, 5.5),
    (1676, 1828, 6.0),
    (1829, 1980, 6.5),
    (1981, 2130, 7.0),
    (2131, 2285, 7.5),
    (2286, 2437, 8.0),
]

def mm_to_allowed_inches_roundup(mm_value):
    """Convert mm -> inches string using allowed fractions, rounding up to next allowed fraction."""
    try:
        if mm_value is None or mm_value == '':
            return ''
        inches_total = float(mm_value) / MM_TO_INCH
    except Exception:
        return ''
    whole = int(math.floor(inches_total))
    frac = inches_total - whole
    chosen_frac = None
    for f in ALLOWED_FRACTIONS:
        if frac <= f + 1e-9:
            chosen_frac = f
            break
    if chosen_frac is None:
        whole += 1
        chosen_frac = 0.0
    if chosen_frac == 0.0:
        return f'{whole}"'
    numerator = int(round(chosen_frac * 8))
    if numerator == 8:
        return f'{whole + 1}"'
    if whole > 0:
        return f'{whole} {numerator}/8"'
    else:
        return f'{numerator}/8"'

def mm_to_feet_by_ranges(mm_value):
    """Map mm to feet using MM_TO_FEET_MAP ranges. If out of ranges, use nearest boundary mapping."""
    try:
        mmv = float(mm_value)
    except Exception:
        return MM_TO_FEET_MAP[0][2]
    for lo, hi, ft in MM_TO_FEET_MAP:
        if lo <= mmv <= hi:
            return ft
    if mmv < MM_TO_FEET_MAP[0][0]:
        return MM_TO_FEET_MAP[0][2]
    return MM_TO_FEET_MAP[-1][2]

# ------------------------------
# Core computations
# ------------------------------
def compute_rows(df):
    """Takes dataframe with columns Sr,H1,W1,Category -> returns df_with_calcs, total_sf"""
    df = df.copy()
    if 'Category' not in df.columns:
        df['Category'] = 'Normal'
    # Ensure numeric H1/W1 (integers)
    df['H1'] = pd.to_numeric(df['H1'], errors='coerce').fillna(0).astype(int)
    df['W1'] = pd.to_numeric(df['W1'], errors='coerce').fillna(0).astype(int)

    df['H2'] = ''
    df['W2'] = ''
    df['H3'] = ''
    df['W3'] = ''
    df['H3 (in)'] = ''
    df['W3 (in)'] = ''
    df['Sf'] = 0.0  # Range as Sf

    for idx, row in df.iterrows():
        cat = str(row.get('Category','')).strip().lower()
        h1 = int(row.get('H1',0))
        w1 = int(row.get('W1',0))

        if cat in ('louver','jina','fix'):
            h2 = ''
            w2 = ''
            h3 = ''
            w3 = ''
        elif cat == 'kitchen':
            h2 = round(h1 - 38)
            w2 = round((w1 - 395) / 2) if (w1 - 395) != 0 else 0
            h3 = round(h2 - 65) if h2 != '' else ''
            w3 = round(w2 + 15) if w2 != '' else ''
        else:
            h2 = round(h1 - 38)
            w2 = round((w1 - 165) / 2) if (w1 - 165) != 0 else 0
            h3 = round(h2 - 64) if h2 != '' else ''
            w3 = round(w2 + 15) if w2 != '' else ''

        df.at[idx,'H2'] = h2 if h2 != '' else ''
        df.at[idx,'W2'] = w2 if w2 != '' else ''
        df.at[idx,'H3'] = h3 if h3 != '' else ''
        df.at[idx,'W3'] = w3 if w3 != '' else ''

        df.at[idx,'H3 (in)'] = mm_to_allowed_inches_roundup(h3) if isinstance(h3,(int,float)) else ''
        df.at[idx,'W3 (in)'] = mm_to_allowed_inches_roundup(w3) if isinstance(w3,(int,float)) else ''

        h_ft = mm_to_feet_by_ranges(h1)
        w_ft = mm_to_feet_by_ranges(w1)
        sf = h_ft * w_ft
        df.at[idx,'Sf'] = round(sf,3)

    total_sf = round(df['Sf'].sum(),3)
    return df, total_sf

# ------------------------------
# OCR helpers (best-effort)
# ------------------------------
def ocr_image_to_rows(img: Image.Image):
    """Try to parse table rows from image using pytesseract (best-effort). 
       Returns list of [Sr,H1,W1] rows.
       Assumes columns: first Sr, second H1, third W1 (as user specified).
    """
    rows_parsed = []
    if not PYTESSERACT_AVAILABLE:
        return rows_parsed
    try:
        txt = pytesseract.image_to_string(img, config='--psm 6')  # psm 6 is assumption of a single uniform block of text
        for line in txt.splitlines():
            if not line.strip():
                continue
            # try to extract integers from line
            parts = []
            for token in line.strip().split():
                cleaned = ''.join(ch for ch in token if (ch.isdigit() or ch=='-' ))
                if cleaned != '' and any(c.isdigit() for c in cleaned):
                    try:
                        parts.append(int(cleaned))
                    except:
                        pass
            # If we got at least 2 numbers, try to interpret
            if len(parts) >= 3:
                # first three numbers: Sr,H1,W1
                rows_parsed.append([parts[0], parts[1], parts[2]])
            elif len(parts) == 2:
                # maybe Sr missing, or Sr/H1; we skip incomplete rows but keep them for manual editing
                rows_parsed.append(parts)
            # else ignore
    except Exception as e:
        st.warning(f"OCR error: {e}")
    return rows_parsed

# ------------------------------
# PDF generation (in-memory)
# ------------------------------
def create_pdf_bytes(owner, contractor, city, mobile, df_calc, total_sf):
    """Return PDF as bytes (A4)"""
    buffer = io.BytesIO()
    doc = SimpleDocTemplate(buffer, pagesize=A4, leftMargin=18*mm, rightMargin=18*mm, topMargin=20*mm, bottomMargin=18*mm)
    elements = []

    # Header (bold centered shop name)
    header_style = ParagraphStyle(name='Header', fontSize=16, alignment=1, leading=18)  # centered
    sub_style = ParagraphStyle(name='Sub', fontSize=11, alignment=1, leading=13)

    elements.append(Paragraph("<b>SAI SHRADDHA ENTERPRISES</b>", header_style))
    # Owner/Contractor/City/Mobile left aligned under header
    details_style = ParagraphStyle(name='Details', fontSize=10, alignment=0, leading=12)
    details_text = f"<b>Owner:</b> {st.escape(owner)} &nbsp;&nbsp; <b>Contractor:</b> {st.escape(contractor)}<br/><b>City:</b> {st.escape(city)} &nbsp;&nbsp; <b>Mobile:</b> {st.escape(mobile)}"
    elements.append(Paragraph(details_text, details_style))
    elements.append(Spacer(1,8))

    # Table header and rows
    headers = ['Sr','H1','W1','H2','W2','H3','W3','H3 (in)','W3 (in)','Sf']
    table_data = [headers]
    for _, r in df_calc.iterrows():
        row = [
            str(r.get('Sr','')),
            str(int(r.get('H1',0))) if r.get('H1','')!='' else '',
            str(int(r.get('W1',0))) if r.get('W1','')!='' else '',
            str(int(r.get('H2'))) if str(r.get('H2','')).strip()!='' else '',
            str(int(r.get('W2'))) if str(r.get('W2','')).strip()!='' else '',
            str(int(r.get('H3'))) if str(r.get('H3','')).strip()!='' else '',
            str(int(r.get('W3'))) if str(r.get('W3','')).strip()!='' else '',
            r.get('H3 (in)','') or '',
            r.get('W3 (in)','') or '',
            f"{r.get('Sf',0):.3f}"
        ]
        table_data.append(row)

    total_row = ['' for _ in headers]
    total_row[0] = 'TOTAL'
    total_row[-1] = f"{total_sf:.3f}"
    table_data.append(total_row)

    col_widths = [30*mm, 20*mm, 20*mm, 20*mm, 20*mm, 20*mm, 20*mm, 28*mm, 28*mm, 28*mm]
    table = Table(table_data, colWidths=col_widths, repeatRows=1)

    style = TableStyle([
        ('FONT', (0,0), (-1,0), 'Helvetica-Bold', 10),
        ('FONT', (0,1), (-1,-1), 'Helvetica', 9),
        ('ALIGN', (0,0), (-1,-1), 'CENTER'),
        ('VALIGN', (0,0), (-1,-1), 'MIDDLE'),
        ('GRID', (0,0), (-1,-1), 0.5, colors.black),
        ('BACKGROUND', (0,0), (-1,0), colors.whitesmoke),
    ])
    # Bold the TOTAL label / last cell
    style.add('FONT', (0, len(table_data)-1), (0, len(table_data)-1), 'Helvetica-Bold', 10)
    style.add('FONT', (len(headers)-1, len(table_data)-1), (len(headers)-1, len(table_data)-1), 'Helvetica-Bold', 10)

    table.setStyle(style)
    elements.append(table)

    # Footer line
    elements.append(Spacer(1,10))
    footer_style = ParagraphStyle(name='Footer', fontSize=9, alignment=1)
    elements.append(Paragraph("Generated by Sai Shraddha Enterprises Window Format App", footer_style))

    doc.build(elements)
    pdf_bytes = buffer.getvalue()
    buffer.close()
    return pdf_bytes

# ------------------------------
# UI: Upload, OCR, Edit, Assign, Generate
# ------------------------------
st.info("Upload an image of your measurement table (first column Sr, second H1 in mm, third W1 in mm). After upload, correct values in the editor if needed, then press 'Generate PDF'.")

col1, col2 = st.columns([2,1])
with col1:
    uploaded_file = st.file_uploader("Upload image (.jpg/.png) — handwriting OK", type=['png','jpg','jpeg','tiff','bmp'])
with col2:
    upload_excel = st.file_uploader("Or upload Excel (.xlsx) if available (optional)", type=['xlsx','csv'])

# Manual inputs
st.write("Enter project details (these will appear in the PDF header):")
owner = st.text_input("Owner Name", value="")
contractor = st.text_input("Contractor", value="")
city = st.text_input("City", value="")
mobile = st.text_input("Mobile Number", value="")

# Dataframe placeholder
df_initial = pd.DataFrame(columns=['Sr','H1','W1','Category'])

# Try to get data from uploaded Excel first (if provided)
if upload_excel is not None:
    try:
        if str(upload_excel.name).lower().endswith('.csv'):
            dfx = pd.read_csv(upload_excel)
        else:
            dfx = pd.read_excel(upload_excel, engine='openpyxl')
        # Map first three columns to Sr,H1,W1 if column names unknown
        cols = list(dfx.columns)
        if len(cols) >= 3:
            df_initial['Sr'] = dfx[cols[0]]
            df_initial['H1'] = dfx[cols[1]]
            df_initial['W1'] = dfx[cols[2]]
        else:
            st.warning("Uploaded Excel has fewer than 3 columns; ensure it contains Sr,H1,W1.")
    except Exception as e:
        st.warning(f"Failed to read Excel: {e}")

# If image uploaded, attempt OCR
ocr_rows = []
if uploaded_file is not None:
    try:
        image = Image.open(uploaded_file).convert('RGB')
        st.image(image, caption="Uploaded image (preview)", use_column_width=True)
        if PYTESSERACT_AVAILABLE:
            with st.spinner("Running OCR on image (best-effort for handwriting)..."):
                rows = ocr_image_to_rows(image)
                if rows:
                    # Build DataFrame: accept rows with 3 entries; incomplete rows will be left for manual edit
                    for r in rows:
                        if len(r) >= 3:
                            df_initial = df_initial.append({'Sr': r[0], 'H1': r[1], 'W1': r[2], 'Category':'Normal'}, ignore_index=True)
                    if df_initial.empty:
                        st.warning("OCR found numbers but could not form Sr,H1,W1 rows reliably. Please correct manually in editor.")
                else:
                    st.warning("OCR couldn't detect rows. Handwritten OCR can be unreliable. Please enter/edit rows below manually.")
        else:
            st.warning("Image OCR (pytesseract) not available on this host. Use manual editing below.")
    except Exception as e:
        st.warning(f"Error reading image: {e}")

# If no uploaded content produced initial rows, show an empty template
if df_initial.empty:
    # Provide a small template with a few empty rows so user can add
    df_initial = pd.DataFrame([
        {'Sr':'','H1':'','W1':'','Category':'Normal'},
        {'Sr':'','H1':'','W1':'','Category':'Normal'},
        {'Sr':'','H1':'','W1':'','Category':'Normal'}
    ])

st.markdown("### Verify or edit the table (first col = Sr, second = H1 mm, third = W1 mm). Set Category if needed.")
# Use Streamlit's experimental data editor so user can edit cells
# Use st.data_editor if available; else fallback to st.text_area CSV input
try:
    edited = st.data_editor(df_initial, num_rows="dynamic")
    df_user = edited.copy()
except Exception:
    st.warning("Editable table interface not available in this Streamlit version. Please paste CSV below (Sr,H1,W1,Category).")
    csv_text = st.text_area("Or paste CSV (Sr,H1,W1,Category) here:")
    if csv_text.strip():
        df_user = pd.read_csv(io.StringIO(csv_text))
    else:
        df_user = df_initial.copy()

# Validate df_user columns
expected_cols = ['Sr','H1','W1','Category']
for c in expected_cols:
    if c not in df_user.columns:
        df_user[c] = ''  # add missing columns

# Recompute with formulas
df_calc, total_sf = compute_rows(df_user)

st.markdown("### Preview of calculated values")
st.dataframe(df_calc)

st.write(f"**Total Sf (sum):** {total_sf:.3f}")

# Category confirmation reminder (final rule)
st.info("Before generating PDF: confirm which Sr numbers are Louver / Kitchen / Jina / Fix by editing the 'Category' column in the table above. These categories affect H2/W2/H3/W3 formulas.")

# Button to generate PDF
if st.button("Generate PDF"):
    # final compute to ensure freshest values
    df_final, total_sf_final = compute_rows(df_user)
    # Ensure Owner name entered manually (user wanted manual entry)
    if not owner.strip():
        st.warning("Please enter Owner Name (manual entry) before generating PDF.")
    else:
        try:
            pdf_bytes = create_pdf_bytes(owner.strip(), contractor.strip(), city.strip(), mobile.strip(), df_final, total_sf_final)
            now = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
            fname = f"WindowFormat_{owner.strip().replace(' ','_')}_{now}.pdf"
            st.success("PDF generated — tap the button below to download.")
            st.download_button("Download PDF", data=pdf_bytes, file_name=fname, mime="application/pdf")
        except Exception as e:
            st.error(f"PDF generation failed: {e}")

# Footer note about OCR availability
if not PYTESSERACT_AVAILABLE:
    st.info("Note: OCR (pytesseract) not available on this host. The app still works — use the editor to enter or paste rows manually.")
