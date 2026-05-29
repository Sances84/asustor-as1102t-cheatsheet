# AS1102T NAS cheatsheet
__Disclaimer__. This project is made only for fun to control your NAS (AS1102T) on SSH.
Note! For all .py files, requires Entware with installed Python.
# for SSH:
# ledset.sh utility, changing brightness in NAS remotely without changing in Asustor Data Manager
```
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
# fancheck.sh (can be buggy) Open source replica to the `fanctrl -getfanspeed` made through exploring in strace with an command
```
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
#Emergency Reboot command, can be runned in the home with NAS (use at your risk, chance of the bios breaking i think is possible on force reboot):
`ssh -p 22 -q -t (username)@(your local nas ip) "echo '(your password)' | sudo -S nohup /bin/sh -c '/usr/sbin/buzzctrl -shutdown & /usr/sbin/ledctrl -status blink4 & [ -f /usr/builtin/etc/init.d/rcK ] && /bin/sh /usr/builtin/etc/init.d/rcK || [ -f /etc/init.d/rcK ] && /bin/sh /etc/init.d/rcK; echo 1 > /proc/sys/kernel/sysrq; for x in e i s u; do echo \$x > /proc/sysrq-trigger; sleep 1; done; /sbin/reboot -f' > /dev/null 2>&1 &""` NOTE: If you are not home or outside the house where the internet router connection ends, and you use an shell in a box if you have it in NAS, use this:`sudo -S nohup /bin/sh -c '/usr/sbin/buzzctrl -shutdown & /usr/sbin/ledctrl -status blink4 & [ -f /usr/builtin/etc/init.d/rcK ] && /bin/sh /usr/builtin/etc/init.d/rcK || [ -f /etc/init.d/rcK ] && /bin/sh /etc/init.d/rcK; echo 1 > /proc/sys/kernel/sysrq; for x in e i s u; do echo \$x > /proc/sysrq-trigger; sleep 1; done; /sbin/reboot -f' > /dev/null 2>&1 &""`

# Funny project-midibuzz.py
```
import mido
import sys
import time

# Path to your MIDI file
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
(requires to run `pip install mido`)
Converts your .mid (MIDI) file into an output, that you should copy and paste to make all sh file.
(try to test it into the https://gsarchive.net/html/sounds/test.mid from https://gsarchive.net/html/midi.html (it's example what you are can convert))
# baycount.py
```
import configparser

config = configparser.ConfigParser()
config.read('/etc/nas.conf')

# Эквивалент функции Hal_Get_Platform_Bay_Count
bay_count = config.getint('Disk', 'Number', fallback=0)
print(f"Bay Count: {bay_count}")
```
Run it and you are can see bay count.
