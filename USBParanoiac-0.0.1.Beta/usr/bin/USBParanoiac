#!/usr/bin/env python3

# Aydınlatan ve geliştiren bilimin adıyla:

import sys
import os
import subprocess
import json
import time
from pathlib import Path
import shutil 
import configparser 
import fnmatch 
import hashlib 
import re
import signal
import threading 

# --- GNOME/Qt Platform Plugin Fix ---
os.environ['QT_QPA_PLATFORM_PLUGIN_PATH'] = ''
os.environ['QT_PLUGIN_PATH'] = ''

# PySide6 Kütüphaneleri
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QFileDialog, QHeaderView, QCheckBox, QProgressBar, QSplitter,
    QGroupBox, QComboBox, QSizePolicy, QMessageBox, QInputDialog,
    QTreeWidget, QTreeWidgetItem, QFileIconProvider
)
from PySide6.QtCore import Qt, QRunnable, QThreadPool, Signal, QObject, QCoreApplication, QThread
from PySide6.QtGui import QColor, QFont, QIcon, QPixmap, QFont
from PySide6.QtGui import QFont

# --- PATH Yardımcı Fonksiyonlar ---
def get_base_path():
    """Uygulamanın çalıştığı ana dizini döndürür. (İkonlar ve Dil dosyaları için kullanılır)"""
    try:
        return os.path.dirname(os.path.abspath(__file__))
    except Exception:
        return os.getcwd()

def get_config_base_path(app_name="USB Paranoiac"):
    """Kullanıcı yapılandırma dosyasının ana dizinini döndürür (~/.config/uygulama_adi)."""
    # XDG Base Directory Specification'a (Linux standardı) uygun olarak
    # varsayılan $HOME/.config olur.
    config_dir = os.environ.get('XDG_CONFIG_HOME')
    if not config_dir:
        config_dir = Path.home() / ".config"
        
    app_config_path = Path(config_dir) / app_name
    
    # Dizin mevcut değilse oluşturulur
    app_config_path.mkdir(parents=True, exist_ok=True)
    
    return str(app_config_path)

def get_icon_path(icon_filename="secure.png"):
    """
    İkon dosyasını arar: Önce uygulama dizininde, sonra sistem dizininde.
    """
    base_path = Path(get_base_path())
    local_path = base_path / icon_filename
    if local_path.exists():
        return str(local_path)
        
    system_path = Path("/usr/share/USBParanoiac/") / icon_filename
    if system_path.exists():
        return str(system_path)
        
    return "" 

# --- DİL YÖNETİMİ SINIFI ---
class LanguageManager:
    # İki ayrı yol alır: 
    # 1. app_base_path (Uygulama dizini, dil dosyaları için)
    # 2. config_base_path (Yapılandırma dizini, config.json için)
    def __init__(self, app_base_path, config_base_path): 
        # Dil dosyaları uygulama dizininden yüklenecek
        self.language_dir = Path(app_base_path) / "languages"
        
        # config.json, yapılandırma dizininde oluşturulacak/aranacak
        self.config_file = Path(config_base_path) / "config.json"
        
        self.current_language = self._load_user_language()
        self.strings = {}
        self.available_languages = self._get_available_languages()
        self.load_language(self.current_language)

    def _get_available_languages(self):
        """'languages' klasöründeki mevcut dilleri bulur ve {kod: Görünen Ad} olarak döndürür."""
        languages = {}
        if not self.language_dir.exists():
             return {"en": "English (Default)"}
             
        for file_path in self.language_dir.glob("*.ini"):
            lang_code = file_path.stem
            config = configparser.ConfigParser()
            try:
                config.read(file_path, encoding='utf-8')
                display_name = config.get("Language", "display_name", fallback=lang_code)
                languages[lang_code] = display_name
            except Exception:
                languages[lang_code] = lang_code 
        
        if not languages:
             languages = {"en": "English (Default)"}
        
        return languages

    def load_language(self, lang_code):
        """Belirtilen dil dosyasını yükler."""
        file_path = self.language_dir / f"{lang_code}.ini"
        config = configparser.ConfigParser()
        
        if not file_path.exists():
            print(f"HATA: Dil dosyası bulunamadı: {file_path}", file=sys.stderr)
            return False

        try:
            config.read(file_path, encoding='utf-8')
            new_strings = {}
            for section in config.sections():
                new_strings[section] = dict(config.items(section))
            
            self.strings = new_strings
            self.current_language = lang_code
            self._save_user_language(lang_code)
            print(f"Dil başarıyla yüklendi: {lang_code}")
            return True
        except Exception as e:
            print(f"KRİTİK HATA: Dil dosyası yüklenemedi: {file_path} - {e}", file=sys.stderr)
            return False

    def get_string(self, section, key):
        """Dil dosyasından dizeyi alır, bulunamazsa hata mesajı döndürür."""
        try:
            return self.strings[section][key]
        except KeyError:
            # ❗ YENİ DÜZELTME: Combobox dizesi sert kodlaması kaldırıldı. Kullanıcı dil dosyasını düzeltecektir.

            # Eksik dize anahtarları için hata önleyici varsayılanlar (KRİTİK HATALARI ENGELLER)
            if section == "Table" and key == "col_address":
                 return "Adres/Yol" 
            if section == "Status" and key == "no_usb_detected_info":
                 return "USB Aygıtı Tespit Edilemedi. Tarama Butonu Pasiftir."
            if section == "Status" and key == "language_load_error":
                 return "Dil yüklenirken hata oluştu."
            if section == "Button" and key == "stop_scan":
                 return "Durdur"
            if section == "DiskInfo" and key == "err_lsblk_code":
                 return "HATA: Disk Listeleme Komutu Başarısız"
            if section == "DiskInfo" and key == "err_lsblk_empty":
                 return "HATA: Disk Listeleme Komutu Boş Çıktı Verdi"
            if section == "DiskInfo" and key == "err_system_list":
                 return "HATA: Sistem Diskleri Listelenemedi"

                 
            return f"[{section}.{key} DİZE HATASI]"
    
    def _load_user_language(self):
        """Kullanıcının kaydettiği dil ayarını yükler."""
        try:
            if self.config_file.exists():
                with open(self.config_file, 'r', encoding='utf-8') as f:
                    config = json.load(f)
                    return config.get('language', 'en')
        except Exception:
            pass
        return 'en'  # Varsayılan İngilizce
    
    def _save_user_language(self, lang_code):
        """Kullanıcının dil ayarını JSON dosyasına kaydeder."""
        try:
            config = {}
            if self.config_file.exists():
                with open(self.config_file, 'r', encoding='utf-8') as f:
                    config = json.load(f)
            
            config['language'] = lang_code
            
            with open(self.config_file, 'w', encoding='utf-8') as f:
                json.dump(config, f, indent=2, ensure_ascii=False)
        except Exception as e:
            print(f"Dil ayarı kaydedilemedi: {e}", file=sys.stderr)

# Global dil yöneticisi nesnesi oluşturulur
APP_BASE_PATH = get_base_path() # Uygulamanın kurulu olduğu yol
CONFIG_BASE_PATH = get_config_base_path() # Kullanıcı config yolunun bulunduğu yer

LANG = LanguageManager(APP_BASE_PATH, CONFIG_BASE_PATH)

# --- Dosya Taşıma/Silme Fonksiyonu (GÜVENLİK) ---
def safe_trash_file(file_path: Path, mount_point: Path) -> str:
    """
    Dosyayı çöp kutusuna taşır. 
    Dönüş: İşlem sonucu (Başarılı/Hata mesajı)
    """
    
    # 1. XDG (Sistem Çöp Kutusu) Standardını Dene (Linux'ta yaygın)
    try:
        subprocess.run(['trash-put', str(file_path)], check=True, capture_output=True, timeout=5)
        return LANG.get_string("Status", "move_trash_xdg_success")
    except (FileNotFoundError, subprocess.CalledProcessError):
        pass # trash-put bulunamadı, taşınabilir çöp kutusuna geç
    except Exception as e:
        return f"{LANG.get_string('Status', 'move_trash_xdg_error')}: {e}"


    # 2. Taşınabilir Çöp Kutusu Yedeği (.Trash-1000)
    try:
        trash_dir_name = ".Trash-1000" 
        
        if not mount_point.is_dir():
             return LANG.get_string("Status", "move_trash_portable_error_mount")
             
        trash_path = mount_point / trash_dir_name
        
        trash_path.mkdir(parents=True, exist_ok=True)
        
        shutil.move(str(file_path), str(trash_path / file_path.name))
        
        return LANG.get_string("Status", "move_trash_portable_success").format(trash_dir_name=trash_dir_name)

    except Exception as e:
        return f"{LANG.get_string('Status', 'move_trash_portable_error')}: {e}"


# --- 1. Sinyal Sınıfı ---
class WorkerSignals(QObject):
    result = Signal(dict)
    finished = Signal()
    progress = Signal(int, str)
    error = Signal(str)

# --- 2. Tarama İş Parçacığı (GERÇEK TARAMA) ---
class ScannerWorker(QThread):
    # Sinyaller
    result = Signal(dict)
    finished = Signal()
    progress = Signal(int, str)
    error = Signal(str)
    
    # --- RİSK SABİTLERİ VE KURAL LİSTELERİ ---
    RISK_EXTS = ['.exe', '.com', '.bat', '.cmd', '.vbs', '.js', '.ps1', '.sh', '.wsf', '.hta', '.lnk', '.scr', '.jse', '.wsh', '.psm1'] 
    RANSOM_EXTS = ['.locky', '.cerber', '.wallet', '.cryp', '.encrypted', '.vvv', '.ccc', '.mljx', '.DJVU', '.djvu', '.xyz', '.YZXXX', '.YFSEK', '.FMOPQ', '.XMEYU', '.WHAUN', '.makop', '.tcvp', '.merl']
    MACRO_EXTS = ['.docm', '.xlsm', '.pptm']
    KEYWORD_RISK = ['crack', 'keygen', 'patch', 'hactivator', 'serial']
    SYSTEM_FOLDERS = ['$RECYCLE.BIN', 'System Volume Information', 'RECYCLER']
    AUTORUN_VARIANTS = ['desktop.ini', 'autorun.exe', 'folder.ico']
    SUSPICIOUS_NAMES = ['new folder', 'yeni klasör', 'untitled folder']
    EMAIL_PATTERN = re.compile(r'.*@.*\.[a-z]{2,}', re.IGNORECASE)
    
    # KANIT TABANLI LİMİTLER
    SMALL_EXE_THRESHOLD_MB = 1     
    LARGE_EXE_THRESHOLD_MB = 30    
    AUTORUN_CONTENT_THRESHOLD_KB = 5 

    def __init__(self, directory, settings):
        super().__init__()
        self.directory = Path(directory)
        self.settings = settings
        self.is_running = True
        self.file_count = 0
        self.scanned_count = 0
        
    def _get_risk_color(self, risk_key):
        """Risk anahtarına göre renk kodunu döndürür."""
        # Renkler kullanıcının verdiği son versiyondan korundu.
        return {
            "CRITICAL": "#FF4500", # Kırmızı-Turuncu (Koyu)
            "HIGH": "#FF6347",     # Kırmızımsı-Somon (Koyu)
            "MEDIUM": "#FFA500",   # Turuncu (Açık)
            "INFO": "#C0C0C0",     # Gri (Açık)
        }.get(risk_key, "#FFFFFF")

    def _get_risk_level(self, base_level, adjustment=0):
        """
        Risk seviyesini (CRITICAL, HIGH, MEDIUM, INFO) döndürür.
        adjustment: +1 (riski artır), -1 (riski düşür).
        """
        levels = ["CRITICAL", "HIGH", "MEDIUM", "INFO"]
        current_index = levels.index(base_level)
        
        new_index = max(0, min(len(levels) - 1, current_index - adjustment))
        
        if new_index == len(levels) - 1 and adjustment < 0:
             return "INFO"
             
        return levels[new_index]

    def _check_file_for_threat(self, file_path: Path):
        """Dosyayı tanımlı kurallara göre kontrol eder."""
        
        file_name = file_path.name
        file_suffix_lower = file_path.suffix.lower()
        
        try:
            if not file_path.is_file():
                return None
            file_size = file_path.stat().st_size 
        except Exception:
            return None 

        if not self.settings.get('check_hidden') and file_name.startswith('.'): 
            return None 

        # --- EN YÜKSEK ÖNCELİK: FİDYE VİRÜSLERİ (NEREDE OLURSA OLSUN) ---
        if self.settings.get('check_ext_anomaly') and any(file_name.lower().endswith(ext) for ext in self.RANSOM_EXTS):
            return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_cerber"), "color": "#800020"}
        
        # Email uzantılı fidye virüsleri
        if self.EMAIL_PATTERN.search(file_name):
            return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_email_ext"), "color": "#800020"}

        # --- ÖNCELİKLİ: SİSTEM KLASÖRÜ KONTROLLERİ ---
        if self.settings.get('check_sys_anomaly'):
            is_in_sys_folder = False
            for folder in self.SYSTEM_FOLDERS:
                if folder in str(file_path.parent):
                    is_in_sys_folder = True
                    break
            if is_in_sys_folder:
                # Sistem klasöründe olmaması gereken dosya türleri
                forbidden_exts = ['.exe', '.com', '.bat', '.cmd', '.vbs', '.js', '.ps1', '.inf', '.scr', '.lnk', '.hta', '.wsf']
                if any(file_name.lower().endswith(ext) for ext in forbidden_exts):
                    return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_sys_folder_forbidden"), "color": self._get_risk_color("CRITICAL")}
                else:
                    return {"risk_key": "HIGH", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "high_sys_folder_anomaly"), "color": self._get_risk_color("HIGH")}

        risk_level_key = "INFO"
        reason_key = "info_default"

        # --- A. KRİTİK VE YÜKSEK RİSK KONTROLLERİ (RISKI ARTIRANLAR) ---

        if self.settings.get('check_ext_anomaly') and len(file_name.split('.')) > 2 and file_suffix_lower != '.lnk':
            return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_double_ext"), "color": self._get_risk_color("CRITICAL")}
        
        # Sahte klasör virüsleri - aynı dizinde klasör adıyla exe varsa
        if file_suffix_lower == '.exe':
            folder_name = file_name[:-4]  # .exe'yi çıkar
            potential_folder = file_path.parent / folder_name
            if potential_folder.exists() and potential_folder.is_dir():
                return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_fake_folder"), "color": self._get_risk_color("CRITICAL")}
        
        # AutoRun varyantları
        if file_name.lower() in [v.lower() for v in self.AUTORUN_VARIANTS]:
            return {"risk_key": "HIGH", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "high_autorun_variant"), "color": self._get_risk_color("HIGH")}
        
        # Şüpheli klasör isimleri + exe
        if file_suffix_lower == '.exe' and any(sus_name in file_name.lower() for sus_name in self.SUSPICIOUS_NAMES):
            return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_suspicious_name"), "color": self._get_risk_color("CRITICAL")}

        if self.settings.get('check_sys_anomaly') and self.settings.get('check_autorun') and file_name.lower() == 'autorun.inf':
            if self.settings.get('check_script_content') and file_size > 0:
                 return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_autorun_content"), "color": self._get_risk_color("CRITICAL")}
            
            return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_autorun"), "color": self._get_risk_color("CRITICAL")}

        is_risk_ext = any(file_name.lower().endswith(ext) for ext in self.RISK_EXTS)
        is_macro_ext = any(file_name.lower().endswith(ext) for ext in self.MACRO_EXTS)
        
        if is_risk_ext or is_macro_ext:
            
            risk_level_key = "HIGH"
            reason_key = "high_exe"
            
            if is_macro_ext:
                reason_key = "high_macro"
            elif file_suffix_lower == '.lnk':
                risk_level_key = "MEDIUM" 
                reason_key = "med_lnk"
            elif file_suffix_lower == '.scr':
                 risk_level_key = "MEDIUM"
                 reason_key = "med_scr"
            elif file_suffix_lower in ['.vbs', '.js', '.ps1', '.hta', '.bat', '.cmd', '.sh', '.wsf', '.jse']:
                 reason_key = "high_script"

            # --- RİSK ARTTIRIMI (YÜKSELTME KANITLARI) ---
            
            # KRİTİK MANTIK KORUNDU: Küçük exe dosyaları CRITICAL
            if file_suffix_lower in ['.exe', '.com'] and file_size > 0 and file_size < self.SMALL_EXE_THRESHOLD_MB * 1024 * 1024:
                return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_small_exe").format(size=round(file_size/(1024*1024), 2)), "color": self._get_risk_color("CRITICAL")}
            
            # Orta boyut exe'ler (300KB-600KB) - şifreli payload olasılığı
            if file_suffix_lower in ['.exe', '.com'] and 300*1024 <= file_size <= 600*1024:
                return {"risk_key": "HIGH", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "high_medium_exe"), "color": self._get_risk_color("HIGH")}

            if self.settings.get('check_crack') and any(kw in file_name.lower() for kw in self.KEYWORD_RISK):
                return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_crack"), "color": self._get_risk_color("CRITICAL")}

            if self.settings.get('check_sys_anomaly'):
                is_in_sys_folder = False
                for folder in self.SYSTEM_FOLDERS:
                    if folder in str(file_path.parent):
                        is_in_sys_folder = True
                        break
                if is_in_sys_folder:
                     return {"risk_key": "CRITICAL", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "crit_sys_folder_suspicious"), "color": self._get_risk_color("CRITICAL")}

            # --- RİSK İNDİRİMİ (GÜVEN KANITLARI) ---

            if file_suffix_lower in ['.exe', '.com'] and file_size >= self.LARGE_EXE_THRESHOLD_MB * 1024 * 1024:
                risk_level_key = "MEDIUM"
                reason_key = "med_large_exe"
                return {"risk_key": risk_level_key, "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", reason_key).format(size=round(file_size/(1024*1024), 2)), "color": self._get_risk_color(risk_level_key)}

            return {"risk_key": risk_level_key, "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", reason_key), "color": self._get_risk_color(risk_level_key)}

        # --- B. SİSTEM KLASÖRÜ ANOMALİ KONTROLÜ (TÜM DOSYALAR) ---
        if self.settings.get('check_sys_anomaly'):
            is_in_sys_folder = False
            for folder in self.SYSTEM_FOLDERS:
                if folder in str(file_path.parent):
                    is_in_sys_folder = True
                    break
            if is_in_sys_folder:
                return {"risk_key": "HIGH", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "high_sys_folder_anomaly"), "color": self._get_risk_color("HIGH")}

        # --- C. DİĞER KONTROLLER (INFO) ---
        
        if self.settings.get('check_script_content') and file_suffix_lower in ['.bat', '.vbs', '.ps1', '.js'] and file_size > self.AUTORUN_CONTENT_THRESHOLD_KB * 1024:
             return {"risk_key": "MEDIUM", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "content_anomaly"), "color": self._get_risk_color("MEDIUM")}

        if self.settings.get('check_sys_anomaly') and file_name.lower() == 'thumbs.db':
             return {"risk_key": "INFO", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "info_thumbs"), "color": self._get_risk_color("INFO")}
        
        # Office geçici dosyaları
        if file_name.startswith('~$'):
             return {"risk_key": "INFO", "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", "info_office_temp"), "color": self._get_risk_color("INFO")}
        
        # --- C. Varsayılan INFO ---
        if file_size > 0:
             return {"risk_key": risk_level_key, "file_name": file_name, "path": str(file_path), "reason": LANG.get_string("Reason", reason_key), "color": self._get_risk_color(risk_level_key)}
             
        return None 
        
    def _count_files(self):
        """Tarama başlamadan önce toplam dosya sayısını hesaplar."""
        count = 0
        for dirpath, dirnames, filenames in os.walk(self.directory):
            for name in filenames:
                count += 1
        self.file_count = count
        
    def run(self):
        """Gerçek dosya sistemi taramasını yapar."""
        
        self.progress.emit(0, LANG.get_string("Status", "counting_files"))
        try:
             self._count_files()
        except Exception as e:
             self.error.emit(f"Dosya sayımı sırasında kritik hata: {e}")
             return

        if self.file_count == 0:
             self.progress.emit(100, LANG.get_string("Status", "scan_finished_clean"))
             self.finished.emit()
             return

        for dirpath, dirnames, filenames in os.walk(self.directory):
            if not self.is_running:
                break
                
            current_path = Path(dirpath)
            
            if not self.settings.get('check_hidden'):
                dirnames[:] = [d for d in dirnames if not d.startswith('.')]
            
            for i, file_name in enumerate(filenames):
                # Her 10 dosyada bir durdur kontrolü yap
                if i % 10 == 0 and not self.is_running:
                    break
                    
                self.scanned_count += 1
                file_path = current_path / file_name
                
                try:
                    result = self._check_file_for_threat(file_path)
                except PermissionError:
                    result = None
                except Exception as e:
                    result = None
                
                if result:
                    self.result.emit(result)
                
                progress_percent = int((self.scanned_count / self.file_count) * 100)
                self.progress.emit(min(progress_percent, 99), LANG.get_string("Status", "scanning_file").format(file_name=file_name))
                
                # Durdur kontrolü - her dosyada
                if not self.is_running:
                    break

        
        if self.is_running:
            self.progress.emit(100, LANG.get_string("Status", "scan_finished_review"))
            self.finished.emit()
        else:
            self.progress.emit(0, LANG.get_string("Status", "scan_canceled"))

    def stop(self):
        self.is_running = False

# --- 3. Disk Bulucu Fonksiyonu (REVİZYON: Sadece Mount Noktası, Boyut ve Label) ---
def find_mountable_disks():
    """lsblk komutunu çalıştırır ve USB/TAŞINABİLİR mount noktalarını bulur."""
    disk_map = {}
    target_fs = ['ntfs', 'vfat', 'exfat', 'fat32', 'fat']
    
    try:
        result = subprocess.run(['lsblk', '-J', '-o', 'NAME,FSTYPE,MOUNTPOINT,SIZE,LABEL,TRAN,RM,RO'], 
                                capture_output=True, text=True, timeout=5, check=False)
        
        if result.returncode != 0:
            # DÜZELTME: Hata durumunda boş string ("") dön
            return {LANG.get_string("DiskInfo", "err_lsblk_code"): ""}
             
        if not result.stdout:
            # DÜZELTME: Boş çıktı durumunda boş string ("") dön
             return {LANG.get_string("DiskInfo", "err_lsblk_empty"): ""}
             
        data = json.loads(result.stdout)
    
    except (FileNotFoundError, json.JSONDecodeError, subprocess.TimeoutExpired, Exception) as e:
        # DÜZELTME: Hata durumunda boş string ("") dön
        return {LANG.get_string("DiskInfo", "err_system_list"): ""}

    def process_block(block):
        mp = block.get('mountpoint')
        fstype_safe = (block.get('fstype') or '').lower()
        size = block.get('size', LANG.get_string("DiskInfo", "unknown_size"))
        label = block.get('label', LANG.get_string("DiskInfo", "no_label"))
        tran_safe = (block.get('tran') or '').lower()
        
        is_removable = bool(block.get('rm'))
        is_usb_device = tran_safe == 'usb' 
        
        is_mounted_valid = mp and mp != '/'
        is_target_fs = fstype_safe in target_fs
        
        if is_mounted_valid and is_target_fs and (is_removable or is_usb_device):
            # Format: Mount Noktası (Boyut) - Label
            display_name = f"{mp} ({size}) - {label}"
            disk_map[display_name] = mp

        if 'children' in block:
            for child in block['children']:
                process_block(child)

    for block in data.get('blockdevices', []):
        process_block(block)

    if not disk_map:
        # KRİTİK DÜZELTME: USB bulunamadığında boş string ("") dön
        return {LANG.get_string("DiskInfo", "no_usb_found"): ""} 
    else:
        return disk_map


# --- 4. Ana Uygulama Sınıfı ---
class AntivirusApp(QMainWindow):
    def __init__(self):
        super().__init__()
        
        QCoreApplication.setApplicationName(LANG.get_string("App", "title"))
        self.icon_path = get_icon_path('secure.png')
        if self.icon_path:
            self.setWindowIcon(QIcon(self.icon_path)) 

        self.setMinimumSize(1050, 700)
        
        self.threadpool = QThreadPool()
        self.current_worker = None
        
        self.mountable_disks = find_mountable_disks() 
        
        # KRİTİK KONTROL: USB bulunamadıysa başlangıç yolu boş string ("") olacak.
        no_usb_key = LANG.get_string("DiskInfo", "no_usb_found")
        # Yolu boş olan tek bir öğe varsa, USB bulunamamış demektir.
        self.is_no_usb_found = (len(self.mountable_disks) == 1 and next(iter(self.mountable_disks.values())) == "")
        
        initial_path = next(iter(self.mountable_disks.values()))
        
        self.current_scan_path = QLabel(initial_path) 
        self.icon_provider = QFileIconProvider() 
        self.results_count = 0 

        central_widget = QWidget()
        self.setCentralWidget(central_widget)
        main_layout = QHBoxLayout(central_widget)

        splitter = QSplitter(Qt.Horizontal)
        
        left_panel = self._create_left_panel()
        # SOL PANEL GENİŞLİK KONTROLÜ
        left_panel.setMaximumWidth(350) 
        right_panel = self._create_right_panel()
        
        splitter.addWidget(left_panel)
        splitter.addWidget(right_panel)
        # BAŞLANGIÇ BOYUTU DAR (220, 700)
        splitter.setSizes([225, 700]) 
        
        main_layout.addWidget(splitter)
        self._initialize_tree()
        
        self._refresh_ui_texts()
        
        # KRİTİK DÜZELTME: USB bulunamadıysa Tarama butonunu hemen pasif et ve stili temizle
        if self.is_no_usb_found:
            self.start_button.setEnabled(False)
            self.status_label.setText(LANG.get_string("Status", "no_usb_detected_info"))
            # ❗ DÜZELTME: Stil temizlenir, sadece temayı kullanır.
            self.status_label.setStyleSheet("") 
        
    # --- DİL YÖNETİMİ METOTLARI ---
    def _refresh_ui_texts(self):
        self.setWindowTitle(LANG.get_string("App", "title"))
        self.logo_label.setText(LANG.get_string("App", "title"))
        self.purpose_label.setText(LANG.get_string("App", "purpose"))
        self.disk_group.setTitle(LANG.get_string("Group", "disk_selection"))
        self.browse_button.setText(LANG.get_string("Button", "browse"))
        self.settings_group.setTitle(LANG.get_string("Group", "rules"))
        
        self.check_ext_anomaly.setText(LANG.get_string("Checkbox", "ext_anomaly"))
        self.check_hidden.setText(LANG.get_string("Checkbox", "hidden"))
        self.check_crack.setText(LANG.get_string("Checkbox", "crack"))
        self.check_autorun.setText(LANG.get_string("Checkbox", "autorun"))
        self.check_sys_anomaly.setText(LANG.get_string("Checkbox", "sys_anomaly"))
        self.check_script_content.setText(LANG.get_string("Checkbox", "script_content"))

        self.status_group.setTitle(LANG.get_string("Group", "status"))
        
        # Tarama durumunu kontrol et ve buton metnini buna göre ayarla
        if self.current_worker and self.current_worker.is_running:
            # Tarama devam ediyorsa buton metnini "Tarama Sürüyor" olarak bırak
            self.start_button.setText(LANG.get_string("Status", "scan_running"))
        else:
            # Tarama yoksa normal "Taramayı Başlat" metni
            self.start_button.setText(LANG.get_string("Button", "start_scan"))
        
        # Durdur butonu kaldırıldı
        
        self.about_button.setText(LANG.get_string("Button", "about_info"))
        self.lang_button.setText(LANG.get_string("Button", "language"))
        
        self.result_title.setText(LANG.get_string("Table", "title_tree")) 
        
        # SÜTUN BAŞLIKLARI (4 Sütun)
        self.results_tree.setHeaderLabels([
            LANG.get_string("Table", "col_action"),    # Sütun 0: Checkbox
            LANG.get_string("Table", "col_file_name"), # Sütun 1: Dosya Adı
            LANG.get_string("Table", "col_address"),   # Sütun 2: Adres/Yol
            LANG.get_string("Table", "col_reason")     # Sütun 3: Neden
        ])
        
        self.select_critical_button.setText(LANG.get_string("Button", "select_critical"))
        self.select_high_button.setText(LANG.get_string("Button", "select_high"))
        self.unselect_all_button.setText(LANG.get_string("Button", "unselect_all"))
        self.delete_button.setText(LANG.get_string("Button", "delete_selected"))
        
        # Durum metnini mevcut duruma göre ayarla
        current_status_text = self.status_label.text()
        
        # Eğer tarama devam ediyorsa durum metnini değiştirme
        if self.current_worker and self.current_worker.is_running:
            # Tarama devam ediyor, durum metnini olduğu gibi bırak
            pass
        elif not self.start_button.isEnabled() and self.is_no_usb_found:
            # USB bulunamadı durumu
            self.status_label.setText(LANG.get_string("Status", "no_usb_detected_info"))
            self.status_label.setStyleSheet("")
        elif self.results_count > 0 and ("tamamlandı" in current_status_text.lower() or "completed" in current_status_text.lower() or "bitti" in current_status_text.lower()):
            # Tarama tamamlanmış ve sonuç var
            self.status_label.setText(LANG.get_string("Status", "scan_finished_found").format(count=self.results_count))
            self.status_label.setStyleSheet("")
        elif self.results_count == 0 and ("tamamlandı" in current_status_text.lower() or "clean" in current_status_text.lower() or "completed" in current_status_text.lower() or "temiz" in current_status_text.lower()):
            # Tarama tamamlanmış ve temiz
            self.status_label.setText(LANG.get_string("Status", "scan_finished_clean"))
            self.status_label.setStyleSheet("")
        # Varsayılan "wait" durumuna dönmeyi kaldırdım - mevcut durumu koru
        
    def _show_about_dialog(self):
        """Hakkında dialog'unu gösterir."""
        about_text = LANG.get_string("About", "full_text")
        # HTML entity'leri ve escape karakterleri düzelt
        about_text = about_text.replace('\\n', '\n').replace('&#39;', "'")
        
        msg_box = QMessageBox(self)
        msg_box.setWindowTitle(LANG.get_string("Menu", "about"))
        msg_box.setText(about_text)
        if self.icon_path:
            msg_box.setIconPixmap(QPixmap(self.icon_path).scaled(64, 64, Qt.KeepAspectRatio, Qt.SmoothTransformation))
        msg_box.exec()

    def _show_language_dialog(self):
        items = list(LANG.available_languages.values())
        if not items:
            QMessageBox.warning(self, LANG.get_string("Menu", "language"), LANG.get_string("Status", "language_file_error"))
            return
            
        current_display_name = LANG.available_languages.get(LANG.current_language, items[0])
        initial_index = items.index(current_display_name) if current_display_name in items else 0

        dialog = QInputDialog()
        selected_display_name, ok = dialog.getItem(self, 
            LANG.get_string("Menu", "language"), 
            LANG.get_string("Status", "language_select_title"), 
            items, initial_index, False)

        if ok and selected_display_name:
            selected_code = next(key for key, value in LANG.available_languages.items() if value == selected_display_name)
            if selected_code != LANG.current_language:
                if LANG.load_language(selected_code):
                    LANG._save_user_language(selected_code)  # Dil tercihini kaydet
                    self._refresh_ui_texts()
                else:
                    QMessageBox.critical(self, LANG.get_string("Menu", "language"), LANG.get_string("Status", "language_load_error").format(code=selected_code))


    # --- SOL PANEL OLUŞTURMA METOTLARI ---
    def _create_left_panel(self):
        left_widget = QWidget()
        layout = QVBoxLayout(left_widget)
        layout.setContentsMargins(15, 15, 15, 15)
        
        title_layout = QHBoxLayout()
        if self.icon_path:
            icon_pixmap = QPixmap(self.icon_path).scaled(64, 64, Qt.KeepAspectRatio, Qt.SmoothTransformation) 
            icon_label = QLabel()
            icon_label.setPixmap(icon_pixmap)
            title_layout.addWidget(icon_label)
        
        self.logo_label = QLabel(LANG.get_string("App", "title")) 
        self.logo_label.setFont(QFont("SansSerif", 14, QFont.Bold)) 
        self.logo_label.setStyleSheet("color: #0093ff;")
        title_layout.addWidget(self.logo_label)
        title_layout.addStretch()
        layout.addLayout(title_layout)
        
        self.purpose_label = QLabel(LANG.get_string("App", "purpose"))
        self.purpose_label.setWordWrap(True)
        layout.addWidget(self.purpose_label)
        layout.addSpacing(20)

        self.disk_group = self._create_disk_selection_group()
        layout.addWidget(self.disk_group)
        layout.addSpacing(10)

        self.settings_group = self._create_settings_group()
        layout.addWidget(self.settings_group)
        layout.addSpacing(20)
        
        self.start_button = QPushButton(LANG.get_string("Button", "start_scan"))
        self.start_button.setFont(QFont("SansSerif", 12, QFont.Bold))
        self.start_button.setStyleSheet("background-color: #0a750c; color: white; padding: 10px;")
        self.start_button.clicked.connect(self._start_scan)
        layout.addWidget(self.start_button)
        layout.addSpacing(20)
        
        self.status_group = self._create_status_group()
        layout.addWidget(self.status_group)
        layout.addLayout(self._create_info_buttons())

        layout.addStretch()
        return left_widget

    def _create_disk_selection_group(self):
        group = QGroupBox(LANG.get_string("Group", "disk_selection"))
        layout = QVBoxLayout(group)
        
        self.disk_combo = QComboBox()
        self.disk_combo.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Fixed) 
        
        for display_name in self.mountable_disks.keys():
            self.disk_combo.addItem(display_name)
            
        layout.addWidget(self.disk_combo)

        manual_layout = QHBoxLayout()
        self.browse_button = QPushButton(LANG.get_string("Button", "browse"))
        self.browse_button.clicked.connect(self._browse_directory)
        manual_layout.addWidget(self.browse_button)
        layout.addLayout(manual_layout)
        
        self.disk_combo.currentIndexChanged.connect(self._update_scan_path)
        
        return group
    
    # _update_scan_path METODU (Tarama Butonunu Kontrol Eder)
    def _update_scan_path(self, index):
        selected_display = self.disk_combo.currentText()
        # Path is now "" if no USB found
        path = self.mountable_disks.get(selected_display, "") 
        self.current_scan_path.setText(path)
        
        # KRİTİK KONTROL: Eğer yol boşsa (USB yoksa veya hata) Tarama butonunu devre dışı bırak.
        if not path:
             self.start_button.setEnabled(False)
             self.status_label.setText(LANG.get_string("Status", "no_usb_detected_info"))
             # DÜZELTME: Stil temizlendi, temaya bırakıldı.
             self.status_label.setStyleSheet("")
        else:
             self.start_button.setEnabled(True)
             self.status_label.setText(LANG.get_string("Status", "wait"))
             self.status_label.setStyleSheet("")

    def _create_settings_group(self):
        group = QGroupBox(LANG.get_string("Group", "rules"))
        layout = QVBoxLayout(group)
        
        self.check_ext_anomaly = QCheckBox(LANG.get_string("Checkbox", "ext_anomaly"))
        self.check_ext_anomaly.setChecked(True)
        layout.addWidget(self.check_ext_anomaly)
        
        self.check_hidden = QCheckBox(LANG.get_string("Checkbox", "hidden"))
        self.check_hidden.setChecked(True)
        layout.addWidget(self.check_hidden)

        self.check_crack = QCheckBox(LANG.get_string("Checkbox", "crack"))
        self.check_crack.setChecked(True)
        layout.addWidget(self.check_crack)
        
        self.check_autorun = QCheckBox(LANG.get_string("Checkbox", "autorun"))
        self.check_autorun.setChecked(True)
        layout.addWidget(self.check_autorun)

        self.check_sys_anomaly = QCheckBox(LANG.get_string("Checkbox", "sys_anomaly"))
        self.check_sys_anomaly.setChecked(True)
        layout.addWidget(self.check_sys_anomaly)

        self.check_script_content = QCheckBox(LANG.get_string("Checkbox", "script_content"))
        self.check_script_content.setChecked(True)
        layout.addWidget(self.check_script_content)
        
        return group
    
    # _create_status_group METODU (Durdur Butonu Eklendi ve Stil Düzenlendi)
    def _create_status_group(self):
        group = QGroupBox(LANG.get_string("Group", "status"))
        layout = QVBoxLayout(group)
        
        font = QFont("SansSerif", 9)
        # YENİ DÜZELTME: Kullanıcının isteği üzerine ince/normal ve italik ayarlandı. Temaya uygunluk için renk ve kalınlık kaldırıldı.
        font.setItalic(True)
        # QFont.setWeight(QFont.Light) de kullanılabilir ancak platforma göre davranması için sadece İtalik bıraktım.
        
        self.status_label = QLabel(LANG.get_string("Status", "wait"))
        self.status_label.setFont(font)
        layout.addWidget(self.status_label)
        
        self.progress_bar = QProgressBar()
        self.progress_bar.setRange(0, 100)
        self.progress_bar.setValue(0)
        layout.addWidget(self.progress_bar)
        
        # Durdur butonu kaldırıldı
        
        return group
    
    def _create_info_buttons(self):
        button_layout = QHBoxLayout()
        
        self.about_button = QPushButton(LANG.get_string("Button", "about_info"))
        self.about_button.clicked.connect(self._show_about_dialog)
        button_layout.addWidget(self.about_button)
        
        self.lang_button = QPushButton(LANG.get_string("Button", "language"))
        self.lang_button.clicked.connect(self._show_language_dialog) 
        button_layout.addWidget(self.lang_button)
        
        return button_layout

    # --- SAĞ PANEL OLUŞTURMA METOTLARI (QTreeWidget) ---
    def _create_right_panel(self):
        # HATA DÜZELTME: right_widget'ı burada tanımlamalıyız.
        right_widget = QWidget()
        layout = QVBoxLayout(right_widget)
        layout.setContentsMargins(15, 15, 15, 15)

        self.result_title = QLabel(LANG.get_string("Table", "title_tree"))
        self.result_title.setFont(QFont("SansSerif", 10, QFont.Bold)) 
        layout.addWidget(self.result_title)
        
        self.results_tree = QTreeWidget()
        self.results_tree.setColumnCount(4) # Sütun sayısı 4 (Risk sütunu kaldırıldı)
        
        self.results_tree.setHeaderLabels([
            LANG.get_string("Table", "col_action"),    # Sütun 0: Checkbox
            LANG.get_string("Table", "col_file_name"), # Sütun 1: Dosya Adı
            LANG.get_string("Table", "col_address"),   # Sütun 2: Adres/Yol
            LANG.get_string("Table", "col_reason")     # Sütun 3: Neden/Tespit Kuralı
        ])
        
        # Sütun boyutlandırma (Elle çekmeye izin veren ve esnek ayarlar)
        header = self.results_tree.header()
        
        # 0: Checkbox - Her zaman içeriğe göre minimal
        header.setSectionResizeMode(0, QHeaderView.ResizeToContents) 
        
        # 1: Dosya Adı - Elle çekilebilir (Interactive)
        header.setSectionResizeMode(1, QHeaderView.Interactive) 
        
        # 2: Adres/Yol - ❗ DÜZELTME: Elle çekilebilir (Interactive). (Önceki Stretch ayarı kaldırılarak boyutlandırma kilitlenmesi önlendi.)
        header.setSectionResizeMode(2, QHeaderView.Interactive)        
        
        # 3: Neden/Tespit Kuralı - ❗ DÜZELTME: Kalan tüm alanı doldurur (Stretch). Son sütun esnek olmalı.
        header.setSectionResizeMode(3, QHeaderView.Stretch)     
        
        # AĞAÇ SİMGE/GİRİNTİLERİNİ GİZLE
        self.results_tree.setIndentation(0) 
        self.results_tree.setRootIsDecorated(False)
        
        layout.addWidget(self.results_tree)

        layout.addLayout(self._create_action_buttons())
        
        return right_widget 

    def _create_action_buttons(self):
        action_layout = QVBoxLayout()
        
        # Risk seviyesi seçim butonları
        risk_layout = QHBoxLayout()
        
        self.select_critical_button = QPushButton(LANG.get_string("Button", "select_critical"))
        self.select_critical_button.clicked.connect(lambda: self._select_by_risk("CRITICAL"))
        self.select_critical_button.setStyleSheet("background-color: #800020; color: white; padding: 4px;")
        risk_layout.addWidget(self.select_critical_button)
        
        self.select_high_button = QPushButton(LANG.get_string("Button", "select_high"))
        self.select_high_button.clicked.connect(lambda: self._select_by_risk("HIGH"))
        self.select_high_button.setStyleSheet("background-color: #FF4500; color: white; padding: 4px;")
        risk_layout.addWidget(self.select_high_button)
        
        self.unselect_all_button = QPushButton(LANG.get_string("Button", "unselect_all"))
        self.unselect_all_button.clicked.connect(self._unselect_all)
        self.unselect_all_button.setStyleSheet("background-color: #484848; color: white; padding: 4px;")
        risk_layout.addWidget(self.unselect_all_button)
        
        action_layout.addLayout(risk_layout)
        action_layout.addSpacing(5)
        
        self.delete_button = QPushButton(LANG.get_string("Button", "delete_selected"))
        self.delete_button.setStyleSheet("background-color: #6a0000; color: white; padding: 5px;")
        self.delete_button.setFont(QFont("SansSerif", 10, QFont.Bold))
        self.delete_button.clicked.connect(self._delete_selected)
        self.delete_button.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Fixed)
        action_layout.addWidget(self.delete_button)
        
        return action_layout

    # --- AKSİYON METODLARI ---
    
    def _initialize_tree(self):
        self.results_tree.clear()
        self.progress_bar.setValue(0)
        self.status_label.setText(LANG.get_string("Status", "wait"))
        self.status_label.setStyleSheet("") # Temaya uyum için stil temizlendi
        self.results_count = 0
        # Durdur butonu kaldırıldı

    def _browse_directory(self):
        directory = QFileDialog.getExistingDirectory(self, LANG.get_string("Button", "browse"))
        if directory:
            # Seçilen dizin için benzersiz bir görünen ad oluştur
            display_name = f"{LANG.get_string('DiskInfo', 'manual_select')}: {directory}"
            
            # Eğer dizin mevcut diskler arasında yoksa ekle
            if directory not in self.mountable_disks.values():
                self.mountable_disks[display_name] = directory
                self.disk_combo.addItem(display_name)
            
            index = self.disk_combo.findText(display_name)
            if index != -1:
                self.disk_combo.setCurrentIndex(index)
                self.current_scan_path.setText(directory)
                
                # DÜZELTME: Manuel seçim yapıldığında Start butonunu aktif et
                self.start_button.setEnabled(True)
                self.status_label.setText(LANG.get_string("Status", "wait"))
                self.status_label.setStyleSheet("") # Temaya uyum için stil temizlendi
    
    # _stop_scan METODU (EKLENDİ ve Stil Temizlendi)
    # Durdur butonu kaldırıldı 

    def _start_scan(self):
        scan_path = self.current_scan_path.text()
        
        # KRİTİK KONTROL: Eğer yol boşsa (USB bulunamadı durumu) taramayı engelle
        if not scan_path:
             self.status_label.setText(LANG.get_string("Status", "no_usb_detected_info"))
             # DÜZELTME: Stil temizlendi, temaya bırakıldı.
             self.status_label.setStyleSheet("")
             self.start_button.setEnabled(False)
             return

        if not os.path.exists(scan_path) or not os.path.isdir(scan_path):
            self.status_label.setText(LANG.get_string("Status", "error_invalid_dir").format(path=scan_path))
            # DÜZELTME: Stil temizlendi, temaya bırakıldı.
            self.status_label.setStyleSheet("")
            QMessageBox.warning(self, LANG.get_string("Menu", "language"), LANG.get_string("Status", "invalid_dir_dialog"))
            return

        self._initialize_tree()
        
        settings = {
            'check_ext_anomaly': self.check_ext_anomaly.isChecked(),
            'check_hidden': self.check_hidden.isChecked(),
            'check_crack': self.check_crack.isChecked(),
            'check_autorun': self.check_autorun.isChecked(),
            'check_sys_anomaly': self.check_sys_anomaly.isChecked(),
            'check_script_content': self.check_script_content.isChecked()
        }

        self.start_button.setEnabled(False)
        self.start_button.setText(LANG.get_string("Status", "scan_running"))
        self.start_button.setStyleSheet("background-color: #ff9800; color: white; padding: 10px;")
        
        # Durdur butonu kaldırıldı 
        
        # *** HIZ OPTİMİZASYONU: Güncellemeleri dondur ***
        self.results_tree.setUpdatesEnabled(False) 

        self.current_worker = ScannerWorker(scan_path, settings)
        self.current_worker.result.connect(self._add_scan_result)
        self.current_worker.finished.connect(self._scan_finished)
        self.current_worker.progress.connect(self._update_progress)
        self.current_worker.error.connect(self._handle_error)
        
        self.current_worker.start()

    # _scan_finished METODU (Durdur butonu pasifleştirildi ve Stil Temizlendi)
    def _scan_finished(self):
        self.current_worker = None
        
        # Durdur butonu kaldırıldı

        # DÜZELTME: USB bulunamadıysa Start butonunu pasif tut.
        if self.is_no_usb_found and not self.current_scan_path.text():
             self.start_button.setEnabled(False)
             self.status_label.setText(LANG.get_string("Status", "no_usb_detected_info"))
             # DÜZELTME: Stil temizlendi, temaya bırakıldı.
             self.status_label.setStyleSheet("")
        else:
             self.start_button.setEnabled(True)
             self.start_button.setText(LANG.get_string("Button", "start_scan"))
             self.start_button.setStyleSheet("background-color: #0a750c; color: white; padding: 10px;")
        
        # *** YENİ OPTİMİZASYON: Sütunları içeriğe göre ayarla ***
        # self._adjust_column_widths() # ❗ SİLİNDİ: Kullanıcının elle yaptığı ayarları sıfırlamaması için kaldırıldı.
        
        # *** HIZ OPTİMİZASYONU: Güncelleyici etkinleştir ve yenile ***
        self.results_tree.setUpdatesEnabled(True)
        self.results_tree.repaint()


        if self.results_count == 0:
            self.status_label.setText(LANG.get_string("Status", "scan_finished_clean"))
            # DÜZELTME: Stil temizlendi.
            self.status_label.setStyleSheet("")
        else:
            self.status_label.setText(LANG.get_string("Status", "scan_finished_found").format(count=self.results_count))
            # DÜZELTME: Stil temizlendi.
            self.status_label.setStyleSheet("")
            
    def _handle_error(self, message):
        # Scan finished'ı çağırıp durumu resetleyelim
        if self.current_worker:
             self.current_worker.is_running = False
             self._scan_finished() 
        
        QMessageBox.critical(self, LANG.get_string("Menu", "about"), LANG.get_string("Status", "scan_error_dialog").format(message=message))

    def _update_progress(self, value, message):
        self.progress_bar.setValue(value)
        self.status_label.setText(message)

    def _add_scan_result(self, result):
        
        self.results_count += 1
        
        file_path_obj = Path(result["path"])
        
        # Dosya Adı ve Adres/Yol ayrımı
        file_name = file_path_obj.name
        scan_root = self.current_scan_path.text()
        
        # Göreceli yolu, os.path.relpath ile hesapla
        try:
             relative_path_str = os.path.relpath(str(file_path_obj.parent), scan_root)
        except ValueError:
             # Unix sistemlerinde Windows yolu formatı nedeniyle hata oluşursa tam yolu kullan
             relative_path_str = str(file_path_obj.parent)
        
        # Kök dizin için "." yerine "/" kullan
        if relative_path_str == '.':
             display_dir_path = "/"
        else:
             display_dir_path = relative_path_str
             
        # DÜZ LİSTE MANTIĞI: 4 sütunlu yeni öğe oluşturulur
        item = QTreeWidgetItem([
            "",                                # Sütun 0: Checkbox için boş
            file_name,                         # Sütun 1: Dosya Adı
            display_dir_path,                  # Sütun 2: Adres/Yol
            #result["risk_key"],                # Sütun 3: Risk << bunu sildim gerek yok.
            result["reason"]                   # Sütun 4: Neden
        ])
        
        # Simge Bulma ve Sütun 1'e Atama
        icon = QIcon.fromTheme(f'text-x-{file_path_obj.suffix.lower().strip(".")}') or QIcon.fromTheme('text-x-generic')
        item.setIcon(1, icon) 
        
        # Checkbox Eklemek için
        checkbox = QCheckBox()
        checkbox.setSizePolicy(QSizePolicy.Fixed, QSizePolicy.Fixed)
        
        widget = QWidget()
        layout = QHBoxLayout(widget)
        layout.addWidget(checkbox)
        layout.setAlignment(Qt.AlignCenter) 
        layout.setContentsMargins(0, 0, 0, 0)
        
        # Renklendirme
        background_color = QColor(result["color"])
        # Yüksek riskli (koyu arka plan) için beyaz metin, orta/düşük riskli için siyah metin
        if result["risk_key"] in ["MEDIUM", "INFO"]:
            text_color = QColor("#000000") 
        else:
            text_color = QColor("#FFFFFF") 

        font = QFont()
        font.setBold(True)

        for i in range(self.results_tree.columnCount()):
            item.setBackground(i, background_color)
            item.setForeground(i, text_color)
            
            #if i == 3: # Risk sütunu (Index 3)
                 #item.setFont(i, font) 


        # Düz listeye (görünmez kök) ekle
        self.results_tree.invisibleRootItem().addChild(item)
        self.results_tree.setItemWidget(item, 0, widget) # Checkbox'ı sıfırıncı sütuna yerleştir
        
        # Dosya yolunu (Path) ve Orijinal Risk Seviyesini Item'a kaydet
        item.setData(0, Qt.UserRole, result["path"]) # Full path
        item.setData(1, Qt.UserRole, result["risk_key"]) # Risk key
        
    # *** YENİ METOT: Sütun Genişliklerini Ayarla *** << Bura iptal hacı. 
    #def _adjust_column_widths(self):
        """
        Tarama bittikten sonra tüm sütunları içeriğin maksimum genişliğine göre ayarlar.
        """
        #header = self.results_tree.header()
        
        # Tüm sütunları içeriğe göre ayarla
        #for i in range(self.results_tree.columnCount()):
            #header.setSectionResizeMode(i, QHeaderView.ResizeToContents)
        
        # Sütunları içeriğe göre yeniden boyutlandırmayı zorla
        #header.resizeSections(QHeaderView.ResizeToContents)
        
        # Adres/Yol (2) ve Neden (4) sütunlarının boşluğu paylaşması için
        # tekrar QHeaderView.Stretch olarak ayarla.
        # Bu, pencere boyutunda yapılacak değişikliklere karşı sütunların dinamik olmasını sağlar.
        #header.setSectionResizeMode(2, QHeaderView.Stretch) # Adres/Yol
        #header.setSectionResizeMode(3, QHeaderView.Stretch) # Neden
        # Bu sırayla önce içeriğe göre sıkıştırılır, sonra kalan boşluk bu iki sütuna dağıtılır.


    def _iter_items(self, root):
        """Ağaçtaki tüm dosya öğelerini (yaprakları) yineleyen yardımcı fonksiyon. Düz liste için basitleştirildi."""
        # Düz liste olduğu için sadece root'un (invisibleRootItem) altındaki öğeler kontrol edilir.
        for i in range(root.childCount()):
            item = root.child(i)
            # Eğer item'ın 0. sütununda bir widget varsa (yani checkbox), bu bir dosyadır.
            if self.results_tree.itemWidget(item, 0):
                yield item
            # Klasör olmadığı varsayımıyla (düz liste) alt öğeler kontrol edilmez.
                
    def _select_by_risk(self, risk_level):
        """Belirtilen risk seviyesindeki tüm dosyaları seçer."""
        root = self.results_tree.invisibleRootItem()
        selected_count = 0
        
        for item in self._iter_items(root):
            checkbox_widget = self.results_tree.itemWidget(item, 0)
            if checkbox_widget:
                checkbox = checkbox_widget.findChild(QCheckBox)
                if checkbox:
                    # Risk seviyesini item'dan al
                    item_risk = item.data(1, Qt.UserRole)
                    
                    if item_risk == risk_level:
                        checkbox.setCheckState(Qt.Checked)
                        selected_count += 1
                    else:
                        checkbox.setCheckState(Qt.Unchecked)
        
        self.status_label.setText(LANG.get_string("Status", "selected_by_risk").format(count=selected_count, risk=risk_level))
        self.status_label.setStyleSheet("")
    
    def _get_risk_from_color(self, color_hex):
        """Renk kodundan risk seviyesini belirler."""
        color_map = {
            "#800020": "CRITICAL",  # Fidye virüsleri bordo
            "#FF4500": "CRITICAL",  # Normal critical
            "#FF6347": "HIGH",      # High
            "#FFA500": "MEDIUM",    # Medium
            "#C0C0C0": "INFO"       # Info
        }
        return color_map.get(color_hex, "INFO")

    def _unselect_all(self):
        """Tüm seçimleri kaldırır."""
        root = self.results_tree.invisibleRootItem()
        unselected_count = 0
        
        for item in self._iter_items(root):
            checkbox_widget = self.results_tree.itemWidget(item, 0)
            if checkbox_widget:
                checkbox = checkbox_widget.findChild(QCheckBox)
                if checkbox and checkbox.isChecked():
                    checkbox.setCheckState(Qt.Unchecked)
                    unselected_count += 1
        
        self.status_label.setText(LANG.get_string("Status", "unselected_all").format(count=unselected_count))
        self.status_label.setStyleSheet("")

    def _delete_selected(self):
        selected_items = []
        root = self.results_tree.invisibleRootItem()
        
        for item in self._iter_items(root):
            checkbox_widget = self.results_tree.itemWidget(item, 0)
            if checkbox_widget:
                checkbox = checkbox_widget.findChild(QCheckBox)
                if checkbox and checkbox.isChecked():
                    file_path = item.data(0, Qt.UserRole) # Full path data from column 0
                    selected_items.append((item, file_path))

        if not selected_items:
            self.status_label.setText(LANG.get_string("Status", "error_select_files"))
            # ❗ DÜZELTME: Stil temizlendi, temaya bırakıldı.
            self.status_label.setStyleSheet("")
            return
        
        current_scan_path = Path(self.current_scan_path.text())
            
        reply = QMessageBox.question(self, LANG.get_string("Button", "delete_selected"),
            LANG.get_string("Status", "move_confirm").format(count=len(selected_items)),
            QMessageBox.Yes | QMessageBox.No, QMessageBox.No)

        if reply == QMessageBox.Yes:
            success_count = 0
            fail_count = 0
            items_to_remove = []
            
            for item, file_path_str in selected_items:
                file_path = Path(file_path_str)
                
                if file_path.exists():
                    result_message = safe_trash_file(file_path, current_scan_path)
                    
                    # Hata kontrolü (Türkçe metin içerdiği için)
                    if "HATA" in result_message or "KRİTİK HATA" in result_message:
                        fail_count += 1
                        print(f"Taşıma Başarısız: {file_path_str} -> {result_message}", file=sys.stderr)
                        # ❗ KRİTİK DÜZELTME: Sütun 4'ten (Risk sonrası) yeni index 3'e düşürüldü.
                        item.setText(3, result_message) 
                        item.setBackground(3, QColor("#FF0000")) 
                        item.setForeground(3, QColor("#FFFFFF"))
                        
                    else:
                        success_count += 1
                        items_to_remove.append(item)
                        self.results_count -= 1 
                else:
                    success_count += 1
                    items_to_remove.append(item)
                    self.results_count -= 1 

            for item in items_to_remove:
                parent = item.parent() or self.results_tree.invisibleRootItem()
                parent.removeChild(item)
                # _cleanup_empty_parents kaldırıldığı için çağrılmayacak.

            if fail_count > 0:
                 self.status_label.setText(LANG.get_string("Status", "move_partial_fail").format(success_count=success_count, fail_count=fail_count))
                 # ❗ DÜZELTME: Stil temizlendi, temaya bırakıldı.
                 self.status_label.setStyleSheet("")
            else:
                 self.status_label.setText(LANG.get_string("Status", "move_success").format(count=success_count))
                 # ❗ DÜZELTME: Stil temizlendi, temaya bırakıldı.
                 self.status_label.setStyleSheet("")
            
            # select_all_button artık yok, bu satır kaldırıldı 
            
        else:
            self.status_label.setText(LANG.get_string("Status", "move_canceled"))
            # ❗ DÜZELTME: Stil temizlendi, temaya bırakıldı.
            self.status_label.setStyleSheet("")
            
# "O ki, karanlıklara ölçümle dokunur; gizliyi görünür kılar.
# Ve gerçeği, iddiadan ayıran teraziyi insanların eline koyar.
# İnsana, her yanılgıdan bir ders; her keşiften bir ufuk verir.
# Şüpheyi rehber kılar; aklı, onunla yükseltir.
# Ve hiçbir iddiayı ayrıcalıklı kılmaz; kanıtsız olana hüküm vermez.
# İnsanın acısını azaltır, ömrünü uzatır; yeryüzünde kolaylığı mümkün kılar.
# O, kulaktan geleni değil; gözle görüleni, akılla sınananı yüceltir.
# Ve insana, imkansızı mümkün kılan emeğin değerini öğretir.
# Hakikati saklamaz; onu arayanı, arayışın sonunda ödüllendirir.
# Gerçekten, ilerlemek isteyen için onda bir yol; duran için onda bir uyarı; merak eden için ışık vardır."
            
# --- Uygulamayı Çalıştır ---
if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = AntivirusApp()
    window.show()
    sys.exit(app.exec())
