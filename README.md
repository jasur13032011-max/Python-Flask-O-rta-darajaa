# Python-Flask-O-rta-darajaa
1. Migratsiyalar ierarxiyasi va Tarixi
Sizda hozirda ikkita migratsiya fayli mavjud. Ularning ketma-ketligini terminalda tekshirish va README uchun ko'rinish hosil qilish uchun quyidagi buyruqlardan foydalanasiz:

Migratsiyalar tarixini ko'rish:

Bash
flask db history
Output quyidagicha ko'rinishda bo'ladi (Screenshot uchun asos):

Plaintext
<rev_id_2> -> <rev_id_1> (head), add is_pinned to notes
base -> <rev_id_2>, initial tables
Hozirgi bazaning holatini ko'rish:

Bash
flask db current
Output (Agar baza oxirgi versiyada bo'lsa):

Plaintext
<rev_id_1> (head)
2. Bosh sahifada Notalarni Saralash (is_pinned)
Notalarni is_pinned=True bo'lganlarini yuqorida, qolganlarini esa yaratilgan vaqti bo'yicha ko'rsatish uchun SQLAlchemy so'rovini (query) quyidagicha yozish kerak:

Python
from models import Note

@app.route('/')
def index():
    # Avval pin qilinganlar, keyin esa ID yoki vaqt bo'yicha teskari saralash
    notes = Note.query.order_by(Note.is_pinned.desc(), Note.id.desc()).all()
    return render_template('index.html', notes=notes)
3. README.md uchun CLI Komandalar To'plami
Loyiha hujjatiga qo'shish uchun tayyor buyruqlar va ularning vazifalari:

Markdown
## Ma'lumotlar bazasi migratsiyalari (Flask-Migrate)

Loyiha ma'lumotlar bazasini boshqarish uchun `Flask-Migrate` paketidan foydalanadi. Quyida asosiy CLI komandalar keltirilgan:

* **Migratsiya muhitini yaratish** (Faqat loyiha boshida 1 marta ishlatiladi):
    ```bash
    flask db init
    ```

* **Yangi migratsiya faylini yaratish** (Modellarga o'zgartirish kiritilganda):
    ```bash
    flask db migrate -m "Migratsiya tavsifi"
    ```

* **O'zgarishlarni bazaga qo'llash** (Baza jadvallarini yangilash):
    ```bash
    flask db upgrade
    ```

* **Orqaga qaytarish (Downgrade)** (Oxirgi migratsiyani bekor qilish):
    ```bash
    flask db downgrade
    ```

* **Migratsiyalar tarixini va joriy holatni tekshirish**:
    ```bash
    flask db history
    flask db current
    ```
4. Git bilan ishlash bo'yicha eslatma
Ikkala migratsiya ham commit qilingani juda yaxshi amaliyot (best practice). Jamoada boshqa dasturchilar loyihani yuklab olganda, ular shunchaki quyidagi buyruqni bajarishsa kifoya:

Bash
flask db upgrade
Bu buyruq migrations/versions/ ichidagi ikkala faylni ham o'qib, bazani eng oxirgi (add is_pinned to notes) holatiga keltirib qo'yadi.
