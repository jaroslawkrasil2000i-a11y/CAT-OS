import os
import json
import subprocess
import webbrowser
import random
import sys
import math
from datetime import datetime
from tkinter import *
from tkinter import ttk, filedialog, messagebox

try:
    import pygame
    pygame.mixer.init()
    MP3_AVAILABLE = True
except ImportError:
    MP3_AVAILABLE = False

APPS_FILE = "catos_apps.json"
SETTINGS_FILE = "catos_settings.json"
USERS_FILE = "catos_users.json"

def load_apps():
    if os.path.exists(APPS_FILE):
        with open(APPS_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return []

def save_apps(apps):
    with open(APPS_FILE, "w", encoding="utf-8") as f:
        json.dump(apps, f, ensure_ascii=False, indent=2)

def load_settings():
    if os.path.exists(SETTINGS_FILE):
        with open(SETTINGS_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {"first_run": True, "theme": "cosmic", "sound_enabled": True, "custom_sound": ""}

def save_settings(settings):
    with open(SETTINGS_FILE, "w", encoding="utf-8") as f:
        json.dump(settings, f, ensure_ascii=False, indent=2)

def load_users():
    if os.path.exists(USERS_FILE):
        with open(USERS_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {}

def save_users(users):
    with open(USERS_FILE, "w", encoding="utf-8") as f:
        json.dump(users, f, ensure_ascii=False, indent=2)

def reset_to_factory():
    for file in [APPS_FILE, SETTINGS_FILE, USERS_FILE]:
        if os.path.exists(file):
            os.remove(file)
    return True


class SoundPlayer:
    @staticmethod
    def play_sound(sound_type="click"):
        settings = load_settings()
        if not settings.get("sound_enabled", True):
            return
        custom_sound = settings.get("custom_sound", "")
        if custom_sound and os.path.exists(custom_sound) and MP3_AVAILABLE:
            try:
                pygame.mixer.music.load(custom_sound)
                pygame.mixer.music.play()
                return
            except:
                pass
        try:
            import winsound
            if sound_type == "click":
                winsound.Beep(880, 80)
            elif sound_type == "startup":
                for freq in [523, 659, 784, 1047]:
                    winsound.Beep(freq, 100)
            elif sound_type == "shutdown":
                for freq in [1047, 784, 659, 523]:
                    winsound.Beep(freq, 120)
            elif sound_type == "error":
                winsound.Beep(440, 300)
            elif sound_type == "success":
                winsound.Beep(1047, 150)
                winsound.Beep(1318, 150)
            elif sound_type == "sleep":
                winsound.Beep(659, 200)
                winsound.Beep(523, 300)
        except:
            pass
    
    @staticmethod
    def click():
        SoundPlayer.play_sound("click")
    @staticmethod
    def startup():
        SoundPlayer.play_sound("startup")
    @staticmethod
    def shutdown():
        SoundPlayer.play_sound("shutdown")


# ========== КАСТОМНОЕ ОКНО В СТИЛЕ CAT OS ==========
class CatOSWindow:
    """Базовый класс для всех окон CAT OS с возможностью перетаскивания"""
    def __init__(self, title, width=800, height=600, parent=None):
        self.parent = parent
        self.window = Toplevel(parent) if parent else Toplevel()
        self.window.title(title)
        self.window.geometry(f"{width}x{height}")
        self.window.configure(bg="#0a0f2c")
        self.window.minsize(400, 300)
        
        # Кастомная верхняя панель
        self.title_bar = Frame(self.window, bg="#1a1f3c", height=45)
        self.title_bar.pack(fill=X)
        
        # Иконка и заголовок
        self.title_label = Label(self.title_bar, text=f"🐱 {title}", bg="#1a1f3c", fg="#f9e2af", 
                                  font=("Segoe UI", 11, "bold"))
        self.title_label.pack(side=LEFT, padx=15)
        
        # Кнопки управления
        self.min_btn = Button(self.title_bar, text="─", command=self.minimize,
                              bg="#2a2f4c", fg="#cdd6f4", relief=FLAT, cursor="hand2", width=3)
        self.min_btn.pack(side=RIGHT, padx=2, pady=5)
        
        self.close_btn = Button(self.title_bar, text="✕", command=self.close,
                                bg="#f38ba8", fg="white", relief=FLAT, cursor="hand2", width=3)
        self.close_btn.pack(side=RIGHT, padx=5, pady=5)
        
        # Перетаскивание окна
        self.drag_data = {"x": 0, "y": 0}
        self.title_bar.bind("<Button-1>", self.start_drag)
        self.title_bar.bind("<B1-Motion>", self.drag)
        self.title_label.bind("<Button-1>", self.start_drag)
        self.title_label.bind("<B1-Motion>", self.drag)
        
        # Основной контейнер
        self.main_frame = Frame(self.window, bg="#0a0f2c")
        self.main_frame.pack(fill=BOTH, expand=True, padx=10, pady=10)
    
    def start_drag(self, event):
        self.drag_data["x"] = event.x
        self.drag_data["y"] = event.y
    
    def drag(self, event):
        x = self.window.winfo_x() + event.x - self.drag_data["x"]
        y = self.window.winfo_y() + event.y - self.drag_data["y"]
        self.window.geometry(f"+{x}+{y}")
    
    def minimize(self):
        self.window.iconify()
    
    def close(self):
        self.window.destroy()
    
    def show(self):
        self.window.mainloop()

class Browser(CatOSWindow):
    def __init__(self):
        super().__init__("CAT Browser", 1000, 650)
        SoundPlayer.click()
        
        # Адресная строка
        top_frame = Frame(self.main_frame, bg="#1a1f3c", height=50)
        top_frame.pack(fill=X, pady=(0, 10))
        
        Label(top_frame, text="🔗", bg="#1a1f3c", fg="#89b4fa", font=("Segoe UI", 16)).pack(side=LEFT, padx=10)
        
        self.url_entry = Entry(top_frame, bg="#2a2f4c", fg="#cdd6f4", font=("Segoe UI", 11),
                                relief=FLAT, insertbackground="#cdd6f4")
        self.url_entry.pack(side=LEFT, fill=X, expand=True, padx=10, ipady=8)
        self.url_entry.insert(0, "https://www.google.com")
        self.url_entry.bind("<Return>", lambda e: self.navigate())
        
        go_btn = Button(top_frame, text="🚀", command=self.navigate,
                        bg="#89b4fa", fg="#1a1f3c", font=("Segoe UI", 12, "bold"),
                        relief=FLAT, cursor="hand2", width=4)
        go_btn.pack(side=LEFT, padx=10)
        
        # Контент
        self.content = Text(self.main_frame, bg="#0a0f2c", fg="#cdd6f4", font=("Segoe UI", 11),
                            relief=FLAT, wrap=WORD, insertbackground="#cdd6f4")
        self.content.pack(fill=BOTH, expand=True)
        
        self.show_welcome()
    
    def show_welcome(self):
        self.content.delete(1.0, END)
        cats = ['🐱', '🐈', '😺', '😸', '😻']
        self.content.insert(1.0, f"""
╔{'═'*60}╗
║{' ' * 18}🐱 КОСМИЧЕСКИЙ БРАУЗЕР 🐱{' ' * 18}║
╠{'═'*60}╣
║                                                          ║
║   🌟 Введи URL и нажми 🚀 или Enter                      ║
║   🐱 {random.choice(cats)} Котики следят за тобой... {random.choice(cats)}        ║
║   🚀 Готов к межгалактическому сёрфингу?                ║
║                                                          ║
╚{'═'*60}╝
        """)
    
    def navigate(self):
        url = self.url_entry.get()
        SoundPlayer.click()
        if messagebox.askyesno("🐱 CAT Browser", f"Открыть {url} в браузере?"):
            webbrowser.open(url)


class Installer(CatOSWindow):
    def __init__(self, desktop):
        self.desktop = desktop
        super().__init__("CAT Installer", 550, 450)
        SoundPlayer.click()
        
        # Иконка
        icon_label = Label(self.main_frame, text="🐱", bg="#0a0f2c", fg="#f9e2af", font=("Segoe UI", 40))
        icon_label.pack(pady=10)
        
        Label(self.main_frame, text="УСТАНОВЩИК ПРИЛОЖЕНИЙ", bg="#0a0f2c", fg="#89b4fa",
              font=("Segoe UI", 16, "bold")).pack()
        
        Label(self.main_frame, text="Выбери .py файл для установки в CAT OS", 
              bg="#0a0f2c", fg="#a6e3a1", font=("Segoe UI", 10)).pack(pady=5)
        
        self.file_label = Label(self.main_frame, text="📄 Файл не выбран", bg="#1a1f3c", fg="#cdd6f4",
                                 font=("Segoe UI", 10), relief=GROOVE, bd=2)
        self.file_label.pack(fill=X, padx=50, pady=20, ipady=15)
        
        btn_frame = Frame(self.main_frame, bg="#0a0f2c")
        btn_frame.pack(pady=10)
        
        Button(btn_frame, text="📂 ВЫБРАТЬ", command=self.select_file,
               bg="#89b4fa", fg="#1a1f3c", font=("Segoe UI", 10, "bold"),
               relief=FLAT, cursor="hand2", padx=20, pady=8).pack(side=LEFT, padx=10)
        
        Button(btn_frame, text="✅ УСТАНОВИТЬ", command=self.install_app,
               bg="#a6e3a1", fg="#1a1f3c", font=("Segoe UI", 10, "bold"),
               relief=FLAT, cursor="hand2", padx=20, pady=8).pack(side=LEFT, padx=10)
        
        self.selected_file = None
    
    def select_file(self):
        SoundPlayer.click()
        file = filedialog.askopenfilename(title="Выбери приложение", filetypes=[("Python", "*.py")])
        if file:
            self.selected_file = file
            self.file_label.config(text=f"📄 {os.path.basename(file)}")
            SoundPlayer.play_sound("success")
    
    def install_app(self):
        SoundPlayer.click()
        if self.selected_file:
            apps = load_apps()
            if self.selected_file not in apps:
                apps.append(self.selected_file)
                save_apps(apps)
                self.desktop.add_app_icon(self.selected_file)
                SoundPlayer.play_sound("success")
                messagebox.showinfo("🐱 CAT OS", "✅ Приложение установлено!")
                self.close()
            else:
                SoundPlayer.play_sound("error")
                messagebox.showwarning("🐱 CAT OS", "⚠️ Это приложение уже установлено!")
        else:
            SoundPlayer.play_sound("error")
            messagebox.showwarning("🐱 CAT OS", "❌ Выбери файл для установки!")


# ========== ПУСК ==========
class StartMenu:
    def __init__(self, desktop, username):
        self.desktop = desktop
        self.username = username
        self.window = None
    
    def show(self):
        SoundPlayer.click()
        if self.window and self.window.winfo_exists():
            self.window.destroy()
            return
        
        self.window = Toplevel(self.desktop.root)
        self.window.title("Меню Пуск")
        self.window.configure(bg="#1a1f3c")
        self.window.overrideredirect(True)
        
        w, h = 340, 600
        x = 10
        y = self.desktop.root.winfo_height() - h - 50
        self.window.geometry(f"{w}x{h}+{x}+{y}")
        
        # Аватар
        profile_frame = Frame(self.window, bg="#2a2f4c", height=80)
        profile_frame.pack(fill=X)
        Label(profile_frame, text="🐱", bg="#2a2f4c", fg="#f9e2af", font=("Segoe UI", 30)).pack(side=LEFT, padx=15, pady=10)
        Label(profile_frame, text=self.username, bg="#2a2f4c", fg="#cdd6f4", font=("Segoe UI", 12, "bold")).pack(side=LEFT)
        
        btn_frame = Frame(self.window, bg="#1a1f3c")
        btn_frame.pack(fill=BOTH, expand=True, padx=10, pady=10)
        
        buttons = [
            ("🌐 Браузер", self.desktop.open_browser),
            ("📦 Установщик", self.desktop.open_installer),
            ("🎨 Сменить тему", self.change_theme),
            ("🔊 Настройки звука", self.sound_settings),
            ("💤 Спящий режим", self.sleep_mode),
            ("🔄 Сброс до заводских", self.factory_reset),
            ("⚡ ВЫКЛЮЧЕНИЕ", self.desktop.shutdown_system)
        ]
        
        for text, cmd in buttons:
            btn_color = "#f38ba8" if text == "⚡ ВЫКЛЮЧЕНИЕ" else "#2a2f4c"
            Button(btn_frame, text=text, command=cmd, bg=btn_color, fg="#cdd6f4",
                   font=("Segoe UI", 10), relief=FLAT, cursor="hand2", pady=8).pack(fill=X, pady=3)
        
        Frame(btn_frame, bg="#45475a", height=2).pack(fill=X, pady=10)
        Label(btn_frame, text="📱 ПРИЛОЖЕНИЯ", bg="#1a1f3c", fg="#a6e3a1", font=("Segoe UI", 9, "bold")).pack()
        
        self.listbox = Listbox(btn_frame, bg="#0a0f2c", fg="#cdd6f4", font=("Segoe UI", 10),
                                relief=FLAT, selectbackground="#89b4fa", selectforeground="#0a0f2c", height=10)
        self.listbox.pack(fill=BOTH, expand=True, pady=5)
        self.listbox.bind("<Double-Button-1>", self.open_app)
        
        self.refresh()
        self.window.bind("<FocusOut>", lambda e: self.window.destroy())
    
    def refresh(self):
        self.listbox.delete(0, END)
        for app in load_apps():
            self.listbox.insert(END, f"🐱 {os.path.basename(app)}")
    
    def change_theme(self):
        SoundPlayer.click()
        settings = load_settings()
        settings['theme'] = 'classic' if settings.get('theme') == 'cosmic' else 'cosmic'
        save_settings(settings)
        SoundPlayer.play_sound("success")
        messagebox.showinfo("🐱", "Тема изменена! Перезапусти систему.")
        self.window.destroy()
    
    def sound_settings(self):
        SoundPlayer.click()
        settings = load_settings()
        dialog = Toplevel(self.window)
        dialog.title("Настройки звука")
        dialog.geometry("400x300")
        dialog.configure(bg="#0a0f2c")
        
        Label(dialog, text="🔊 НАСТРОЙКИ ЗВУКА", bg="#0a0f2c", fg="#f9e2af", font=("Segoe UI", 14, "bold")).pack(pady=20)
        
        sound_var = BooleanVar(value=settings.get("sound_enabled", True))
        Checkbutton(dialog, text="Включить звуки", variable=sound_var, bg="#0a0f2c", fg="#cdd6f4", selectcolor="#0a0f2c").pack(pady=10)
        
        Label(dialog, text="Свой MP3 файл:", bg="#0a0f2c", fg="#cdd6f4").pack(pady=5)
        path_label = Label(dialog, text=settings.get("custom_sound", "Не выбран")[:40], bg="#1a1f3c", fg="#a6e3a1")
        path_label.pack(pady=5)
        
        def select_sound():
            file = filedialog.askopenfilename(title="Выбери MP3", filetypes=[("MP3", "*.mp3"), ("WAV", "*.wav")])
            if file:
                settings['custom_sound'] = file
                save_settings(settings)
                path_label.config(text=os.path.basename(file)[:40])
                SoundPlayer.play_sound("success")
        
        Button(dialog, text="📂 Выбрать", command=select_sound, bg="#89b4fa", fg="#1a1f3c", relief=FLAT).pack(pady=5)
        
        def save_sound():
            settings['sound_enabled'] = sound_var.get()
            save_settings(settings)
            dialog.destroy()
        
        Button(dialog, text="💾 СОХРАНИТЬ", command=save_sound, bg="#a6e3a1", fg="#1a1f3c", relief=FLAT, padx=20, pady=8).pack(pady=20)
    
    def sleep_mode(self):
        SoundPlayer.click()
        SoundPlayer.play_sound("sleep")
        self.window.destroy()
        self.desktop.sleep_mode()
    
    def factory_reset(self):
        SoundPlayer.click()
        if messagebox.askyesno("🐱 CAT OS", "Сбросить до заводских настроек? Все данные будут удалены!"):
            reset_to_factory()
            SoundPlayer.play_sound("shutdown")
            messagebox.showinfo("🐱", "Сброс выполнен! Система перезагрузится.")
            self.desktop.root.quit()
    
    def open_app(self, event):
        selection = self.listbox.curselection()
        if selection:
            apps = load_apps()
            self.window.destroy()
            SoundPlayer.click()
            subprocess.Popen(["python", apps[selection[0]]])

class DraggableIcon:
    def __init__(self, parent, text, command, x, y):
        self.parent = parent
        self.command = command
        self.frame = Frame(parent, bg='')
        self.frame.place(x=x, y=y)
        
        self.btn = Button(self.frame, text=text, command=self.on_click,
                         bg='#0a0f2c', fg='#cdd6f4', font=("Segoe UI", 20),
                         relief=FLAT, cursor="hand2", width=3)
        self.btn.pack()
        
        self.label = Label(self.frame, text=command.__name__ if hasattr(command, '__name__') else "App",
                          bg='#0a0f2c', fg='#89b4fa', font=("Segoe UI", 8))
        self.label.pack()
        
        self.drag_data = {"x": 0, "y": 0, "dragging": False}
        self.btn.bind("<Button-1>", self.start_drag)
        self.btn.bind("<B1-Motion>", self.drag)
        self.btn.bind("<ButtonRelease-1>", self.stop_drag)
        self.label.bind("<Button-1>", self.start_drag)
        self.label.bind("<B1-Motion>", self.drag)
        self.label.bind("<ButtonRelease-1>", self.stop_drag)
    
    def on_click(self):
        SoundPlayer.click()
        if self.command:
            self.command()
    
    def start_drag(self, event):
        self.drag_data["x"] = event.x
        self.drag_data["y"] = event.y
        self.drag_data["dragging"] = True
    
    def drag(self, event):
        if self.drag_data["dragging"]:
            new_x = self.frame.winfo_x() + event.x - self.drag_data["x"]
            new_y = self.frame.winfo_y() + event.y - self.drag_data["y"]
            self.frame.place(x=new_x, y=new_y)
    
    def stop_drag(self, event):
        self.drag_data["dragging"] = False

class CosmicBackground:
    def __init__(self, canvas, width, height):
        self.canvas = canvas
        self.width = width
        self.height = height
        self.stars = []
        self.flying_cats = []
        self.create_stars()
        self.create_flying_cats()
        self.animate()
    
    def create_stars(self):
        for _ in range(150):
            x = random.randint(0, self.width)
            y = random.randint(0, self.height)
            size = random.randint(1, 3)
            brightness = random.choice(['#ffffff', '#ffffdd', '#ffeedd', '#bbddff'])
            star = self.canvas.create_oval(x, y, x+size, y+size, fill=brightness, outline='')
            self.stars.append(star)
    
    def create_flying_cats(self):
        cats = ['🐱', '🐈', '😺', '😸', '🐾']
        for _ in range(10):
            x = random.randint(0, self.width)
            y = random.randint(0, self.height)
            cat = self.canvas.create_text(x, y, text=random.choice(cats),
                                           font=("Segoe UI", random.randint(14, 24)),
                                           fill='#f9e2af')
            self.flying_cats.append({'id': cat, 'x': x, 'y': y,
                                     'dx': random.choice([-0.5, 0.5, -0.3, 0.3]),
                                     'dy': random.choice([-0.3, 0.3, -0.2, 0.2])})
    
    def animate(self):
        for star in self.stars:
            self.canvas.move(star, 0, 0.3)
            coords = self.canvas.coords(star)
            if coords and coords[1] > self.height:
                self.canvas.move(star, 0, -self.height - 10)
        
        for cat in self.flying_cats:
            self.canvas.move(cat['id'], cat['dx'], cat['dy'])
            coords = self.canvas.coords(cat['id'])
            if coords:
                if coords[0] < 0 or coords[0] > self.width:
                    cat['dx'] = -cat['dx']
                if coords[1] < 0 or coords[1] > self.height:
                    cat['dy'] = -cat['dy']
        
        self.canvas.tag_raise('all')
        self.canvas.after(50, self.animate)


# ========== ПАНЕЛЬ ЗАДАЧ ==========
class Win11Taskbar:
    def __init__(self, parent, toggle_start_callback, username):
        self.parent = parent
        self.frame = Frame(parent, bg="#1a1f3c", height=45)
        self.frame.pack(side=BOTTOM, fill=X)
        
        self.start_btn = Button(self.frame, text="🐱", command=toggle_start_callback,
                                bg="#89b4fa", fg="#1a1f3c", font=("Segoe UI", 16, "bold"),
                                relief=FLAT, cursor="hand2", width=4)
        self.start_btn.pack(side=LEFT, padx=10, pady=5)
        
        self.user_btn = Button(self.frame, text=f"🐱 {username}", 
                               command=lambda: messagebox.showinfo("Пользователь", f"Вы вошли как {username}"),
                               bg="#2a2f4c", fg="#cdd6f4", font=("Segoe UI", 9),
                               relief=FLAT, cursor="hand2")
        self.user_btn.pack(side=RIGHT, padx=10, pady=5)
        
        self.clock_label = Label(self.frame, bg="#1a1f3c", fg="#cdd6f4", font=("Segoe UI", 10))
        self.clock_label.pack(side=RIGHT, padx=10)
        
        self.update_clock()
    
    def update_clock(self):
        now = datetime.now()
        self.clock_label.config(text=f"{now.strftime('%H:%M')}")
        self.parent.after(1000, self.update_clock)

class Desktop:
    def __init__(self, root, settings, username):
        self.root = root
        self.settings = settings
        self.username = username
        self.root.title(f"🐱 CAT OS - {username}")
        self.root.geometry("1300x750")
        self.root.configure(bg="#0a0f2c")
        
        self.canvas = Canvas(self.root, highlightthickness=0, bg="#0a0f2c")
        self.canvas.pack(fill=BOTH, expand=True)
        
        if settings.get("theme") == "cosmic":
            self.background = CosmicBackground(self.canvas, 1300, 750)
        
        self.icons = []
        self.create_system_icons()
        self.load_app_icons()
        
        self.taskbar = Win11Taskbar(self.root, self.toggle_start, username)
        self.start_menu = StartMenu(self, username)
        
        self.root.bind("<KeyPress>", self.on_key_press)
        
        self.browser = None
        self.installer = None
    
    def on_key_press(self, event):
        if event.keysym == 'Super_L' or event.keysym == 'Super_R':
            self.toggle_start()
        elif event.keysym == 'q' and (event.state & 0x4):
            self.shutdown_system()
    
    def toggle_start(self):
        self.start_menu.show()
    
    def shutdown_system(self):
        SoundPlayer.click()
        if messagebox.askyesno("🐱 CAT OS", "Выключить систему? Мяу..."):
            SoundPlayer.shutdown()
            self.root.quit()
    
    def sleep_mode(self):
        self.root.iconify()
        self.root.after(100, lambda: messagebox.showinfo("💤", "Система в спящем режиме. Нажми OK для пробуждения."))
        self.root.deiconify()
    
    def create_system_icons(self):
        icons = [("🐱", "Браузер", self.open_browser, 50, 50),
                 ("📦", "Установщик", self.open_installer, 50, 150),
                 ("🐟", "Корзина", lambda: messagebox.showinfo("🐟", "Корзина пуста"), 50, 250)]
        for icon in icons:
            self.icons.append(DraggableIcon(self.canvas, icon[0], icon[2], icon[3], icon[4]))
    
    def load_app_icons(self):
        apps = load_apps()
        for i, app in enumerate(apps):
            x = 150 + (i % 4) * 100
            y = 50 + (i // 4) * 100
            self.icons.append(DraggableIcon(self.canvas, "📄", lambda a=app: subprocess.Popen(["python", a]), x, y))
    
    def add_app_icon(self, app_path):
        x = 150 + (len(self.icons) % 4) * 100
        y = 50 + (len(self.icons) // 4) * 100
        self.icons.append(DraggableIcon(self.canvas, "📄", lambda a=app_path: subprocess.Popen(["python", a]), x, y))
    
    def open_browser(self):
        if self.browser is None or not hasattr(self.browser, 'window') or not self.browser.window.winfo_exists():
            self.browser = Browser()
            self.browser.show()
    
    def open_installer(self):
        if self.installer is None or not hasattr(self.installer, 'window') or not self.installer.window.winfo_exists():
            self.installer = Installer(self)
            self.installer.show()


class LoginScreen:
    def __init__(self, root, on_success):
        self.root = root
        self.on_success = on_success
        self.show()
    
    def show(self):
        for widget in self.root.winfo_children():
            widget.destroy()
        
        self.root.configure(bg="#0a0f2c")
        self.root.geometry("500x450")
        self.root.title("🐱 CAT OS - Вход")
        
        main = Frame(self.root, bg="#0a0f2c")
        main.pack(expand=True)
        
        Label(main, text="🐱 CAT OS", bg="#0a0f2c", fg="#f9e2af", font=("Segoe UI", 28, "bold")).pack(pady=30)
        Label(main, text="Вход в систему", bg="#0a0f2c", fg="#89b4fa", font=("Segoe UI", 14)).pack(pady=5)
        
        users = load_users()
        
        if users:
            Label(main, text="Выбери пользователя:", bg="#0a0f2c", fg="#cdd6f4", font=("Segoe UI", 11)).pack(pady=15)
            self.user_var = StringVar()
            for user in users.keys():
                Radiobutton(main, text=user, variable=self.user_var, value=user, 
                           bg="#0a0f2c", fg="#cdd6f4", selectcolor="#0a0f2c", font=("Segoe UI", 11)).pack()
            
            Label(main, text="Пароль:", bg="#0a0f2c", fg="#cdd6f4", font=("Segoe UI", 11)).pack(pady=10)
            self.pwd_entry = Entry(main, show="*", bg="#1a1f3c", fg="#cdd6f4", relief=FLAT, font=("Segoe UI", 11))
            self.pwd_entry.pack(pady=5)
            self.pwd_entry.bind("<Return>", lambda e: self.do_login())
            
            Button(main, text="🚀 ВОЙТИ", command=self.do_login,
                   bg="#89b4fa", fg="#1a1f3c", font=("Segoe UI", 11, "bold"),
                   relief=FLAT, cursor="hand2", padx=30, pady=8).pack(pady=20)
            Button(main, text="➕ НОВЫЙ ПОЛЬЗОВАТЕЛЬ", command=self.new_user,
                   bg="#2a2f4c", fg="#cdd6f4", font=("Segoe UI", 10),
                   relief=FLAT, cursor="hand2").pack()
        else:
            self.new_user()
    
    def do_login(self):
        SoundPlayer.click()
        user = self.user_var.get()
        pwd = self.pwd_entry.get()
        users = load_users()
        
        if user in users and users[user] == pwd:
            SoundPlayer.play_sound("success")
            self.on_success(user)
        else:
            SoundPlayer.play_sound("error")
            messagebox.showerror("Ошибка", "Неверный логин или пароль!")
    
    def new_user(self):
        SoundPlayer.click()
        dialog = Toplevel(self.root)
        dialog.title("Новый пользователь")
        dialog.geometry("400x350")
        dialog.configure(bg="#0a0f2c")
        dialog.resizable(False, False)
        
        Label(dialog, text="🐱 СОЗДАНИЕ УЧЁТНОЙ ЗАПИСИ", bg="#0a0f2c", fg="#f9e2af", 
              font=("Segoe UI", 14, "bold")).pack(pady=20)
        
        Label(dialog, text="Логин:", bg="#0a0f2c", fg="#cdd6f4", font=("Segoe UI", 11)).pack()
        login_entry = Entry(dialog, bg="#1a1f3c", fg="#cdd6f4", relief=FLAT, font=("Segoe UI", 11))
        login_entry.pack(pady=5)
        
        Label(dialog, text="Пароль:", bg="#0a0f2c", fg="#cdd6f4", font=("Segoe UI", 11)).pack()
        pwd_entry = Entry(dialog, show="*", bg="#1a1f3c", fg="#cdd6f4", relief=FLAT, font=("Segoe UI", 11))
        pwd_entry.pack(pady=5)
        
        def create():
            login = login_entry.get()
            pwd = pwd_entry.get()
            if login and pwd:
                users = load_users()
                users[login] = pwd
                save_users(users)
                dialog.destroy()
                self.show()
                SoundPlayer.play_sound("success")
                messagebox.showinfo("Успех", f"Пользователь {login} создан!")
            else:
                SoundPlayer.play_sound("error")
                messagebox.showerror("Ошибка", "Заполните все поля!")
        
        Button(dialog, text="✅ СОЗДАТЬ", command=create,
               bg="#a6e3a1", fg="#1a1f3c", font=("Segoe UI", 11, "bold"),
               relief=FLAT, cursor="hand2", padx=30, pady=8).pack(pady=20)

class SetupWizard:
    def __init__(self, root, on_finish):
        self.root = root
        self.on_finish = on_finish
        self.show()
    
    def show(self):
        for widget in self.root.winfo_children():
            widget.destroy()
        
        self.root.configure(bg="#0a0f2c")
        self.root.geometry("600x450")
        self.root.title("🐱 CAT OS - Установка")
        
        main = Frame(self.root, bg="#0a0f2c")
        main.pack(expand=True)
        
        Label(main, text="🐱 ДОБРО ПОЖАЛОВАТЬ В CAT OS!", bg="#0a0f2c", fg="#f9e2af",
              font=("Segoe UI", 20, "bold")).pack(pady=40)
        
        Label(main, text="Космическая операционная система для настоящих котиков", 
              bg="#0a0f2c", fg="#a6e3a1", font=("Segoe UI", 11)).pack(pady=10)
        
        Label(main, text="Нажми СТАРТ для начала", bg="#0a0f2c", fg="#cdd6f4", font=("Segoe UI", 12)).pack(pady=20)
        
        Button(main, text="🚀 СТАРТ", command=self.finish,
               bg="#89b4fa", fg="#1a1f3c", font=("Segoe UI", 14, "bold"),
               relief=FLAT, cursor="hand2", padx=50, pady=12).pack(pady=30)
    
    def finish(self):
        SoundPlayer.click()
        settings = load_settings()
        settings['first_run'] = False
        save_settings(settings)
        self.on_finish()

class BootScreen:
    def __init__(self, root, on_finish):
        self.root = root
        self.on_finish = on_finish
        self.show()
    
    def show(self):
        self.root.configure(bg="#0a0f2c")
        self.root.overrideredirect(True)
        
        w, h = 500, 400
        sw = self.root.winfo_screenwidth()
        sh = self.root.winfo_screenheight()
        self.root.geometry(f"{w}x{h}+{sw//2 - w//2}+{sh//2 - h//2}")
        
        canvas = Canvas(self.root, width=w, height=h, bg="#0a0f2c", highlightthickness=0)
        canvas.pack()
        
        canvas.create_text(w//2, 100, text="🐱", font=("Segoe UI", 60), fill="#f9e2af")
        canvas.create_text(w//2, 170, text="CAT OS", font=("Segoe UI", 28, "bold"), fill="#89b4fa")
        canvas.create_text(w//2, 220, text="Космическая версия", font=("Segoe UI", 12), fill="#a6e3a1")
        
        self.spinner_active = True
        self.spinner_angle = 0
        self.spinner_canvas = canvas
        self.animate_spinner()
        
        self.root.after(2500, self.finish)
    
    def animate_spinner(self):
        if not self.spinner_active:
            return
        self.spinner_angle = (self.spinner_angle + 15) % 360
        self.spinner_canvas.delete("spinner_dot")
        for i in range(8):
            rad = self.spinner_angle + i * 45
            x = 250 + 20 * math.cos(math.radians(rad))
            y = 305 + 20 * math.sin(math.radians(rad))
            self.spinner_canvas.create_oval(x-2, y-2, x+2, y+2, fill="#89b4fa", tags="spinner_dot")
        self.root.after(50, self.animate_spinner)
    
    def finish(self):
        self.spinner_active = False
        self.root.overrideredirect(False)
        self.root.geometry("1300x750")
        sw = self.root.winfo_screenwidth()
        sh = self.root.winfo_screenheight()
        self.root.geometry(f"{1300}x{750}+{sw//2 - 650}+{sh//2 - 375}")
        self.on_finish()

class CATOS:
    def __init__(self):
        self.root = Tk()
        self.root.title("CAT OS")
    
    def run(self):
        settings = load_settings()
        if settings.get("first_run", True):
            BootScreen(self.root, self.show_setup)
        else:
            self.show_login()
        self.root.mainloop()
    
    def show_setup(self):
        SetupWizard(self.root, self.show_login)
    
    def show_login(self):
        LoginScreen(self.root, self.show_desktop)
    
    def show_desktop(self, username):
        for widget in self.root.winfo_children():
            widget.destroy()
        settings = load_settings()
        self.desktop = Desktop(self.root, settings, username)


if __name__ == "__main__":
    os_system = CATOS()
    os_system.run()
