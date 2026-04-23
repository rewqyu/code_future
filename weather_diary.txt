import tkinter as tk
from tkinter import ttk, messagebox
import json
import os
from datetime import datetime

class WeatherDiaryApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Дневник погоды")
        self.root.geometry("900x600")
        self.records = []
        self.data_file = "weather_data.json"

        self.setup_ui()

    def setup_ui(self):
        # === Панель ввода ===
        input_frame = ttk.LabelFrame(self.root, text="Добавить запись")
        input_frame.pack(fill="x", padx=10, pady=10)

        ttk.Label(input_frame, text="Дата (YYYY-MM-DD):").grid(row=0, column=0, padx=5, pady=5, sticky="w")
        self.date_entry = ttk.Entry(input_frame, width=15)
        self.date_entry.grid(row=0, column=1, padx=5, pady=5)

        ttk.Label(input_frame, text="Температура (°C):").grid(row=0, column=2, padx=5, pady=5, sticky="w")
        self.temp_entry = ttk.Entry(input_frame, width=10)
        self.temp_entry.grid(row=0, column=3, padx=5, pady=5)

        ttk.Label(input_frame, text="Описание:").grid(row=0, column=4, padx=5, pady=5, sticky="w")
        self.desc_entry = ttk.Entry(input_frame, width=25)
        self.desc_entry.grid(row=0, column=5, padx=5, pady=5)

        ttk.Label(input_frame, text="Осадки:").grid(row=0, column=6, padx=5, pady=5, sticky="w")
        self.precip_var = tk.StringVar(value="Нет")
        ttk.Combobox(input_frame, textvariable=self.precip_var, values=["Да", "Нет"], 
                     width=6, state="readonly").grid(row=0, column=7, padx=5, pady=5)

        ttk.Button(input_frame, text="➕ Добавить запись", command=self.add_record).grid(row=0, column=8, padx=10, pady=5)

        # === Панель фильтрации ===
        filter_frame = ttk.LabelFrame(self.root, text="Фильтрация")
        filter_frame.pack(fill="x", padx=10, pady=10)

        ttk.Label(filter_frame, text="Точная дата:").grid(row=0, column=0, padx=5, pady=5)
        self.filter_date_entry = ttk.Entry(filter_frame, width=12)
        self.filter_date_entry.grid(row=0, column=1, padx=5, pady=5)

        ttk.Label(filter_frame, text="Мин. температура (>°C):").grid(row=0, column=2, padx=5, pady=5)
        self.filter_temp_entry = ttk.Entry(filter_frame, width=10)
        self.filter_temp_entry.grid(row=0, column=3, padx=5, pady=5)

        ttk.Button(filter_frame, text="🔍 Применить", command=self.apply_filter).grid(row=0, column=4, padx=5, pady=5)
        ttk.Button(filter_frame, text="❌ Сбросить", command=self.reset_filter).grid(row=0, column=5, padx=5, pady=5)

        # === Таблица записей ===
        table_frame = ttk.Frame(self.root)
        table_frame.pack(fill="both", expand=True, padx=10, pady=10)

        columns = ("date", "temp", "desc", "precip")
        self.tree = ttk.Treeview(table_frame, columns=columns, show="headings")
        self.tree.heading("date", text="Дата")
        self.tree.heading("temp", text="Температура (°C)")
        self.tree.heading("desc", text="Описание погоды")
        self.tree.heading("precip", text="Осадки")

        self.tree.column("date", width=120, anchor="center")
        self.tree.column("temp", width=100, anchor="center")
        self.tree.column("desc", width=400)
        self.tree.column("precip", width=80, anchor="center")

        scrollbar = ttk.Scrollbar(table_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        scrollbar.pack(side="right", fill="y")
        self.tree.pack(side="left", fill="both", expand=True)

        # === Кнопки сохранения/загрузки ===
        io_frame = ttk.Frame(self.root)
        io_frame.pack(fill="x", padx=10, pady=10)
        ttk.Button(io_frame, text="💾 Сохранить в JSON", command=self.save_to_json).pack(side="left", padx=5)
        ttk.Button(io_frame, text="📂 Загрузить из JSON", command=self.load_from_json).pack(side="left", padx=5)

    def validate_input(self):
        date_str = self.date_entry.get().strip()
        temp_str = self.temp_entry.get().strip()
        desc = self.desc_entry.get().strip()

        try:
            datetime.strptime(date_str, "%Y-%m-%d")
        except ValueError:
            messagebox.showerror("Ошибка ввода", "Неверный формат даты. Используйте YYYY-MM-DD")
            return False

        try:
            float(temp_str)
        except ValueError:
            messagebox.showerror("Ошибка ввода", "Температура должна быть числом")
            return False

        if not desc:
            messagebox.showerror("Ошибка ввода", "Описание погоды не может быть пустым")
            return False

        return True

    def add_record(self):
        if not self.validate_input():
            return

        record = {
            "date": self.date_entry.get().strip(),
            "temp": float(self.temp_entry.get().strip()),
            "desc": self.desc_entry.get().strip(),
            "precip": self.precip_var.get()
        }
        self.records.append(record)
        self.update_table()
        self.clear_inputs()
        messagebox.showinfo("Успех", "Запись успешно добавлена!")

    def clear_inputs(self):
        self.date_entry.delete(0, tk.END)
        self.temp_entry.delete(0, tk.END)
        self.desc_entry.delete(0, tk.END)
        self.precip_var.set("Нет")

    def update_table(self, data=None):
        for item in self.tree.get_children():
            self.tree.delete(item)
        display_data = data if data is not None else self.records
        for rec in display_data:
            self.tree.insert("", "end", values=(rec["date"], f"{rec['temp']:.1f}", rec["desc"], rec["precip"]))

    def apply_filter(self):
        date_filter = self.filter_date_entry.get().strip()
        temp_filter = self.filter_temp_entry.get().strip()

        filtered = []
        for rec in self.records:
            match = True
            if date_filter:
                try:
                    # Приводим ввод к стандартному формату для точного сравнения
                    formatted_date = datetime.strptime(date_filter, "%Y-%m-%d").strftime("%Y-%m-%d")
                    if rec["date"] != formatted_date:
                        match = False
                except ValueError:
                    messagebox.showerror("Ошибка фильтра", "Неверный формат даты в фильтре")
                    return

            if temp_filter:
                try:
                    if rec["temp"] <= float(temp_filter):
                        match = False
                except ValueError:
                    messagebox.showerror("Ошибка фильтра", "Значение температуры фильтра должно быть числом")
                    return

            if match:
                filtered.append(rec)

        self.update_table(filtered)

    def reset_filter(self):
        self.filter_date_entry.delete(0, tk.END)
        self.filter_temp_entry.delete(0, tk.END)
        self.update_table()

    def save_to_json(self):
        try:
            with open(self.data_file, "w", encoding="utf-8") as f:
                json.dump(self.records, f, ensure_ascii=False, indent=4)
            messagebox.showinfo("Успех", f"Данные сохранены в {self.data_file}")
        except Exception as e:
            messagebox.showerror("Ошибка сохранения", str(e))

    def load_from_json(self):
        if not os.path.exists(self.data_file):
            messagebox.showwarning("Внимание", "Файл данных не найден. Будет создан новый при сохранении.")
            return
        try:
            with open(self.data_file, "r", encoding="utf-8") as f:
                self.records = json.load(f)
            self.update_table()
            messagebox.showinfo("Успех", "Данные успешно загружены!")
        except Exception as e:
            messagebox.showerror("Ошибка загрузки", str(e))

if __name__ == "__main__":
    root = tk.Tk()
    app = WeatherDiaryApp(root)
    root.mainloop()