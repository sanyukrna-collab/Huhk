#!/usr/bin/env python3
"""
รันมอเตอร์ 4 ตัวพร้อมกัน บน Raspberry Pi 3
- Motor A, B -> ขับผ่าน L298N
- Motor C, D -> ขับผ่าน BTS7960 (คนละบอร์ด)

รันแล้วมอเตอร์ทั้ง 4 ตัวจะหมุนไปข้างหน้าพร้อมกันตามเวลาที่กำหนด แล้วหยุดเอง
"""

import RPi.GPIO as GPIO
import time

# ========================= ผังพิน GPIO =========================

# L298N: Motor A
A_ENA, A_IN1, A_IN2 = 18, 23, 24
# L298N: Motor B
B_ENB, B_IN3, B_IN4 = 13, 27, 22
# BTS7960 #1: Motor C
C_RPWM, C_LPWM, C_R_EN, C_L_EN = 12, 19, 5, 6
# BTS7960 #2: Motor D
D_RPWM, D_LPWM, D_R_EN, D_L_EN = 16, 20, 21, 26

PWM_FREQ = 1000
RUN_TIME = 3        # วิ่งกี่วินาที
SPEED = 70           # ความเร็ว 0-100

# ========================= ตั้งค่า GPIO =========================

GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

all_pins = [A_ENA, A_IN1, A_IN2, B_ENB, B_IN3, B_IN4,
            C_RPWM, C_LPWM, C_R_EN, C_L_EN,
            D_RPWM, D_LPWM, D_R_EN, D_L_EN]

for pin in all_pins:
    GPIO.setup(pin, GPIO.OUT)

# เปิด enable ของ BTS7960 ค้างไว้ตลอด
for en in [C_R_EN, C_L_EN, D_R_EN, D_L_EN]:
    GPIO.output(en, GPIO.HIGH)

pwm_A = GPIO.PWM(A_ENA, PWM_FREQ)
pwm_B = GPIO.PWM(B_ENB, PWM_FREQ)
pwm_C_R = GPIO.PWM(C_RPWM, PWM_FREQ)
pwm_C_L = GPIO.PWM(C_LPWM, PWM_FREQ)
pwm_D_R = GPIO.PWM(D_RPWM, PWM_FREQ)
pwm_D_L = GPIO.PWM(D_LPWM, PWM_FREQ)

for p in [pwm_A, pwm_B, pwm_C_R, pwm_C_L, pwm_D_R, pwm_D_L]:
    p.start(0)

# ========================= ฟังก์ชัน =========================

def all_forward(speed):
    # Motor A
    GPIO.output(A_IN1, GPIO.HIGH)
    GPIO.output(A_IN2, GPIO.LOW)
    pwm_A.ChangeDutyCycle(speed)
    # Motor B
    GPIO.output(B_IN3, GPIO.HIGH)
    GPIO.output(B_IN4, GPIO.LOW)
    pwm_B.ChangeDutyCycle(speed)
    # Motor C
    pwm_C_R.ChangeDutyCycle(speed)
    pwm_C_L.ChangeDutyCycle(0)
    # Motor D
    pwm_D_R.ChangeDutyCycle(speed)
    pwm_D_L.ChangeDutyCycle(0)

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

# ========================= โปรแกรมหลัก =========================

try:
    print(f"มอเตอร์ทั้ง 4 ตัวหมุนไปข้างหน้าพร้อมกัน (speed={SPEED}%) เป็นเวลา {RUN_TIME} วินาที")
    all_forward(SPEED)
    time.sleep(RUN_TIME)

    print("หยุดมอเตอร์ทั้งหมด")
    stop_all()

except KeyboardInterrupt:
    print("\nถูกยกเลิกกลางทาง")

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

