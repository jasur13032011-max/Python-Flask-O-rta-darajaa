# Python-Flask-O-rta-darajaa
Mana, barcha talablarga to'liq javob beradigan, loyiha tuzilishi toza va professional tarzda tashkil etilgan Flask ilovasi.

Ushbu loyihada create_app() factory funksiyasi, konfiguratsiya klasslari, Blueprint'lar, Many-to-Many munosabatlar, qidiruv va paginatsiya tizimi to'liq realizatsiya qilingan.

Loyiha Tuzilishi (Directory Structure)
Plaintext
my_flask_app/
│
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── models.py
│   ├── extensions.py
│   └── blueprints/
│       ├── main/
│       │   ├── __init__.py
│       │   └── routes.py
│       └── items/
│           ├── __init__.py
│           └── routes.py
│
├── templates/
│   ├── base.html
│   ├── index.html
│   ├── item_form.html
│   └── tag_items.html
│
├── migrations/          # Flask-Migrate tomonidan yaratiladi
├── README.md
├── requirements.txt
└── run.py
1. Konfiguratsiya va Ilova Factory (app/)
app/config.py
Python
import os

BASE_DIR = os.path.abspath(os.path.dirname(__file__))

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY', 'default-super-secret-key')
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    ITEMS_PER_PAGE = 10

class DevelopmentConfig(Config):
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = f"sqlite:///{os.path.join(BASE_DIR, 'dev_db.sqlite')}"

class TestingConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"

class ProductionConfig(Config):
    DEBUG = False
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL', f"sqlite:///{os.path.join(BASE_DIR, 'prod_db.sqlite')}")

config_by_name = {
    'dev': DevelopmentConfig,
    'test': TestingConfig,
    'prod': ProductionConfig
}
app/extensions.py
Python
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()
app/__init__.py
Python
from flask import Flask
from app.config import config_by_name
from app.extensions import db, migrate

def create_app(config_name='dev'):
    app = Flask(__name__, template_folder='../templates')
    app.config.from_object(config_by_name[config_name])

    # Extensionlarni initsializatsiya qilish
    db.init_app(app)
    migrate.init_app(app, db)

    # Blueprintlarni ro'yxatdan o'tkazish
    from app.blueprints.main.routes import main_bp
    from app.blueprints.items.routes import items_bp

    app.register_blueprint(main_bp)
    app.register_blueprint(items_bp, url_prefix='/items')

    return app
2. Ma'lumotlar Modellari (app/models.py)
Many-to-Many bog'lanish va cascade='all, delete-orphan' talabi to'g'ri bajarilgan model kodi:

Python
from app.extensions import db

# Many-to-Many yordamchi jadvali (Item <-> Tag)
item_tags = db.Table('item_tags',
    db.Column('item_id', db.Integer, db.ForeignKey('items.id', ondelete='CASCADE'), primary_key=True),
    db.Column('tag_id', db.Integer, db.ForeignKey('tags.id', ondelete='CASCADE'), primary_key=True)
)

class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    
    # User o'chib ketganda uning barcha e'lonlari ham o'chib ketishi uchun cascade qo'shildi
    items = db.relationship('Item', backref='author', lazy='dynamic', cascade='all, delete-orphan')

    def __repr__(self):
        return f'<User {self.username}>'

class Item(db.Model):
    __tablename__ = 'items'
    
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id', ondelete='CASCADE'), nullable=False)
    
    # Many-to-Many munosabati
    tags = db.relationship('Tag', secondary=item_tags, backref=db.backref('items', lazy='dynamic'))

    def __repr__(self):
        return f'<Item {self.title}>'

class Tag(db.Model):
    __tablename__ = 'tags'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)

    def __repr__(self):
        return f'<Tag {self.name}>'
3. Blueprint'lar va Controller'lar
app/blueprints/main/routes.py (Bosh sahifa, Qidiruv, Tag sahifasi)
Python
from flask import Blueprint, render_template, request, current_app
from app.models import Item, Tag

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
def index():
    page = request.args.get('page', 1, type=int)
    search_query = request.args.get('q', '', type=str)
    
    query = Item.query
    
    if search_query:
        # Sarlavha yoki tavsif bo'yicha qidirish
        query = query.filter(
            (Item.title.ilike(f'%{search_query}%')) | 
            (Item.description.ilike(f'%{search_query}%'))
        )
    
    # Paginatsiya: har bir sahifaga belgilangan miqdorda (10 ta) chiqarish
    pagination = query.order_by(Item.id.desc()).paginate(
        page=page, 
        per_page=current_app.config['ITEMS_PER_PAGE'], 
        error_out=False
    )
    
    return render_template('index.html', pagination=pagination, search_query=search_query)

@main_bp.route('/tag/<string:name>')
def tag_items(name):
    tag = Tag.query.filter_by(name=name).first_or_404()
    # Ushbu tagga tegishli barcha e'lonlarni olish
    items = tag.items.all()
    return render_template('tag_items.html', tag=tag, items=items)
app/blueprints/items/routes.py (Yangi e'lon qo'shish)
Python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from app.extensions import db
from app.models import Item, Tag, User

items_bp = Blueprint('items', __name__)

@items_bp.route('/add', methods=['GET', 'POST'])
def add_item():
    if request.method == 'POST':
        title = request.form.get('title')
        description = request.form.get('description')
        tags_input = request.form.get('tags', '')
        
        if not title or not description:
            flash("Sarlavha va tavsif to'ldirilishi shart!", "danger")
            return redirect(url_for('items.add_item'))
        
        # Hozircha autentifikatsiya yo'qligi sababli, test uchun birinchi foydalanuvchini olamiz
        # (Agar baza bo'sh bo'lsa, xato bermasligi uchun default foydalanuvchi yaratiladi)
        test_user = User.query.first()
        if not test_user:
            test_user = User(username="testuser", email="test@example.com")
            db.session.add(test_user)
            db.session.commit()

        new_item = Item(title=title, description=description, author=test_user)
        
        # Taglarni qayta ishlash (vergul bilan ajratilgan)
        if tags_input:
            tag_names = [t.strip().lower() for t in tags_input.split(',') if t.strip()]
            for name in tag_names:
                tag = Tag.query.filter_by(name=name).first()
                if not tag:
                    tag = Tag(name=name)
                    db.session.add(tag)
                new_item.tags.append(tag)
        
        db.session.add(new_item)
        db.session.commit()
        
        flash("E'lon muvaffaqiyatli qo'shildi!", "success")
        return redirect(url_for('main.index'))
        
    return render_template('item_form.html')
4. HTML Shablonlar (Templates)
templates/base.html
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>E'lonlar Doskasi</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
</head>
<body class="bg-light">
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark mb-4">
        <div class="container">
            <a class="navbar-brand" href="{{ url_for('main.index') }}">E'lonlar</a>
            <div class="navbar-nav ms-auto">
                <a class="nav-link btn btn-primary text-white px-3" href="{{ url_for('items.add_item') }}">Yangi E'lon</a>
            </div>
        </div>
    </nav>

    <div class="container">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }}">{{ message }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        {% block content %}{% endblock %}
    </div>
</body>
</html>
templates/index.html
HTML
{% extends 'base.html' %}

{% block content %}
<div class="row mb-4">
    <div class="col-md-6 offset-md-3">
        <form method="GET" action="{{ url_for('main.index') }}" class="d-flex">
            <input type="text" name="q" class="form-control me-2" placeholder="Qidiruv..." value="{{ search_query }}">
            <button type="submit" class="btn btn-outline-success">Izlash</button>
        </form>
    </div>
</div>

<h2>Barcha E'lonlar</h2>
<div class="row">
    {% for item in pagination.items %}
    <div class="col-md-12 mb-3">
        <div class="card shadow-sm">
            <div class="card-body">
                <h5 class="card-title">{{ item.title }}</h5>
                <p class="card-text">{{ item.description }}</p>
                <p class="text-muted small">Muallif: {{ item.author.username }}</p>
                <div>
                    {% for tag in item.tags %}
                        <a href="{{ url_for('main.tag_items', name=tag.name) }}" class="badge bg-secondary text-decoration-none">#{{ tag.name }}</a>
                    {% endfor %}
                </div>
            </div>
        </div>
    </div>
    {% else %}
    <p>Hech qanday e'lon topilmadi.</p>
    {% endfor %}
</div>

{% if pagination.pages > 1 %}
<nav class="mt-4">
    <ul class="pagination justify-content-center">
        {% for page_num in pagination.iter_pages(left_edge=1, right_edge=1, left_current=1, right_current=2) %}
            {% if page_num %}
                <li class="page-item {% if page_num == pagination.page %}active{% endif %}">
                    <a class="page-item" href="{{ url_for('main.index', page=page_num, q=search_query) }}"><span class="page-link">{{ page_num }}</span></a>
                </li>
            {% else %}
                <li class="page-item disabled"><span class="page-link">...</span></li>
            {% endif %}
        {% endfor %}
    </ul>
</nav>
{% endif %}
{% endblock %}
templates/item_form.html
HTML
{% extends 'base.html' %}

{% block content %}
<div class="row">
    <div class="col-md-8 offset-md-2">
        <div class="card p-4">
            <h3>Yangi e'lon qo'shish</h3>
            <hr>
            <form method="POST">
                <div class="mb-3">
                    <label class="form-label">Sarlavha</label>
                    <input type="text" name="title" class="form-control" required>
                </div>
                <div class="mb-3">
                    <label class="form-label">Tavsif</label>
                    <textarea name="description" rows="5" class="form-control" required></textarea>
                </div>
                <div class="mb-3">
                    <label class="form-label">Taglar (vergul bilan ajrating)</label>
                    <input type="text" name="tags" class="form-control" placeholder="avto, sotuv, yangi">
                </div>
                <button type="submit" class="btn btn-success">E'lonni joylashtirish</button>
                <a href="{{ url_for('main.index') }}" class="btn btn-secondary">Bekor qilish</a>
            </form>
        </div>
    </div>
</div>
{% endblock %}
templates/tag_items.html
HTML
{% extends 'base.html' %}

{% block content %}
<h3 class="mb-4">#{{ tag.name }} tagiga tegishli e'lonlar</h3>
<div class="row">
    {% for item in items %}
    <div class="col-md-12 mb-3">
        <div class="card">
            <div class="card-body">
                <h5>{{ item.title }}</h5>
                <p>{{ item.description }}</p>
                <p class="text-muted small">Muallif: {{ item.author.username }}</p>
            </div>
        </div>
    </div>
    {% else %}
    <p>Bu tag bilan hech qanday e'lon topilmadi.</p>
    {% endfor %}
</div>
<a href="{{ url_for('main.index') }}" class="btn btn-primary">Bosh sahifaga qaytish</a>
{% endblock %}
5. Loyihani Ishga Tushirish Fayli (run.py)
Python
from app import create_app

app = create_app('dev')  # 'dev', 'test', yoki 'prod' berish mumkin

if __name__ == '__main__':
    app.run()
6. Yo'riqnoma (README.md va requirements.txt)
requirements.txt
Plaintext
Flask==3.0.2
Flask-SQLAlchemy==3.1.1
Flask-Migrate==4.0.7
README.md
Markdown
# Flask E'lonlar Tizimi Ilovasi

Ushbu loyiha e'lonlarni boshqarish, qidiruv tizimi, ko'pga-ko'p (many-to-many) bog'lanishlar hamda paginatsiyani o'z ichiga oladi.

## Loyihani O'rnatish va Ishga Tushirish Qadamlari

### 1. Virtual Muhitni Yaratish va Faollashtirish
```bash
# Virtual muhit yaratish
python -m venv venv

# Faollashtirish (Windows)
venv\Scripts\activate

# Faollashtirish (Linux / MacOS)
source venv/bin/activate
2. Kutubxonalarni O'rnatish
Bash
pip install -r requirements.txt
3. Ma'lumotlar Bazasi Migratsiyasi (Flask-Migrate qadamlari)
Loyihada ma'lumotlar bazasini sozlash va birinchi migratsiyani amalga oshirish uchun quyidagi buyruqlarni ketma-ket bajaring:

Bash
# Migratsiya muhitini initsializatsiya qilish (faqat bir marta bajariladi)
flask db init

# Birinchi migratsiya faylini yaratish
flask db migrate -m "Initial migration with User, Item, Tag models"

# Migratsiyani bazaga qo'llash (dev_db.sqlite hosil bo'ladi)
flask db upgrade
4. Loyihani Ishga Tushirish
Bash
python run.py
Ilova avtomatik ravishda http://127.0.0.1:5000/ manzilida ishga tushadi.
