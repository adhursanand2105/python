#!/usr/bin/env python3
"""
Color-coded Notes App (Tkinter + SQLite)
Features:
- Create, edit, delete notes
- Assign colors (palette + custom color chooser)
- Search and filter by color
- Persist notes in SQLite (notes.db)
- Export notes to .txt or .md
- Keyboard shortcuts: Ctrl+N (new), Ctrl+S (save), Del (delete), Ctrl+E (export)
"""
import tkinter as tk
from tkinter import ttk, messagebox, filedialog, colorchooser
import sqlite3
import datetime
import os

DB_PATH = "notes.db"
PALETTE = [
    ("None", None),
    ("Red", "#ff6b6b"),
    ("Orange", "#ff9f43"),
    ("Yellow", "#ffd93d"),
    ("Green", "#4cd137"),
    ("Teal", "#34ace0"),
    ("Blue", "#5567ff"),
    ("Purple", "#9b59b6"),
    ("Pink", "#ff7ab6"),
    ("Gray", "#95a5a6"),
]

def now_iso():
    return datetime.datetime.utcnow().isoformat(sep=" ", timespec="seconds")

class NotesDB:
    def __init__(self, path=DB_PATH):
        self.path = path
        self.conn = sqlite3.connect(self.path)
        self._ensure_table()

    def _ensure_table(self):
        cur = self.conn.cursor()
        cur.execute("""
        CREATE TABLE IF NOT EXISTS notes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT,
            content TEXT,
            color TEXT,
            created TEXT,
            modified TEXT
        )
        """)
        self.conn.commit()

    def list_notes(self, search=None, color=None):
        cur = self.conn.cursor()
        sql = "SELECT id, title, color, created, modified FROM notes"
        params = []
        clauses = []
        if search:
            clauses.append("(title LIKE ? OR content LIKE ?)")
            params.extend([f"%{search}%", f"%{search}%"])
        if color and color != "None":
            clauses.append("color = ?")
            params.append(color)
        if clauses:
            sql += " WHERE " + " AND ".join(clauses)
        sql += " ORDER BY modified DESC"
        cur.execute(sql, params)
        return cur.fetchall()

    def get_note(self, note_id):
        cur = self.conn.cursor()
        cur.execute("SELECT id, title, content, color, created, modified FROM notes WHERE id = ?", (note_id,))
        return cur.fetchone()

    def create_note(self, title="New note", content="", color=None):
        cur = self.conn.cursor()
        t = now_iso()
        cur.execute("INSERT INTO notes (title, content, color, created, modified) VALUES (?, ?, ?, ?, ?)",
                    (title, content, color, t, t))
        self.conn.commit()
        return cur.lastrowid

    def update_note(self, note_id, title, content, color):
        cur = self.conn.cursor()
        t = now_iso()
        cur.execute("UPDATE notes SET title = ?, content = ?, color = ?, modified = ? WHERE id = ?",
                    (title, content, color, t, note_id))
        self.conn.commit()

    def delete_note(self, note_id):
        cur = self.conn.cursor()
        cur.execute("DELETE FROM notes WHERE id = ?", (note_id,))
        self.conn.commit()

    def export_note_to_file(self, note_id, path):
        note = self.get_note(note_id)
        if not note:
            raise ValueError("Note not found")
        _, title, content, color, created, modified = note
        with open(path, "w", encoding="utf-8") as f:
            f.write(f"# {title}\n\n")
            f.write(content)
            f.write("\n")
        return path

class ColorNotesApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Color Notes — v1")
        self.geometry("1000x640")
        self.db = NotesDB()
        self.current_id = None
        self._build_ui()
        self._bind_shortcuts()
        self._load_list()

    def _build_ui(self):
        # Top toolbar
        toolbar = ttk.Frame(self)
        toolbar.pack(side=tk.TOP, fill=tk.X, padx=4, pady=4)

        btn_new = ttk.Button(toolbar, text="New (Ctrl+N)", command=self.new_note)
        btn_delete = ttk.Button(toolbar, text="Delete (Del)", command=self.delete_note)
        btn_save = ttk.Button(toolbar, text="Save (Ctrl+S)", command=self.save_note)
        btn_export = ttk.Button(toolbar, text="Export (Ctrl+E)", command=self.export_note)

        btn_new.pack(side=tk.LEFT, padx=2)
        btn_delete.pack(side=tk.LEFT, padx=2)
        btn_save.pack(side=tk.LEFT, padx=2)
        btn_export.pack(side=tk.LEFT, padx=2)

        # Search & filter
        ttk.Label(toolbar, text=" Search:").pack(side=tk.LEFT, padx=(12,0))
        self.search_var = tk.StringVar()
        search_entry = ttk.Entry(toolbar, textvariable=self.search_var, width=30)
        search_entry.pack(side=tk.LEFT, padx=4)
        search_entry.bind("<Return>", lambda e: self._load_list())

        ttk.Label(toolbar, text=" Filter:").pack(side=tk.LEFT, padx=(12,0))
        self.filter_var = tk.StringVar(value="All")
        colors = ["All"] + [name for name,_ in PALETTE]
        filter_combo = ttk.Combobox(toolbar, values=colors, state="readonly", width=12, textvariable=self.filter_var)
        filter_combo.pack(side=tk.LEFT, padx=4)
        filter_combo.bind("<<ComboboxSelected>>", lambda e: self._load_list())

        btn_refresh = ttk.Button(toolbar, text="Refresh", command=self._load_list)
        btn_refresh.pack(side=tk.LEFT, padx=8)

        # Main paned window
        paned = ttk.PanedWindow(self, orient=tk.HORIZONTAL)
        paned.pack(fill=tk.BOTH, expand=1, padx=6, pady=(0,6))

        # Left: notes list
        left_frame = ttk.Frame(paned, width=320)
        paned.add(left_frame, weight=1)

        self.tree = ttk.Treeview(left_frame, columns=("title","color","modified"), show="headings", selectmode="browse", height=25)
        self.tree.heading("title", text="Title")
        self.tree.heading("color", text="Color")
        self.tree.heading("modified", text="Modified")
        self.tree.column("title", width=200)
        self.tree.column("color", width=70, anchor="center")
        self.tree.column("modified", width=110, anchor="center")
        self.tree.pack(fill=tk.BOTH, expand=1, side=tk.LEFT)
        self.tree.bind("<<TreeviewSelect>>", lambda e: self._on_select())

        # add scrollbar
        vs = ttk.Scrollbar(left_frame, orient="vertical", command=self.tree.yview)
        vs.pack(side=tk.RIGHT, fill=tk.Y)
        self.tree.configure(yscrollcommand=vs.set)

        # Right: editor
        right_frame = ttk.Frame(paned)
        paned.add(right_frame, weight=3)

        # Title entry
        ttk.Label(right_frame, text="Title").pack(anchor="w", padx=6, pady=(6,0))
        self.title_var = tk.StringVar()
        self.title_entry = ttk.Entry(right_frame, textvariable=self.title_var, font=("TkDefaultFont", 12, "bold"))
        self.title_entry.pack(fill=tk.X, padx=6)

        # Color buttons / picker
        color_bar = ttk.Frame(right_frame)
        color_bar.pack(fill=tk.X, padx=6, pady=6)
        ttk.Label(color_bar, text="Color:").pack(side=tk.LEFT)
        self.selected_color = None
        for name, hexv in PALETTE:
            display = name if hexv is None else ""
            b = tk.Button(color_bar, text=display, width=3, relief=tk.FLAT,
                          command=lambda c=hexv: self._set_selected_color(c))
            if hexv:
                b.configure(bg=hexv, activebackground=hexv)
            b.pack(side=tk.LEFT, padx=2)
        ttk.Button(color_bar, text="Custom...", command=self._choose_custom_color).pack(side=tk.LEFT, padx=(6,0))

        # Content Text area
        ttk.Label(right_frame, text="Content").pack(anchor="w", padx=6)
        self.content_text = tk.Text(right_frame, wrap="word", undo=True)
        self.content_text.pack(fill=tk.BOTH, expand=1, padx=6, pady=(0,6))
        self.content_text.bind("<<Modified>>", self._on_text_modified)

        # Status bar
        self.status_var = tk.StringVar(value="Ready")
        status = ttk.Label(self, textvariable=self.status_var, relief=tk.SUNKEN, anchor="w")
        status.pack(side=tk.BOTTOM, fill=tk.X)

    # -- DB / UI sync --
    def _load_list(self):
        search = self.search_var.get().strip() or None
        filter_name = self.filter_var.get()
        color_filter = None
        if filter_name and filter_name != "All":
            # map palette name -> color hex
            for n,h in PALETTE:
                if n == filter_name:
                    color_filter = h
                    break
        notes = self.db.list_notes(search=search, color=color_filter)
        # clear tree
        for r in self.tree.get_children():
            self.tree.delete(r)
        for nid, title, color, created, modified in notes:
            color_display = color if color else "None"
            item = self.tree.insert("", "end", iid=str(nid), values=(title, color_display, modified))
            # tag style: apply background to color column by setting tags and styles
            if color:
                # Make a tag for this color if not exists
                tagname = f"c_{color.replace('#','')}"
                if not tagname in self.tree.tag_has(tagname):
                    # define row style using tag — style through tags needs map in ttk; fallback: we will set tags and set the 'foreground' maybe
                    self.tree.tag_configure(tagname, background=color)
                self.tree.item(item, tags=(tagname,))

        self.status_var.set(f"Loaded {len(notes)} notes")

    def _on_select(self):
        sel = self.tree.selection()
        if not sel:
            return
        nid = int(sel[0])
        note = self.db.get_note(nid)
        if note:
            self.current_id = nid
            _, title, content, color, created, modified = note
            self.title_var.set(title or "")
            self.content_text.delete("1.0", tk.END)
            self.content_text.insert("1.0", content or "")
            self.selected_color = color
            self._apply_content_bg()
            self.status_var.set(f"Loaded note {nid}")

    def new_note(self):
        nid = self.db.create_note(title="New note", content="", color=None)
        self._load_list()
        # select it
        self.tree.selection_set(str(nid))
        self.tree.see(str(nid))
        self._on_select()

    def delete_note(self):
        sel = self.tree.selection()
        if not sel:
            messagebox.showinfo("Delete", "No note selected.")
            return
        nid = int(sel[0])
        if messagebox.askyesno("Delete", "Delete the selected note?"):
            self.db.delete_note(nid)
            self.current_id = None
            self.title_var.set("")
            self.content_text.delete("1.0", tk.END)
            self.selected_color = None
            self._load_list()
            self.status_var.set("Deleted note")

    def save_note(self):
        title = self.title_var.get().strip() or "Untitled"
        content = self.content_text.get("1.0", tk.END).rstrip("\n")
        color = self.selected_color
        if self.current_id:
            self.db.update_note(self.current_id, title, content, color)
            self.status_var.set("Saved")
            self._load_list()
        else:
            nid = self.db.create_note(title=title, content=content, color=color)
            self.current_id = nid
            self._load_list()
            self.tree.selection_set(str(nid))
            self.status_var.set("Saved (new)")

    def export_note(self):
        if not self.current_id:
            messagebox.showinfo("Export", "No note selected.")
            return
        note = self.db.get_note(self.current_id)
        if not note:
            messagebox.showerror("Export", "Selected note not found.")
            return
        title = note[1] or "note"
        default_name = title.replace(" ", "_")[:40]
        path = filedialog.asksaveasfilename(defaultextension=".md", initialfile=default_name + ".md",
                                            filetypes=[("Markdown", "*.md"), ("Text", "*.txt"), ("All", "*.*")])
        if path:
            try:
                self.db.export_note_to_file(self.current_id, path)
                self.status_var.set(f"Exported to {os.path.basename(path)}")
            except Exception as e:
                messagebox.showerror("Export error", str(e))

    # -- color helpers --
    def _set_selected_color(self, hexcolor):
        # hexcolor may be None for "None"
        self.selected_color = hexcolor
        self._apply_content_bg()
        self.status_var.set(f"Color set: {hexcolor or 'None'}")

    def _choose_custom_color(self):
        c = colorchooser.askcolor(title="Choose note color")
        if c and c[1]:
            self._set_selected_color(c[1])

    def _apply_content_bg(self):
        # set background of content text to selected color for visual cue (and adjust fg)
        if self.selected_color:
            try:
                self.content_text.config(bg=self.selected_color)
                # choose black/white foreground based on luminance
                r = int(self.selected_color[1:3],16)
                g = int(self.selected_color[3:5],16)
                b = int(self.selected_color[5:7],16)
                lum = 0.299*r + 0.587*g + 0.114*b
                fg = "#000000" if lum > 160 else "#ffffff"
                self.content_text.config(fg=fg)
            except Exception:
                pass
        else:
            # restore defaults
            self.content_text.config(bg="white", fg="black")

    # -- autosave / text modified --
    def _on_text_modified(self, event=None):
        # reset flag
        try:
            self.content_text.edit_modified(False)
        except Exception:
            pass
        # mark as unsaved in status
        self.status_var.set("Modified (unsaved)")
        # optionally autosave after inactivity: for simplicity, we won't autosave automatically here
        # but save on focus out or explicit save

    # -- shortcuts --
    def _bind_shortcuts(self):
        self.bind_all("<Control-n>", lambda e: self.new_note())
        self.bind_all("<Control-N>", lambda e: self.new_note())
        self.bind_all("<Control-s>", lambda e: self.save_note())
        self.bind_all("<Control-S>", lambda e: self.save_note())
        self.bind_all("<Delete>", lambda e: self.delete_note())
        self.bind_all("<Control-e>", lambda e: self.export_note())
        self.bind_all("<Control-E>", lambda e: self.export_note())
        # save on close
        self.protocol("WM_DELETE_WINDOW", self._on_close)

    def _on_close(self):
        if messagebox.askyesno("Quit", "Save before quitting?"):
            self.save_note()
        self.destroy()


if __name__ == "__main__":
    app = ColorNotesApp()
    app.mainloop()