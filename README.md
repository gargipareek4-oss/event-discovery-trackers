# event-discovery-trackers
Automated event discovery and tracking tool with scraping, scheduling, and Excel-based storage.
#!/usr/bin/env python3
"""
Event Discovery & Tracking Tool – Desktop app.
Double-click or run: python app.py
"""
import os
import sys
import threading
from pathlib import Path

# Run from project root
PROJECT_ROOT = Path(__file__).resolve().parent
os.chdir(PROJECT_ROOT)
sys.path.insert(0, str(PROJECT_ROOT))

import tkinter as tk
from tkinter import ttk, messagebox, scrolledtext, filedialog


def get_cities():
    """Return list of city names for dropdown."""
    from src.extractors.bookmyshow import BookMyShowExtractor
    extractor = BookMyShowExtractor()
    cities = extractor.get_cities()
    return [c["name"] for c in cities]


def run_fetch(city: str, output_file: str, log_widget=None) -> tuple[bool, str]:
    """Fetch events and update Excel. Returns (success, message)."""
    from src.extractors.bookmyshow import BookMyShowExtractor
    from src.storage.excel_store import ExcelEventStore
    try:
        extractor = BookMyShowExtractor()
        events = extractor.fetch_events(city)
        store = ExcelEventStore(output_file)
        store.upsert(events, mark_expired=True)
        msg = f"Done. Fetched {len(events)} events for {city}. Updated {output_file}"
        return True, msg
    except Exception as e:
        return False, str(e)


def open_excel(path: str):
    """Open the Excel file with default app."""
    path = Path(path)
    if not path.exists():
        return False
    if sys.platform == "win32":
        os.startfile(path)
    elif sys.platform == "darwin":
        os.system(f'open "{path}"')
    else:
        os.system(f'xdg-open "{path}"')
    return True


def build_ui():
    root = tk.Tk()
    root.title("Event Discovery & Tracking – Pixie")
    root.geometry("640x420")
    root.minsize(520, 360)

    # Modern-looking themed widgets
    style = ttk.Style(root)
    try:
        # Use a nicer theme when available
        if "clam" in style.theme_names():
            style.theme_use("clam")
    except Exception:
        pass

    style.configure("TFrame", background="#0f172a")
    style.configure("Card.TFrame", background="#0b1120", relief="flat")
    style.configure("Header.TLabel", font=("Segoe UI", 14, "bold"), foreground="#e5e7eb", background="#0f172a")
    style.configure("SubHeader.TLabel", font=("Segoe UI", 9), foreground="#9ca3af", background="#0f172a")
    style.configure("TLabel", foreground="#e5e7eb", background="#0b1120")
    style.configure("Accent.TButton", font=("Segoe UI", 9, "bold"))

    # Config
    from src.config import load_config
    cfg = load_config()
    default_city = cfg.get("city") or "Jaipur"
    output_file = cfg.get("output_file") or "events.xlsx"
    output_path = Path(output_file)
    if not output_path.is_absolute():
        output_path = PROJECT_ROOT / output_path

    # Main background
    main = ttk.Frame(root, padding=16, style="TFrame")
    main.pack(fill=tk.BOTH, expand=True)

    # Header
    header = ttk.Frame(main, style="TFrame")
    header.grid(row=0, column=0, columnspan=2, sticky=tk.EW)
    ttk.Label(header, text="Event Discovery & Tracking", style="Header.TLabel").grid(
        row=0, column=0, sticky=tk.W
    )
    ttk.Label(
        header,
        text="Select a city, then fetch & review upcoming events.",
        style="SubHeader.TLabel",
    ).grid(row=1, column=0, sticky=tk.W, pady=(4, 0))

    # Card container
    card = ttk.Frame(main, padding=12, style="Card.TFrame")
    card.grid(row=1, column=0, columnspan=2, sticky=tk.NSEW, pady=(16, 8))
    main.rowconfigure(1, weight=1)
    main.columnconfigure(0, weight=1)

    # --- Settings area ---
    settings = ttk.LabelFrame(card, text=" Settings ", padding=10)
    settings.grid(row=0, column=0, columnspan=2, sticky=tk.EW, pady=(0, 10))

    # City
    ttk.Label(settings, text="City:").grid(row=0, column=0, sticky=tk.W, pady=(0, 4))
    city_var = tk.StringVar(value=default_city)
    cities = ["Jaipur", "Mumbai", "Delhi", "Bangalore", "Hyderabad", "Chennai", "Pune", "Kolkata", "Ahmedabad"]
    try:
        cities = get_cities()
    except Exception:
        pass
    city_combo = ttk.Combobox(settings, textvariable=city_var, values=cities, width=28, state="readonly")
    city_combo.grid(row=0, column=1, sticky=tk.EW, padx=(8, 0), pady=(0, 4))
    settings.columnconfigure(1, weight=1)

    # Output file
    ttk.Label(settings, text="Excel file:").grid(row=1, column=0, sticky=tk.W, pady=(0, 4))
    out_var = tk.StringVar(value=str(output_path))
    ttk.Entry(settings, textvariable=out_var, width=35).grid(
        row=1, column=1, sticky=tk.EW, padx=(8, 0), pady=(0, 4)
    )
    def browse():
        p = filedialog.asksaveasfilename(
            defaultextension=".xlsx",
            filetypes=[("Excel", "*.xlsx"), ("All", "*.*")],
            initialfile=Path(out_var.get()).name,
        )
        if p:
            out_var.set(p)
    ttk.Button(settings, text="Browse…", command=browse).grid(
        row=1, column=2, padx=(4, 0), pady=(0, 4)
    )

    # --- Actions ---
    actions = ttk.Frame(card)
    actions.grid(row=1, column=0, columnspan=2, sticky=tk.EW, pady=(0, 10))
    card.columnconfigure(0, weight=1)

    # Progress bar
    progress = ttk.Progressbar(actions, mode="indeterminate", length=150)
    progress.grid(row=0, column=0, sticky=tk.W)

    # Buttons row
    # (Buttons created later, just reserve placement)

    # --- Log area ---
    log_frame = ttk.LabelFrame(card, text=" Activity ", padding=6)
    log_frame.grid(row=2, column=0, columnspan=2, sticky=tk.NSEW)
    card.rowconfigure(2, weight=1)

    log = scrolledtext.ScrolledText(
        log_frame,
        height=8,
        width=60,
        state=tk.DISABLED,
        wrap=tk.WORD,
    )
    log.grid(row=0, column=0, columnspan=2, sticky=tk.NSEW)
    log_frame.rowconfigure(0, weight=1)
    log_frame.columnconfigure(0, weight=1)

    def log_msg(msg: str):
        log.config(state=tk.NORMAL)
        log.insert(tk.END, msg + "\n")
        log.see(tk.END)
        log.config(state=tk.DISABLED)

    # --- Fetch & open actions ---
    def do_fetch():
        city = city_var.get().strip()
        out = out_var.get().strip()
        if not city:
            messagebox.showwarning("City required", "Please select a city.")
            return
        if not out:
            messagebox.showwarning("Output required", "Please set the Excel file path.")
            return
        log_msg(f"Fetching events for {city}…")
        btn_fetch.config(state=tk.DISABLED)
        btn_open.config(state=tk.DISABLED)
        progress.start(12)
        def run():
            ok, msg = run_fetch(city, out, log)
            root.after(0, lambda: _done_fetch(ok, msg))
        threading.Thread(target=run, daemon=True).start()

    def _done_fetch(ok: bool, msg: str):
        progress.stop()
        btn_fetch.config(state=tk.NORMAL)
        btn_open.config(state=tk.NORMAL)
        log_msg(msg)
        if ok:
            log_msg("You can open the Excel file with the button below.")
        else:
            messagebox.showerror("Error", msg)

    btn_fetch = ttk.Button(actions, text="Fetch events now", style="Accent.TButton", command=do_fetch)
    btn_fetch.grid(row=0, column=1, padx=(12, 0), pady=(0, 4), sticky=tk.W)

    # Open Excel button
    def do_open():
        p = out_var.get().strip()
        if open_excel(p):
            log_msg(f"Opened {p}")
        else:
            messagebox.showinfo("File not found", f"File not found:\n{p}\nRun Fetch first.")
    btn_open = ttk.Button(actions, text="Open Excel file", command=do_open)
    btn_open.grid(row=0, column=2, padx=(8, 0), pady=(0, 4), sticky=tk.E)

    # Status bar
    status_var = tk.StringVar(value="Ready. Select a city and click Fetch.")
    status = ttk.Label(root, textvariable=status_var, anchor="w", padding=(8, 3))
    status.pack(fill=tk.X, side=tk.BOTTOM)

    def log_msg(msg: str):
        log.config(state=tk.NORMAL)
        log.insert(tk.END, msg + "\n")
        log.see(tk.END)
        log.config(state=tk.DISABLED)
        status_var.set(msg)

    log_msg("Ready. Select a city and click 'Fetch events now'.")

    root.mainloop()


if __name__ == "__main__":
    build_ui()
