# Drawing-Pad
This is my first git hub mini project
import tkinter as tk
from tkinter import colorchooser, messagebox
from PIL import Image, ImageDraw, ImageTk
import os

# ─────────────────────────────────────────────
#  Drawing Pad  –  Tkinter + Canvas + PIL
# ─────────────────────────────────────────────

class DrawingPad:
    # ── Palette shown in the toolbar ──────────────────────────────────────
    PALETTE = [
        "#1a1a1a", "#e24b4a", "#ef9f27", "#f5e642",
        "#639922", "#1d9e75", "#378add", "#7f77dd",
        "#d4537e", "#ffffff",
    ]

    def __init__(self, root: tk.Tk):
        self.root = root
        self.root.title("🖌️  Drawing Pad")
        self.root.resizable(False, False)

        # ── State ─────────────────────────────────────────────────────────
        self.current_color = "#1a1a1a"
        self.brush_size    = tk.IntVar(value=5)
        self.eraser_on     = False
        self.last_x        = None
        self.last_y        = None
        self.history: list[Image.Image] = []   # undo stack (PIL images)

        # Canvas dimensions
        self.CW, self.CH = 800, 500

        # PIL image that mirrors the canvas (for saving + undo)
        self.pil_image  = Image.new("RGB", (self.CW, self.CH), "white")
        self.pil_draw   = ImageDraw.Draw(self.pil_image)

        self._build_ui()

    # ── UI Construction ────────────────────────────────────────────────────

    def _build_ui(self):
        # ── Toolbar ───────────────────────────────────────────────────────
        toolbar = tk.Frame(self.root, bg="#f5f5f0", pady=6, padx=8)
        toolbar.pack(fill=tk.X)

        # Color palette
        tk.Label(toolbar, text="Color:", bg="#f5f5f0", font=("Segoe UI", 9)).pack(side=tk.LEFT, padx=(0, 4))
        for hex_color in self.PALETTE:
            relief = "solid" if hex_color == self.current_color else "flat"
            btn = tk.Button(
                toolbar, bg=hex_color, width=2, height=1,
                relief=relief, bd=2, cursor="hand2",
                command=lambda c=hex_color: self._set_color(c),
            )
            btn.pack(side=tk.LEFT, padx=2)
            if hex_color == self.current_color:
                self._active_color_btn = btn

        # Custom color picker
        tk.Button(
            toolbar, text="⊕ Custom", font=("Segoe UI", 9),
            relief="flat", cursor="hand2", bg="#f5f5f0",
            command=self._pick_color,
        ).pack(side=tk.LEFT, padx=6)

        # Separator
        tk.Frame(toolbar, width=1, bg="#cccccc", height=24).pack(side=tk.LEFT, padx=6)

        # Brush size
        tk.Label(toolbar, text="Size:", bg="#f5f5f0", font=("Segoe UI", 9)).pack(side=tk.LEFT)
        size_slider = tk.Scale(
            toolbar, from_=1, to=50, orient=tk.HORIZONTAL,
            variable=self.brush_size, length=110,
            bg="#f5f5f0", bd=0, highlightthickness=0,
            troughcolor="#d0d0d0", font=("Segoe UI", 8),
        )
        size_slider.pack(side=tk.LEFT, padx=4)

        # Separator
        tk.Frame(toolbar, width=1, bg="#cccccc", height=24).pack(side=tk.LEFT, padx=6)

        # Action buttons
        for label, cmd in [
            ("🧹 Eraser", self._toggle_eraser),
            ("↩ Undo",   self._undo),
            ("🗑 Clear",  self._clear),
            ("💾 Save",   self._save),
        ]:
            tk.Button(
                toolbar, text=label, font=("Segoe UI", 9),
                relief="flat", cursor="hand2", bg="#f5f5f0",
                padx=6, pady=2, command=cmd,
            ).pack(side=tk.LEFT, padx=3)

        # ── Canvas ────────────────────────────────────────────────────────
        canvas_frame = tk.Frame(self.root, bg="#e8e8e8", padx=8, pady=8)
        canvas_frame.pack()

        self.canvas = tk.Canvas(
            canvas_frame, width=self.CW, height=self.CH,
            bg="white", cursor="crosshair",
            highlightthickness=1, highlightbackground="#cccccc",
        )
        self.canvas.pack()

        # ── Status bar ────────────────────────────────────────────────────
        self.status_var = tk.StringVar(value="✏️  Drawing  |  x: —  y: —")
        status = tk.Label(
            self.root, textvariable=self.status_var,
            bg="#eaeae5", font=("Segoe UI", 8),
            anchor="w", padx=10, pady=3,
        )
        status.pack(fill=tk.X)

        # ── Colour preview swatch (bottom-left of canvas frame) ───────────
        self.swatch = tk.Label(
            canvas_frame, bg=self.current_color,
            width=3, height=1, relief="solid", bd=1,
        )
        self.swatch.place(x=0, y=0)   # floated top-left of frame

        # ── Bind mouse events ─────────────────────────────────────────────
        self.canvas.bind("<ButtonPress-1>",   self._on_press)
        self.canvas.bind("<B1-Motion>",       self._on_drag)
        self.canvas.bind("<ButtonRelease-1>", self._on_release)
        self.canvas.bind("<Motion>",          self._on_hover)

    # ── Colour helpers ─────────────────────────────────────────────────────

    def _set_color(self, color: str):
        self.current_color = color
        self.eraser_on     = False
        self.swatch.config(bg=color)
        self.status_var.set(f"✏️  Drawing  |  color: {color}")

    def _pick_color(self):
        result = colorchooser.askcolor(color=self.current_color, title="Pick a colour")
        if result and result[1]:
            self._set_color(result[1])

    # ── Tool toggles ───────────────────────────────────────────────────────

    def _toggle_eraser(self):
        self.eraser_on = not self.eraser_on
        label = "🧹  Eraser active" if self.eraser_on else "✏️  Drawing"
        self.status_var.set(f"{label}  |  x: —  y: —")
        self.canvas.config(cursor="dotbox" if self.eraser_on else "crosshair")

    # ── Canvas / PIL drawing ───────────────────────────────────────────────

    def _stroke_color(self):
        return "white" if self.eraser_on else self.current_color

    def _draw_segment(self, x0, y0, x1, y1):
        """Draw on both the Tk canvas and the PIL mirror image."""
        size  = self.brush_size.get()
        color = self._stroke_color()
        r     = size // 2

        # Tkinter canvas
        self.canvas.create_line(
            x0, y0, x1, y1,
            fill=color, width=size,
            capstyle=tk.ROUND, smooth=True,
        )

        # PIL (for save / undo)
        self.pil_draw.line([(x0, y0), (x1, y1)], fill=color, width=size)
        # round caps
        self.pil_draw.ellipse([x1 - r, y1 - r, x1 + r, y1 + r], fill=color)

    def _dot(self, x, y):
        """Single tap / click — draw a filled circle."""
        size  = self.brush_size.get()
        color = self._stroke_color()
        r     = size // 2

        self.canvas.create_oval(
            x - r, y - r, x + r, y + r,
            fill=color, outline=color,
        )
        self.pil_draw.ellipse([x - r, y - r, x + r, y + r], fill=color)

    # ── Mouse event handlers ───────────────────────────────────────────────

    def _on_press(self, event):
        # Save state for undo before the stroke begins
        self.history.append(self.pil_image.copy())
        if len(self.history) > 30:
            self.history.pop(0)

        self.last_x, self.last_y = event.x, event.y
        self._dot(event.x, event.y)

    def _on_drag(self, event):
        if self.last_x is None:
            return
        self._draw_segment(self.last_x, self.last_y, event.x, event.y)
        self.last_x, self.last_y = event.x, event.y
        self.status_var.set(
            f"{'🧹  Eraser' if self.eraser_on else '✏️  Drawing'}  "
            f"|  x: {event.x}  y: {event.y}"
        )

    def _on_release(self, event):
        self.last_x = self.last_y = None

    def _on_hover(self, event):
        if self.last_x is None:   # not dragging
            self.status_var.set(
                f"{'🧹  Eraser' if self.eraser_on else '✏️  Drawing'}  "
                f"|  x: {event.x}  y: {event.y}"
            )

    # ── Undo / Clear ───────────────────────────────────────────────────────

    def _undo(self):
        if not self.history:
            return
        self.pil_image = self.history.pop()
        self.pil_draw  = ImageDraw.Draw(self.pil_image)
        # Re-draw canvas from PIL image
        self._redraw_canvas_from_pil()

    def _clear(self):
        if not messagebox.askyesno("Clear", "Clear the canvas?"):
            return
        self.history.append(self.pil_image.copy())
        self.canvas.delete("all")
        self.pil_image = Image.new("RGB", (self.CW, self.CH), "white")
        self.pil_draw  = ImageDraw.Draw(self.pil_image)

    def _redraw_canvas_from_pil(self):
        """Restore the Tk canvas from the PIL image (used by undo)."""
        self.canvas.delete("all")
        tk_img = ImageTk.PhotoImage(self.pil_image)
        self.canvas.create_image(0, 0, anchor=tk.NW, image=tk_img)
        # Keep a reference so GC doesn't collect it
        self.canvas._undo_img = tk_img

    # ── Save ───────────────────────────────────────────────────────────────

    def _save(self):
        from tkinter import filedialog
        path = filedialog.asksaveasfilename(
            defaultextension=".png",
            filetypes=[("PNG image", "*.png"), ("JPEG image", "*.jpg"), ("All files", "*.*")],
            title="Save drawing",
        )
        if path:
            self.pil_image.save(path)
            self.status_var.set(f"💾  Saved → {os.path.basename(path)}")


# ── Entry point ────────────────────────────────────────────────────────────

def main():
    try:
        from PIL import Image  # noqa – verify Pillow is available
    except ImportError:
        import sys, subprocess
        subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow", "-q"])

    root = tk.Tk()
    app  = DrawingPad(root)
    root.mainloop()


if __name__ == "__main__":
    main()
