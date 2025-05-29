import os
import time
import json
import sqlite3

import cv2
import numpy as np
from picamera2 import Picamera2
import busio
import board
import adafruit_mlx90640
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

from flask import (
    Flask, render_template, request,
    redirect, url_for, session, flash,
    g, Response, jsonify
)

app = Flask(
    __name__,
    template_folder='templates',
    static_folder='static'
)
app.secret_key = 'votre_cle_secrete'  # À remplacer en production

# --- INITIALISATION LAZY DE PICAMERA2 ---
picam2 = None
def get_picam2():
    global picam2
    if picam2 is None:
        picam2 = Picamera2()
        cfg = picam2.create_video_configuration(main={"size": (320, 240)})
        picam2.configure(cfg)
        picam2.start()
        time.sleep(1)
    return picam2

# --- HELPER SQLITE ---
def get_db():
    if 'db' not in g:
        g.db = sqlite3.connect('app.db')
        g.db.row_factory = sqlite3.Row
        g.db.execute('''
          CREATE TABLE IF NOT EXISTS patients(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            prenom TEXT,
            nom TEXT,
            dob TEXT,
            cin TEXT,
            date_consult TEXT
          )
        ''')
        g.db.execute('''
          CREATE TABLE IF NOT EXISTS captures(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            patient_id INTEGER,
            type TEXT,
            filename TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
          )
        ''')
        g.db.commit()
    return g.db

@app.teardown_appcontext
def close_db(exc):
    db = g.pop('db', None)
    if db:
        db.close()

# --- AUTHENTIFICATION ---
def check_credentials(user, pwd):
    return user == 'admin' and pwd == 'password123'

@app.route('/login', methods=['GET','POST'])
def login():
    if request.method == 'POST':
        u = request.form['username']
        p = request.form['password']
        if check_credentials(u, p):
            session['user'] = u
            return redirect(url_for('patients'))
        flash('Identifiants incorrects', 'danger')
    return render_template('login.html')

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('login'))

# --- LISTE DES PATIENTS ---
@app.route('/patients')
def patients():
    if 'user' not in session:
        return redirect(url_for('login'))
    db = get_db()
    pts = db.execute(
        'SELECT id, prenom, nom FROM patients'
    ).fetchall()
    return render_template('patients.html', patients=pts)

# --- AJOUT D'UN NOUVEAU PATIENT ---
@app.route('/patients/new', methods=['GET','POST'])
def new_patient():
    if 'user' not in session:
        return redirect(url_for('login'))
    if request.method == 'POST':
        db = get_db()
        cur = db.execute(
            'INSERT INTO patients(prenom,nom,dob,cin,date_consult) VALUES (?,?,?,?,?)',
            (
                request.form['prenom'],
                request.form['nom'],
                request.form['dob'],
                request.form['cin'],
                request.form['date_consult']
            )
        )
        db.commit()
        return redirect(url_for('capture', patient_id=cur.lastrowid))
    return render_template('new_patient.html')

# --- MODIFIER LES INFOS D’UN PATIENT EXISTANT ---
@app.route('/patients/edit/<int:patient_id>', methods=['GET','POST'])
def edit_patient(patient_id):
    if 'user' not in session:
        return redirect(url_for('login'))
    db = get_db()
    if request.method == 'POST':
        db.execute(
            'UPDATE patients SET prenom=?, nom=?, dob=?, cin=?, date_consult=? WHERE id=?',
            (
                request.form['prenom'],
                request.form['nom'],
                request.form['dob'],
                request.form['cin'],
                request.form['date_consult'],
                patient_id
            )
        )
        db.commit()
        flash("Informations du patient mises à jour.", "success")
        return redirect(url_for('patients'))
    p = db.execute(
        'SELECT * FROM patients WHERE id = ?', (patient_id,)
    ).fetchone()
    return render_template('edit_patient.html', patient=p)

# --- SUPPRIMER UN PATIENT ---
@app.route('/patients/delete/<int:patient_id>', methods=['POST'])
def delete_patient(patient_id):
    if 'user' not in session:
        return redirect(url_for('login'))
    db = get_db()
    db.execute('DELETE FROM captures WHERE patient_id = ?', (patient_id,))
    db.execute('DELETE FROM patients WHERE id = ?', (patient_id,))
    db.commit()
    flash('Patient et captures supprimés.', 'success')
    return redirect(url_for('patients'))

# --- FLUX PI CAMERA ---
def gen_pi_stream():
    cam = get_picam2()
    while True:
        frame = cam.capture_array()
        _, jpg = cv2.imencode('.jpg', frame)
        yield (
            b'--frame\r\n'
            b'Content-Type: image/jpeg\r\n\r\n' +
            jpg.tobytes() +
            b'\r\n'
        )

@app.route('/pi_feed')
def pi_feed():
    return Response(
        gen_pi_stream(),
        mimetype='multipart/x-mixed-replace;boundary=frame'
    )

# --- FLUX THERMIQUE ---
def gen_thermal_stream():
    i2c = busio.I2C(board.SCL, board.SDA, frequency=400000)
    mlx = adafruit_mlx90640.MLX90640(i2c)
    mlx.refresh_rate = adafruit_mlx90640.RefreshRate.REFRESH_2_HZ
    frame = [0] * 768
    while True:
        mlx.getFrame(frame)
        data = np.reshape(frame, (24,32))
        fig, ax = plt.subplots(figsize=(3.2,2.4), dpi=100)
        ax.imshow(data, cmap='hot', vmin=20, vmax=40)
        ax.axis('off')
        fig.canvas.draw()
        h, w = fig.canvas.get_width_height()
        img = np.frombuffer(
            fig.canvas.tostring_rgb(),
            dtype=np.uint8
        ).reshape((h,w,3))
        plt.close(fig)
        _, jpg = cv2.imencode('.jpg', img)
        yield (
            b'--frame\r\n'
            b'Content-Type: image/jpeg\r\n\r\n' +
            jpg.tobytes() +
            b'\r\n'
        )

@app.route('/thermal_feed')
def thermal_feed():
    return Response(
        gen_thermal_stream(),
        mimetype='multipart/x-mixed-replace;boundary=frame'
    )

# --- PAGE DE CAPTURE ---
@app.route('/capture/<int:patient_id>')
def capture(patient_id):
    if 'user' not in session:
        return redirect(url_for('login'))
    db = get_db()
    caps = db.execute(
        'SELECT type,filename,timestamp FROM captures '
        'WHERE patient_id = ? ORDER BY timestamp DESC',
        (patient_id,)
    ).fetchall()
    return render_template(
        'capture.html',
        patient_id=patient_id,
        captures=caps
    )

@app.route('/capture_eye/<int:patient_id>')
def capture_eye(patient_id):
    cam = get_picam2()
    frame = cam.capture_array()
    _, jpg = cv2.imencode('.jpg', frame)
    fname = f"eye_{patient_id}_{int(time.time())}.jpg"
    path  = os.path.join('static/eye_captures', fname)
    with open(path, 'wb') as f:
        f.write(jpg.tobytes())
    db = get_db()
    db.execute(
        'INSERT INTO captures(patient_id,type,filename) VALUES (?,?,?)',
        (patient_id, 'eye', fname)
    )
    db.commit()
    return fname, 200

@app.route('/thermal_data/<int:patient_id>')
def thermal_data(patient_id):
    frame = [0] * 768
    i2c = busio.I2C(board.SCL, board.SDA, frequency=400000)
    mlx = adafruit_mlx90640.MLX90640(i2c)
    mlx.refresh_rate = adafruit_mlx90640.RefreshRate.REFRESH_2_HZ
    mlx.getFrame(frame)
    arr = np.reshape(frame, (24,32))
    return json.dumps({
        'min':  float(arr.min()),
        'max':  float(arr.max()),
        'mean': float(arr.mean())
    })

# --- GALERIE DE PHOTOS ---
@app.route('/photos/<int:patient_id>')
def photos(patient_id):
    if 'user' not in session:
        return redirect(url_for('login'))
    db = get_db()
    caps = db.execute(
        'SELECT filename,timestamp FROM captures '
        'WHERE patient_id = ? ORDER BY timestamp DESC',
        (patient_id,)
    ).fetchall()
    return render_template(
        'photos.html',
        patient_id=patient_id,
        captures=caps
    )

# --- SUPPRESSION DÉFINITIVE DE PHOTOS ---
@app.route('/photos/delete', methods=['POST'])
def delete_photos():
    if 'user' not in session:
        return jsonify({'error': 'Not authenticated'}), 403

    data = request.get_json(force=True)
    filenames = data.get('files', [])

    db = get_db()
    db.executemany(
        'DELETE FROM captures WHERE filename = ?',
        [(fn,) for fn in filenames]
    )
    db.commit()

    for fn in filenames:
        path = os.path.join('static', 'eye_captures', fn)
        if os.path.exists(path):
            os.remove(path)

    return jsonify({'removed': filenames})

if __name__ == '__main__':
    os.makedirs(os.path.join('static','eye_captures'), exist_ok=True)
    app.run(host='0.0.0.0', port=5000, debug=True)
