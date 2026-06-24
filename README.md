# Python-Flask-O-rta-darajaa
├── __init__.py
    ├── main/
    │   ├── __init__.py
    │   └── routes.py
    ├── notes/
    │   ├── __init__.py
    │   └── routes.py
    └── templates/
        ├── base.html
        ├── main/
        │   └── index.html
        └── notes/
            └── list.html
1. Kodlarni Yangilash va Blueprint'larni Kiritish
app/main/__init__.py
main blueprint'ini e'lon qilamiz:

Python
from flask import Blueprint

main_bp = Blueprint('main', __name__)

from app.main import routes
app/main/routes.py
Bosh sahifa logikasi:

Python
from flask import render_template
from app.main import main_bp

@main_bp.route('/')
def index():
    return render_template('main/index.html')
app/notes/__init__.py
notes blueprint'ini url_prefix bilan e'lon qilish uchun sozlaymiz:

Python
from flask import Blueprint

notes_bp = Blueprint('notes', __name__)

from app.notes import routes
app/notes/routes.py
Kamida 5 ta dastlabki eslatmaga ega ro'yxat va eslatmalar sahifasi:

Python
from flask import render_template
from app.notes import notes_bp

# Xotira ichidagi dummy ma'lumotlar (Python list)
NOTES_DATA = [
    {"id": 1, "title": "Xaridlar ro'yxati", "content": "Sut, non, tuxum va mevalar olish kerak."},
    {"id": 2, "title": "Dasturlash", "content": "Flask Blueprint mavzusini takrorlash."},
    {"id": 3, "title": "Sport", "content": "Bugun soat 18:00 da yugurish bor."},
    {"id": 4, "title": "Kitobxonlik", "content": "Har kuni kamida 20 sahifa kitob o'qish."},
    {"id": 5, "title": "Eslatma", "content": "Yangi loyiha arxitekturasini chizish."}
]

@notes_bp.route('/')
def list_notes():
    return render_template('notes/list.html', notes=NOTES_DATA)
2. Factory funksiyada Blueprint'larni ro'yxatdan o'tkazish
app/__init__.py
Eski holatdagi oddiy route o'rniga, endi blueprint'larni registratsiya qilamiz va notes_bp uchun /notes prefiksini beramiz:

Python
from flask import Flask
from config import config_by_name

def create_app(config_name='development'):
    app = Flask(__name__)
    app.config.from_object(config_by_name[config_name])
    
    # Blueprint'larni import qilish
    from app.main import main_bp
    from app.notes import notes_bp
    
    # Ro'yxatdan o'tkazish
    app.register_blueprint(main_bp)
    app.register_blueprint(notes_bp, url_prefix='/notes')  # /notes prefiksi shu yerda o'rnatiladi
        
    return app
3. Jinja2 Shablonlari (Templates)
Barcha o'tish havolalari xavfsiz va dinamik bo'lishi uchun url_for('blueprint_nomi.funksiya_nomi') ko'rinishida yoziladi.

app/templates/base.html
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Mening Bloknotim</title>
</head>
<body>
    <nav>
        <a href="{{ url_for('main.index') }}">Bosh sahifa</a> | 
        <a href="{{ url_for('notes.list_notes') }}">Eslatmalar</a>
    </nav>
    <hr>
    {% block content %} {% endblock %}
</body>
</html>
app/templates/main/index.html
HTML
{% extends "base.html" %}
{% block content %}
    <h1>Bosh sahifaga xush kelibsiz!</h1>
    <p>Eslatmalarni ko'rish uchun yuqoridagi menyudan foydalaning.</p>
{% endblock %}
app/templates/notes/list.html
HTML
{% extends "base.html" %}
{% block content %}
    <h1>Mening Eslatmalarim</h1>
    <ul>
        {% for note in notes %}
            <li>
                <strong>{{ note.title }}</strong>: {{ note.content }}
            </li>
        {% endfor %}
    </ul>
{% endblock %}
