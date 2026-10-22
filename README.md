import json
import os
import time
from tkinter import *
from tkinter import ttk, messagebox

APP_DIR = os.path.join(os.path.expanduser("~"), ".focusforge")
DATA_FILE = os.path.join(APP_DIR, "tasks.json")

if not os.path.exists(APP_DIR):
    os.makedirs(APP_DIR)


def load_data():
    if not os.path.exists(DATA_FILE):
        return []
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)
            if isinstance(data, list):
                return data
    except Exception:
        pass
    return []


def save_data(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2)


def add_task():
    title = task_entry.get().strip()
    category = category_entry.get().strip() or "General"
    try:
        minutes = int(minutes_entry.get())
    except ValueError:
        messagebox.showerror("Error", "Minutes must be a number.")
        return
    if not title:
        messagebox.showerror("Error", "Task title is required.")
        return
    if minutes <= 0:
        messagebox.showerror("Error", "Minutes must be greater than 0.")
        return

    tasks = load_data()
    tasks.insert(0, {
        "id": int(time.time() * 1000),
        "title": title,
        "category": category,
        "minutes": minutes,
        "completed": False,
        "created_at": time.strftime("%Y-%m-%d %H:%M:%S")
    })
    save_data(tasks)
    refresh_tasks()
    clear_form()


def clear_form():
    task_entry.delete(0, END)
    category_entry.delete(0, END)
    minutes_entry.delete(0, END)
    task_entry.focus()


def get_selected_task():
    selected = tasks_tree.selection()
    if not selected:
        return None
    task_id = int(selected[0])
    tasks = load_data()
    for task in tasks:
        if task["id"] == task_id:
            return task
    return None


def refresh_tasks():
    for row in tasks_tree.get_children():
        tasks_tree.delete(row)

    tasks = load_data()
    for task in tasks:
        status = "Done" if task.get("completed") else "Open"
        tasks_tree.insert(
            "",
            "end",
            iid=str(task["id"]),
            values=(
                task["title"],
                task["category"],
                f"{task['minutes']} min",
                status,
                task["created_at"]
            )
        )


def complete_selected():
    task = get_selected_task()
    if not task:
        messagebox.showinfo("Info", "Select a task first.")
        return

    tasks = load_data()
    for item in tasks:
        if item["id"] == task["id"]:
            item["completed"] = True
            break

    save_data(tasks)
    refresh_tasks()


def delete_selected():
    task = get_selected_task()
    if not task:
        messagebox.showinfo("Info", "Select a task first.")
        return

    if not messagebox.askyesno("Delete", f"Delete '{task['title']}'?"):
        return

    tasks = load_data()
    tasks = [t for t in tasks if t["id"] != task["id"]]
    save_data(tasks)
    refresh_tasks()


def set_timer_from_selected():
    task = get_selected_task()
    if task:
        minutes = task["minutes"]
        timer_seconds.set(minutes * 60)
        timer_mode.set("Focus")
        update_timer_label()


def start_timer():
    if timer_running.get():
        return

    task = get_selected_task()
    if task and not timer_seconds.get():
        timer_seconds.set(task["minutes"] * 60)

    timer_running.set(True)
    countdown()


def stop_timer():
    timer_running.set(False)
    timer_button.config(text="Start Timer")


def countdown():
    if not timer_running.get():
        timer_button.config(text="Start Timer")
        return

    current = timer_seconds.get()
    if current > 0:
        timer_seconds.set(current - 1)
        update_timer_label()
        root.after(1000, countdown)
    else:
        timer_running.set(False)
        timer_button.config(text="Start Timer")
        messagebox.showinfo("Time's up!", "Nice work! Take a quick break.")
        update_timer_label()


def update_timer_label():
    total = timer_seconds.get()
    minutes, seconds = divmod(total, 60)
    timer_label.config(text=f"{timer_mode.get()}: {minutes:02d}:{seconds:02d}")


root = Tk()
root.title("FocusForge")
root.geometry("920x560")
root.configure(bg="#121826")

timer_seconds = IntVar(value=25 * 60)
timer_running = BooleanVar(value=False)
timer_mode = StringVar(value="Focus")

header = Label(root, text="FocusForge", font=("Segoe UI", 24, "bold"), bg="#121826", fg="#F2F7FF")
header.pack(pady=(20, 10))

main_frame = Frame(root, bg="#121826")
main_frame.pack(fill=BOTH, expand=True, padx=20, pady=10)

left = Frame(main_frame, bg="#1c2434", padx=16, pady=16)
left.pack(side=LEFT, fill=BOTH, expand=True)

right = Frame(main_frame, bg="#1c2434", padx=16, pady=16)
right.pack(side=RIGHT, fill=BOTH, expand=True)

Label(left, text="Add New Task", font=("Segoe UI", 18, "bold"), bg="#1c2434", fg="#F2F7FF").pack(anchor=W, pady=(0, 10))

Label(left, text="Task", bg="#1c2434", fg="#dfeafc").pack(anchor=W)
task_entry = Entry(left, width=35, font=("Segoe UI", 12))
task_entry.pack(fill=X, pady=(0, 10))

Label(left, text="Category", bg="#1c2434", fg="#dfeafc").pack(anchor=W)
category_entry = Entry(left, width=35, font=("Segoe UI", 12))
category_entry.pack(fill=X, pady=(0, 10))

Label(left, text="Minutes", bg="#1c2434", fg="#dfeafc").pack(anchor=W)
minutes_entry = Entry(left, width=35, font=("Segoe UI", 12))
minutes_entry.pack(fill=X, pady=(0, 16))

button_row = Frame(left, bg="#1c2434")
button_row.pack(fill=X)

add_btn = Button(button_row, text="Add Task", command=add_task, bg="#38bdf8", fg="white", font=("Segoe UI", 11, "bold"))
add_btn.pack(side=LEFT, padx=(0, 10))

timer_card = Frame(right, bg="#172033", padx=16, pady=16)
timer_card.pack(fill=BOTH, expand=True)

timer_label = Label(timer_card, text="Focus: 25:00", font=("Segoe UI", 26, "bold"), bg="#172033", fg="#7dd3fc")
timer_label.pack(pady=(10, 20))

timer_button = Button(timer_card, text="Start Timer", command=start_timer, bg="#22c55e", fg="white", font=("Segoe UI", 11, "bold"), width=16)
timer_button.pack(pady=8)

stop_button = Button(timer_card, text="Stop", command=stop_timer, bg="#ef4444", fg="white", font=("Segoe UI", 11, "bold"), width=16)
stop_button.pack(pady=8)

load_selected_btn = Button(timer_card, text="Load Selected Task", command=set_timer_from_selected, bg="#8b5cf6", fg="white", font=("Segoe UI", 11, "bold"), width=16)
load_selected_btn.pack(pady=8)

update_timer_label()

tasks_frame = Frame(right, bg="#172033", padx=16, pady=16)
tasks_frame.pack(fill=BOTH, expand=True, pady=(15, 0))

Label(tasks_frame, text="Tasks", font=("Segoe UI", 18, "bold"), bg="#172033", fg="#F2F7FF").pack(anchor=W)

columns = ("Title", "Category", "Length", "Status", "Created")
tasks_tree = ttk.Treeview(tasks_frame, columns=columns, show="headings", height=10)
for col in columns:
    tasks_tree.heading(col, text=col)
    tasks_tree.column(col, anchor="center", width=140)

tasks_tree.pack(fill=BOTH, expand=True, pady=(10, 10))

task_action_row = Frame(tasks_frame, bg="#172033")
task_action_row.pack(fill=X)

complete_btn = Button(task_action_row, text="Mark Done", command=complete_selected, bg="#f59e0b", fg="white", font=("Segoe UI", 10, "bold"))
complete_btn.pack(side=LEFT, padx=(0, 10))

delete_btn = Button(task_action_row, text="Delete", command=delete_selected, bg="#ef4444", fg="white", font=("Segoe UI", 10, "bold"))
delete_btn.pack(side=LEFT)

refresh_tasks()

root.mainloop()





































































































































































































































""" 




"""




















































































""" 



"""



















































































































































































































































































































































































































































































































































































































































































","content":"# FocusForge\nCross-platform productivity desktop app built with Python for Windows and Linux.\n\n## Features\n- Add tasks with category and time estimate\n- Mark tasks complete\n- Pomodoro-style timer\n- Local JSON storage\n- Simple UI\n\n## Run\n1. Install Python 3.9+\n2. Run:\n   python main.py\n\n## Linux note\nIf Tkinter is missing:\n   sudo apt install python3-tk\n\n## Build executable\npip install pyinstaller\n\nWindows:\n   pyinstaller --onefile --windowed main.py\n\nLinux:\n   pyinstaller --onefile main.py\n"},{"content":"# No external dependencies required\n# Python 3.9+ recommended\n"},{"path":".gitignore