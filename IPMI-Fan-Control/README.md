# FIX Fan ramping issue on X10DRL-i
Since putting in Noctua NF-R8 redux-1800 PWM fans, speed thresholds need to be adjusted

To get list of fans etc
```sh
ipmitool -H 'address' -U 'username' -P 'password' sensor 
```

Then to adjust for example.
```sh
ipmitool -H 'address' -U 'username' -P 'password' sensor thresh FAN6 lower 100 150 250
```

The values 100 150 250 seem to work for these fans.

Reboot the BMC when complete.