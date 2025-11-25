# Smart-becon-color-sorting-machine

import cv2
import numpy as np
import asyncio
import random
import PySimpleGUI as sg
import pyttsx3
from asyncua import Client, ua
import mysql.connector
from mysql.connector import Error
from datetime import date
import matplotlib.pyplot as plt

# ===== Color Dictionaries and HSV Ranges =====
workpiece_color_dict = {
    1: "GREEN",
    2: "RED",
    3: "YELLOW",
    4: "BLUE",
    5: "ORANGE"
}

color_name_to_id = {v: k for k, v in workpiece_color_dict.items()}

color_ranges = {
    'RED':     [(172, 100, 100), (179, 255, 255)],
    'ORANGE':  [(00, 165, 150), (21, 255, 255)],
    'YELLOW':  [(22, 150, 150), (30, 255, 255)],
    'GREEN':   [(45, 100, 100), (85, 255, 255)],
    'BLUE':    [(83, 100, 100), (105, 255, 255)],
}

# ===== UI/Plot Color Map (added) =====
COLOR_HEX = {
    'RED':    '#FF4D4D',
    'ORANGE': '#FFA500',
    'YELLOW': '#FFD700',
    'GREEN':  '#32CD32',
    'BLUE':   '#1E90FF',
}
DIM_HEX = '#2B2B2B'  # unlit circle fill

# ===== TTS Function =====
def speak(text):
    try:
        engine = pyttsx3.init(driverName='sapi5')
        engine.say(text)
        engine.runAndWait()
        engine.stop()
    except Exception as e:
        print(f"TTS Error: {e}")

# ===== OPC UA Control =====
async def beacon_control(segment, color):
    url = "opc.tcp://192.168.0.26:4840/"
    try:
        async with Client(url=url) as client:
            light_segment = client.get_node("ns=4;s=MAIN.lightSegment")
            workpiece_color = client.get_node("ns=4;s=MAIN.workpieceColor")
            await workpiece_color.set_value(ua.DataValue(ua.Variant(color, ua.VariantType.Int16)))
            await light_segment.set_value(ua.DataValue(ua.Variant(segment, ua.VariantType.Int16)))
    except:
        print("Simulated beacon write (server not connected)")

async def blink_beacon():
    for _ in range(5):
        await beacon_control(6, 1)
        await asyncio.sleep(0.5)
        await beacon_control(7, 1)
        await asyncio.sleep(0.5)

# ===== MYSQL Database Update =====
def connect():
    try:
        conn = mysql.connector.connect(
            host="localhost",
            database="workpiece_color_database",
            user="me2631",
            password="me2631P2447029"
        )
        return conn
    except (Exception, Error) as error:
        print("DB Connection Error:", error)
        return None

def update_color_counts(color_list):
    conn = connect()
    if conn is None:
        print("Cannot update color counts - DB not connected.")
        return

    today = date.today()
    try:
        cursor = conn.cursor()
        for color in color_list:
            cursor.execute("SELECT Count FROM workpiece_color_table WHERE Date=%s AND Color=%s", (today, color))
            result = cursor.fetchone()
            if result:
                cursor.execute(
                    "UPDATE workpiece_color_table SET Count = Count + 1 WHERE Date=%s AND Color=%s",
                    (today, color)
                )
            else:
                cursor.execute(
                    "INSERT INTO workpiece_color_table (Date, Color, Count) VALUES (%s, %s, 1)",
                    (today, color)
                )
        conn.commit()
        print("Updated workpiece counts for:", color_list)
    except Error as e:
        print("MySQL Update Error:", e)
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

# ===== Plotting Function =====
def plot_workpiece_summary():
    conn = connect()
    if conn is None:
        print("Cannot fetch data - DB not connected.")
        return

    try:
        cursor = conn.cursor()
        cursor.execute("SELECT Date, Color, Count FROM workpiece_color_table ORDER BY Date, Color")
        rows = cursor.fetchall()
        if not rows:
            print("No records found in database.")
            return

        summary = {}
        for d, color, count in rows:
            d = str(d)
            if d not in summary:
                summary[d] = {}
            summary[d][color] = count

        for date_str, color_counts in summary.items():
            colors = list(color_counts.keys())
            counts = list(color_counts.values())
            # map bar colors by name (changed from single 'skyblue')
            bar_colors = [COLOR_HEX.get(c.upper(), '#87CEEB') for c in colors]

            plt.figure()
            plt.bar(colors, counts, color=bar_colors)
            plt.title(f'Workpieces Sorted on {date_str}')
            plt.xlabel('Color')
            plt.ylabel('Count')
            plt.tight_layout()

        plt.show()

    except Error as e:
        print("Plotting error:", e)
    finally:
        cursor.close()
        conn.close()

# ===== Detection =====
def generate_random_colors():
    return [random.randint(1, 5) for _ in range(5)]

def detect_colors(frame):
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    for color_name, (lower, upper) in color_ranges.items():
        mask = cv2.inRange(hsv, np.array(lower), np.array(upper))
        contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
        for cnt in contours:
            if cv2.contourArea(cnt) > 3000:
                x, y, w, h = cv2.boundingRect(cnt)
                return color_name, (x, y, x + w, y + h)
    return None, None

# ===== Indicator Drawing Helper (added) =====
def draw_indicators(graph_elem, target_sequence, current_index):
    """
    Draw 5 circles representing the required sequence.
    A circle lights up (filled with its color) only when the correct color has been accepted.
    Unlit circles are dark; outlines always show the target color.
    """
    if graph_elem is None:
        return
    g = graph_elem
    g.erase()

    width, height = 520, 120
    cx_start = 60
    spacing = 100
    radius = 30
    # Draw title
    g.draw_text("Tray Progress", location=(width/2, height-18), color="#DDDDDD", font="Helvetica 12 bold")

    for i, color_id in enumerate(target_sequence):
        name = workpiece_color_dict[color_id]  # e.g., "GREEN"
        outline = COLOR_HEX.get(name, "#CCCCCC")
        # lit if this index already matched (i < current_index)
        fill = COLOR_HEX.get(name, "#CCCCCC") if i < current_index else DIM_HEX

        cx = cx_start + i * spacing
        cy = 55

        # circle
        g.draw_circle(center_location=(cx, cy), radius=radius, fill_color=fill, line_color=outline, line_width=3)
        # label (1..5) below
        g.draw_text(f"{i+1}:{name}", location=(cx, cy - radius - 14), color=outline, font="Helvetica 9 bold")

# ===== Main GUI Application =====
def run_gui():
    sg.theme("DarkGrey12")

    layout = [
        [sg.Text("Smart Beacon Sorting System", font=("Helvetica", 16))],
        [sg.Image(filename="", key="-IMAGE-")],
        [sg.Text("Target Colors:"), sg.Text("", size=(40, 1), key="-TARGET-")],
        [sg.Text("Status:"), sg.Text("", size=(40, 1), key="-STATUS-")],
        # ---- added in-window indicator (graph) ----
        [sg.Graph(canvas_size=(520, 120), graph_bottom_left=(0, 0), graph_top_right=(520, 120),
                  background_color="#1E1E1E", key="-GRAPH-")],
        [sg.Button("Start Sorting"), sg.Button("Scan Workpiece"), sg.Button("Stop Sorting"), sg.Button("Exit"), sg.Button("Show Workpiece Summary")]
    ]

    window = sg.Window("Smart Beacon Sorting", layout, finalize=True)

    cap = cv2.VideoCapture(1)
    target_sequence = []
    current_index = 0
    sorting = False

    while True:
        event, _ = window.read(timeout=10)

        ret, frame = cap.read()
        if ret:
            imgbytes = cv2.imencode('.png', cv2.resize(frame, (480, 320)))[1].tobytes()
            window["-IMAGE-"].update(data=imgbytes)

        if event == sg.WIN_CLOSED or event == "Exit":
            break

        elif event == "Start Sorting":
            target_sequence = generate_random_colors()
            current_index = 0
            sorting = True
            window["-TARGET-"].update(", ".join(workpiece_color_dict[c] for c in target_sequence))
            window["-STATUS-"].update("Sorting started.")
            speak("Sorting started.")
            # draw/reset indicators
            draw_indicators(window["-GRAPH-"], target_sequence, current_index)

        elif event == "Scan Workpiece" and sorting and current_index < 5:
            color_name, box = detect_colors(frame)
            expected_color = target_sequence[current_index]

            if color_name:
                speak(f"{color_name} detected")
                detected_id = color_name_to_id[color_name]
                if detected_id == expected_color:
                    asyncio.run(beacon_control(current_index + 1, detected_id))
                    window["-STATUS-"].update(f"Segment {current_index + 1}: {color_name} ✓")
                    speak(f"{color_name} accepted")
                    current_index += 1
                    # update indicators: light up the newly correct circle
                    draw_indicators(window["-GRAPH-"], target_sequence, current_index)

                    if current_index == 5:
                        window["-STATUS-"].update("Tray full. Blinking...")
                        speak("Sorting complete.")
                        asyncio.run(blink_beacon())
                        sorted_colors = [workpiece_color_dict[c] for c in target_sequence]
                        update_color_counts(sorted_colors)
                else:
                    expected_name = workpiece_color_dict[expected_color]
                    window["-STATUS-"].update(f"Detected: {color_name}, Expected: {expected_name}")
                    speak(f"Rejected. Expected {expected_name}")
            else:
                window["-STATUS-"].update("No color detected.")
                speak("No color detected.")

        elif event == "Stop Sorting":
            sorting = False
            asyncio.run(beacon_control(0, 1))
            window["-STATUS-"].update("Sorting stopped. Beacon cleared.")
            speak("Sorting stopped.")
            # optional: clear indicators
            draw_indicators(window["-GRAPH-"], target_sequence, 0)

        elif event == "Show Workpiece Summary":
            plot_workpiece_summary()

    cap.release()
    window.close()

if __name__ == "__main__":
    run_gui()
