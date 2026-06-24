# Python-Flask-O-rta-darajaa
Loyiha strukturasi
Sizda loyiha quyidagi ko'rinishda shakllanadi:

Plaintext
my_flask_app/
├── config.py
├── wsgi.py
├── README.md
└── app/
    └── __init__.py
1. config.py
Barcha konfiguratsiyalar bitta joyda jamlanadi va SECRET_KEY muhit o'zgaruvchilaridan (os.environ) xavfsiz o'qiladi.

Python
import os

class Config:
    """Asosiy (baza) konfiguratsiya klassi"""
    SECRET_KEY = os.environ.get('SECRET_KEY', 'default-super-secret-key')
    SQLALCHEMY_TRACK_MODIFICATIONS = False

class DevelopmentConfig(Config):
    """Developerlik muhiti uchun"""
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('DEV_DATABASE_URL', 'sqlite:///dev.db')

class TestingConfig(Config):
    """Testlar muhiti uchun"""
    TESTING = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('TEST_DATABASE_URL', 'sqlite:///test.db')

class ProductionConfig(Config):
    """Jonli (Production) muhit uchun"""
    DEBUG = False
    # Productionda default qiymat berilmaydi, SECRET_KEY shart bo'lishi kerak!
    SECRET_KEY = os.environ.get('SECRET_KEY') 
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')

# Konfiguratsiyalarni nom bo'yicha chaqirish uchun lug'at
config_by_name = {
    'development': DevelopmentConfig,
    'testing': TestingConfig,
    'production': ProductionConfig
}
2. app/__init__.py
Ilovani yaratuvchi va unga konfiguratsiyani yuklovchi create_app factory funksiyasi.

Python
from flask import Flask
from config import config_by_name

def create_app(config_name='development'):
    """Application Factory funksiyasi"""
    app = Flask(__name__)
    
    # Berilgan nom bo'yicha konfiguratsiyani yuklash
    app.config.from_object(config_by_name[config_name])
    
    # Oddiy test route
    @app.route('/')
    def index():
        return f"Hozirgi muhit: {config_name.upper()}"
        
    return app
3. wsgi.py
Faqat factory funksiyani chaqiradigan va server ishga tushishi uchun kirish nuqtasi bo'lgan ixcham fayl.

Python
import os
from app import create_app

env = os.environ.get('FLASK_ENV', 'development')
app = create_app(config_name=env)
4. README.md
Loyiha muhitlarini ishga tushirish yo'riqnomasi.

Markdown
# Flask Ilovasi

Ushbu loyiha Application Factory pattern asosida qurilgan.

## Ishga tushirish ko'rsatmalari

### 1. Development (Dasturlash) muhitida ishga tushirish
Development rejimida xatolar brauzerda ko'rinadi (Debug=True) va kod o'zgarganda server avtomatik qayta yuklanadi.

```bash
# Muhit o'zgaruvchilarini sozlash
export FLASK_ENV=development
export SECRET_KEY=mening_yashirin_kalitim_123

# Serverni ishga tushirish
flask run
2. Production (Jonli) muhitda ishga tushirish
Production rejimida xavfsizlik va tezlik uchun debug rejim o'chiriladi. Server gunicorn yoki uWSGI kabi WSGI serverlar orqali ishga tushiriladi.

Bash
# Muhit o'zgaruvchilarini sozlash (Majburiy!)
export FLASK_ENV=production
export SECRET_KEY=juda_kuchli_va_uzun_tasodifiy_satr_2026
export DATABASE_URL=postgresql://user:password@localhost/dbname

# Gunicorn orqali ishga tushirish (wsgi fayliga bog'lanadi)
gunicorn wsgi:app
