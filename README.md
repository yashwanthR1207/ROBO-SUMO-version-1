
<div align="center">

```
██████╗  ██████╗ ██████╗  ██████╗      ███████╗██╗   ██╗███╗   ███╗ ██████╗ 
██╔══██╗██╔═══██╗██╔══██╗██╔═══██╗     ██╔════╝██║   ██║████╗ ████║██╔═══██╗
██████╔╝██║   ██║██████╔╝██║   ██║     ███████╗██║   ██║██╔████╔██║██║   ██║
██╔══██╗██║   ██║██╔══██╗██║   ██║     ╚════██║██║   ██║██║╚██╔╝██║██║   ██║
██║  ██║╚██████╔╝██████╔╝╚██████╔╝     ███████║╚██████╔╝██║ ╚═╝ ██║╚██████╔╝
╚═╝  ╚═╝ ╚═════╝ ╚═════╝  ╚═════╝      ╚══════╝ ╚═════╝ ╚═╝     ╚═╝ ╚═════╝ 
```

#  ROBO-SUMO — Version 2 ⚔️
### *Enter the ring. Dominate the dohyō.*

---

[![Arduino](https://img.shields.io/badge/Arduino-Uno-00979D?style=for-the-badge&logo=arduino&logoColor=white)](https://www.arduino.cc/)
[![Language](https://img.shields.io/badge/Language-C%2B%2B-00599C?style=for-the-badge&logo=cplusplus&logoColor=white)]()
[![RC](https://img.shields.io/badge/RC-FlySky%20FS--i6-E8491D?style=for-the-badge)]()
[![Status](https://img.shields.io/badge/Status-Ring%20Ready-2ea44f?style=for-the-badge)]()
[![Drive](https://img.shields.io/badge/Drive-Differential-purple?style=for-the-badge)]()

> **Two-wheel RC sumo robot · Arduino Uno · FlySky FS-i6 · BTS7960**  
> *Built to push. Built to win. Built for the dohyō.*

</div>

---

## What This Is

A two-wheel differential drive sumo robot controlled over FlySky PWM radio — designed to shove opponents out of the ring with full-speed precision control.

The main problem with most beginner code is `pulseIn()` — it reads one channel, **blocks for 25 ms**, then tries to read the second. By then that pulse is long gone, so steering never responds. In a sumo match, that delay costs you the round.

This code reads **both channels simultaneously** using a tight polling loop.  
**No blocking. No missed pulses. No excuses.**

```
CH2  (throttle) ──┬──► left  motor  BTS7960 #1
CH4  (steering) ──┘──► right motor  BTS7960 #2
```

---

## Hardware

| Qty | Part |
|:---:|------|
| 1 | Arduino Uno |
| 1 | FlySky FS-i6 transmitter + FS-iA6B receiver |
| 2 | BTS7960 43 A H-bridge motor driver |
| 2 | DC gear motors (6 V – 12 V) |
| 1 | 7.4 V 2S or 11.1 V 3S LiPo battery |
| 1 | Buck converter — output set to 5 V (for Arduino logic) |

---

##  Wiring

### Receiver → Arduino

| Receiver | Arduino | What |
|:---:|:---:|---|
| CH2 | `A0` | Throttle — forward / back |
| CH4 | `A1` | Steering — left / right |
| GND | `GND` | Common ground (required) |
| +5V | `5V` | Receiver power |

> CH4 is the VrA knob by default. To use a stick instead:  
> `Menu → Functions → Aux. channels → Channel 4 → assign axis`

---

### Arduino → Left BTS7960

| Arduino | BTS7960 |
|:---:|:---:|
| `D5` | `RPWM` |
| `D6` | `LPWM` |
| `D7` | `R_EN` |
| `D8` | `L_EN` |
| `5V` | `VCC` |
| `GND` | `GND` |

---

### Arduino → Right BTS7960

| Arduino | BTS7960 |
|:---:|:---:|
| `D9` | `RPWM` |
| `D10` | `LPWM` |
| `D11` | `R_EN` |
| `D12` | `L_EN` |
| `5V` | `VCC` |
| `GND` | `GND` |

---

### Power

| From | To |
|---|---|
| Battery `+` | `B+` on both BTS7960 boards |
| Battery `−` | `B−` on both BTS7960 boards |
| Battery `+` | Buck converter input |
| Buck converter output (5 V) | Arduino `5V` |
| Battery `−` | Arduino `GND` — tie all grounds together |
| BTS7960 `M+` / `M−` | Motor terminals (swap to reverse direction) |

>  **Never** power motors from the Arduino 5 V pin. Always use the buck converter for logic and connect battery power directly to the BTS7960 B+ / B−.

---

##  Channel Map

```
CH1   right stick  left/right    — not used
CH2   right stick  up/down       — THROTTLE → A0
CH3   left stick   up/down       — not used
CH4   VrA / axis   left/right    — STEERING → A1
```

---

## 🎛️ Calibration

Open `sumo_bot.ino` and update these six values at the top to match your receiver:

```cpp
const int CH2_MIN    =  997;
const int CH2_CENTER = 1496;
const int CH2_MAX    = 1988;

const int CH4_MIN    =  997;
const int CH4_CENTER = 1496;
const int CH4_MAX    = 1988;
```

To find the real values: enable `Serial.begin(115200)`, print `ch2Raw` and `ch4Raw` in `loop()`, open Serial Monitor, and move each stick to its extremes and centre. Write down what you see.

---

## Drive Mixing

```
left  speed = throttle + steering
right speed = throttle − steering
```

| Stick | Left wheel | Right wheel |
|---|:---:|:---:|
| Forward | fwd | fwd |
| Backward | rev | rev |
| Steer right | fwd | rev |
| Steer left | rev | fwd |
| Forward + right | faster | slower |
| Forward + left | slower | faster |

> If one motor spins backwards, swap its `M+` and `M−` wires on the BTS7960. No code change needed.

---

## 💻 The Code

```cpp
// ============================================================
//  ROBO-SUMO v2  —  sumo_bot.ino
//  Two-wheel differential drive · FlySky FS-i6 · BTS7960
//  Reads CH2 (throttle) + CH4 (steering) simultaneously
//  No pulseIn() blocking — both channels polled in tight loop
// ============================================================

// ---------- Calibration: update these for your receiver -----
const int CH2_MIN    =  997;
const int CH2_CENTER = 1496;
const int CH2_MAX    = 1988;

const int CH4_MIN    =  997;
const int CH4_CENTER = 1496;
const int CH4_MAX    = 1988;

const int DEADBAND   =  40;   // raise to 50-70 if motors twitch at rest
// ------------------------------------------------------------

// Left motor (BTS7960 #1)
const int L_RPWM = 5;
const int L_LPWM = 6;
const int L_REN  = 7;
const int L_LEN  = 8;

// Right motor (BTS7960 #2)
const int R_RPWM = 9;
const int R_LPWM = 10;
const int R_REN  = 11;
const int R_LEN  = 12;

// Receiver input pins
const int CH2_PIN = A0;
const int CH4_PIN = A1;

// ── Read both PWM channels simultaneously (non-blocking) ────
void readChannels(int &ch2, int &ch4) {
  bool ch2Done = false, ch4Done = false;
  unsigned long ch2Start = 0, ch4Start = 0;
  unsigned long ch2Val   = 1500, ch4Val = 1500;

  unsigned long timeout = micros() + 25000UL;

  bool prevCH2 = digitalRead(CH2_PIN);
  bool prevCH4 = digitalRead(CH4_PIN);

  while ((!ch2Done || !ch4Done) && (micros() < timeout)) {
    bool curCH2 = digitalRead(CH2_PIN);
    bool curCH4 = digitalRead(CH4_PIN);

    if (!ch2Done) {
      if (!prevCH2 && curCH2)        ch2Start = micros();       // rising edge
      else if (prevCH2 && !curCH2) { ch2Val = micros() - ch2Start; ch2Done = true; }
    }
    if (!ch4Done) {
      if (!prevCH4 && curCH4)        ch4Start = micros();
      else if (prevCH4 && !curCH4) { ch4Val = micros() - ch4Start; ch4Done = true; }
    }
    prevCH2 = curCH2;
    prevCH4 = curCH4;
  }
  ch2 = (int)ch2Val;
  ch4 = (int)ch4Val;
}

// ── Drive one BTS7960 channel ────────────────────────────────
void driveMotor(int rpwm, int lpwm, int speed) {
  // speed: -255 (full reverse) … 0 … +255 (full forward)
  speed = constrain(speed, -255, 255);
  if (speed > 0) {
    analogWrite(rpwm, speed);
    analogWrite(lpwm, 0);
  } else if (speed < 0) {
    analogWrite(rpwm, 0);
    analogWrite(lpwm, -speed);
  } else {
    analogWrite(rpwm, 0);
    analogWrite(lpwm, 0);
  }
}

// ── Map raw PWM to -255…+255, with deadband ──────────────────
int pwmToSpeed(int raw, int minV, int center, int maxV) {
  raw = constrain(raw, minV, maxV);
  int val;
  if (raw > center + DEADBAND)
    val = map(raw, center + DEADBAND, maxV, 0, 255);
  else if (raw < center - DEADBAND)
    val = map(raw, minV, center - DEADBAND, -255, 0);
  else
    val = 0;
  return val;
}

void setup() {
  // Enable BTS7960 bridges
  pinMode(L_REN, OUTPUT); digitalWrite(L_REN, HIGH);
  pinMode(L_LEN, OUTPUT); digitalWrite(L_LEN, HIGH);
  pinMode(R_REN, OUTPUT); digitalWrite(R_REN, HIGH);
  pinMode(R_LEN, OUTPUT); digitalWrite(R_LEN, HIGH);

  // PWM outputs
  pinMode(L_RPWM, OUTPUT); pinMode(L_LPWM, OUTPUT);
  pinMode(R_RPWM, OUTPUT); pinMode(R_LPWM, OUTPUT);

  // Receiver inputs
  pinMode(CH2_PIN, INPUT);
  pinMode(CH4_PIN, INPUT);

  // Uncomment to calibrate:
  // Serial.begin(115200);
}

void loop() {
  int ch2Raw, ch4Raw;
  readChannels(ch2Raw, ch4Raw);

  // Uncomment to calibrate:
  // Serial.print("CH2: "); Serial.print(ch2Raw);
  // Serial.print("  CH4: "); Serial.println(ch4Raw);

  int throttle = pwmToSpeed(ch2Raw, CH2_MIN, CH2_CENTER, CH2_MAX);
  int steering  = pwmToSpeed(ch4Raw, CH4_MIN, CH4_CENTER, CH4_MAX);

  // Differential mixing
  int leftSpeed  = constrain(throttle + steering, -255, 255);
  int rightSpeed = constrain(throttle - steering, -255, 255);

  driveMotor(L_RPWM, L_LPWM, leftSpeed);
  driveMotor(R_RPWM, R_LPWM, rightSpeed);
}
```

---

##  Upload

```bash
# 1. Clone
git clone https://github.com/YOUR_USERNAME/robo-sumo-v2.git

# 2. Open in Arduino IDE
File → Open → sumo_bot/sumo_bot.ino

# 3. Board + port
Tools → Board → Arduino Uno
Tools → Port  → your port

# 4. Upload
Ctrl + U
```

>  Power-on order: **transmitter first**, then robot. Wait for receiver LED to go solid before moving sticks.

---

##  Troubleshooting

| Problem | Fix |
|---|---|
| No movement at all | Check CH2 → A0, CH4 → A1, receiver GND |
| Steering does nothing | Make sure you are using `readChannels()`, not `pulseIn()` |
| Motors twitch at rest | Raise `DEADBAND` to 50–70 |
| One motor reversed | Swap `M+` / `M−` on that BTS7960 |
| Creeps with sticks centred | Re-calibrate `CH2_CENTER` / `CH4_CENTER` |
| Signal drops under load | Add 470 µF cap across Arduino 5 V and GND |
| BTS7960 overheats | Add heatsink; check motor stall current |
| Arduino resets on motor start | Use separate power paths; shared buck converter for logic only |

---

<div align="center">

```
  ⚔️  Step into the dohyō. Push hard. Never yield.  ⚔️
```

**Built for the ring.**  
If this helped you dominate, leave a ⭐ star.



</div>
