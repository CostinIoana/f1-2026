import os
import requests
import pandas as pd
from flask import Flask, request, render_template, redirect, send_file
from flask_sqlalchemy import SQLAlchemy
from io import BytesIO

app = Flask(__name__)

DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///f1.db")
ADMIN_KEY = os.getenv("ADMIN_KEY", "F12026admin")

app.config["SQLALCHEMY_DATABASE_URI"] = DATABASE_URL
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

db = SQLAlchemy(app)

# -------------------
# MODELS
# -------------------

class Season(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    year = db.Column(db.Integer, unique=True)

class Player(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String, unique=True)

class Driver(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    ref = db.Column(db.String, unique=True)
    name = db.Column(db.String)

class PlayerDriver(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    season_id = db.Column(db.Integer)
    player_id = db.Column(db.Integer)
    driver_id = db.Column(db.Integer)

class Race(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    season_id = db.Column(db.Integer)
    round = db.Column(db.Integer)
    name = db.Column(db.String)

class Result(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    race_id = db.Column(db.Integer)
    driver_id = db.Column(db.Integer)
    points = db.Column(db.Float)

# -------------------
# INIT DATA
# -------------------

def init_2025():
    if not Season.query.filter_by(year=2025).first():
        season = Season(year=2025)
        db.session.add(season)
        db.session.commit()

    if Driver.query.count() == 0:
        drivers_2025 = [
            ("verstappen","Max Verstappen"),
            ("perez","Sergio Perez"),
            ("leclerc","Charles Leclerc"),
            ("sainz","Carlos Sainz"),
            ("hamilton","Lewis Hamilton"),
            ("russell","George Russell"),
            ("norris","Lando Norris"),
            ("piastri","Oscar Piastri"),
            ("alonso","Fernando Alonso"),
            ("stroll","Lance Stroll"),
            ("gasly","Pierre Gasly"),
            ("ocon","Esteban Ocon"),
            ("tsunoda","Yuki Tsunoda"),
            ("ricciardo","Daniel Ricciardo"),
            ("albon","Alexander Albon"),
            ("sargeant","Logan Sargeant"),
            ("bottas","Valtteri Bottas"),
            ("zhou","Guanyu Zhou"),
            ("hulkenberg","Nico Hulkenberg"),
            ("magnussen","Kevin Magnussen")
        ]
        for ref,name in drivers_2025:
            db.session.add(Driver(ref=ref,name=name))
        db.session.commit()

# -------------------
# PUBLIC
# -------------------

@app.route("/")
def index():
    year = int(request.args.get("season",2025))
    season = Season.query.filter_by(year=year).first()

    players = Player.query.all()
    table = []

    for p in players:
        total = 0

        pds = PlayerDriver.query.filter_by(player_id=p.id,season_id=season.id).all()
        driver_ids = [pd.driver_id for pd in pds]

        races = Race.query.filter_by(season_id=season.id).all()
        for r in races:
            results = Result.query.filter(Result.race_id==r.id, Result.driver_id.in_(driver_ids)).all()
            total += sum(res.points for res in results)

        table.append((p.name,total))

    table.sort(key=lambda x: x[1], reverse=True)

    seasons = Season.query.all()
    return render_template("index.html",table=table,seasons=seasons,current=year)

# -------------------
# ADMIN
# -------------------

@app.route("/admin")
def admin():
    if request.args.get("key") != ADMIN_KEY:
        return "Unauthorized"

    seasons = Season.query.all()
    players = Player.query.all()
    drivers = Driver.query.all()

    return render_template("admin.html",seasons=seasons,players=players,drivers=drivers)

@app.route("/add_player", methods=["POST"])
def add_player():
    name = request.form["name"]
    if not Player.query.filter_by(name=name).first():
        db.session.add(Player(name=name))
        db.session.commit()
    return redirect("/admin?key="+ADMIN_KEY)

@app.route("/create_season", methods=["POST"])
def create_season():
    year = int(request.form["year"])
    if not Season.query.filter_by(year=year).first():
        db.session.add(Season(year=year))
        db.session.commit()
    return redirect("/admin?key="+ADMIN_KEY)

@app.route("/clone_players", methods=["POST"])
def clone_players():
    from_year = int(request.form["from"])
    to_year = int(request.form["to"])

    s_from = Season.query.filter_by(year=from_year).first()
    s_to = Season.query.filter_by(year=to_year).first()

    for p in Player.query.all():
        existing = PlayerDriver.query.filter_by(player_id=p.id,season_id=s_to.id).first()
        if not existing:
            pass

    return redirect("/admin?key="+ADMIN_KEY)

# -------------------
# IMPORT ERGAST
# -------------------

@app.route("/import")
def import_last():
    if request.args.get("key") != ADMIN_KEY:
        return "Unauthorized"

    year = int(request.args.get("season",2025))
    season = Season.query.filter_by(year=year).first()

    url = f"https://ergast.com/api/f1/{year}/last/results.json"
    data = requests.get(url).json()

    race_data = data["MRData"]["RaceTable"]["Races"][0]
    round_no = int(race_data["round"])
    race_name = race_data["raceName"]

    existing = Race.query.filter_by(season_id=season.id,round=round_no).first()
    if existing:
        return "Race already imported"

    race = Race(season_id=season.id,round=round_no,name=race_name)
    db.session.add(race)
    db.session.commit()

    for r in race_data["Results"]:
        ref = r["Driver"]["driverId"]
        points = float(r["points"])
        driver = Driver.query.filter_by(ref=ref).first()
        if driver:
            db.session.add(Result(race_id=race.id,driver_id=driver.id,points=points))

    db.session.commit()
    return redirect("/admin?key="+ADMIN_KEY)

# -------------------
# EXPORT
# -------------------

@app.route("/export")
def export_excel():
    year = int(request.args.get("season",2025))
    season = Season.query.filter_by(year=year).first()

    data = []

    for p in Player.query.all():
        total = 0
        pds = PlayerDriver.query.filter_by(player_id=p.id,season_id=season.id).all()
        driver_ids = [pd.driver_id for pd in pds]

        races = Race.query.filter_by(season_id=season.id).all()
        for r in races:
            results = Result.query.filter(Result.race_id==r.id, Result.driver_id.in_(driver_ids)).all()
            total += sum(res.points for res in results)

        data.append({"Player":p.name,"Points":total})

    df = pd.DataFrame(data).sort_values("Points",ascending=False)

    output = BytesIO()
    df.to_excel(output,index=False)
    output.seek(0)

    return send_file(output, download_name=f"F1_{year}.xlsx", as_attachment=True)

# -------------------

with app.app_context():
    db.create_all()
    init_2025()

if __name__ == "__main__":
    app.run()
