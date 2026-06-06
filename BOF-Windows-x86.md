## Folow this steps
```html
1.Fuzzing Parameters
2.Controlling EIP
3.Identifying Bad Characters
4.Finding a Return Instruction
5.Jumping to Shellcode
```
## Fuzzing usually this parametrs
```bash
Text Input Fields 	# - Program's "license registration" field. - Various text fields found in the program's preferences.
Opened Files 	    # Any file that the program can open.
Program Arguments 	# Various arguments accepted by the program during runtime.
Remote Resources 	# Any files or resources loaded by the program on run time or on a certain condition.
```

## Tools in Windows
```bash
x32dbg
x64dbg
```

## rdp connection to vuln machine
```bash
xfreerdp /v:<target IP address> /u:<username> /p:<password>
```

## Commands in Linux
```bash
/usr/bin/msf-pattern_offset -q <data>
/usr/bin/msf-pattern_create -l 5000
```

## ERC
```bash
ERC --config SetWorkingDirectory C:\Users\<Username>\Desktop\ # example path(console)
ERC --pattern o <hex value> # example 1hF0
ERC --findNRP # find the offset automatically
ERC --bytearray # generates all ASCII elements from 00 to FF
ERC --compare <ESP_value> C:\Users\<Username>\Desktop\<*>.bin # compares byte-by-byte both our input in ESP
ERC --bytearray -bytes 0x00 # deletes specified bytes and generate all list without deleted
```

## Making paylods in cmd or terminal
```bash
python -c "print('A'*10000)" # creates a string with A 10000 times
python -c "print('A'*10000, file=open('fuzz.wav', 'w'))" # the same but writes into a file
```






