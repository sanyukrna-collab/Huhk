#!/usr/bin/env python3
"""
รันมอเตอร์ 4 ตัวพร้อมกัน บน Raspberry Pi 3
ใช้ L298N 2 บอร์ด (บอร์ดละ 2 มอเตอร์)
- บอร์ด 1: Motor A, Motor B
- บอร์ด 2: Motor C, Motor D
"""

import RPi.GPIO as GPIO
import time

# ========================= ผังพิน GPIO =========================

# บอร์ด 1
A_ENA, A_IN1, A_IN2 = 18, 23, 24
B_ENB, B_IN3, B_IN4 = 13, 27, 22

# บอร์ด 2
C_ENA, C_IN1, C_IN2 = 12, 5, 6
D_ENB, D_IN3, D_IN4 = 16, 21, 26

PWM_FREQ = 1000
RUN_TIME = 3
SPEED = 70   # 0-100

# ========================= ตั้งค่า GPIO =========================

GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

all_pins = [A_ENA, A_IN1, A_IN2, B_ENB, B_IN3, B_IN4,
            C_ENA, C_IN1, C_IN2, D_ENB, D_IN3, D_IN4]

for pin in all_pins:
    GPIO.setup(pin, GPIO.OUT)

pwm_A = GPIO.PWM(A_ENA, PWM_FREQ)
pwm_B = GPIO.PWM(B_ENB, PWM_FREQ)
pwm_C = GPIO.PWM(C_ENA, PWM_FREQ)
pwm_D = GPIO.PWM(D_ENB, PWM_FREQ)

for p in [pwm_A, pwm_B, pwm_C, pwm_D]:
    p.start(0)

# ========================= ฟังก์ชันแต่ละมอเตอร์ =========================

def motor(pwm_obj, in1, in2, speed, forward=True):
    GPIO.output(in1, GPIO.HIGH if forward else GPIO.LOW)
    GPIO.output(in2, GPIO.LOW if forward else GPIO.HIGH)
    pwm_obj.ChangeDutyCycle(abs(speed))

def all_forward(speed):
    motor(pwm_A, A_IN1, A_IN2, speed, True)
    motor(pwm_B, B_IN3, B_IN4, speed, True)
    motor(pwm_C, C_IN1, C_IN2, speed, True)
    motor(pwm_D, D_IN3, D_IN4, speed, True)

def stop_all():
    for in_pin in [A_IN1, A_IN2, B_IN3, B_IN4, C_IN1, C_IN2, D_IN3, D_IN4]:
        GPIO.output(in_pin, GPIO.LOW)
    for p in [pwm_A, pwm_B, pwm_C, pwm_D]:
        p.ChangeDutyCycle(0)

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
    for p in [pwm_A, pwm_B, pwm_C, pwm_D]:
        p.stop()
    GPIO.cleanup()
    print("เคลียร์ GPIO เรียบร้อย")
