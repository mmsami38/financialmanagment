import streamlit as st
import pandas as pd
import sqlite3
import datetime
import json
import plotly.express as px
import plotly.graph_objects as go
from datetime import datetime, timedelta
import io

# Page config
st.set_page_config(
    page_title="Mining Business Financial Manager",
    page_icon="💰",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Dark theme CSS
st.markdown("""
<style>
    .stApp {
        background-color: #0e1117;
        color: #fafafa;
    }

    .metric-card {
        background-color: #1e2130;
        padding: 1rem;
        border-radius: 0.5rem;
        border: 1px solid #30363d;
        margin: 0.5rem 0;
    }

    .metric-value {
        font-size: 2rem;
        font-weight: bold;
        color: #00d4aa;
    }

    .metric-label {
        font-size: 0.9rem;
        color: #8b949e;
    }

    .alert-box {
        background-color: #f85149;
        color: white;
        padding: 0.75rem;
        border-radius: 0.25rem;
        margin: 0.5rem 0;
    }

    .success-box {
        background-color: #238636;
        color: white;
        padding: 0.75rem;
        border-radius: 0.25rem;
        margin: 0.5rem 0;
    }
</style>
""", unsafe_allow_html=True)


# Database initialization
def init_database():
    conn = sqlite3.connect('financial_manager.db')
    c = conn.cursor()

    # Transactions table
    c.execute('''CREATE TABLE IF NOT EXISTS transactions
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  date TEXT NOT NULL,
                  description TEXT NOT NULL,
                  amount REAL NOT NULL,
                  transaction_type TEXT NOT NULL,
                  primary_category TEXT NOT NULL,
                  sub_category TEXT,
                  payment_method TEXT NOT NULL,
                  contract_name TEXT,
                  notes TEXT,
                  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')

    # Categories table
    c.execute('''CREATE TABLE IF NOT EXISTS categories
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  primary_category TEXT NOT NULL,
                  sub_category TEXT,
                  keywords TEXT,
                  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')

    # Contacts table
    c.execute('''CREATE TABLE IF NOT EXISTS contacts
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  name TEXT NOT NULL UNIQUE,
                  mobile TEXT,
                  category TEXT,
                  notes TEXT,
                  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')

    # Contracts table
    c.execute('''CREATE TABLE IF NOT EXISTS contracts
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  project_name TEXT NOT NULL UNIQUE,
                  client_name TEXT,
                  start_date TEXT,
                  end_date TEXT,
                  payment_terms TEXT,
                  total_value REAL,
                  status TEXT DEFAULT 'Active',
                  notes TEXT,
                  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')

    # Alerts table
    c.execute('''CREATE TABLE IF NOT EXISTS alerts
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  title TEXT NOT NULL,
                  description TEXT,
                  alert_date TEXT NOT NULL,
                  alert_type TEXT NOT NULL,
                  is_completed INTEGER DEFAULT 0,
                  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')

    # Initialize default categories
    default_categories = [
        ("Personal Dealings", "Individual Loans", "loan,personal,individual"),
        ("Personal Dealings", "Individual Payments", "payment,individual,personal"),
        ("Contract Payments", "Project Income", "project,contract,income,payment"),
        ("Contract Payments", "Contractor Payments", "contractor,payment,expense"),
        ("Partner Transactions", "Partner Investment", "partner,investment"),
        ("Partner Transactions", "Partner Withdrawal", "partner,withdrawal"),
        ("Mutual Expenses", "Equipment", "equipment,machinery,tool"),
        ("Mutual Expenses", "Materials", "cement,limestone,material,supply"),
        ("Mutual Expenses", "Transportation", "transport,fuel,vehicle"),
        ("Loans Given", "Business Loans", "business,loan,given"),
        ("Loans Received", "Bank Loans", "bank,loan,received"),
        ("New Project Expense", "Site Preparation", "site,preparation,clearing"),
        ("New Project Expense", "Equipment Purchase", "equipment,purchase,new"),
        ("Upcoming Events", "Maintenance", "maintenance,service,repair"),
    ]

    for primary, sub, keywords in default_categories:
        c.execute("INSERT OR IGNORE INTO categories (primary_category, sub_category, keywords) VALUES (?, ?, ?)",
                  (primary, sub, keywords))

    conn.commit()
    conn.close()


# Auto-categorization function
def auto_categorize_transaction(description):
    conn = sqlite3.connect('financial_manager.db')
    c = conn.cursor()

    c.execute("SELECT primary_category, sub_category, keywords FROM categories")
    categories = c.fetchall()
    conn.close()

    description_lower = description.lower()

    for primary, sub, keywords in categories:
        if keywords:
            keyword_list = [k.strip().lower() for k in keywords.split(',')]
            for keyword in keyword_list:
                if keyword in description_lower:
                    return primary, sub

    return "Miscellaneous", "Other"


# Transaction functions
def add_transaction(date, description, amount, transaction_type, payment_method,
                    primary_category=None, sub_category=None, contract_name=None, notes=None):
    conn = sqlite3.connect('financial_manager.db')
    c = conn.cursor()

    # Auto-categorize if not provided
    if not primary_category:
        primary_category, sub_category = auto_categorize_transaction(description)

    c.execute('''INSERT INTO transactions 
                 (date, description, amount, transaction_type, primary_category, 
                  sub_category, payment_method, contract_name, notes)
                 VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)''',
              (date, description, amount, transaction_type, primary_category,
               sub_category, payment_method, contract_name, notes))

    conn.commit()
    conn.close()


def get_transactions(limit=None):
    conn = sqlite3.connect('financial_manager.db')
    query = "SELECT * FROM transactions ORDER BY date DESC, created_at DESC"
    if limit:
        query += f" LIMIT {limit}"
    df = pd.read_sql_query(query, conn)
    conn.close()
    return df


def update_transaction(transaction_id, **kwargs):
    conn = sqlite3.connect('financial_manager.db')
    c = conn.cursor()

    set_clause = ', '.join([f"{key} = ?" for key in kwargs.keys()])
    values = list(kwargs.values()) + [transaction_id]

    c.execute(f"UPDATE transactions SET {set_clause} WHERE id = ?", values)
    conn.commit()
    conn.close()


def delete_transaction(transaction_id):
    conn = sqlite3.connect('financial_manager.db')
    c = conn.cursor()
    c.execute("DELETE FROM transactions WHERE id = ?", (transaction_id,))
    conn.commit()
    conn.close()


# Dashboard functions
def get_dashboard_metrics():
    conn = sqlite3.connect('financial_manager.db')

    # Total income vs expenses
    income_df = pd.read_sql_query(
        "SELECT SUM(amount) as total FROM transactions WHERE transaction_type = 'Income'", conn)
    expense_df = pd.read_sql_query(
        "SELECT SUM(amount) as total FROM transactions WHERE transaction_type = 'Expense'", conn)

    # Loan balances
    loans_given_df = pd.read_sql_query(
        "SELECT SUM(amount) as total FROM transactions WHERE primary_category = 'Loans Given'", conn)
    loans_received_df = pd.read_sql_query(
        "SELECT SUM(amount) as total FROM transactions WHERE primary_category = 'Loans Received'", conn)

    # Contract payments
    contract_income_df = pd.read_sql_query(
        "SELECT SUM(amount) as total FROM transactions WHERE primary_category = 'Contract Payments' AND transaction_type = 'Income'",
        conn)

    conn.close()

    total_income = income_df['total'].iloc[0] or 0
    total_expense = expense_df['total'].iloc[0] or 0
    loans_given = loans_given_df['total'].iloc[0] or 0
    loans_received = loans_received_df['total'].iloc[0] or 0
    contract_income = contract_income_df['total'].iloc[0] or 0

    return {
        'net_balance': total_income - total_expense,
        'total_income': total_income,
        'total_expense': total_expense,
        'net_loan_balance': loans_received - loans_given,
        'loans_given': loans_given,
        'loans_received': loans_received,
        'contract_income': contract_income
    }


# Contact management
def add_contact(name, mobile=None, category=None, notes=None):
    conn = sqlite3.connect('financial_manager.db')
    c = conn.cursor()
    try:
        c.execute("INSERT INTO contacts (name, mobile, category, notes) VALUES (?, ?, ?, ?)",
                  (name, mobile, category, notes))
        conn.commit()
        return True
    except sqlite3.IntegrityError:
        return False
    finally:
        conn.close()


def get_contacts():
    conn = sqlite3.connect('financial_manager.db')
    df = pd.read_sql_query("SELECT * FROM contacts ORDER BY name", conn)
    conn.close()
    return df


# Contract management
def add_contract(project_name, client_name=None, start_date=None, end_date=None,
                 payment_terms=None, total_value=None, notes=None):
    conn = sqlite3.connect('financial_manager.db')
    c = conn.cursor()
    try:
        c.execute('''INSERT INTO contracts 
                     (project_name, client_name, start_date, end_date, payment_terms, total_value, notes)
                     VALUES (?, ?, ?, ?, ?, ?, ?)''',
                  (project_name, client_name, start_date, end_date, payment_terms, total_value, notes))
        conn.commit()
        return True
    except sqlite3.IntegrityError:
        return False
    finally:
        conn.close()


def get_contracts():
    conn = sqlite3.connect('financial_manager.db')
    df = pd.read_sql_query("SELECT * FROM contracts ORDER BY created_at DESC", conn)
    conn.close()
    return df


# Alert management
def add_alert(title, description, alert_date, alert_type):
    conn = sqlite3.connect('financial_manager.db')
    c = conn.cursor()
    c.execute("INSERT INTO alerts (title, description, alert_date, alert_type) VALUES (?, ?, ?, ?)",
              (title, description, alert_date, alert_type))
    conn.commit()
    conn.close()


def get_upcoming_alerts():
    conn = sqlite3.connect('financial_manager.db')
    today = datetime.now().strftime('%Y-%m-%d')
    df = pd.read_sql_query(
        "SELECT * FROM alerts WHERE alert_date >= ? AND is_completed = 0 ORDER BY alert_date",
        conn, params=(today,))
    conn.close()
    return df


# Initialize database
init_database()

# Sidebar navigation
st.sidebar.title("💰 Financial Manager")
page = st.sidebar.selectbox("Navigate", [
    "📊 Dashboard",
    "➕ Add Transaction",
    "📋 View Transactions",
    "👥 Contacts",
    "📄 Contracts",
    "🔔 Alerts",
    "📊 Reports",
    "⚙️ Settings"
])

# Dashboard Page
if page == "📊 Dashboard":
    st.title("📊 Financial Dashboard")

    metrics = get_dashboard_metrics()

    col1, col2, col3, col4 = st.columns(4)

    with col1:
        st.markdown(f"""
        <div class="metric-card">
            <div class="metric-label">Net Balance</div>
            <div class="metric-value">₹{metrics['net_balance']:,.2f}</div>
        </div>
        """, unsafe_allow_html=True)

    with col2:
        st.markdown(f"""
        <div class="metric-card">
            <div class="metric-label">Total Income</div>
            <div class="metric-value">₹{metrics['total_income']:,.2f}</div>
        </div>
        """, unsafe_allow_html=True)

    with col3:
        st.markdown(f"""
        <div class="metric-card">
            <div class="metric-label">Total Expenses</div>
            <div class="metric-value">₹{metrics['total_expense']:,.2f}</div>
        </div>
        """, unsafe_allow_html=True)

    with col4:
        st.markdown(f"""
        <div class="metric-card">
            <div class="metric-label">Net Loan Balance</div>
            <div class="metric-value">₹{metrics['net_loan_balance']:,.2f}</div>
        </div>
        """, unsafe_allow_html=True)

    # Recent transactions
    st.subheader("📝 Recent Transactions")
    recent_transactions = get_transactions(limit=10)
    if not recent_transactions.empty:
        st.dataframe(recent_transactions[['date', 'description', 'amount', 'transaction_type', 'primary_category']],
                     use_container_width=True)
    else:
        st.info("No transactions found. Add your first transaction!")

    # Upcoming alerts
    st.subheader("🔔 Upcoming Alerts")
    upcoming_alerts = get_upcoming_alerts()
    if not upcoming_alerts.empty:
        for _, alert in upcoming_alerts.iterrows():
            days_until = (datetime.strptime(alert['alert_date'], '%Y-%m-%d') - datetime.now()).days
            if days_until <= 7:
                st.markdown(f"""
                <div class="alert-box">
                    <strong>{alert['title']}</strong><br>
                    {alert['description']}<br>
                    Due: {alert['alert_date']} ({days_until} days)
                </div>
                """, unsafe_allow_html=True)
    else:
        st.info("No upcoming alerts.")

# Add Transaction Page
elif page == "➕ Add Transaction":
    st.title("➕ Add New Transaction")

    with st.form("add_transaction_form"):
        col1, col2 = st.columns(2)

        with col1:
            transaction_date = st.date_input("Date", datetime.now())
            description = st.text_input("Description",
                                        placeholder="e.g., Payment from Contractor A - Limestone Project")
            amount = st.number_input("Amount (₹)", min_value=0.01, step=0.01)
            transaction_type = st.selectbox("Transaction Type", ["Income", "Expense"])

        with col2:
            payment_method = st.selectbox("Payment Method", ["Bank", "Cash", "Cheque", "Digital Transfer"])

            # Get categories for dropdown
            conn = sqlite3.connect('financial_manager.db')
            categories_df = pd.read_sql_query("SELECT DISTINCT primary_category FROM categories", conn)
            conn.close()

            primary_category = st.selectbox("Primary Category",
                                            ["Auto-Categorize"] + categories_df['primary_category'].tolist())

            sub_category = st.text_input("Sub Category (optional)")
            contract_name = st.text_input("Contract Name (optional)")

        notes = st.text_area("Notes (optional)")

        submit_button = st.form_submit_button("Add Transaction")

        if submit_button:
            if description and amount > 0:
                # Handle auto-categorization
                if primary_category == "Auto-Categorize":
                    primary_category = None

                add_transaction(
                    transaction_date.strftime('%Y-%m-%d'),
                    description,
                    amount,
                    transaction_type,
                    payment_method,
                    primary_category,
                    sub_category if sub_category else None,
                    contract_name if contract_name else None,
                    notes if notes else None
                )

                st.markdown("""
                <div class="success-box">
                    Transaction added successfully!
                </div>
                """, unsafe_allow_html=True)

                st.experimental_rerun()
            else:
                st.error("Please fill in all required fields.")

# View Transactions Page
elif page == "📋 View Transactions":
    st.title("📋 Transaction History")

    transactions_df = get_transactions()

    if not transactions_df.empty:
        # Filters
        col1, col2, col3, col4 = st.columns(4)

        with col1:
            date_filter = st.date_input("Filter by Date (optional)")

        with col2:
            category_filter = st.selectbox("Filter by Category",
                                           ["All"] + transactions_df['primary_category'].unique().tolist())

        with col3:
            type_filter = st.selectbox("Filter by Type", ["All", "Income", "Expense"])

        with col4:
            search_term = st.text_input("Search Description")

        # Apply filters
        filtered_df = transactions_df.copy()

        if category_filter != "All":
            filtered_df = filtered_df[filtered_df['primary_category'] == category_filter]

        if type_filter != "All":
            filtered_df = filtered_df[filtered_df['transaction_type'] == type_filter]

        if search_term:
            filtered_df = filtered_df[filtered_df['description'].str.contains(search_term, case=False)]

        # Display transactions
        st.dataframe(filtered_df, use_container_width=True)

        # Export functionality
        if st.button("📥 Export to Excel"):
            output = io.BytesIO()
            with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
                filtered_df.to_excel(writer, index=False, sheet_name='Transactions')

            st.download_button(
                label="Download Excel File",
                data=output.getvalue(),
                file_name=f"transactions_{datetime.now().strftime('%Y%m%d')}.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
            )
    else:
        st.info("No transactions found.")

# Contacts Page
elif page == "👥 Contacts":
    st.title("👥 Contact Management")

    tab1, tab2 = st.tabs(["View Contacts", "Add Contact"])

    with tab1:
        contacts_df = get_contacts()
        if not contacts_df.empty:
            st.dataframe(contacts_df, use_container_width=True)
        else:
            st.info("No contacts found.")

    with tab2:
        with st.form("add_contact_form"):
            name = st.text_input("Name *")
            mobile = st.text_input("Mobile Number")
            category = st.selectbox("Category", ["Individual", "Contractor", "Partner", "Supplier", "Other"])
            notes = st.text_area("Notes")

            if st.form_submit_button("Add Contact"):
                if name:
                    if add_contact(name, mobile, category, notes):
                        st.success("Contact added successfully!")
                        st.experimental_rerun()
                    else:
                        st.error("Contact with this name already exists.")
                else:
                    st.error("Name is required.")

# Contracts Page
elif page == "📄 Contracts":
    st.title("📄 Contract Management")

    tab1, tab2 = st.tabs(["View Contracts", "Add Contract"])

    with tab1:
        contracts_df = get_contracts()
        if not contracts_df.empty:
            st.dataframe(contracts_df, use_container_width=True)
        else:
            st.info("No contracts found.")

    with tab2:
        with st.form("add_contract_form"):
            col1, col2 = st.columns(2)

            with col1:
                project_name = st.text_input("Project Name *")
                client_name = st.text_input("Client Name")
                start_date = st.date_input("Start Date")
                end_date = st.date_input("End Date")

            with col2:
                payment_terms = st.text_area("Payment Terms")
                total_value = st.number_input("Total Value (₹)", min_value=0.0, step=0.01)
                notes = st.text_area("Notes")

            if st.form_submit_button("Add Contract"):
                if project_name:
                    if add_contract(
                            project_name,
                            client_name if client_name else None,
                            start_date.strftime('%Y-%m-%d'),
                            end_date.strftime('%Y-%m-%d'),
                            payment_terms if payment_terms else None,
                            total_value if total_value > 0 else None,
                            notes if notes else None
                    ):
                        st.success("Contract added successfully!")
                        st.experimental_rerun()
                    else:
                        st.error("Contract with this project name already exists.")
                else:
                    st.error("Project name is required.")

# Alerts Page
elif page == "🔔 Alerts":
    st.title("🔔 Alert Management")

    tab1, tab2 = st.tabs(["View Alerts", "Add Alert"])

    with tab1:
        alerts_df = get_upcoming_alerts()
        if not alerts_df.empty:
            st.dataframe(alerts_df, use_container_width=True)
        else:
            st.info("No upcoming alerts.")

    with tab2:
        with st.form("add_alert_form"):
            title = st.text_input("Alert Title *")
            description = st.text_area("Description")
            alert_date = st.date_input("Alert Date", min_value=datetime.now().date())
            alert_type = st.selectbox("Alert Type", ["Loan Repayment", "Maintenance", "Contract Deadline", "Other"])

            if st.form_submit_button("Add Alert"):
                if title:
                    add_alert(title, description, alert_date.strftime('%Y-%m-%d'), alert_type)
                    st.success("Alert added successfully!")
                    st.experimental_rerun()
                else:
                    st.error("Alert title is required.")

# Reports Page
elif page == "📊 Reports":
    st.title("📊 Financial Reports")

    transactions_df = get_transactions()

    if not transactions_df.empty:
        # Monthly summary
        st.subheader("📈 Monthly Summary")
        transactions_df['date'] = pd.to_datetime(transactions_df['date'])
        transactions_df['month_year'] = transactions_df['date'].dt.to_period('M')

        monthly_summary = transactions_df.groupby(['month_year', 'transaction_type'])['amount'].sum().unstack(
            fill_value=0)
        st.dataframe(monthly_summary)

        # Category breakdown
        st.subheader("🥧 Category Breakdown")
        category_summary = transactions_df.groupby(['primary_category', 'transaction_type'])['amount'].sum().unstack(
            fill_value=0)
        st.dataframe(category_summary)

        # Charts
        if len(transactions_df) > 0:
            fig = px.bar(transactions_df.groupby('primary_category')['amount'].sum().reset_index(),
                         x='primary_category', y='amount',
                         title="Spending by Category")
            fig.update_layout(template="plotly_dark")
            st.plotly_chart(fig, use_container_width=True)
    else:
        st.info("No data available for reports.")

# Settings Page
elif page == "⚙️ Settings":
    st.title("⚙️ Settings")

    st.subheader("📊 Database Statistics")

    conn = sqlite3.connect('financial_manager.db')

    # Get counts
    transaction_count = pd.read_sql_query("SELECT COUNT(*) as count FROM transactions", conn)['count'].iloc[0]
    contact_count = pd.read_sql_query("SELECT COUNT(*) as count FROM contacts", conn)['count'].iloc[0]
    contract_count = pd.read_sql_query("SELECT COUNT(*) as count FROM contracts", conn)['count'].iloc[0]
    alert_count = pd.read_sql_query("SELECT COUNT(*) as count FROM alerts", conn)['count'].iloc[0]

    conn.close()

    col1, col2, col3, col4 = st.columns(4)

    with col1:
        st.metric("Transactions", transaction_count)
    with col2:
        st.metric("Contacts", contact_count)
    with col3:
        st.metric("Contracts", contract_count)
    with col4:
        st.metric("Alerts", alert_count)

    st.subheader("💾 Data Management")

    if st.button("📥 Export All Data"):
        conn = sqlite3.connect('financial_manager.db')

        # Export all tables
        with io.BytesIO() as output:
            with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
                transactions_df = pd.read_sql_query("SELECT * FROM transactions", conn)
                contacts_df = pd.read_sql_query("SELECT * FROM contacts", conn)
                contracts_df = pd.read_sql_query("SELECT * FROM contracts", conn)
                alerts_df = pd.read_sql_query("SELECT * FROM alerts", conn)

                transactions_df.to_excel(writer, sheet_name='Transactions', index=False)
                contacts_df.to_excel(writer, sheet_name='Contacts', index=False)
                contracts_df.to_excel(writer, sheet_name='Contracts', index=False)
                alerts_df.to_excel(writer, sheet_name='Alerts', index=False)

            st.download_button(
                label="Download Complete Data Export",
                data=output.getvalue(),
                file_name=f"financial_data_backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
            )

        conn.close()

    st.warning("⚠️ Always backup your data regularly!")

# Footer
st.sidebar.markdown("---")
st.sidebar.markdown("🏗️ Mining Business Financial Manager")
st.sidebar.markdown("Built with Streamlit")
