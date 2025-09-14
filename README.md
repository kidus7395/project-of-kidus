import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import psycopg2
from PIL import Image, ImageTk
import os

# --- Configuration ---
# NOTE: You will need to configure these details to match your PostgreSQL setup.
db_detail = {
    "dbname": "food_menu_db",
    "user": "postgres",
    "password": "st123",
    "host": "localhost",
    "port": "5432"
}

BG_COLOR = "#F4F4F4"
PRIMARY_COLOR = "#2C3E50"
ACCENT_COLOR = "#E74C3C"
HEADING_COLOR = "#3498DB"

# --- Database Functions ---
def connect_to_db():
    """Establishes a connection to the PostgreSQL database."""
    try:
        conn = psycopg2.connect(**db_detail)
        return conn
    except psycopg2.OperationalError as e:
        messagebox.showerror("Database Error", f"Unable to connect to the database.\n{e}")
        return None

def get_menu_items(cuisine_type, search_query=""):
    """Fetches menu items from the database based on cuisine and search query."""
    conn = connect_to_db()
    if conn is None:
        return []
    items = []
    cursor = conn.cursor()
    try:
        if search_query:
            query = "SELECT name, image_path, cooking_steps FROM menu_items WHERE cuisine = %s AND name ILIKE %s;"
            cursor.execute(query, (cuisine_type, f"%{search_query}%"))
        else:
            query = "SELECT name, image_path, cooking_steps FROM menu_items WHERE cuisine = %s;"
            cursor.execute(query, (cuisine_type,))
        results = cursor.fetchall()
        for row in results:
            items.append({'name': row[0], 'image_path': row[1], 'cooking_steps': row[2]})
    except Exception as e:
        messagebox.showerror("Query Error", f"Error fetching data: {e}")
    finally:
        cursor.close()
        conn.close()
    return items

def add_new_recipe_to_db(name, cuisine, image_path, cooking_steps):
    """Adds a new recipe to the database."""
    conn = connect_to_db()
    if conn is None:
        return False
    try:
        cursor = conn.cursor()
        query = "INSERT INTO menu_items (name, cuisine, image_path, cooking_steps) VALUES (%s, %s, %s, %s);"
        cursor.execute(query, (name, cuisine, image_path, cooking_steps))
        conn.commit()
        return True
    except Exception as e:
        messagebox.showerror("Error", f"Failed to add recipe: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
    return False

# --- GUI Class ---
class FoodMenuApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Online Food Menu")
        self.geometry("1000x800")
        self.config(bg=BG_COLOR)
        self.current_cuisine = 'Ethiopian'

        self.style = ttk.Style(self)
        self.style.theme_use('clam')
        self.style.configure('TButton', font=('Helvetica', 12, 'bold'), padding=10, background=ACCENT_COLOR, foreground="white")
        self.style.map('TButton', background=[('active', '#C0392B')])
        self.style.configure('TLabel', font=('Helvetica', 14, 'bold'), foreground=PRIMARY_COLOR)
        self.style.configure('Card.TFrame', background='white', relief='raised', borderwidth=2)
        self.style.configure('CardHover.TFrame', background='#F0F0F0')

        self.create_widgets()
        self.show_menu(self.current_cuisine)

    def create_widgets(self):
        """Creates the main GUI widgets and layout."""
        header_frame = ttk.Frame(self, padding=20)
        header_frame.pack(fill=tk.X)
        
        title_label = ttk.Label(header_frame, text="Online Food Menu", font=('Helvetica', 24, 'bold'), foreground=HEADING_COLOR)
        title_label.pack(side=tk.LEFT)

        self.search_var = tk.StringVar()
        self.search_var.trace_add("write", self.search_menu)
        self.search_entry = ttk.Entry(header_frame, textvariable=self.search_var, width=30, font=('Helvetica', 12))
        self.search_entry.pack(side=tk.RIGHT, padx=10)
        self.search_entry.insert(0, "Search food...")
        self.search_entry.bind("<FocusIn>", lambda e: self.search_entry.delete(0, "end"))

        button_frame = ttk.Frame(self, padding=(20, 0))
        button_frame.pack(fill=tk.X)

        self.ethiopian_btn = ttk.Button(button_frame, text="Ethiopian Menu", command=lambda: self.switch_menu('Ethiopian'))
        self.ethiopian_btn.pack(side=tk.LEFT, padx=5)

        self.european_btn = ttk.Button(button_frame, text="European Menu", command=lambda: self.switch_menu('European'))
        self.european_btn.pack(side=tk.LEFT, padx=5)
        
        add_recipe_btn = ttk.Button(button_frame, text="Add New Recipe", command=self.open_add_recipe_window)
        add_recipe_btn.pack(side=tk.RIGHT, padx=5)

        self.main_frame = ttk.Frame(self, padding=20, style='Card.TFrame')
        self.main_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=20)

        self.canvas = tk.Canvas(self.main_frame, bg=BG_COLOR)
        self.scrollbar = ttk.Scrollbar(self.main_frame, orient="vertical", command=self.canvas.yview)
        self.scrollable_frame = ttk.Frame(self.canvas)

        self.scrollable_frame.bind("<Configure>", lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")))
        self.canvas.create_window((0, 0), window=self.scrollable_frame, anchor="nw")
        self.canvas.configure(yscrollcommand=self.scrollbar.set)
        
        self.canvas.pack(side="left", fill="both", expand=True)
        self.scrollbar.pack(side="right", fill="y")

    def switch_menu(self, cuisine_type):
        """Switches the displayed menu to a different cuisine type."""
        self.current_cuisine = cuisine_type
        self.search_var.set("")
        self.show_menu(cuisine_type)

    def search_menu(self, *args):
        """Filters the menu items based on the search query."""
        self.show_menu(self.current_cuisine, self.search_var.get())

    def show_menu(self, cuisine_type, search_query=""):
        """Displays menu items in a grid layout."""
        for widget in self.scrollable_frame.winfo_children():
            widget.destroy()

        items = get_menu_items(cuisine_type, search_query)
        if not items:
            ttk.Label(self.scrollable_frame, text=f"No {cuisine_type} items found.", font=('Helvetica', 16), foreground=PRIMARY_COLOR).pack(pady=50)
            return

        row, col = 0, 0
        for item in items:
            item_frame = ttk.Frame(self.scrollable_frame, style='Card.TFrame', padding=10)
            item_frame.grid(row=row, column=col, padx=15, pady=15, sticky="nsew")
            
            item_frame.bind("<Enter>", lambda e, f=item_frame: f.configure(style='CardHover.TFrame'))
            item_frame.bind("<Leave>", lambda e, f=item_frame: f.configure(style='Card.TFrame'))
            item_frame.bind("<Button-1>", lambda e, item_data=item: self.show_details(item_data))
            
            try:
                img = Image.open(item['image_path'])
                img = img.resize((150, 100), Image.LANCZOS)
                photo = ImageTk.PhotoImage(img)
                image_label = ttk.Label(item_frame, image=photo, background='white')
                image_label.image = photo
            except (FileNotFoundError, IOError):
                image_label = ttk.Label(item_frame, text="[No Image]", width=20, anchor='center', background='white')
            image_label.pack(pady=5)
            
            name_label = ttk.Label(item_frame, text=item['name'], font=('Arial', 11, 'bold'), background='white', wraplength=140, anchor='center')
            name_label.pack(pady=5)
            
            col += 1
            if col > 3:
                col = 0
                row += 1

        for i in range(4):
            self.scrollable_frame.grid_columnconfigure(i, weight=1)
            
    def show_details(self, item_data):
        """Opens a new window to display detailed recipe information."""
        detail_window = tk.Toplevel(self)
        detail_window.title(item_data['name'])
        detail_window.geometry("600x650")
        detail_window.config(bg=BG_COLOR)

        main_frame = ttk.Frame(detail_window, padding=20, style='Card.TFrame')
        main_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=20)

        title_label = ttk.Label(main_frame, text=item_data['name'], font=('Helvetica', 22, 'bold'), foreground=HEADING_COLOR)
        title_label.pack(pady=10)

        try:
            img = Image.open(item_data['image_path'])
            img = img.resize((300, 200), Image.LANCZOS)
            photo = ImageTk.PhotoImage(img)
            image_label = ttk.Label(main_frame, image=photo, relief='solid', borderwidth=2)
            image_label.image = photo
        except (FileNotFoundError, IOError):
            image_label = ttk.Label(main_frame, text="[Image Not Found]", font=('Helvetica', 14, 'italic'))
        image_label.pack(pady=10)
        
        steps_title = ttk.Label(main_frame, text="Cooking Steps", font=('Helvetica', 16, 'underline'), foreground=PRIMARY_COLOR)
        steps_title.pack(pady=10)

        steps_text = tk.Text(main_frame, wrap=tk.WORD, height=15, width=60, font=('Helvetica', 10), padx=10, pady=10, relief="flat", bg=BG_COLOR)
        steps_text.insert(tk.END, item_data['cooking_steps'])
        steps_text.config(state=tk.DISABLED)
        steps_text.pack(pady=10)
        
    def open_add_recipe_window(self):
        """Opens a new window for adding a new recipe."""
        add_window = tk.Toplevel(self)
        add_window.title("Add New Recipe")
        add_window.geometry("450x550")
        add_window.config(bg=BG_COLOR)

        main_frame = ttk.Frame(add_window, padding=20)
        main_frame.pack(fill=tk.BOTH, expand=True)

        ttk.Label(main_frame, text="Recipe Name:").pack(pady=5)
        name_entry = ttk.Entry(main_frame)
        name_entry.pack(fill=tk.X, padx=10)

        ttk.Label(main_frame, text="Cuisine (Ethiopian/European):").pack(pady=5)
        cuisine_entry = ttk.Entry(main_frame)
        cuisine_entry.pack(fill=tk.X, padx=10)

        # --- MODIFIED: Image selection with Browse button ---
        image_frame = ttk.Frame(main_frame)
        image_frame.pack(fill=tk.X, padx=10, pady=5)
        
        ttk.Label(image_frame, text="Image Path:").pack(side=tk.LEFT)
        
        self.image_path_var = tk.StringVar()
        image_entry = ttk.Entry(image_frame, textvariable=self.image_path_var)
        image_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=(0, 5))
        
        browse_button = ttk.Button(image_frame, text="Browse...", command=self.browse_image_path)
        browse_button.pack(side=tk.RIGHT)
        
        # --- End of Modification ---
        
        ttk.Label(main_frame, text="Cooking Steps:").pack(pady=5)
        steps_text = tk.Text(main_frame, height=15, width=40)
        steps_text.pack(fill=tk.BOTH, expand=True, padx=10)

        def save_recipe():
            """Saves the new recipe to the database."""
            name = name_entry.get()
            cuisine = cuisine_entry.get()
            image_path = self.image_path_var.get()
            steps = steps_text.get("1.0", tk.END).strip()

            if name and cuisine and steps:
                if add_new_recipe_to_db(name, cuisine, image_path, steps):
                    messagebox.showinfo("Success", "Recipe added successfully!")
                    add_window.destroy()
                    self.show_menu(self.current_cuisine) # Refresh the menu
            else:
                messagebox.showerror("Error", "Please fill in all fields.")

        save_button = ttk.Button(main_frame, text="Save Recipe", command=save_recipe)
        save_button.pack(pady=20)
        
    def browse_image_path(self):
        """Opens a file dialog to select an image file."""
        file_path = filedialog.askopenfilename(
            filetypes=[("Image Files", "*.png;*.jpg;*.jpeg;*.gif;*.bmp")]
        )
        if file_path:
            self.image_path_var.set(file_path)

if __name__ == "__main__":
    app = FoodMenuApp()
    app.mainloop()
