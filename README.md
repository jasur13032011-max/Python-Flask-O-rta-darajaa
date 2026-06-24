# Python-Flask-O-rta-darajaa
Loyiha tarkibiga ma'lumotlar bazasini (SQLAlchemy) integratsiya qilamiz va Note modelini factory pattern asosida to'g'ri shakllantiramiz.

Yangilangan loyiha strukturasi
Plaintext
my_flask_app/
├── config.py
├── wsgi.py
├── app/
    ├── __init__.py
    ├── extensions.py  # db obyekti shu yerda yaratiladi
    ├── models.py      # Ma'lumotlar modellari
    ├── main/
    │   ├── __init__.py
    │   └── routes.py
    └── templates/
        ├── base.html
        └── main/
            └── index.html
1. Ma'lumotlar bazasi va Modellarni sozlash
app/extensions.py
Sirkulyar importlarning (circular imports) oldini olish uchun db obyektini alohida faylda e'lon qilamiz:

Python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()
app/models.py
Note modelini yaratamiz. Vaqtni avtomatik belgilash uchun func.now() funksiyasidan foydalanamiz:

Python
from app.extensions import db
from sqlalchemy import func

class Note(db.Model):
    __tablename__ = 'notes'
    
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    body = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, server_default=func.now())

    def __repr__(self):
        return f'<Note {self.title}>'
2. Factory funksiyani yangilash
app/__init__.py
db obyektini ilovaga bog'laymiz va jadval avtomatik yaratilishini ta'minlaymiz:

Python
from flask import Flask
from config import config_by_name
from app.extensions import db

def create_app(config_name='development'):
    app = Flask(__name__)
    app.config.from_object(config_by_name[config_name])
    
    # DBni init qilish
    db.init_app(app)
    
    # Blueprintlarni ulash
    from app.main import main_bp
    app.register_blueprint(main_bp)
    
    # Jadvallarni yaratish (Faqat test/dev muhitida qulaylik uchun)
    with app.app_context():
        db.create_all()
        
    return app
3. Logika va Formani boshqarish (Routes)
app/main/routes.py
Bosh sahifada ma'lumotlarni yangidan-eskiga qarab tartiblaymiz (desc()), yangi nota qo'shishda esa xatolik yuz bersa db.session.rollback() yordamida sessiyani tozalaymiz.

Python
from flask import render_template, request, redirect, url_for, flash
from app.main import main_bp
from app.extensions import db
from app.models import Note

@main_bp.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        title = request.form.get('title')
        body = request.form.get('body')
        
        if title and body:
            new_note = Note(title=title, body=body)
            try:
                db.session.add(new_note)
                db.session.commit()
                return redirect(url_for('main.index'))
            except Exception as e:
                db.session.rollback()  # Xatolik bo'lsa, sessiyani orqaga qaytaramiz
                flash("Xatolik yuz berdi. Ma'lumot saqlanmadi!")
        else:
            flash("Sarlavha va matn bo'sh bo'lishi mumkin emas!")

    # Yangidan-eskiga qarab tartiblab olish (.desc())
    notes = Note.query.order_by(Note.created_at.desc()).all()
    return render_template('main/index.html', notes=notes)
4. Jinja2 shablonini yangilash
app/templates/main/index.html
Yangi eslatma qo'shish formasi va eslatmalar ro'yxati:

HTML
{% extends "base.html" %}
{% block content %}
    <h2>Yangi Eslatma Qo'shish</h2>
    
    {% with messages = get_flashed_messages() %}
      {% if messages %}
        {% for message in messages %}
          <p style="color: red;">{{ message }}</p>
        {% endfor %}
      {% endif %}
    {% endwith %}

    <form method="POST" action="{{ url_for('main.index') }}">
        <div>
            <label>Sarlavha:</label><br>
            <input type="text" name="title" style="width: 100%; max-width: 400px;" required>
        </div>
        <br>
        <div>
            <label>Matn:</label><br>
            <textarea name="body" rows="5" style="width: 100%; max-width: 400px;" required></textarea>
        </div>
        <br>
        <button type="submit">Saqlash</button>
    </form>

    <hr>

    <h2>Barcha Eslatmalar (Yangi -> Eski)</h2>
    {% if notes %}
        {% for note in notes %}
            <div style="border: 1px solid #ccc; padding: 10px; margin-bottom: 10px; border-radius: 5px;">
                <h3>{{ note.title }}</h3>
                <p>{{ note.body }}</p>
                <small style="color: gray;">Yaratilgan vaqti: {{ note.created_at.strftime('%Y-%m-%d %H:%M') }}</small>
            </div>
        {% endfor %}
    {% else %}
        <p>Hozircha eslatmalar yo'q.</p>
    {% endif %}
{% endblock %}
5. Flask Shell orqali bazani Seed qilish (Boshlang'ich ma'lumotlar)
Terminalda loyiha ildiz papkasida turib, quyidagi buyruqlar orqali Flask shell muhitiga kiramiz va kamida 5 ta dastlabki eslatmani bazaga yozamiz:

Bash
# Terminalda:
flask shell
Ochilgan Python interaktiv muhitida quyidagi kodni bajaring:

Python
from app.extensions import db
from app.models import Note

# 5 ta yangi obyekt yaratamiz
n1 = Note(title="Birinchi eslatma", body="Bu flask shell orqali qo'shilgan 1-nota.")
n2 = Note(title="Rejalar", body="Bugun kechki payt Flask darsini yakunlash.")
n3 = Note(title="Sotib olish kerak", body="Klaviatura va sichqoncha sotib olish lozim.")
n4 = Note(title="Eslatma 4", body="Dasturni test rejimida ishga tushirib ko'rish.")
n5 = Note(title="Yakuniy eslatma", body="Ma'lumotlar bazasi muvaffaqiyatli ulandi.")

# Sessiyaga qo'shish va saqlash
db.session.add_all([n1, n2, n3, n4, n5])
db.session.commit()

# Shell'dan chiqish
exit()
Endi brauzerda loyihani ochganingizda ushbu 5 ta eslatma sanasi bo'yicha eng yangisidan boshlab teskari tartibda ko'rinadi va forma orqali yangilarini muammosiz qo'shishingiz mumkin bo'ladi.
