# Banking_App_with_hash_pin
I have created a banking app using TKinter, pyodbc ( SQL Server connection ) and bcrypt to encrypt pin entered by customer. It's a side project I always wanted to do and recently finished it. A few kinks here and there which i'll update soon. 

# sql statements used to create tables.

```
CREATE TABLE Customers (
    customer_id INT PRIMARY KEY,
    pin_hash VARCHAR(255) NOT NULL
);
```
```
CREATE TABLE Transactions (
    transaction_id INT IDENTITY(1,1) PRIMARY KEY,
    customer_id INT FOREIGN KEY REFERENCES Customers(customer_id),
    amount DECIMAL(10, 2) NOT NULL,
    balance DECIMAL(10, 2) NOT NULL,
    transaction_type VARCHAR(50) NOT NULL,  -- 'credit' or 'debit'
    timestamp DATETIME DEFAULT GETDATE()
);
```
# Python script

```
import tkinter as tk
from tkinter import messagebox
import pyodbc
import bcrypt
from decimal import Decimal


# Database connection (SQL Server)
conn = pyodbc.connect('Driver={SQL Server};'
                     'Server=SQLEXPRESS;'
                     'Database=Banking;'
                      'Trusted_Connection=yes;')
cursor = conn.cursor()


# Hashing the PIN using bcrypt
def hash_pin(pin):
    salt = bcrypt.gensalt()
    hashed_pin = bcrypt.hashpw(pin.encode('utf-8'), salt)
    return hashed_pin

# Function to register a new customer
def register_customer(customer_id, pin):
    hashed_pin = hash_pin(pin)
    
    query = "INSERT INTO Customers (customer_id, pin_hash) VALUES (?, ?)"
    try:
        cursor.execute(query, (customer_id, hashed_pin))
        conn.commit()
        return True
    except Exception as e:
        print(f"Error during registration: {e}")
        return False

# Function to authenticate user (check PIN)
def authenticate(account_number, entered_pin):
    query = "SELECT pin_hash FROM Customers WHERE customer_id = ?"
    cursor.execute(query, (account_number,))
    result = cursor.fetchone()
    
    if result:
        stored_pin_hash = result[0]
        if bcrypt.checkpw(entered_pin.encode('utf-8'), stored_pin_hash.encode('utf-8')):
            return True
        else:
            return False
    else:
        return False

# Function to check current balance
def check_balance(customer_id):
    query = """
    SELECT TOP 1 balance 
    FROM Transactions 
    WHERE customer_id = ? 
    ORDER BY timestamp DESC;
    """
    cursor.execute(query, (customer_id,))
    result = cursor.fetchone()
    return result[0] if result else 0.00

# Function to deposit money
def deposit(customer_id, amount):
    current_balance = Decimal(check_balance(customer_id))  # Ensure balance is Decimal
    new_balance = current_balance + Decimal(amount)        # Convert amount to Decimal

    # Format amount and new_balance to have 2 decimal places (matching DECIMAL(10, 2))
    formatted_amount = round(Decimal(amount), 2)
    formatted_balance = round(new_balance, 2)

    query = """
    INSERT INTO Transactions (customer_id, amount, balance, transaction_type) 
    VALUES (?, ?, ?, 'credit');
    """
    cursor.execute(query, (customer_id, formatted_amount, formatted_balance))
    conn.commit()


# Function to withdraw money
def withdraw(customer_id, amount):
    current_balance = Decimal(check_balance(customer_id))  # Ensure balance is Decimal

    # Round both current balance and amount to 2 decimal places for accurate comparison
    amount = round(Decimal(amount), 2)
    current_balance = round(current_balance, 2)

    if current_balance < amount:
        print("Insufficient funds")
        return False
    
    new_balance = current_balance - amount  # Subtract amount from balance

    # Format new_balance to have 2 decimal places (matching DECIMAL(10, 2))
    formatted_balance = round(new_balance, 2)

    query = """
    INSERT INTO Transactions (customer_id, amount, balance, transaction_type) 
    VALUES (?, ?, ?, 'debit');
    """
    cursor.execute(query, (customer_id, amount, formatted_balance))
    conn.commit()
    return True

# GUI: Handle customer registration
def show_register_form():
    register_window = tk.Toplevel()
    register_window.title("Register Customer")

    # Customer ID entry
    customer_id_label = tk.Label(register_window, text="Account Number:")
    customer_id_label.pack()
    customer_id_entry = tk.Entry(register_window)
    customer_id_entry.pack()

    # PIN entry
    pin_label = tk.Label(register_window, text="PIN:")
    pin_label.pack()
    pin_entry = tk.Entry(register_window, show="*")
    pin_entry.pack()

    # Register button
    def register_action():
        customer_id = customer_id_entry.get()
        pin = pin_entry.get()

        if not customer_id or not pin:
            messagebox.showerror("Error", "Please enter both account number and PIN.")
            return

        # Try to register the customer
        if register_customer(customer_id, pin):
            messagebox.showinfo("Success", "Customer registered successfully!")
            register_window.destroy()
        else:
            messagebox.showerror("Error", "Failed to register customer.")

    register_btn = tk.Button(register_window, text="Register", command=register_action)
    register_btn.pack()

# GUI: Handle login and user authentication
def login():
    account_number = account_entry.get()
    pin = pin_entry.get()
    
    if authenticate(account_number, pin):
        messagebox.showinfo("Login Success", "Welcome!")
        show_dashboard(account_number)
    else:
        messagebox.showerror("Login Failed", "Invalid account number or PIN")

# Show account dashboard (after login)
def show_dashboard(account_number):
    dashboard = tk.Toplevel()
    dashboard.title("Account Dashboard")
    
    # Set the dashboard window size
    dashboard.geometry("400x400")  # Width x Height

    balance = check_balance(account_number)
    balance_label = tk.Label(dashboard, text=f"Your balance: {balance:.2f}")
    balance_label.pack(pady=10)

    deposit_label = tk.Label(dashboard, text="Deposit Amount:")
    deposit_label.pack()
    
    deposit_entry = tk.Entry(dashboard)
    deposit_entry.pack(pady=5)

    withdraw_label = tk.Label(dashboard, text="Withdraw Amount:")
    withdraw_label.pack()
    
    withdraw_entry = tk.Entry(dashboard)
    withdraw_entry.pack(pady=5)

    # Deposit and Withdraw buttons
    deposit_btn = tk.Button(dashboard, text="Deposit", command=lambda: deposit_funds(account_number, deposit_entry.get()))
    withdraw_btn = tk.Button(dashboard, text="Withdraw", command=lambda: withdraw_funds(account_number, withdraw_entry.get()))

    deposit_btn.pack(pady=5)
    withdraw_btn.pack(pady=5)

    # Button to exit the dashboard
    exit_btn = tk.Button(dashboard, text="Exit", command=dashboard.destroy)
    exit_btn.pack(pady=10)

# Deposit funds
def deposit_funds(account_number, amount):
    amount = amount.strip()  # Strip whitespace
    if amount:  # Check if amount is not empty
        deposit(account_number, amount)
        messagebox.showinfo("Success", "Deposit successful!")
        show_dashboard(account_number)  # Refresh dashboard

# Withdraw funds
def withdraw_funds(account_number, amount):
    amount = amount.strip()  # Strip whitespace
    if amount:  # Check if amount is not empty
        if withdraw(account_number, amount):
            messagebox.showinfo("Success", "Withdrawal successful!")
        else:
            messagebox.showerror("Error", "Insufficient funds")
        show_dashboard(account_number)  # Refresh dashboard

# GUI setup for the main window
root = tk.Tk()
root.title("Banking App")

# Set the window size
root.geometry("400x400")  # Width x Height

# Add Register Button to open the registration form
register_btn = tk.Button(root, text="Register New Customer", command=show_register_form)
register_btn.pack(pady=10)

# Account Number and PIN fields for login
account_label = tk.Label(root, text="Account Number")
account_label.pack()

account_entry = tk.Entry(root)
account_entry.pack(pady=5)

pin_label = tk.Label(root, text="PIN")
pin_label.pack()

pin_entry = tk.Entry(root, show="*")  # Mask PIN input
pin_entry.pack(pady=5)

# Login button
login_btn = tk.Button(root, text="Login", command=login)
login_btn.pack(pady=10)

root.mainloop()
```
