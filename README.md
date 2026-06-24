# Python-Flask-O-rta-darajaa
Mana, siz so'ragan barcha talablar asosida tayyorlangan Flask-SQLAlchemy modellari va ma'lumotlarni qo'shish uchun Flask shell skripti.

1. Modellar va Ma'lumotlar Bazasi Strukturasi
Barcha munosabatlar (One-to-Many, Many-to-Many kaskadli o'chirish bilan) quyidagi kodda shakllantirilgan:

Python
from flask import Flask, render_template
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///notes.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Many-to-Many munosabati uchun oraliq jadval (Note <-> Tag)
note_tags = db.Table('note_tags',
    db.Column('note_id', db.Integer, db.ForeignKey('note.id', ondelete='CASCADE'), primary_key=True),
    db.Column('tag_id', db.Integer, db.ForeignKey('tag.id', ondelete='CASCADE'), primary_key=True)
)

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    
    # Cascade 'all, delete-orphan' - User o'chganda uning barcha notalari ham o'chadi
    notes = db.relationship('Note', back_populates='author', cascade='all, delete-orphan')

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    
    # ForeignKey munosabati
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    author = db.relationship('User', back_populates='notes')
    
    # Many-to-Many munosabati backref bilan
    tags = db.relationship('Tag', secondary=note_tags, backref=db.backref('notes', lazy='dynamic'))

class Tag(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)
2. Flask Shell Skripti (Ma'lumotlarni yaratish)
Terminalda flask shell ni oching va quyidagi kodni ketma-ketlikda bajaring (yoki alohida seed.py skripti sifatida ishlating):

Python
# Bazani yaratish
db.create_all()

# 1. 3 ta User yaratish
u1 = User(username='ali')
u2 = User(username='vali')
u3 = User(username='guli')

db.session.add_all([u1, u2, u3])
db.session.commit()

# 2. 4 ta Tag yaratish
t1 = Tag(name='Dasturlash')
t2 = Tag(name='Hayotiy')
t3 = Tag(name='Eslatma')
t4 = Tag(name='Muhim')

db.session.add_all([t1, t2, t3, t4])
db.session.commit()

# 3. 6 ta Note yaratish va ularni User hamda Taglarga bog'lash
n1 = Note(title='Python o\'rganish', content='Har kuni 2 soat kod yozish kerak.', author=u1)
n1.tags.append(t1)
n1.tags.append(t4)

n2 = Note(title='Bozorlik ro\'yxati', content='Sut, non, mevalar olish kerak.', author=u1)
n2.tags.append(t3)

n3 = Note(title='Ertalabki yugurish', content='Sog\'liq uchun foydali odat.', author=u2)
n3.tags.append(t2)

n4 = Note(title='Flask Framework', content='Veb ilovalar yaratish uchun qulay.', author=u2)
n4.tags.append(t1)

n5 = Note(title='Kitob o\'qish', content='Shaytanat romanini yakunlash.', author=u3)
n5.tags.append(t2)
n5.tags.append(t3)

n6 = Note(title='Yig\'ilish', content='Dushanba kuni soat 10:00 da muhokama.', author=u3)
n6.tags.append(t4)

db.session.add_all([n1, n2, n3, n4, n5, n6])
db.session.commit()

print("Ma'lumotlar muvaffaqiyatli qo'shildi!")
3. Bosh Sahifa (Route va HTML)
Bosh sahifada har bir notaning yonida uning muallifi (author) va tegishli taglarini ko'rsatish:

Python Route (app.py ichida):
Python
@app.route('/')
def index():
    # Notalarni bazadan yuklab olamiz
    notes = Note.query.all()
    return render_template('index.html', notes=notes)

if __name__ == '__main__':
    app.run(debug=True)
HTML Shablon (templates/index.html):
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Bosh Sahifa - Notalar</title>
    <style>
        .note-card { border: 1px solid #ccc; padding: 15px; margin-bottom: 15px; border-radius: 5px; }
        .author { color: #555; font-style: italic; }
        .tag { background-color: #e0e0e0; padding: 2px 8px; margin-right: 5px; border-radius: 3px; font-size: 12px; }
    </style>
</head>
<body>

    <h1>Barcha Notalar</h1>

    {% for note in notes %}
        <div class="note-card">
            <h2>{{ note.title }}</h2>
            <p>{{ note.content }}</p>
            
            <p class="author">Muallif: <strong>{{ note.author.username }}</strong></p>
            
            <div class="tags">
                Teghlar:
                {% for tag in note.tags %}
                    <span class="tag">#{{ tag.name }}</span>
                {% else %}
                    <span style="color: gray;">Teglar yo'q</span>
                {% endfor %}
            </div>
        </div>
    {% endfor %}

</body>
</html>
