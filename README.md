# filtro-retrato-opencv
Aplicación en Python que aplica un filtro de retrato en tiempo real usando OpenCV, morfología matemática, y procesamiento de imágenes.
# Importar las librerías necesarias para el procesamiento de imágenes, la interfaz gráfica y el manejo de hilos
import cv2
import numpy as np
import tkinter as tk
from tkinter import ttk
from threading import Thread
from PIL import Image, ImageTk

# Clase principal de la aplicación
class FiltroRetratoApp:
    def __init__(self):
        # Configuración inicial de la ventana
        self.root = tk.Tk()
        self.root.title("Filtro de Retrato en Tiempo Real")
        self.root.state('zoomed')  # abre la ventana maximizada
        self.root.configure(bg='#E3F2FD')  # color de fondo pastel

        # Resolución del video
        self.width, self.height = 640, 480

        # Variables para los filtros
        self.desenfoque = tk.IntVar(value=15)
        self.gris = tk.BooleanVar()
        self.erosionar = tk.BooleanVar()
        self.ecualizar = tk.BooleanVar()
        self.brillo = tk.IntVar(value=0)
        self.contraste = tk.DoubleVar(value=1.0)

        # Variables de estado
        self.cap = None
        self.running = False
        self.fondo = None
        self.frame_actual = None
        self.frame_filtrado = None

        # Llamadas a funciones de configuración
        self.setup_styles()
        self.setup_ui()

        # Cierra correctamente la cámara al cerrar ventana
        self.root.protocol("WM_DELETE_WINDOW", self.cerrar)
        self.root.mainloop()

    # Configura estilos de la interfaz usando colores armónicos
    def setup_styles(self):
        estilo = ttk.Style()
        estilo.theme_use('clam')

        # Paleta de colores y fuentes
        self.color_fondo = '#E3F2FD'
        self.color_panel = '#FFFFFF'
        self.color_texto = '#263238'
        self.color_boton = '#64B5F6'
        self.color_boton_hover = '#42A5F5'
        self.color_borde = '#B0BEC5'

        self.fuente_general = ('Segoe UI', 11)
        self.fuente_titulo = ('Segoe UI Semibold', 22)
        self.fuente_labelframe = ('Segoe UI Semibold', 13)
        self.fuente_boton = ('Segoe UI Semibold', 12)

        # Aplicar los estilos a los widgets
        estilo.configure('TFrame', background=self.color_fondo)
        estilo.configure('TLabelframe', background=self.color_panel, foreground=self.color_texto, font=self.fuente_labelframe, bordercolor=self.color_borde, borderwidth=2)
        estilo.configure('TLabelframe.Label', background=self.color_panel, foreground=self.color_texto)
        estilo.configure('TLabel', background=self.color_panel, foreground=self.color_texto, font=self.fuente_general)
        estilo.configure('TCheckbutton', background=self.color_panel, foreground=self.color_texto)
        estilo.configure('TScale', background=self.color_panel)
        estilo.configure('TButton', font=self.fuente_boton, foreground='white', background=self.color_boton, borderwidth=0, padding=8)

        estilo.map('TButton',
            background=[('active', self.color_boton_hover), ('!active', self.color_boton)],
            foreground=[('disabled', '#90A4AE'), ('!disabled', 'white')],
            relief=[('pressed', 'sunken'), ('!pressed', 'flat')])

    # Configura toda la interfaz de usuario
    def setup_ui(self):
        # Título principal
        titulo = ttk.Label(self.root, text="FILTRO DE RETRATO EN TIEMPO REAL", font=self.fuente_titulo, background=self.color_fondo, foreground=self.color_texto)
        titulo.pack(pady=25)

        # Frame principal que contiene todo
        main_frame = ttk.Frame(self.root)
        main_frame.pack(expand=True, fill=tk.BOTH, padx=30, pady=15)
        main_frame.columnconfigure(0, weight=1, uniform='a')
        main_frame.columnconfigure(1, weight=2, uniform='a')

        # Panel lateral de controles
        controles_frame = ttk.Labelframe(main_frame, text="Controles de Filtro", padding=20)
        controles_frame.grid(row=0, column=0, sticky='nswe', padx=(0,20), pady=10)
        controles_frame.columnconfigure(0, weight=1)

        # Sliders para filtros
        ttk.Label(controles_frame, text="Nivel de desenfoque:").grid(row=0, column=0, sticky='w', pady=(0,8))
        ttk.Scale(controles_frame, from_=1, to=51, variable=self.desenfoque, orient=tk.HORIZONTAL).grid(row=1, column=0, sticky='we', pady=(0,15))

        ttk.Label(controles_frame, text="Brillo:").grid(row=2, column=0, sticky='w', pady=(0,8))
        ttk.Scale(controles_frame, from_=-100, to=100, variable=self.brillo, orient=tk.HORIZONTAL).grid(row=3, column=0, sticky='we', pady=(0,15))

        ttk.Label(controles_frame, text="Contraste:").grid(row=4, column=0, sticky='w', pady=(0,8))
        ttk.Scale(controles_frame, from_=0.5, to=3.0, variable=self.contraste, orient=tk.HORIZONTAL).grid(row=5, column=0, sticky='we', pady=(0,15))

        # Checkboxes con opciones adicionales
        opciones_frame = ttk.Labelframe(controles_frame, text="Opciones Adicionales", padding=15)
        opciones_frame.grid(row=6, column=0, sticky='we', pady=(20,0))

        ttk.Checkbutton(opciones_frame, text="Escala de grises", variable=self.gris).pack(anchor='w', pady=6)
        ttk.Checkbutton(opciones_frame, text="Erosionar imagen", variable=self.erosionar).pack(anchor='w', pady=6)
        ttk.Checkbutton(opciones_frame, text="Ecualizar histograma", variable=self.ecualizar).pack(anchor='w', pady=6)

        # Botones de control de cámara
        botones_frame = ttk.Frame(controles_frame)
        botones_frame.grid(row=7, column=0, pady=25, sticky='we')
        botones_frame.columnconfigure((0,1,2), weight=1)

        ttk.Button(botones_frame, text="Iniciar Cámara", command=self.iniciar_video).grid(row=0, column=0, padx=5, sticky='we')
        ttk.Button(botones_frame, text="Detener Cámara", command=self.detener_video).grid(row=0, column=1, padx=5, sticky='we')
        ttk.Button(botones_frame, text="Guardar Imagen", command=self.capturar_imagen).grid(row=0, column=2, padx=5, sticky='we')

        # Panel derecho para mostrar video original y filtrado
        video_frame = ttk.Frame(main_frame, padding=10, relief='ridge', borderwidth=2, style='TFrame')
        video_frame.grid(row=0, column=1, sticky='nswe')
        video_frame.columnconfigure(0, weight=1)
        video_frame.columnconfigure(1, weight=1)
        video_frame.rowconfigure(0, weight=1)

        self.canvas_original = tk.Label(video_frame, bg='black', bd=2, relief='sunken', width=self.width, height=self.height)
        self.canvas_original.grid(row=0, column=0, padx=10, pady=10, sticky='nsew')

        self.canvas_filtrado = tk.Label(video_frame, bg='black', bd=2, relief='sunken', width=self.width, height=self.height)
        self.canvas_filtrado.grid(row=0, column=1, padx=10, pady=10, sticky='nsew')

        # Etiqueta de estado de la cámara
        self.estado_label = ttk.Label(self.root, text="Cámara: Detenida", font=('Segoe UI', 10, 'italic'), background=self.color_fondo, foreground='#FF5252')
        self.estado_label.pack(side='bottom', pady=8)

    # Inicia la captura de video
    def iniciar_video(self):
        if not self.running:
            self.cap = cv2.VideoCapture(0)
            self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, self.width)
            self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, self.height)
            self.running = True
            self.estado_label.config(text="Cámara: Activa", foreground='#388E3C')
            Thread(target=self.procesar_video, daemon=True).start()
            self.root.after(1000, self.capturar_fondo_automatico)
            self.actualizar_frame()

    # Detiene la cámara
    def detener_video(self):
        self.running = False
        self.estado_label.config(text="Cámara: Detenida", foreground='#FF5252')
        if self.cap:
            self.cap.release()
            self.cap = None

    # Captura el fondo automáticamente al iniciar la cámara
    def capturar_fondo_automatico(self):
        if self.cap and self.running:
            ret, frame = self.cap.read()
            if ret:
                self.fondo = cv2.resize(frame, (self.width, self.height))
                print("Fondo capturado automáticamente.")

    # Guarda la imagen procesada
    def capturar_imagen(self):
        if self.frame_filtrado is not None:
            nombre_archivo = "captura_filtro_retrato.png"
            cv2.imwrite(nombre_archivo, cv2.cvtColor(self.frame_filtrado, cv2.COLOR_BGR2RGB))
            print(f"Imagen guardada como {nombre_archivo}")

    # Hilo que procesa cada cuadro del video
    def procesar_video(self):
        while self.running and self.cap.isOpened():
            ret, frame = self.cap.read()
            if not ret:
                break
            frame = cv2.resize(frame, (self.width, self.height))
            self.frame_actual = frame.copy()

            if self.fondo is None:
                self.fondo = frame.copy()

            mask = self.calcular_mascara(frame, self.fondo)
            self.frame_filtrado = self.aplicar_filtros(frame, mask)

    # Calcula la máscara de diferencia entre fondo y persona
    def calcular_mascara(self, frame, fondo):
        diff = cv2.absdiff(frame, fondo)
        gris = cv2.cvtColor(diff, cv2.COLOR_BGR2GRAY)
        _, mask = cv2.threshold(gris, 30, 255, cv2.THRESH_BINARY)
        mask = cv2.medianBlur(mask, 7)
        return mask

    # Aplica los filtros seleccionados al cuadro actual
    def aplicar_filtros(self, frame, mask):
        ksize = self.desenfoque.get()
        if ksize % 2 == 0:
            ksize += 1
        ksize = max(1, ksize)

        _, mask_binaria = cv2.threshold(mask, 127, 255, cv2.THRESH_BINARY)
        mask_binaria = mask_binaria.astype(np.uint8)

        blurred = cv2.GaussianBlur(frame, (ksize, ksize), 0)

        mask_3ch = cv2.cvtColor(mask_binaria, cv2.COLOR_GRAY2BGR)
        inverted_mask_3ch = cv2.bitwise_not(mask_3ch)

        fondo_desenfocado = cv2.bitwise_and(blurred, inverted_mask_3ch)
        persona = cv2.bitwise_and(frame, mask_3ch)
        resultado = cv2.add(fondo_desenfocado, persona)

        # Aplicar ajustes
        resultado = cv2.convertScaleAbs(resultado, alpha=self.contraste.get(), beta=self.brillo.get())

        if self.gris.get():
            gris = cv2.cvtColor(resultado, cv2.COLOR_BGR2GRAY)
            resultado = cv2.cvtColor(gris, cv2.COLOR_GRAY2BGR)

        if self.erosionar.get():
            kernel = np.ones((5, 5), np.uint8)
            resultado = cv2.erode(resultado, kernel, iterations=1)

        if self.ecualizar.get():
            ycrcb = cv2.cvtColor(resultado, cv2.COLOR_BGR2YCrCb)
            ycrcb[:, :, 0] = cv2.equalizeHist(ycrcb[:, :, 0])
            resultado = cv2.cvtColor(ycrcb, cv2.COLOR_YCrCb2BGR)

        return resultado

    # Actualiza constantemente los cuadros del canvas
    def actualizar_frame(self):
        if self.running and self.frame_actual is not None:
            img_rgb = cv2.cvtColor(self.frame_actual, cv2.COLOR_BGR2RGB)
            img_pil = Image.fromarray(img_rgb)
            imgtk = ImageTk.PhotoImage(image=img_pil)
            self.canvas_original.imgtk = imgtk
            self.canvas_original.configure(image=imgtk)

            if self.frame_filtrado is not None:
                img_rgb_f = cv2.cvtColor(self.frame_filtrado, cv2.COLOR_BGR2RGB)
                img_pil_f = Image.fromarray(img_rgb_f)
                imgtk_f = ImageTk.PhotoImage(image=img_pil_f)
                self.canvas_filtrado.imgtk = imgtk_f
                self.canvas_filtrado.configure(image=imgtk_f)

        self.root.after(20, self.actualizar_frame)  # actualización cada 20ms

    # Cierra la aplicación correctamente
    def cerrar(self):
        self.running = False
        if self.cap:
            self.cap.release()
        self.root.destroy()

# Ejecutar la app
if __name__ == "__main__":
    app = FiltroRetratoApp()
