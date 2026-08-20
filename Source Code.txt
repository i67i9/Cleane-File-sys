import os
import sys
import shutil
import time
import threading
import ctypes
import subprocess
import customtkinter as ctk

def is_admin():
    try:
        return ctypes.windll.shell32.IsUserAnAdmin()
    except:
        return False

class ModernCleanerApp(ctk.CTk):
    def __init__(self):
        super().__init__()

        self.title("System Cleaner Pro")
        self.geometry("780x660")
        self.resizable(False, False)

        # Default Appearance
        ctk.set_appearance_mode("Dark")
        ctk.set_default_color_theme("blue")

        # Cleanup Paths
        self.user_profile = os.environ.get('USERPROFILE', '')
        self.local_appdata = os.environ.get('LOCALAPPDATA', '')
        self.windir = os.environ.get('WINDIR', 'C:\\Windows')

        self.targets = {
            "temp": [os.path.join(self.windir, 'Temp')],
            "%temp%": [os.path.join(self.local_appdata, 'Temp')],
            "prefetch": [os.path.join(self.windir, 'Prefetch')],
            "directx": [
                os.path.join(self.local_appdata, 'D3DSCache'),
                os.path.join(self.local_appdata, 'NVIDIA', 'DXCache'),
                os.path.join(self.local_appdata, 'AMD', 'DxCache')
            ],
            "delivery_opt": [os.path.join(self.windir, 'SoftwareDistribution', 'DeliveryOptimization')],
            "thumbnails": [os.path.join(self.local_appdata, 'Microsoft', 'Windows', 'Explorer')],
        }

        self.setup_ui()
        
        # Start background scan on launch
        threading.Thread(target=self.scan_total_files, daemon=True).start()

    def setup_ui(self):
        # 1. Top Bar
        self.top_bar = ctk.CTkFrame(self, fg_color="transparent")
        self.top_bar.pack(pady=(15, 5), padx=25, fill="x")

        self.title_label = ctk.CTkLabel(
            self.top_bar, 
            text="System Optimizer", 
            font=ctk.CTkFont(size=22, weight="bold")
        )
        self.title_label.pack(side="left")

        self.theme_switch = ctk.CTkSwitch(
            self.top_bar, 
            text="Dark Mode", 
            command=self.toggle_theme,
            font=ctk.CTkFont(size=12, weight="bold")
        )
        self.theme_switch.pack(side="right")
        self.theme_switch.select()

        # 2. Hero Orange Card (File Counter & Status)
        self.hero_card = ctk.CTkFrame(self, fg_color="#FF6B00", corner_radius=18)
        self.hero_card.pack(pady=10, padx=25, fill="x")

        self.hero_title = ctk.CTkLabel(
            self.hero_card,
            text="TEMPORARY JUNK FILES DETECTED",
            font=ctk.CTkFont(size=12, weight="bold"),
            text_color="#FFE0B2"
        )
        self.hero_title.pack(anchor="w", padx=20, pady=(15, 2))

        self.file_count_label = ctk.CTkLabel(
            self.hero_card,
            text="Scanning files...",
            font=ctk.CTkFont(size=34, weight="bold"),
            text_color="#FFFFFF"
        )
        self.file_count_label.pack(anchor="w", padx=20, pady=(0, 2))

        self.hero_sub = ctk.CTkLabel(
            self.hero_card,
            text="Scan completed. Click clean to free up system resources.",
            font=ctk.CTkFont(size=12),
            text_color="#FFF3E0"
        )
        self.hero_sub.pack(anchor="w", padx=20, pady=(0, 15))

        # 3. Individual Clean Options Grid
        self.grid_frame = ctk.CTkFrame(self, fg_color="transparent")
        self.grid_frame.pack(pady=10, padx=20, fill="both", expand=True)

        categories = [
            ("Clean Temp", "temp"),
            ("Clean %Temp%", "%temp%"),
            ("Clean Prefetch", "prefetch"),
            ("Clean DirectX Shader", "directx"),
            ("Clean Delivery Opt.", "delivery_opt"),
            ("Clean Thumbnails", "thumbnails"),
            ("Empty Recycle Bin", "recycle_bin")
        ]

        row, col = 0, 0
        for name, key in categories:
            btn = ctk.CTkButton(
                self.grid_frame,
                text=name,
                font=ctk.CTkFont(size=13, weight="bold"),
                height=42,
                corner_radius=10,
                command=lambda k=key: self.start_clean_thread(k)
            )
            btn.grid(row=row, column=col, padx=8, pady=8, sticky="ew")
            col += 1
            if col > 1:
                col = 0
                row += 1

        self.grid_frame.grid_columnconfigure(0, weight=1)
        self.grid_frame.grid_columnconfigure(1, weight=1)

        # 4. Action Buttons
        self.actions_frame = ctk.CTkFrame(self, fg_color="transparent")
        self.actions_frame.pack(pady=5, padx=25, fill="x")

        self.btn_clean_all = ctk.CTkButton(
            self.actions_frame,
            text="🔥 Full System Cleanup",
            fg_color="#D32F2F",
            hover_color="#9A0007",
            height=45,
            corner_radius=12,
            font=ctk.CTkFont(size=14, weight="bold"),
            command=lambda: self.start_clean_thread("all")
        )
        self.btn_clean_all.pack(side="left", padx=(0, 5), expand=True, fill="x")

        self.btn_christitus = ctk.CTkButton(
            self.actions_frame,
            text="🚀 Launch Chris Titus Tool",
            fg_color="#2E7D32",
            hover_color="#1B5E20",
            height=45,
            corner_radius=12,
            font=ctk.CTkFont(size=14, weight="bold"),
            command=self.run_chris_titus
        )
        self.btn_christitus.pack(side="right", padx=(5, 0), expand=True, fill="x")

        # 5. Status & Progress Card
        self.status_card = ctk.CTkFrame(self, corner_radius=12)
        self.status_card.pack(pady=(10, 20), padx=25, fill="x")

        self.status_label = ctk.CTkLabel(
            self.status_card, 
            text="Status: Idle", 
            font=ctk.CTkFont(size=12, weight="bold")
        )
        self.status_label.pack(anchor="w", padx=15, pady=(8, 2))

        self.progress_bar = ctk.CTkProgressBar(self.status_card, height=10, corner_radius=5)
        self.progress_bar.pack(fill="x", padx=15, pady=5)
        self.progress_bar.set(0)

        self.time_label = ctk.CTkLabel(
            self.status_card, 
            text="Time Remaining: --:--", 
            font=ctk.CTkFont(size=11),
            text_color="gray"
        )
        self.time_label.pack(anchor="e", padx=15, pady=(2, 8))

    def toggle_theme(self):
        if self.theme_switch.get() == 1:
            ctk.set_appearance_mode("Dark")
        else:
            ctk.set_appearance_mode("Light")

    def run_chris_titus(self):
        try:
            # استخدام أمر PowerShell مباشر ومتوافق لتخطي سياسات الحماية
            ps_cmd = "Set-ExecutionPolicy Bypass -Scope Process -Force; [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; iwr -useb https://christitus.com/win | iex"
            subprocess.Popen(["powershell.exe", "-NoExit", "-ExecutionPolicy", "Bypass", "-Command", ps_cmd])
        except Exception as e:
            print(f"Error launching Chris Titus tool: {e}")

    def empty_recycle_bin(self):
        try:
            ctypes.windll.shell32.SHEmptyRecycleBinW(None, None, 7)
        except Exception:
            pass

    def get_files_list(self, target_keys):
        files_to_delete = []
        for key in target_keys:
            if key in self.targets:
                for path in self.targets[key]:
                    if os.path.exists(path):
                        if key == "thumbnails":
                            for root, _, files in os.walk(path):
                                for f in files:
                                    if f.endswith('.db') and 'thumbcache' in f.lower():
                                        files_to_delete.append(os.path.join(root, f))
                        else:
                            for root, dirs, files in os.walk(path):
                                for f in files:
                                    files_to_delete.append(os.path.join(root, f))
                                for d in dirs:
                                    files_to_delete.append(os.path.join(root, d))
        return files_to_delete

    def scan_total_files(self):
        all_files = self.get_files_list(list(self.targets.keys()))
        total_count = len(all_files)
        self.file_count_label.configure(text=f"{total_count:,} Files Found")

    def start_clean_thread(self, target_type):
        threading.Thread(target=self.clean_process, args=(target_type,), daemon=True).start()

    def clean_process(self, target_type):
        self.progress_bar.set(0)
        self.status_label.configure(text="Status: Scanning files...")
        self.time_label.configure(text="Time Remaining: Calculating...")

        if target_type == "recycle_bin":
            self.empty_recycle_bin()
            self.progress_bar.set(1.0)
            self.status_label.configure(text="Status: Recycle Bin Emptied Successfully!")
            self.time_label.configure(text="Time Remaining: 00:00")
            return

        if target_type == "all":
            keys = list(self.targets.keys())
            self.empty_recycle_bin()
        else:
            keys = [target_type]

        items_to_delete = self.get_files_list(keys)
        total_items = len(items_to_delete)

        if total_items == 0:
            self.progress_bar.set(1.0)
            self.status_label.configure(text="Status: No Junk Files Found!")
            self.time_label.configure(text="Time Remaining: 00:00")
            self.scan_total_files()
            return

        start_time = time.time()
        deleted_count = 0

        for idx, item in enumerate(items_to_delete, 1):
            try:
                if os.path.isfile(item) or os.path.islink(item):
                    os.remove(item)
                elif os.path.isdir(item):
                    shutil.rmtree(item, ignore_errors=True)
                deleted_count += 1
            except Exception:
                pass

            progress = idx / total_items
            self.progress_bar.set(progress)

            elapsed_time = time.time() - start_time
            avg_time_per_item = elapsed_time / idx
            remaining_items = total_items - idx
            eta_seconds = int(remaining_items * avg_time_per_item)

            mins, secs = divmod(eta_seconds, 60)
            self.time_label.configure(text=f"Time Remaining: {mins:02d}m {secs:02d}s")
            self.status_label.configure(text=f"Status: Cleaning... ({idx}/{total_items})")

            time.sleep(0.001)

        self.progress_bar.set(1.0)
        self.status_label.configure(text=f"Status: Done! Cleaned {deleted_count:,} items.")
        self.time_label.configure(text="Time Remaining: 00:00")
        
        self.scan_total_files()

if __name__ == "__main__":
    if not is_admin():
        script_path = os.path.abspath(__file__)
        ctypes.windll.shell32.ShellExecuteW(
            None, "runas", sys.executable, f'"{script_path}"', None, 1
        )
        sys.exit(0)
    else:
        app = ModernCleanerApp()
        app.mainloop()