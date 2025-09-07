# 📊 Banking Risk Analytics Dashboard

[![Power BI](https://img.shields.io/badge/Power%20BI-DAX-blue)]()
[![Python](https://img.shields.io/badge/Python-Pandas-green)]()
[![SQL](https://img.shields.io/badge/SQL-Relational%20Joins-orange)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-lightgrey.svg)]()

A data analytics project to evaluate **loan risk, deposits, and client engagement** for a banking dataset using **Pandas, SQL, and Power BI (DAX)**. The dashboard helps stakeholders make **data-driven lending decisions** and reduce risk.

---

## 🎯 Problem Statement
Banks must assess whether an applicant is **likely to repay** before approving loans. This project:
- Builds an **analytics pipeline** to transform raw banking data into usable insights.
- Surfaces **KPIs** around loans, deposits, and client engagement.
- Highlights **risk indicators** to support **smarter approvals**.

---

## 🗂️ Dataset
Multi-table dataset linked by primary/foreign keys (examples below):
- `Clients_Banking` — client id, join date, loan/deposit/account fields  
- `Banking_Relationship` — bank type, relationship flags  
- `Gender` — client gender map  
- `Investment_Advisor` — advisor assignments  
- `Period` — date/calendar mapping  

---

## ⚙️ Tech Stack
- **Python (Pandas, NumPy)** → Data cleaning & transformation  
- **SQL** → Querying and relational joins  
- **Power BI** → Interactive dashboards & KPIs  
- **DAX** → Custom calculations & measures  

---

## 🧹 Data Cleaning (Pandas)

```python
import pandas as pd
import numpy as np
from datetime import datetime

# Input CSV paths
clients = pd.read_csv("data/raw/Clients_Banking.csv")

# Basic cleaning
clients.columns = [c.strip().replace(" ", "_").lower() for c in clients.columns]
clients["joined_bank"] = pd.to_datetime(clients["joined_bank"], errors="coerce")

# Numeric columns
num_cols = ["bank_loans","business_lending","credit_card_balance","bank_deposits",
            "checking_accounts","saving_accounts","foreign_currency_account",
            "amount_of_credit_cards","estimated_income"]
for c in num_cols:
    if c in clients.columns:
        clients[c] = pd.to_numeric(clients[c], errors="coerce")

# Feature engineering
today = pd.Timestamp.today().normalize()
clients["engagement_days"] = (today - clients["joined_bank"]).dt.days

bins = [-np.inf, 180, 365, 730, np.inf]
labels = ["<6m", "6-12m", "1-2y", "2y+"]
clients["engagement_timeframe"] = pd.cut(clients["engagement_days"], bins=bins, labels=labels)

clients["income_band"] = pd.cut(clients["estimated_income"],
                                bins=[-np.inf,100000,300000,np.inf],
                                labels=["Low","Mid","High"])

def fee_rate(x):
    if pd.isna(x): return np.nan
    s = str(x).lower()
    if s=="high": return 0.05
    if s=="medium": return 0.03
    if s=="low": return 0.01
    try:
        v=float(s); return v if v<=1 else v/100.0
    except: return np.nan

clients["processing_fee_rate"] = clients.get("fee_structure","").apply(fee_rate)

# KPIs
clients["total_loan"] = clients.get("bank_loans",0) + clients.get("business_lending",0) + clients.get("credit_card_balance",0)
clients["total_deposit"] = clients.get("bank_deposits",0) + clients.get("saving_accounts",0) + clients.get("foreign_currency_account",0) + clients.get("checking_accounts",0)
clients["total_fees"] = clients["total_loan"] * clients.get("processing_fee_rate",0)

# Save cleaned
clients.to_csv("data/clean/Clients_Banking_Clean.csv", index=False)
print("✅ Cleaned data exported")

Total Clients = DISTINCTCOUNT('Clients_Banking'[client_id])

Bank Loan = SUM('Clients_Banking'[bank_loans])
Business Lending = SUM('Clients_Banking'[business_lending])
Credit Cards Balance = SUM('Clients_Banking'[credit_card_balance])

Total Loan = [Bank Loan] + [Business Lending] + [Credit Cards Balance]

Bank Deposit = SUM('Clients_Banking'[bank_deposits])
Savings Account = SUM('Clients_Banking'[saving_accounts])
Foreign Currency Account = SUM('Clients_Banking'[foreign_currency_account])
Checking Accounts = SUM('Clients_Banking'[checking_accounts])

Total Deposit = [Bank Deposit] + [Savings Account] + [Foreign Currency Account] + [Checking Accounts]

Processing Fee Rate =
SWITCH(
    TRUE(),
    SELECTEDVALUE('Clients_Banking'[fee_structure]) = "high", 0.05,
    SELECTEDVALUE('Clients_Banking'[fee_structure]) = "medium", 0.03,
    SELECTEDVALUE('Clients_Banking'[fee_structure]) = "low", 0.01,
    0
)

Total Fees = SUMX('Clients_Banking', [Total Loan] * [Processing Fee Rate])

Engagement Days = DATEDIFF(
    MIN('Clients_Banking'[joined_bank]),
    TODAY(),
    DAY
)

CREATE TABLE clients_banking (
  client_id INT,
  joined_bank DATE,
  bank_loans DECIMAL(18,2),
  business_lending DECIMAL(18,2),
  credit_card_balance DECIMAL(18,2),
  bank_deposits DECIMAL(18,2),
  checking_accounts DECIMAL(18,2),
  saving_accounts DECIMAL(18,2),
  foreign_currency_account DECIMAL(18,2),
  amount_of_credit_cards DECIMAL(18,2),
  estimated_income DECIMAL(18,2),
  fee_structure VARCHAR(20)
);

CREATE VIEW v_clients_kpi AS
SELECT
  client_id,
  bank_loans + business_lending + credit_card_balance AS total_loan,
  bank_deposits + saving_accounts + foreign_currency_account + checking_accounts AS total_deposit
FROM clients_banking;
