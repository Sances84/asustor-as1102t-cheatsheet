# AS1102T NAS Cheatsheet

> ⚠️ **Disclaimer:** This project is made only for fun to control your NAS (AS1102T) via SSH. Use at your own risk. This project respects the ASUSTOR End User License Agreement (EULA). No proprietary binaries have been permanently modified, patched, or illegally re-distributed. We are explored proprietary binaries, then put into commands, can be not work on some models of NAS.
> 
> 💡 **Note:** All `.py` files require Entware with Python installed.

---

## 🛠️ SSH Utilities

### 💡 ledset.sh
Changes NAS LED brightness remotely without altering settings in the Asustor Data Manager.

```sh
#!/bin/sh
if [ -z "$1" ]; then
    echo "Warning: ledset.sh <number of brightness>"
    echo "Example: ledset.sh 100"
    exit 1
fi
confutil -set /usr/etc/emboard.conf Led Brightness "$1"
ledctrl -led_brightness $(confutil -get /usr/etc/emboard.conf Led Brightness)
echo "Brightness set to $1 and saved in emboard.conf"
```

### 🌪️ fancheck.sh
An open-source replica of `fanctrl -getfanspeed`, created by exploring `strace`. *Note: Can be buggy or slightly inaccurate.*

```sh
#!/bin/sh

# Setup serial port - DISABLE ECHO is the most important part!
# We also use 'min 0' and 'time 1' to ensure we don't hang.
stty -F /dev/ttyS1 115200 raw -echo -echoe -echok -onlcr min 0 time 1

query_hw() {
    # 1. Clear any junk currently in the serial buffer
    dd if=/dev/ttyS1 bs=1 count=100 timeout=0.1 > /dev/null 2>&1

    # 2. Send the 3-byte command
    printf "$1" > /dev/ttyS1

    # 3. Wait a tiny bit for the PIC to process
    usleep 50000

    # 4. Read the 6-byte response
    # Packet format: [3A] [CMD] [SUB] [31] [CMD] [VALUE]
    HEX_DATA=$(dd if=/dev/ttyS1 bs=1 count=6 2>/dev/null | hexdump -ve '1/1 "%02x "')

    # 5. Extract the 6th byte (the actual value)
    VAL_HEX=$(echo $HEX_DATA | awk '{print $6}')

    if [ -z "$VAL_HEX" ] || [ "$VAL_HEX" = " " ]; then
        echo 0
    else
        echo $((16#$VAL_HEX))
    fi
}

echo "Reading ASUSTOR AS1102T Sensors..."

# Get PWM
PWM=$(query_hw "\x31\x00\x00")

# Get RPM High Byte
HI=$(query_hw "\x31\x11\x00")

# Get RPM Low Byte
LO=$(query_hw "\x31\x10\x00")

# Calculate RPM
RPM=$(( ($HI << 8) | $LO ))

echo "-------------------------------"
echo "Real RPM : $RPM"
echo "Real PWM : $PWM %"
echo "-------------------------------"
```

---

## 🚨 Emergency Reboot Commands

> 💥 **Risk Warning:** Force rebooting carries a potential risk of breaking or corrupting the BIOS.

### Option A: Running from Home (Local Network)
```sh
ssh -p 22 -q -t (username)@(your local nas ip) "echo '(your password)' | sudo -S nohup /bin/sh -c '/usr/sbin/buzzctrl -shutdown & /usr/sbin/ledctrl -status blink4 & [ -f /usr/builtin/etc/init.d/rcK ] && /bin/sh /usr/builtin/etc/init.d/rcK || [ -f /etc/init.d/rcK ] && /bin/sh /etc/init.d/rcK; echo 1 > /proc/sys/kernel/sysrq; for x in e i s u; do echo \$x > /proc/sysrq-trigger; sleep 1; done; /sbin/reboot -f' > /dev/null 2>&1 &"
```

### Option B: Running Outside Home (via Shellinabox)
Use this variant if you are connected outside your local router environment:
```sh
sudo -S nohup /bin/sh -c '/usr/sbin/buzzctrl -shutdown & /usr/sbin/ledctrl -status blink4 & [ -f /usr/builtin/etc/init.d/rcK ] && /bin/sh /usr/builtin/etc/init.d/rcK || [ -f /etc/init.d/rcK ] && /bin/sh /etc/init.d/rcK; echo 1 > /proc/sys/kernel/sysrq; for x in e i s u; do echo \$x > /proc/sysrq-trigger; sleep 1; done; /sbin/reboot -f' > /dev/null 2>&1 &
```

---

## 🎹 Python Projects

### midibuzz.py
Converts a `.mid` (MIDI) file into a shell script output that plays the melody using the NAS internal buzzer.

```bash
pip install mido
```

```python
import mido
import sys
import time

if len(sys.argv) < 2:
    print("Usage: python3 midibuzz.py <file.mid>")
    sys.exit(1)

mid = mido.MidiFile(sys.argv[1])

print("#!/bin/sh")
print("# Generated Buzz Script")

for msg in mid.play():
    # msg.time is the duration to wait in seconds since the last event
    if msg.time > 0:
        print(f"usleep {int(msg.time * 1000000)}")

    if msg.type == 'note_on' and msg.velocity > 0:
        # Start Beep (Silent background mode)
        print('sh -c "buzzctrl -poweron >/dev/null 2>&1 &"')

    elif msg.type == 'note_off' or (msg.type == 'note_on' and msg.velocity == 0):
        # Stop Beep
        print('buzzctrl -disable poweron >/dev/null 2>&1')
```

* **Test Asset:** You can download a sample track from the [GS Archive Test MIDI](https://gsarchive.net/html/sounds/test.mid) hosted on the [GS Archive Index](https://gsarchive.net/html/midi.html).

### baycount.py
Retrieves the hard drive bay count configuration directly from the system files.

```python
import configparser

config = configparser.ConfigParser()
config.read('/etc/nas.conf')

# Equivalent to the Hal_Get_Platform_Bay_Count function
bay_count = config.getint('Disk', 'Number', fallback=0)
print(f"Bay Count: {bay_count}")
```
```text
Output Example:
Bay Count: 2
```
