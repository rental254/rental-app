import streamlit as st
import json
import os
import uuid
from datetime import datetime
from fpdf import FPDF
import base64

st.set_page_config(page_title="Rental Payments Portal", page_icon="🏠", layout="wide")

# GREEN & WHITE THEME
st.markdown("""
<style>
    .stApp {background-color: #FFFFFF;}
    h1, h2, h3 {color: #0A7F2E;}
    .stButton>button {background-color: #0A7F2E; color: white; border-radius: 8px; border: none;}
    .stButton>button:hover {background-color: #0C9E39;}
    .metric-box {background-color: #E8F5E9; padding: 15px; border-radius: 10px; border-left: 5px solid #0A7F2E;}
    .receipt {background-color: #F1F8F4; padding: 20px; border-radius: 10px; border: 2px dashed #0A7F2E;}
</style>
""", unsafe_allow_html=True)

# Load data
def load_data(filename):
    if not os.path.exists(filename):
        return []
    with open(filename, "r") as f:
        return json.load(f)

def save_data(filename, data):
    with open(filename, "w") as f:
        json.dump(data, f, indent=2)

tenants = load_data("tenants.json")
payments = load_data("payments.json")

if "user" not in st.session_state:
    st.session_state.user = None

# PDF RECEIPT GENERATOR
def create_pdf_receipt(payment):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", "B", 16)
    pdf.set_text_color(10, 127, 46)
    pdf.cell(200, 10, "RENT PAYMENT RECEIPT", ln=True, align="C")
    pdf.ln(5)
    pdf.set_draw_color(10, 127, 46)
    pdf.line(10, 25, 200, 25)
    pdf.ln(5)
    
    pdf.set_font("Arial", "", 12)
    pdf.set_text_color(0,0,0)
    pdf.cell(100, 10, f"Receipt No: {payment['id']}", ln=True)
    pdf.cell(100, 10, f"Date of Payment: {payment['date']}", ln=True)
    pdf.cell(100, 10, f"Tenant Name: {payment['tenant']}", ln=True)
    pdf.cell(100, 10, f"Room Type: {payment.get('room_type', 'N/A')}", ln=True)
    pdf.cell(100, 10, f"Room Number: {payment['room']}", ln=True)
    pdf.cell(100, 10, f"Amount Paid: KES {payment['amount']:,}", ln=True)
    pdf.cell(100, 10, f"Method of Payment: {payment['method']}", ln=True)
    pdf.cell(100, 10, f"Phone: {payment['phone']}", ln=True)
    pdf.ln(10)
    pdf.cell(200, 10, "Thank you for your payment!", ln=True, align="C")
    
    pdf_file = f"receipt_{payment['id']}.pdf"
    pdf.output(pdf_file)
    return pdf_file

def get_pdf_download_link(pdf_file):
    with open(pdf_file, "rb") as f:
        data = f.read()
    b64 = base64.b64encode(data).decode()
    return f'<a href="data:application/pdf;base64,{b64}" download="{pdf_file}">📄 Download Receipt PDF</a>'

# LOGIN
def login():
    st.title("🏠 Rental Payments Portal")
    col1, col2, col3 = st.columns([1,2,1])
    with col2:
        st.subheader("Login")
        username = st.text_input("Username")
        password = st.text_input("Password", type="password")
        if st.button("Login", use_container_width=True):
            if username == "admin" and password == "admin123":
                st.session_state.user = {"role": "admin", "name": "Landlord"}
                st.rerun()
            for t in tenants:
                if t["username"] == username and t["password"] == password:
                    st.session_state.user = t
                    st.rerun()
            st.error("Invalid username or password")

# TENANT REGISTRATION FORM
def tenant_registration():
    st.subheader("📝 New Tenant Registration")
    with st.form("tenant_form"):
        name = st.text_input("Full Name")
        phone = st.text_input("Phone Number")
        room_type = st.selectbox("Room Type", ["Single", "Bedsitter", "1 Bedroom", "2 Bedroom"])
        room = st.text_input("Room Number")
        rent = st.number_input("Monthly Rent KES", min_value=1000, step=500)
        username = st.text_input("Choose Username")
        password = st.text_input("Choose Password", type="password")
        submitted = st.form_submit_button("Register Tenant")
        if submitted:
            new_tenant = {
                "username": username,
                "password": password,
                "name": name,
                "phone": phone,
                "room_type": room_type,
                "room": room,
                "rent": rent,
                "balance": rent,
                "due_date": "2026-08-05"
            }
            tenants.append(new_tenant)
            save_data("tenants.json", tenants)
            st.success(f"Tenant {name} registered successfully!")

# PAYMENT + RECEIPT
def make_payment(tenant):
    st.subheader("💳 Pay Rent")
    amount = st.number_input("Amount KES", min_value=100, value=tenant["balance"])
    method = st.radio("Payment Method", ["M-PESA", "Bank Transfer"])
    
    if method == "M-PESA":
        phone = st.text_input("M-PESA Phone", value=tenant["phone"])
    else:
        phone = st.text_input("Bank Reference / Phone", value=tenant["phone"])
        st.info("Bank Details: Account: 12345678 | Bank: KCB | Name: Rental Properties Ltd")

    if st.button("Pay Now", use_container_width=True):
        receipt_id = str(uuid.uuid4())[:8].upper()
        payment = {
            "id": receipt_id,
            "tenant": tenant["name"],
            "room_type": tenant["room_type"],
            "room": tenant["room"],
            "amount": amount,
            "phone": phone,
            "method": method,
            "date": datetime.now().strftime("%Y-%m-%d %H:%M")
        }
        payments.append(payment)
        save_data("payments.json", payments)
        
        tenant["balance"] -= amount
        save_data("tenants.json", tenants)
        
        st.success("Payment Recorded Successfully!")
        show_receipt(payment)

def show_receipt(payment):
    st.markdown(f"""
    <div class="receipt">
    <h3 style="text-align:center; color:#0A7F2E;">RENT PAYMENT RECEIPT</h3>
    <hr>
    <p><b>Receipt No:</b> {payment['id']}</p>
    <p><b>Date of Payment:</b> {payment['date']}</p>
    <p><b>Tenant Name:</b> {payment['tenant']}</p>
    <p><b>Room Type:</b> {payment.get('room_type', 'N/A')}</p>
    <p><b>Room Number:</b> {payment['room']}</p>
    <p><b>Amount Paid:</b> KES {payment['amount']:,}</p>
    <p><b>Method of Payment:</b> {payment['method']}</p>
    <p><b>Phone/Ref:</b> {payment['phone']}</p>
    <hr>
    <p style="text-align:center;">Thank you for your payment!</p>
    </div>
    """, unsafe_allow_html=True)
    
    pdf_file = create_pdf_receipt(payment)
    st.markdown(get_pdf_download_link(pdf_file), unsafe_allow_html=True)

# LANDLORD DASHBOARD
def admin_dashboard():
    st.title("🏠 Landlord Dashboard")
    
    col1, col2, col3 = st.columns(3)
    total_rent = sum([t["rent"] for t in tenants])
    total_collected = sum([p["amount"] for p in payments])
    outstanding = total_rent - total_collected
    
    with col1:
        st.markdown(f'<div class="metric-box"><h4>Total Monthly Rent</h4><h2>KES {total_rent:,}</h2></div>', unsafe_allow_html=True)
    with col2:
        st.markdown(f'<div class="metric-box"><h4>Collected</h4><h2>KES {total_collected:,}</h2></div>', unsafe_allow_html=True)
    with col3:
        st.markdown(f'<div class="metric-box"><h4>Outstanding</h4><h2>KES {outstanding:,}</h2></div>', unsafe_allow_html=True)
    
    tab1, tab2, tab3 = st.tabs(["Tenants", "Payments", "Add Tenant"])
    
    with tab1:
        st.subheader("All Tenants")
        for t in tenants:
            st.write(f"**{t['name']}** | {t['room_type']} | Room: {t['room']} | Balance: KES {t['balance']:,} | Phone: {t['phone']}")
    
    with tab2:
        st.subheader("Payment History & Receipts")
        for p in payments[::-1]:
            with st.expander(f"Receipt {p['id']} - {p['tenant']} - {p['method']} - KES {p['amount']:,}"):
                show_receipt(p)
    
    with tab3:
        tenant_registration()
    
    if st.button("Logout"):
        st.session_state.user = None
        st.rerun()

# TENANT DASHBOARD
def tenant_dashboard(tenant):
    st.title(f"Welcome, {tenant['name']}")
    
    col1, col2, col3 = st.columns(3)
    with col1:
        st.metric("Room Type", tenant["room_type"])
    with col2:
        st.metric("Room", tenant["room"])
    with col3:
        st.metric("Balance", f"KES {tenant['balance']:,}")
    
    st.metric("Due Date", tenant["due_date"])
    
    img_path = f"room_images/{tenant['room']}.jpg"
    if os.path.exists(img_path):
        st.image(img_path, caption=f"{tenant['room_type']} - Room {tenant['room']}", use_column_width=True)
    
    make_payment(tenant)
    
    st.subheader("Your Payment History")
    my_payments = [p for p in payments if p["tenant"] == tenant["name"]]
    for p in my_payments[::-1]:
        show_receipt(p)
    
    if st.button("Logout"):
        st.session_state.user = None
        st.rerun()

# ROUTING
if st.session_state.user is None:
    login()
elif st.session_state.user.get("role") == "admin":
    admin_dashboard()
else:
    tenant_dashboard(st.session_state.user) 
    streamlit
fpdf2
