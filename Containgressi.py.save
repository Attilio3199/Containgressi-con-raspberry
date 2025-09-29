import time
import sqlite3
import VL53L0X
import RPi.GPIO as GPIO

# === Configurazione Database ===
db_path = "/home/BET/containgressi/eventi.db"
conn = sqlite3.connect(db_path)
cursor = conn.cursor()

cursor.execute('''
CREATE TABLE IF NOT EXISTS eventi (
    id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
    Neg TEXT,
    IdSensore TEXT,
    Momento DATETIME,
    TipoIngresso INTEGER
)
''')
conn.commit()

def salva_evento(log_str):
    campi = [campo.strip() for campo in log_str.split("|")]
    if len(campi) == 4:
        neg, idsensore, momento, tipo = campi
        cursor.execute(
            "INSERT INTO eventi (Neg, IdSensore, Momento, TipoIngresso) VALUES (?, ?, ?, ?)",
            (neg, idsensore, momento, int(tipo))
        )
        conn.commit()

# === Configurazione GPIO ===
sensor1_shutdown = 20
sensor2_shutdown = 16

GPIO.setwarnings(False)
GPIO.setmode(GPIO.BCM)
GPIO.setup(sensor1_shutdown, GPIO.OUT)
GPIO.setup(sensor2_shutdown, GPIO.OUT)

GPIO.output(sensor1_shutdown, GPIO.LOW)
GPIO.output(sensor2_shutdown, GPIO.LOW)
time.sleep(0.5)

# === Inizializza sensori ===
tof1 = VL53L0X.VL53L0X(i2c_bus=1, i2c_address=0x29)
tof2 = VL53L0X.VL53L0X(i2c_bus=1, i2c_address=0x29)

GPIO.output(sensor1_shutdown, GPIO.HIGH)
time.sleep(1)
tof1.change_address(0x2B)

GPIO.output(sensor2_shutdown, GPIO.HIGH)
time.sleep(1)
tof2.change_address(0x2D)

tof1.open()
tof1.start_ranging(VL53L0X.Vl53l0xAccuracyMode.BETTER)
tof2.open()
tof2.start_ranging(VL53L0X.Vl53l0xAccuracyMode.BETTER)

timing = 100000  # 100 ms
soglia = 150

base_distance1 = tof1.get_distance()
base_distance2 = tof2.get_distance()

print("Distanze di riferimento memorizzate")
print(f"Sensore 1: {base_distance1} mm | Sensore 2: {base_distance2} mm")

try:
    while True:
        distanza1 = tof1.get_distance()
        distanza2 = tof2.get_distance()
        var1 = base_distance1 - distanza1
        var2 = base_distance2 - distanza2
        evento_valido = True

        # Entrata
        if var1 >= soglia:
            time.sleep(0.2)
            distanza2 = tof2.get_distance()
            var2 = base_distance2 - distanza2

            if var2 >= soglia:
                tempo_inizio = time.time()
                while var1 >= soglia and var2 >= soglia:
                    if time.time() - tempo_inizio > 3:
                        evento_valido = False
                        break
                    distanza1 = tof1.get_distance()
                    distanza2 = tof2.get_distance()
                    var1 = base_distance1 - distanza1
                    var2 = base_distance2 - distanza2
                    time.sleep(0.1)

                if evento_valido:
                    timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
                    log_str = f"BET | 01 | {timestamp} | 1"
                    print(f"{log_str}  -->  D1: {distanza1} mm | D2: {distanza2} mm")
                    salva_evento(log_str)

        # Uscita
        elif var2 >= soglia:
            time.sleep(0.2)
            distanza1 = tof1.get_distance()
            var1 = base_distance1 - distanza1

            if var1 >= soglia:
                tempo_inizio = time.time()
                while var1 >= soglia and var2 >= soglia:
                    if time.time() - tempo_inizio > 3:
                        evento_valido = False
                        break
                    distanza1 = tof1.get_distance()
                    distanza2 = tof2.get_distance()
                    var1 = base_distance1 - distanza1
                    var2 = base_distance2 - distanza2
                    time.sleep(0.1)

                if evento_valido:
                    timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
                    log_str = f"BET | 01 | {timestamp} | 0"
                    print(f"{log_str}  -->  D1: {distanza1} mm | D2: {distanza2} mm")
                    salva_evento(log_str)

        time.sleep(timing / 1_000_000.0)

except KeyboardInterrupt:
    print("\n🛑 Rilevamento interrotto dall'utente.")
finally:
    tof1.stop_ranging()
    tof2.stop_ranging()
    GPIO.cleanup()
    conn.close()
