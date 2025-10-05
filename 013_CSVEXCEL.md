

# Working with CSV and Excel Files in Python — updated Oct 5, 2025

Beginner‑friendly guide to reading, writing, and transforming **CSV**, **XLS**, **XLSX**, and **XLSB** files with Python. Covers the standard library `csv` module, the **pandas** toolbox, Excel engines (**openpyxl**, **xlsxwriter**, **pyxlsb**), encodings, dates, dtypes, memory‑safe patterns, formatting, and troubleshooting. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13 in a venv.  
- Small examples use the stdlib; bigger tasks use pandas.  
- Paths shown are relative to your project root.

---

## Table of Contents
- [0) Install packages](#0-install-packages)
- [1) CSV with the standard library](#1-csv-with-the-standard-library)
- [2) CSV with pandas](#2-csv-with-pandas)
- [3) Excel with pandas (XLSX/XLS)](#3-excel-with-pandas-xlsxxls)
- [4) Writing Excel with formatting (xlsxwriter)](#4-writing-excel-with-formatting-xlsxwriter)
- [5) Reading XLSB and very old XLS](#5-reading-xlsb-and-very-old-xls)
- [6) Dates, timezones, and types](#6-dates-timezones-and-types)
- [7) Encodings and delimiters](#7-encodings-and-delimiters)
- [8) Large files: chunks, memory, and speed](#8-large-files-chunks-memory-and-speed)
- [9) Common transforms](#9-common-transforms)
- [10) Validation and data cleaning](#10-validation-and-data-cleaning)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Install packages
```sh
pip install pandas openpyxl xlsxwriter pyxlsb pyarrow python-dateutil
# optional: older xls support, encrypted files, fast csv
pip install "xlrd<2.0" msoffcrypto-tool
```
Notes:
- **pandas** handles CSV/Excel I/O and dataframe transforms.  
- **openpyxl** reads/writes **.xlsx**; **xlsxwriter** writes **.xlsx** with rich formatting; **pyxlsb** reads **.xlsb**; **xlrd<2.0** reads legacy **.xls** only.  
- **pyarrow** accelerates CSV and Parquet I/O in recent pandas.

---

## 1) CSV with the standard library
**What**: Minimal dependencies; great for tiny scripts or strict control of quoting and dialects.

Read rows:
```python
import csv
from pathlib import Path

with Path('data.csv').open(newline='', encoding='utf-8') as f:
    reader = csv.reader(f)  # list-of-lists
    header = next(reader)
    for row in reader:
        print(row)
```
Dict style:
```python
with open('data.csv', newline='', encoding='utf-8') as f:
    reader = csv.DictReader(f)  # maps header→value
    for rec in reader:
        print(rec['name'], rec.get('price'))
```
Write:
```python
rows = [
    {"name": "Widget", "price": 9.99},
    {"name": "Gadget", "price": 12.5},
]
with open('out.csv', 'w', newline='', encoding='utf-8') as f:
    w = csv.DictWriter(f, fieldnames=['name','price'])
    w.writeheader(); w.writerows(rows)
```
Dialects, quoting, delimiter:
```python
csv.reader(f, delimiter=';', quotechar='"', quoting=csv.QUOTE_MINIMAL)
```
Auto‑sniff:
```python
sample = Path('mystery.csv').read_text(1024, encoding='utf-8', errors='ignore')
dialect = csv.Sniffer().sniff(sample)
```

---

## 2) CSV with pandas
**What**: Fast parsing, types, filtering, groupby, joins.
```python
import pandas as pd

# basic
df = pd.read_csv('data.csv')
print(df.head())

# control dtypes and dates
df = pd.read_csv(
    'data.csv',
    dtype={'id': 'Int64', 'price': 'float64'},
    parse_dates=['sold_at'],
    na_values=['', 'NA', 'null'],
    encoding='utf-8'
)

# write
df.to_csv('out.csv', index=False)
```
Select columns and filter:
```python
small = df[['id','name','price']].query('price > 10')
small.to_csv('premium.csv', index=False)
```

---

## 3) Excel with pandas (XLSX/XLS)
**What**: Read/write Excel with sheet selection and dtype control.
```python
import pandas as pd

# read specific sheet by name or index
sales = pd.read_excel('book.xlsx', sheet_name='Sales')
first = pd.read_excel('book.xlsx', sheet_name=0)

# choose engine explicitly if needed
sales = pd.read_excel('book.xlsx', engine='openpyxl')

# write multiple sheets
with pd.ExcelWriter('report.xlsx', engine='xlsxwriter') as xw:
    sales.to_excel(xw, sheet_name='Sales', index=False)
    summary = sales.groupby('Region', as_index=False)['Amount'].sum()
    summary.to_excel(xw, sheet_name='Summary', index=False)
```
Sheet names:
```python
xls = pd.ExcelFile('book.xlsx')
print(xls.sheet_names)
```

---

## 4) Writing Excel with formatting (xlsxwriter)
**What**: Add number formats, column widths, autofilter, freeze panes, and formulas.
```python
import pandas as pd

with pd.ExcelWriter('styled.xlsx', engine='xlsxwriter') as xw:
    df = pd.DataFrame({
        'Date': pd.date_range('2025-01-01', periods=7),
        'Sales': [100,120,90,130,95,110,150],
        'Region': ['AU','NZ','US','AU','US','NZ','AU']
    })
    df.to_excel(xw, sheet_name='Data', index=False)

    wb  = xw.book
    ws  = xw.sheets['Data']
    money = wb.add_format({'num_format': '$#,##0'})
    datef = wb.add_format({'num_format': 'yyyy-mm-dd'})

    ws.set_column('A:A', 12, datef)
    ws.set_column('B:B', 10, money)
    ws.set_column('C:C', 10)
    ws.autofilter(0, 0, len(df), len(df.columns)-1)
    ws.freeze_panes(1, 0)  # freeze header

    # simple formula total
    ws.write(len(df)+1, 0, 'Total')
    ws.write_formula(len(df)+1, 1, f'=SUM(B2:B{len(df)+1})', money)
```

---

## 5) Reading XLSB and very old XLS
- **XLSB**: binary Excel. Use `pyxlsb` reader.
```python
import pandas as pd
from pyxlsb import open_workbook

# pandas helper
df = pd.read_excel('data.xlsb', engine='pyxlsb')
```
- **Legacy XLS (BIFF)**: modern `xlrd` (v2+) dropped .xls support. Install an older version only if needed:
```sh
pip install "xlrd<2.0"  # security/maintenance caveats
```
Alternative: save as .xlsx if possible, or use LibreOffice headless conversion.

---

## 6) Dates, timezones, and types
Excel stores dates as serial numbers; CSV stores text. Normalize on read.
```python
import pandas as pd

# parse and localise
orders = pd.read_csv('orders.csv', parse_dates=['created_at'])
orders['created_at'] = orders['created_at'].dt.tz_localize('Australia/Melbourne')

# ensure dtypes
orders = orders.astype({'order_id':'Int64', 'amount':'float64'})

# Excel date parsing
x = pd.read_excel('book.xlsx', parse_dates=['Date'])
```
Tip: Avoid implicit dtype changes; specify `dtype=` or convert explicitly. Use `errors="coerce"` with `to_numeric`/`to_datetime` when cleaning.

---

## 7) Encodings and delimiters
```python
# UTF‑8 with BOM sometimes appears from Excel → handle explicitly
pd.read_csv('export.csv', encoding='utf-8-sig')

# Windows CSV often uses cp1252
pd.read_csv('export.csv', encoding='cp1252', sep=';')

# Robust stdlib open for unknowns
open('file.csv', encoding='utf-8', errors='replace')
```
Use the stdlib `csv.Sniffer()` to detect delimiter/quote if unknown.

---

## 8) Large files: chunks, memory, and speed
Read in chunks to limit RAM, or use Arrow for speed.
```python
import pandas as pd

# stream in 100k‑row chunks
it = pd.read_csv('big.csv', chunksize=100_000)
for chunk in it:
    # process each chunk
    pass

# Arrow engine (pandas>=2) can speed up I/O
pd.read_csv('big.csv', engine='pyarrow')
```
Tips:
- Select only needed columns with `usecols=`.  
- Use `dtype=` to avoid object fallbacks.  
- Convert to **Parquet** for analytics workloads: `df.to_parquet('data.parquet')`.

---

## 9) Common transforms
```python
import pandas as pd

sales = pd.read_csv('sales.csv', parse_dates=['date'])

# filter and compute
au = sales.query("region == 'AU' & sales > 100")
au['week'] = au['date'].dt.to_period('W').apply(lambda r: r.start_time)
weekly = au.groupby('week', as_index=False)['sales'].sum()

# join lookup
regions = pd.read_csv('regions.csv')
merged = sales.merge(regions, on='region_id', how='left')

# pivot
pivot = sales.pivot_table(index='date', columns='region', values='sales', aggfunc='sum').fillna(0)

# write back
weekly.to_csv('weekly.csv', index=False)
pivot.to_excel('pivot.xlsx', index=True)
```

---

## 10) Validation and data cleaning
```python
import pandas as pd
from pandas.api.types import is_numeric_dtype

 df = pd.read_csv('data.csv')
 # required columns
required = {'id','name','price'}
missing = required - set(df.columns)
if missing:
    raise ValueError(f"Missing columns: {missing}")

# strip whitespace, normalize case
for c in ['name']:
    df[c] = df[c].astype(str).str.strip()

# numeric conversion with coercion
df['price'] = pd.to_numeric(df['price'], errors='coerce')
if not is_numeric_dtype(df['price']):
    raise TypeError('price must be numeric')

# drop impossible rows
df = df[df['price'].between(0, 1_000_000, inclusive='both')]
```

---

## 11) Troubleshooting
- **`ValueError: Excel file format cannot be determined`**: specify `engine=` or install the required engine (`openpyxl`, `pyxlsb`, `xlrd<2`).  
- **Garbled characters**: wrong encoding; try `encoding='utf-8-sig'` or `cp1252`.  
- **Numbers read as text**: pass `dtype=` or convert with `to_numeric(errors='coerce')`.  
- **Dates off by 4 years in Excel**: check 1900/1904 date system; export to ISO dates or force `parse_dates=`.  
- **MemoryError on big CSV**: use `chunksize`, `usecols`, and Arrow engine.
- **Formulas shown as text in Excel**: write with `xlsxwriter` and use `write_formula` or ensure the cell starts with `=`.
- **Encrypted Excel**: use `msoffcrypto-tool` to decrypt to a temp file, then read with pandas.

---

## 12) Recap
```plaintext
Small CSV → use csv module → DictReader/Writer
Data wrangling → pandas read_csv/read_excel → transform → to_csv/to_excel
Excel formatting → xlsxwriter via ExcelWriter
Legacy or special formats → xlrd<2 for .xls, pyxlsb for .xlsb
Large data → chunksize, usecols, Arrow → consider Parquet
```

**Next**: Add logging, command‑line args (`argparse`) to batch convert files, and unit tests for your transforms. For BI dashboards, export tidy CSV/Parquet and visualize with the tools in `011_GRAPHS.md`. 