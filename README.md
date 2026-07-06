#!/usr/bin/env python3
"""
โค้ดควบคุมหุ่นยนต์ 4 มอเตอร์ บน Raspberry Pi 3
- Motor A, B  -> ขับผ่าน L298N   (1 บอร์ด คุมได้ 2 มอเตอร์)
- Motor C, D  -> ขับผ่าน BTS7960 (คนละบอร์ด ตัวละ 1 มอเตอร์)

ควบคุมด้วยคีย์บอร์ด:
    w = เดินหน้า        s = ถอยหลัง
    a = เลี้ยวซ้าย       d = เลี้ยวขวา
    x = หยุด
    + / - = เพิ่ม/ลดความเร็ว
    q = ออกจากโปรแกรม

หมายเหตุ: ปรับผังพิน GPIO ด้านล่างให้ตรงกับที่ต่อจริงก่อนใช้งาน
"""

import RPi.GPIO as GPIO
import time
import sys
import tty
import termios

# ========================= ผังพิน GPIO =========================

# --- L298N: Motor A (ล้อหน้าซ้าย) ---
A_ENA, A_IN1, A_IN2 = 18, 23, 24

# --- L298N: Motor B (ล้อหน้าขวา) ---
B_ENB, B_IN3, B_IN4 = 13, 27, 22

# --- BTS7960 #1: Motor C (ล้อหลังซ้าย) ---
C_RPWM, C_LPWM, C_R_EN, C_L_EN = 12, 19, 5, 6

# --- BTS7960 #2: Motor D (ล้อหลังขวา) ---
D_RPWM, D_LPWM, D_R_EN, D_L_EN = 16, 20, 21, 26

PWM_FREQ = 1000  # Hz

# ========================= ตั้งค่า GPIO =========================

GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

l298n_pins = [A_ENA, A_IN1, A_IN2, B_ENB, B_IN3, B_IN4]
bts_pins = [C_RPWM, C_LPWM, C_R_EN, C_L_EN, D_RPWM, D_LPWM, D_R_EN, D_L_EN]

for pin in l298n_pins + bts_pins:
    GPIO.setup(pin, GPIO.OUT)

# เปิด enable ของ BTS7960 ค้างไว้ตลอด
for en in [C_R_EN, C_L_EN, D_R_EN, D_L_EN]:
    GPIO.output(en, GPIO.HIGH)

# PWM ของ L298N (ควบคุมความเร็วผ่านขา EN)
pwm_A = GPIO.PWM(A_ENA, PWM_FREQ)
pwm_B = GPIO.PWM(B_ENB, PWM_FREQ)
pwm_A.start(0)
pwm_B.start(0)

# PWM ของ BTS7960 (ควบคุมทิศ+ความเร็วผ่าน RPWM/LPWM โดยตรง)
pwm_C_R = GPIO.PWM(C_RPWM, PWM_FREQ)
pwm_C_L = GPIO.PWM(C_LPWM, PWM_FREQ)
pwm_D_R = GPIO.PWM(D_RPWM, PWM_FREQ)
pwm_D_L = GPIO.PWM(D_LPWM, PWM_FREQ)
for p in [pwm_C_R, pwm_C_L, pwm_D_R, pwm_D_L]:
    p.start(0)

# ========================= ฟังก์ชันควบคุมมอเตอร์แต่ละตัว =========================

def motor_A(speed, forward=True):
    GPIO.output(A_IN1, GPIO.HIGH if forward else GPIO.LOW)
    GPIO.output(A_IN2, GPIO.LOW if forward else GPIO.HIGH)
    pwm_A.ChangeDutyCycle(abs(speed))

def motor_B(speed, forward=True):
    GPIO.output(B_IN3, GPIO.HIGH if forward else GPIO.LOW)
    GPIO.output(B_IN4, GPIO.LOW if forward else GPIO.HIGH)
    pwm_B.ChangeDutyCycle(abs(speed))

def motor_C(speed, forward=True):
    if forward:
        pwm_C_R.ChangeDutyCycle(abs(speed))
        pwm_C_L.ChangeDutyCycle(0)
    else:
        pwm_C_R.ChangeDutyCycle(0)
        pwm_C_L.ChangeDutyCycle(abs(speed))

def motor_D(speed, forward=True):
    if forward:
        pwm_D_R.ChangeDutyCycle(abs(speed))
        pwm_D_L.ChangeDutyCycle(0)
    else:
        pwm_D_R.ChangeDutyCycle(0)
        pwm_D_L.ChangeDutyCycle(abs(speed))

def stop_all():
    GPIO.output(A_IN1, GPIO.LOW)
    GPIO.output(A_IN2, GPIO.LOW)
    GPIO.output(B_IN3, GPIO.LOW)
    GPIO.output(B_IN4, GPIO.LOW)
    pwm_A.ChangeDutyCycle(0)
    pwm_B.ChangeDutyCycle(0)
    pwm_C_R.ChangeDutyCycle(0)
    pwm_C_L.ChangeDutyCycle(0)
    pwm_D_R.ChangeDutyCycle(0)
    pwm_D_L.ChangeDutyCycle(0)

# ========================= ฟังก์ชันควบคุมทิศทางหุ่นยนต์ =========================
# สมมติ A,C = ฝั่งซ้าย (หน้า/หลัง)   B,D = ฝั่งขวา (หน้า/หลัง)

def go_forward(speed):
    motor_A(speed, True)
    motor_C(speed, True)
    motor_B(speed, True)
    motor_D(speed, True)

def go_backward(speed):
    motor_A(speed, False)
    motor_C(speed, False)
    motor_B(speed, False)
    motor_D(speed, False)

def turn_left(speed):
    # ฝั่งซ้ายถอย ฝั่งขวาเดินหน้า -> หมุนซ้ายอยู่กับที่
    motor_A(speed, False)
    motor_C(speed, False)
    motor_B(speed, True)
    motor_D(speed, True)

def turn_right(speed):
    # ฝั่งซ้ายเดินหน้า ฝั่งขวาถอย -> หมุนขวาอยู่กับที่
    motor_A(speed, True)
    motor_C(speed, True)
    motor_B(speed, False)
    motor_D(speed, False)

# ========================= อ่านคีย์บอร์อดแบบ real-time =========================

def get_key():
    fd = sys.stdin.fileno()
    old_settings = termios.tcgetattr(fd)
    try:
        tty.setraw(fd)
        key = sys.stdin.read(1)
    finally:
        termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)
    return key

# ========================= โปรแกรมหลัก =========================

def main():
    speed = 70  # ความเร็วเริ่มต้น (0-100)

    print("=== ควบคุมหุ่นยนต์ ===")
    print("w=หน้า  s=ถอยหลัง  a=เลี้ยวซ้าย  d=เลี้ยวขวา")
    print("x=หยุด  +=เพิ่มความเร็ว  -=ลดความเร็ว  q=ออก")
    print(f"ความเร็วเริ่มต้น: {speed}%\n")

    try:
        while True:
            key = get_key().lower()

            if key == "w":
                go_forward(speed)
                print(f"เดินหน้า (speed={speed})")
            elif key == "s":
                go_backward(speed)
                print(f"ถอยหลัง (speed={speed})")
            elif key == "a":
                turn_left(speed)
                print(f"เลี้ยวซ้าย (speed={speed})")
            elif key == "d":
                turn_right(speed)
                print(f"เลี้ยวขวา (speed={speed})")
            elif key == "x":
                stop_all()
                print("หยุด")
            elif key == "+":
                speed = min(100, speed + 10)
                print(f"ความเร็ว = {speed}")
            elif key == "-":
                speed = max(0, speed - 10)
                print(f"ความเร็ว = {speed}")
            elif key == "q":
                print("ออกจากโปรแกรม...")
                break

    except KeyboardInterrupt:
        pass
    finally:
        stop_all()
        pwm_A.stop()
        pwm_B.stop()
        pwm_C_R.stop()
        pwm_C_L.stop()
        pwm_D_R.stop()
        pwm_D_L.stop()
        GPIO.cleanup()
        print("เคลียร์ GPIO เรียบร้อย")


if __name__ == "__main__":
    main()
