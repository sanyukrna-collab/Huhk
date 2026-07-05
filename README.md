import RPi.GPIO as GPIO
import time

# Motor A
ENA, IN1, IN2 = 18, 23, 24
# Motor B
ENB, IN3, IN4 = 13, 27, 22

GPIO.setmode(GPIO.BCM)
for pin in [ENA, IN1, IN2, ENB, IN3, IN4]:
    GPIO.setup(pin, GPIO.OUT)

pwmA = GPIO.PWM(ENA, 1000)
pwmB = GPIO.PWM(ENB, 1000)
pwmA.start(0)
pwmB.start(0)

def motorA(speed, forward=True):
    GPIO.output(IN1, GPIO.HIGH if forward else GPIO.LOW)
    GPIO.output(IN2, GPIO.LOW if forward else GPIO.HIGH)
    pwmA.ChangeDutyCycle(speed)

def motorB(speed, forward=True):
    GPIO.output(IN3, GPIO.HIGH if forward else GPIO.LOW)
    GPIO.output(IN4, GPIO.LOW if forward else GPIO.HIGH)
    pwmB.ChangeDutyCycle(speed)

def stop_all():
    for pin in [IN1, IN2, IN3, IN4]:
        GPIO.output(pin, GPIO.LOW)
    pwmA.ChangeDutyCycle(0)
    pwmB.ChangeDutyCycle(0)

try:
    motorA(70, forward=True)
    motorB(70, forward=True)
    time.sleep(3)
    stop_all()

except KeyboardInterrupt:
    pass
finally:
    pwmA.stop()
    pwmB.stop()
    GPIO.cleanup()
