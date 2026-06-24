# Python-Flask-O-rta-darajaa
Mana, siz aytgan barcha talablarga javob beradigan, Flask, Flask-SQLAlchemy va Jinja2 andozalari asosida yozilgan to'liq kod strukturasi.

Bu yerda N+1 muammosi joinedload orqali hal qilingan, qidiruv, paginatsiya va seed ma'lumotlar to'liq qamrab olingan.

1. Flask Ilovasi va Ma'lumotlar Bazasi Modeli (app.py)
Python
import random
from flask import Flask, render_template, request
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import or_
from sqlalchemy.orm import joinedload

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///notes.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# --- Modellar ---

# Note va Tag o'rtasidagi ko'pga-ko'p (Many-to-Many) bog'liqlik jadvali
note_tags = db.Table('note_tags',
    db.Column('note_id', db.Integer, db.ForeignKey('note.id'), primary_key=True),
    db.Column('tag_id', db.Integer, db.ForeignKey('tag.id'), primary_key=True)
)

class User(db.Model):
    id = db.開設 = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)
    notes = db.relationship('Note', backref='user', lazy=True)

class Tag(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(30), unique=True, nullable=False)

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    body = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    tags = db.relationship('Tag', secondary=note_tags, backref=db.backref('notes', lazy='dynamic'))

# --- Marshrutlar (Routes) ---

@app.route('/')
def index():
    page = request.args.get('page', 1, type=int)
    search_query = request.args.get('q', '', type=str)
    
    # N+1 muammosini oldini olish uchun joinedload(Note.user) ishlatamiz
    query = Note.query.options(joinedload(Note.user))
    
    # qidiruv parametri bo'lsa, title VA body bo'yicha qidiramiz
    if search_query:
        query = query.filter(
            or_(
                Note.title.ilike(f"%{search_query}%"),
                Note.body.ilike(f"%{search_query}%")
            )
        )
    
    # Paginatsiya: har sahifada 10 tadan nota
    pagination = query.paginate(page=page, per_page=10, error_out=False)
    return render_template('index.html', pagination=pagination, search_query=search_query, endpoint='index')


@app.route('/tag/<name>')
def tag_notes(name):
    page = request.args.get('page', 1, type=int)
    tag = Tag.query.filter_by(name=name).first_or_404()
    
    # Tegga tegishli jurnallarni paginatsiya bilan yuklash
    pagination = Note.query.options(joinedload(Note.user))\
        .filter(Note.tags.contains(tag))\
        .paginate(page=page, per_page=10, error_out=False)
        
    return render_template('index.html', pagination=pagination, endpoint='tag_notes', name=name)


@app.route('/user/<username>')
def user_notes(username):
    page = request.args.get('page', 1, type=int)
    user = User.query.filter_by(username=username).first_or_404()
    
    # Foydalanuvchining notalarini yuklash
    pagination = Note.query.options(joinedload(Note.user))\
        .filter_by(user_id=user.id)\
        .paginate(page=page, per_page=10, error_out=False)
        
    return render_template('index.html', pagination=pagination, endpoint='user_notes', username=username)


# --- Seed ma'lumotlar (Testlash uchun) ---
def seed_data():
    db.create_all()
    if User.query.first() is None:
        # Foydalanuvchilar va Teglar yaratish
        users = [User(username=f"user_{i}") for i in range(1, 5)]
        tags = [Tag(name=f"tag_{i}") for i in range(1, 6)]
        db.session.add_all(users + tags)
        db.session.commit()

        # 35 ta Nota generatsiya qilish
        for i in range(1, 36):
            note = Note(
                title=f"Nota sarlavhasi #{i}",
                body=f"Bu #{i}-notaning matni hisoblanadi. Bu yerda qidiruv so'zlari bo'lishi mumkin.",
                user=random.choice(users)
            )
            # Tasodifiy 1-2 ta teg biriktirish
            note.tags.extend(random.sample(tags, k=random.randint(1, 2)))
            db.session.add(note)
        
        db.session.commit()
        print("Baza muvaffaqiyatli seed qilindi (35 ta nota yaratildi)!")

if __name__ == '__main__':
    with app.app_context():
        seed_data()
    app.run(debug=True)
2. Jinja2 Andozasi (templates/index.html)
Barcha sahifalar (Bosh sahifa, Tag va User sahifalari) uchun bitta umumiy andozadan foydalanamiz. Sahifa pastidagi navigatsiya dinamik ishlaydi.

HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Notalar Tizimi</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .note { border: 1px solid #ccc; padding: 15px; margin-bottom: 10px; border-radius: 5px; }
        .pagination { margin-top: 20px; display: flex; gap: 10px; align-items: center; }
        .pagination a { text-decoration: none; padding: 5px 10px; border: 1px solid #007BFF; color: #007BFF; border-radius: 3px; }
        .pagination span.current { padding: 5px 10px; background-color: #007BFF; color: white; border-radius: 3px; }
        .tags { margin-top: 5px; }
        .tag { background: #e0e0e0; padding: 2px 6px; margin-right: 5px; border-radius: 3px; font-size: 12px; text-decoration: none; color: #333; }
    </style>
</head>
<body>

    <h1><a href="/" style="text-decoration: none; color: black;">Notalar Bosh Sahifasi</a></h1>

    <form method="GET" action="/">
        <input type="text" name="q" value="{{ search_query or '' }}" placeholder="Sarlavha yoki matn bo'yicha qidirish...">
        <button type="submit">Qidirish</button>
    </form>

    <hr>

    {% if pagination.items %}
        {% for note in pagination.items %}
            <div class="note">
                <h3>{{ note.title }}</h3>
                <p>{{ note.body }}</p>
                <small>Muallif: <a href="{{ url_for('user_notes', username=note.user.username) }}">{{ note.user.username }}</a></small>
                <div class="tags">
                    {% for tag in note.tags %}
                        <a href="{{ url_for('tag_notes', name=tag.name) }}" class="tag">#{{ tag.name }}</a>
                    {% endfor %}
                </div>
            </div>
        {% endfor %}
    {% else %}
        <p>Hech qanday nota topilmadi.</p>
    {% endif %}

    {% if pagination.pages > 1 %}
    <div class="pagination">
        
        {% if pagination.has_prev %}
            <a href="{{ url_for(endpoint, page=pagination.prev_num, q=search_query, name=name, username=username) }}">Oldingi</a>
        {% else %}
            <span style="color: gray;">Oldingi</span>
        {% endif %}

        {% for page_num in pagination.iter_pages(left_edge=2, left_current=2, right_current=3, right_edge=2) %}
            {% if page_num %}
                {% if page_num == pagination.page %}
                    <span class="current">{{ page_num }}</span>
                {% else %}
                    <a href="{{ url_for(endpoint, page=page_num, q=search_query, name=name, username=username) }}">{{ page_num }}</a>
                {% endif %}
            {% else %}
                <span>...</span>
            {% endif %}
        {% endfor %}

        {% if pagination.has_next %}
            <a href="{{ url_for(endpoint, page=pagination.next_num, q=search_query, name=name, username=username) }}">Keyingi</a>
        {% else %}
            <span style="color: gray;">Keyingi</span>
        {% endif %}

    </div>
    <p><small>Jami sahifalar: {{ pagination.page }} / {{ pagination.pages }}</small></p>
    {% endif %}

</body>
</html>
Kodning muhim qismlari tushuntirishi:
N+1 muammosi yechimi (joinedload): Note.query.options(joinedload(Note.user)) yordamida SQL-da avtomatik ravishda JOIN amalga oshiriladi. Natijada har bir notaning muallifini tekshirganda bazaga qayta-qayta so'rov (N ta so'rov) yuborilmaydi, hammasi 1 ta so'rovda hal bo'ladi.

Qidiruv (or_ + ilike): Note.title.ilike(f"%{search_query}%") va Note.body.ilike(...) usullari registrni inobatga olmagan holda (ilike), sarlavha yoki matn ichidan qidiruv so'zini qidiradi.

Dinamik Paginatsiya: url_for(endpoint, ...) qismi orqali bitta HTML andozaning o'zi ham qidiruv natijalarida, ham foydalanuvchi yoki teg sahifalarida to'g'ri linklarni shakllantirib bera oladi.
