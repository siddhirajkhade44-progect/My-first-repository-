# My-first-repository-
I make this repository for training 
my-small-repo/
├── .gitignore
├── LICENSE
├── README.md
├── requirements.txt
└── main.py
import tkinter as tk
from tkinter import messagebox

# Predefined login credentials
USERNAME = "admin"
PASSWORD = "1234"

def login():
    user = entry_user.get()
    pwd = entry_pass.get()

    if user == USERNAME and pwd == PASSWORD:
        messagebox.showinfo("Login", f"✅ Welcome {user}!")
    else:
        messagebox.showerror("Login", "❌ Invalid username or password")

# Window
root = tk.Tk()
root.title("Login Page")
root.geometry("300x200")

# Labels & Entries
tk.Label(root, text="Username:").pack(pady=5)
entry_user = tk.Entry(root)
entry_user.pack()

tk.Label(root, text="Password:").pack(pady=5)
entry_pass = tk.Entry(root, show="*")
entry_pass.pack()

# Login Button
tk.Button(root, text="Login", command=login).pack(pady=15)

root.mainloop()