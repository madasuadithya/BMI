# BMI
import tkinter as tk
from tkinter import messagebox
import sqlite3
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

# Database setup
conn = sqlite3.connect('bmi_data.db')
cursor = conn.cursor()
cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        weight REAL,
        height REAL,
        bmi REAL,
        timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
    )
''')
conn.commit()

# GUI setup
class BMIApp:
    def __init__(self, master):
        self.master = master
        self.master.title("BMI Calculator")
        
        # Variables
        self.name_var = tk.StringVar()
        self.weight_var = tk.DoubleVar()
        self.height_var = tk.DoubleVar()
        
        # Widgets
        tk.Label(master, text="Name:").grid(row=0, column=0, sticky=tk.E)
        tk.Entry(master, textvariable=self.name_var).grid(row=0, column=1)
        
        tk.Label(master, text="Weight (kg):").grid(row=1, column=0, sticky=tk.E)
        tk.Entry(master, textvariable=self.weight_var).grid(row=1, column=1)
        
        tk.Label(master, text="Height (m):").grid(row=2, column=0, sticky=tk.E)
        tk.Entry(master, textvariable=self.height_var).grid(row=2, column=1)
        
        tk.Button(master, text="Calculate BMI", command=self.calculate_bmi).grid(row=3, column=0, columnspan=2, pady=10)
        tk.Button(master, text="View History", command=self.view_history).grid(row=4, column=0, columnspan=2, pady=10)
        
    def calculate_bmi(self):
        try:
            weight = self.weight_var.get()
            height = self.height_var.get()
            bmi = round(weight / (height ** 2), 2)
            
            messagebox.showinfo("BMI Result", f"Your BMI is: {bmi}")
            
            # Save data to the database
            cursor.execute("INSERT INTO users (name, weight, height, bmi) VALUES (?, ?, ?, ?)",
                           (self.name_var.get(), weight, height, bmi))
            conn.commit()
            
        except Exception as e:
            messagebox.showerror("Error", f"An error occurred: {str(e)}")

    def view_history(self):
        # Fetch historical data from the database
        cursor.execute("SELECT name, weight, height, bmi, timestamp FROM users ORDER BY timestamp DESC")
        data = cursor.fetchall()

        if not data:
            messagebox.showinfo("History", "No data available.")
            return

        # Create a new window to display the historical data
        history_window = tk.Toplevel(self.master)
        history_window.title("BMI History")

        # Display historical data in a listbox
        listbox = tk.Listbox(history_window, width=50)
        listbox.grid(row=0, column=0, padx=10, pady=10)

        for entry in data:
            listbox.insert(tk.END, f"{entry[0]} - Weight: {entry[1]} kg, Height: {entry[2]} m, BMI: {entry[3]}, Time: {entry[4]}")

        # Add a button to show BMI trend graph
        tk.Button(history_window, text="Show BMI Trend", command=lambda: self.show_bmi_trend(data)).grid(row=1, column=0, pady=10)

    def show_bmi_trend(self, data):
        # Extract BMI values and corresponding timestamps for plotting
        timestamps = [entry[4] for entry in data]
        bmi_values = [entry[3] for entry in data]

        # Create a new window for the graph
        trend_window = tk.Toplevel(self.master)
        trend_window.title("BMI Trend Analysis")

        # Plot BMI trend using Matplotlib
        fig, ax = plt.subplots(figsize=(8, 5))
        ax.plot(timestamps, bmi_values, marker='o', linestyle='-', color='b')
        ax.set_title("BMI Trend Analysis")
        ax.set_xlabel("Timestamp")
        ax.set_ylabel("BMI Value")

        # Embed the Matplotlib figure in the Tkinter window
        canvas = FigureCanvasTkAgg(fig, master=trend_window)
        canvas.draw()
        canvas.get_tk_widget().pack(side=tk.TOP, fill=tk.BOTH, expand=1)

        # Add a toolbar for Matplotlib
        toolbar = NavigationToolbar2Tk(canvas, trend_window)
        toolbar.update()
        canvas.get_tk_widget().pack(side=tk.TOP, fill=tk.BOTH, expand=1)

# Main application
root = tk.Tk()
app = BMIApp(root)
root.mainloop()

# Close the database connection when the application exits
conn.close()
