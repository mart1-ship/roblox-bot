import time
import re
import pyautogui
import pytesseract
from PIL import ImageGrab, Image
import tkinter as tk
from tkinter import ttk
import threading
import sys

pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

# ============================================================
#  CONFIGURATION — MODIFIE CES VALEURS SELON TON ÉCRAN
# ============================================================

# Zone de capture où apparaît "Prix actuel : XX$"
# Format : (x_gauche, y_haut, x_droite, y_bas)
# Lance le script une fois, regarde les coordonnées avec SETUP MODE
ZONE_PRIX = (769, 330, 1149, 369)   # <-- À AJUSTER

# Coordonnées du bouton "Vendre" (x, y)
BOUTON_VENDRE = (952, 777)           # <-- À AJUSTER

# Prix cibles qui déclenchent la vente
PRIX_CIBLES = [13, 14, 15]

# Délai entre chaque scan (en secondes)
DELAI_SCAN = 0.5

# ============================================================

running = False
log_lines = []


def capturer_prix(zone):
    """Capture une zone de l'écran et extrait le prix via OCR."""
    try:
        screenshot = pyautogui.screenshot(region=(zone[0], zone[1], zone[2]-zone[0], zone[3]-zone[1]))
    except Exception:
        screenshot = ImageGrab.grab(bbox=zone)
    # Agrandir pour améliorer l'OCR
    screenshot = screenshot.resize(
        (screenshot.width * 3, screenshot.height * 3),
        Image.LANCZOS
    )
    texte = pytesseract.image_to_string(screenshot, config='--psm 6')
    return texte


def extraire_prix(texte):
    """Cherche un nombre dans le texte OCR (ex: 'Prix actuel : 13$')."""
    # Cherche des patterns comme "13", "13$", "13,00", "13.00"
    matches = re.findall(r'\b(\d{1,3})[,.]?\d*\s*\$?', texte)
    for m in matches:
        try:
            return int(m)
        except ValueError:
            continue
    return None


def vendre():
    """Déplace la souris sur le bouton vendre et clique."""
    pyautogui.moveTo(BOUTON_VENDRE[0], BOUTON_VENDRE[1], duration=0.3)
    time.sleep(0.1)
    pyautogui.click()


def loop_scan(log_callback, status_callback):
    """Boucle principale de scan."""
    global running
    while running:
        try:
            texte = capturer_prix(ZONE_PRIX)
            prix = extraire_prix(texte)

            if prix is not None:
                log_callback(f"Prix détecté : {prix}$")
                status_callback(f"Prix actuel : {prix}$", "green")

                if prix in PRIX_CIBLES:
                    log_callback(f"✅ VENTE déclenchée à {prix}$ !")
                    status_callback(f"VENTE à {prix}$ !", "lime")
                    vendre()
                    time.sleep(2)
            else:
                log_callback(f"OCR: {texte.strip()[:40]!r} — prix non trouvé")
                status_callback("Scan en cours...", "yellow")

        except Exception as e:
            import traceback
            log_callback(f"Erreur: {traceback.format_exc()}")
            running = False

        time.sleep(DELAI_SCAN)


# ============================================================
#  INTERFACE GRAPHIQUE
# ============================================================

class App:
    def __init__(self, root):
        self.root = root
        root.title("🛢️ Oil Price Bot")
        root.geometry("500x450")
        root.configure(bg="#1a1a2e")
        root.resizable(False, False)

        # Titre
        tk.Label(root, text="🛢️ OIL PRICE BOT",
                 font=("Consolas", 18, "bold"),
                 bg="#1a1a2e", fg="#e94560").pack(pady=10)

        # Status
        self.status_var = tk.StringVar(value="En attente...")
        self.status_label = tk.Label(root, textvariable=self.status_var,
                                     font=("Consolas", 12),
                                     bg="#1a1a2e", fg="yellow")
        self.status_label.pack()

        # Info config
        info = (f"Zone scan: {ZONE_PRIX}\n"
                f"Bouton vendre: {BOUTON_VENDRE}\n"
                f"Prix cibles: {PRIX_CIBLES}")
        tk.Label(root, text=info, font=("Consolas", 9),
                 bg="#1a1a2e", fg="#aaa", justify="left").pack(pady=5)

        # Boutons Start/Stop
        btn_frame = tk.Frame(root, bg="#1a1a2e")
        btn_frame.pack(pady=5)

        self.btn_start = tk.Button(btn_frame, text="▶ START",
                                   command=self.start,
                                   font=("Consolas", 11, "bold"),
                                   bg="#16213e", fg="#00ff88",
                                   activebackground="#0f3460",
                                   relief="flat", padx=20, pady=8)
        self.btn_start.pack(side="left", padx=10)

        self.btn_stop = tk.Button(btn_frame, text="⏹ STOP",
                                  command=self.stop,
                                  font=("Consolas", 11, "bold"),
                                  bg="#16213e", fg="#e94560",
                                  activebackground="#0f3460",
                                  relief="flat", padx=20, pady=8,
                                  state="disabled")
        self.btn_stop.pack(side="left", padx=10)

        # Bouton pour trouver les coordonnées
        tk.Button(root, text="🖱️ Trouver coordonnées souris",
                  command=self.show_coords,
                  font=("Consolas", 9),
                  bg="#0f3460", fg="white",
                  relief="flat", padx=10, pady=4).pack(pady=3)

        # Log
        tk.Label(root, text="LOG:", font=("Consolas", 9),
                 bg="#1a1a2e", fg="#888").pack(anchor="w", padx=10)

        self.log_text = tk.Text(root, height=10, width=60,
                                font=("Consolas", 8),
                                bg="#0d0d1a", fg="#00ff88",
                                insertbackground="white",
                                relief="flat", state="disabled")
        self.log_text.pack(padx=10, pady=5)

    def log(self, msg):
        timestamp = time.strftime("%H:%M:%S")
        line = f"[{timestamp}] {msg}\n"
        self.log_text.config(state="normal")
        self.log_text.insert("end", line)
        self.log_text.see("end")
        self.log_text.config(state="disabled")

    def set_status(self, msg, color="yellow"):
        self.status_var.set(msg)
        self.status_label.config(fg=color)

    def start(self):
        global running
        running = True
        self.btn_start.config(state="disabled")
        self.btn_stop.config(state="normal")
        self.log("Bot démarré !")
        t = threading.Thread(
            target=loop_scan,
            args=(lambda m: self.root.after(0, self.log, m),
                  lambda m, c: self.root.after(0, self.set_status, m, c)),
            daemon=True
        )
        t.start()

    def stop(self):
        global running
        running = False
        self.btn_start.config(state="normal")
        self.btn_stop.config(state="disabled")
        self.log("Bot arrêté.")
        self.set_status("Arrêté", "red")

    def show_coords(self):
        """Affiche les coordonnées de la souris en temps réel pendant 5s."""
        win = tk.Toplevel(self.root)
        win.title("Coordonnées souris")
        win.geometry("250x100")
        win.configure(bg="#1a1a2e")
        lbl = tk.Label(win, text="", font=("Consolas", 14),
                       bg="#1a1a2e", fg="lime")
        lbl.pack(expand=True)
        tk.Label(win, text="Bouge ta souris sur l'écran",
                 font=("Consolas", 9), bg="#1a1a2e", fg="#aaa").pack()

        def update():
            x, y = pyautogui.position()
            lbl.config(text=f"X: {x}   Y: {y}")
            win.after(100, update)

        update()


if __name__ == "__main__":
    root = tk.Tk()
    app = App(root)
    root.mainloop()
