from flask import Flask, render_template, request, redirect, url_for, session
import sqlite3

app = Flask(__name__)
app.secret_key = 'mysecretkey'  # تستخدم لتخزين الجلسات بشكل آمن

# إعداد قاعدة بيانات SQLite للمستخدمين والمهام
def init_db():
    conn = sqlite3.connect('app.db')
    cursor = conn.cursor()
    
    # إنشاء جدول المستخدمين إذا لم يكن موجودًا
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE,
            password TEXT
        )
    ''')
    
    # إنشاء جدول المهام
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            task_name TEXT,
            task_type TEXT,
            user_id INTEGER,
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    ''')
    conn.commit()
    conn.close()

# استدعاء دالة إعداد قاعدة البيانات عند بداية التشغيل
init_db()

# مسار الصفحة الرئيسية لعرض النموذج
@app.route('/')
def index():
    return render_template('index.html')

# مسار لتسجيل الدخول
@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']
    conn = sqlite3.connect('app.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM users WHERE username = ? AND password = ?', (username, password))
    user = cursor.fetchone()
    conn.close()
    if user:
        session['user_id'] = user[0]  # تخزين معرف المستخدم في الجلسة
        return redirect(url_for('tasks_page'))
    else:
        return "اسم المستخدم أو كلمة المرور غير صحيحة!"

# مسار لعرض وإضافة المهام
@app.route('/tasks', methods=['GET', 'POST'])
def tasks_page():
    if 'user_id' not in session:
        return redirect(url_for('index'))

    conn = sqlite3.connect('app.db')
    cursor = conn.cursor()
    
    if request.method == 'POST':
        task_name = request.form['task_name']
        task_type = request.form['task_type']
        user_id = session['user_id']
        cursor.execute('INSERT INTO tasks (task_name, task_type, user_id) VALUES (?, ?, ?)',
                       (task_name, task_type, user_id))
        conn.commit()

    cursor.execute('SELECT task_name, task_type FROM tasks WHERE user_id = ?', (session['user_id'],))
    tasks = cursor.fetchall()
    conn.close()
    
    return render_template('tasks.html', tasks=tasks)

if __name__ == '__main__':
    app.run(debug=True)
