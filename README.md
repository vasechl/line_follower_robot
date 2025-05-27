from machine import Pin, PWM
import time

# Ρύθμιση κινητήρων
left_motor_fwd = PWM(Pin(9))
right_motor_fwd = PWM(Pin(11))
for m in [left_motor_fwd, right_motor_fwd]:
    m.freq(1000)

# Αισθητήρες D1–D5
d1 = Pin(26, Pin.IN)
d2 = Pin(6, Pin.IN)
d3 = Pin(17, Pin.IN)
d4 = Pin(16, Pin.IN)
d5 = Pin(5, Pin.IN)

# Ρυθμίσεις ταχύτητας
BASE_SPEED = 60000
MAX_SPEED = 65000
MIN_SPEED = 0
Kp = 15000

# Μνήμη τελευταίας κατεύθυνσης
# -1 = αριστερά, 1 = δεξιά, 0 = ευθεία
last_direction = 0

# Χρόνος απο την τελευταία φορά που είδε μαύρο
last_black_time = time.ticks_ms()

while True:
    # Διαβάζω τους αισθητήρες
    s1 = d1.value()
    s2 = d2.value()
    s3 = d3.value()
    s4 = d4.value()
    s5 = d5.value()

    # Ελέγχω αν βλέπει μαύρο
    sees_black = s1 or s2 or s3 or s4 or s5

    current_time = time.ticks_ms()

    if sees_black:
        last_black_time = current_time  # ανανεώνω το χρονόμετρο

        # Υπολογισμός σφάλματος
        error = (-2 * s1) + (-1 * s2) + (0 * s3) + (1 * s4) + (2 * s5)

        # κρατάει στημ μνήμη την τελευταία κατεύθηνση  τελευταία κατεύθυνση
        if error < 0:
            last_direction = -1
        elif error > 0:
            last_direction = 1
        else:
            last_direction = 0

        # Υπολογισμός διόρθωσης
        correction = Kp * error

        left_speed = BASE_SPEED + correction
        right_speed = BASE_SPEED - correction

    else:
        # οταν χάνει την γραμμή να ψάχνει προς την τελευταία κατεύθυνση
        if last_direction == -1:
            left_speed = 0
            right_speed = MAX_SPEED
        elif last_direction == 1:
            left_speed = MAX_SPEED
            right_speed = 0
        else:
            left_speed = BASE_SPEED
            right_speed = BASE_SPEED

    # Ελέγχω αν έχει περάσει 1 δευτερόλεπτο χωρίς μαύρο. Αμα έχει περάσει τότε σταματάει
    if time.ticks_diff(current_time, last_black_time) > 1000:
        left_speed = 0
        right_speed = 0

    # Περιορισμός ταχυτήτων
    left_speed = max(MIN_SPEED, min(MAX_SPEED, left_speed))
    right_speed = max(MIN_SPEED, min(MAX_SPEED, right_speed))

    # Εφαρμογή στα μοτέρ
    left_motor_fwd.duty_u16(int(left_speed))
    right_motor_fwd.duty_u16(int(right_speed))

    time.sleep(0.01)

