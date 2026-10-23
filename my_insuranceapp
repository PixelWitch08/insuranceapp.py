import streamlit as st
import pandas as pd
import jdatetime
import sqlite3
from datetime import timedelta

# --- تنظیمات دیتابیس (جایگزین اکسل برای پایداری در ابر) ---
def create_connection():
    conn = sqlite3.connect('insurance_data.db', check_same_thread=False)
    return conn

def init_db():
    conn = create_connection()
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS policies (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            b_no TEXT, c_name TEXT, c_nid TEXT, p_type TEXT,
            total_amt REAL, advance_amt REAL, total_inst INTEGER,
            paid_inst INTEGER, inst_amt REAL, is_supplementary TEXT,
            due_date TEXT, status TEXT
        )
    ''')
    conn.commit()
    conn.close()

init_db()

# --- توابع کمکی داده‌ها ---
def load_data_from_db():
    conn = create_connection()
    df = pd.read_sql_query("SELECT * FROM policies", conn)
    conn.close()
    if df.empty:
        return pd.DataFrame(columns=["id", "b_no", "c_name", "c_nid", "p_type", "total_amt", "advance_amt", "total_inst", "paid_inst", "inst_amt", "is_supplementary", "due_date", "status"])
    return df

def save_to_db(b_no, c_name, c_nid, p_type, total_amt, advance_amt, total_inst, paid_inst, inst_amt, is_supp, due_date, status):
    conn = create_connection()
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO policies (b_no, c_name, c_nid, p_type, total_amt, advance_amt, total_inst, paid_inst, inst_amt, is_supplementary, due_date, status)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    ''', (b_no, c_name, c_nid, p_type, total_amt, advance_amt, total_inst, paid_inst, inst_amt, is_supp, due_date, status))
    conn.commit()
    conn.close()

def delete_from_db(policy_id):
    conn = create_connection()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM policies WHERE id=?", (policy_id,))
    conn.commit()
    conn.close()

def update_paid_inst(policy_id, new_paid):
    conn = create_connection()
    cursor = conn.cursor()
    cursor.execute("UPDATE policies SET paid_inst=? WHERE id=?", (new_paid, policy_id))
    conn.commit()
    conn.close()

# --- تنظیمات صفحه و تم (همان تم قبلی) ---
st.set_page_config(page_title="مدیریت بیمه - نسخه ابری", page_icon="🛡️", layout="wide")

def apply_soft_light_theme():
    st.markdown("""
    <style>
    .stApp { background-color: #f4f7f9; color: #334155; }
    h1, h2, h3 { color: #1e293b !important; }
    div[data-testid="stForm"] { background-color: #ffffff !important; border-radius: 12px !important; border: 1px solid #e2e8f0 !important; padding: 20px !important; }
    .stButton>button { background: linear-gradient(90deg, #3b82f6 0%, #2563eb 100%) !important; color: white !important; border-radius: 8px !important; }
    </style>
    """, unsafe_allow_html=True)

apply_soft_light_theme()

# --- منطق اصلی برنامه ---
def calculate_status_logic(row):
    if row['paid_inst'] >= row['total_inst']:
        return "تسویه کامل"
    try:
        date_parts = list(map(int, str(row['due_date']).split('/')))
        due_j_date = jdatetime.date(date_parts[0], date_parts[1], date_parts[2])
        if due_j_date < jdatetime.date.today():
            return "⚠️ معوق (دیرکرد)"
    except: pass
    return "در جریان پرداخت"

# لود کردن داده‌ها در Session
if "df" not in st.session_state:
    st.session_state.df = load_data_from_db()

# --- بخش نمایش هشدارها ---
st.subheader("🚨 وضعیت معوقات و هشدارها")
if not st.session_state.df.empty:
    st.session_state.df["وضعیت"] = st.session_state.df.apply(calculate_status_logic, axis=1)
    alerts = st.session_state.df[st.session_state.df["status"] == "⚠️ معوق (دیرکرد)"]
    if not alerts.empty:
        for _, row in alerts.iterrows():
            st.error(f"🔴 هشدار: مشتری `{row['c_name']}` | شماره بیمه: `{row['b_no']}`")
    else:
        st.success("✅ وضعیت همه بیمه‌ها نرمال است.")
else:
    st.info("داده‌ای وجود ندارد.")

st.markdown("---")

# --- فرم ثبت ---
with st.form("add_form", clear_on_submit=True):
    st.subheader("➕ ثبت بیمه‌نامه جدید")
    col1, col2, col3 = st.columns(3)
    with col1:
        b_no = st.text_input("شماره بیمه‌نامه")
        c_name = st.text_input("نام مشتری")
        c_nid = st.text_input("کد ملی")
        p_type = st.selectbox("نوع پرداخت", ["نقدی (کامل)", "شرایطی (اقساطی)", "نمایندگان بیمه", "کارکنان آموزش و پرورش"])
    with col2:
        base_amount = st.number_input("مبلغ پایه (ریال)", min_value=0, step=100000)
        has_supp = st.checkbox("بیمه مکمل؟ (+۲,۰۰۰,۰۰۰ ریال)")
        total_amt = base_amount + (2000000 if has_supp else 0)
        st.info(f"مبلغ کل: {total_amt:,} ریال")
        advance_amt = st.number_input("پیش‌پرداخت (ریال)", min_value=0)
    with col3:
        total_inst = st.number_input("تعداد اقساط", min_value=1, value=1)
        paid_inst = st.number_input("اقساط پرداخت شده", min_value=0, value=0)
        inst_amt = st.number_input("مبلغ هر قسط (ریال)", min_value=0)
        due_date_str = (jdatetime.date.today() + timedelta(days=40)).strftime("%Y/%m/%d")
        st.caption(f"تاریخ سررسید اول: {due_date_str}")

    if st.form_submit_button("ثبت اطلاعات"):
        if b_no and c_name:
            save_to_db(b_no, c_name, c_nid, p_type, total_amt, advance_amt, total_inst, paid_inst, inst_amt, "دارد" if has_supp else "ندارد", due_date_str, "در جریان پرداخت")
            st.session_state.df = load_data_from_db()
            st.success("ثبت شد!")
            st.rerun()
        else:
            st.error("نام و شماره بیمه اجباری است.")

# --- نمایش و جستجو ---
st.subheader("📋 لیست بیمه‌نامه‌ها")
search = st.text_input("🔍 جستجو...")
display_df = st.session_state.df.copy()
if search:
    q = search.lower()
    display_df = display_df[display_df.astype(str).apply(lambda x: x.str.contains(q, case=False)).any(axis=1)]

if not display_df.empty:
    # نمایش با فرمت بهتر
    view_df = display_df.copy()
    # تبدیل ستون‌های عددی به فرمت با کاما برای نمایش
    for col in ["total_amt", "advance_amt", "inst_amt"]:
        view_df[col] = view_df[col].apply(lambda x: f"{int(x):,}")
    st.dataframe(view_df, use_container_width=True)
else:
    st.warning("موردی یافت نشد.")

# --- مدیریت ویرایش و حذف ---
if not st.session_state.df.empty:
    st.markdown("---")
    st.subheader("⚙️ مدیریت")
    ec1, ec2 = st.columns(2)
    with ec1:
        eid = st.selectbox("انتخاب شناسه برای ویرایش اقساط:", st.session_state.df["id"].tolist())
        new_p = st.number_input("تعداد اقساط پرداخت شده جدید:", min_value=0, max_value=12, value=0)
        if st.button("🔄 به‌روزرسانی"):
            update_paid_inst(eid, new_p)
            st.session_state.df = load_data_from_db()
            st.rerun()
    with ec2:
        if st.button("🗑️ حذف پرونده", type="primary"):
            delete_from_db(eid)
            st.session_state.df = load_data_from_db()
            st.rerun()
