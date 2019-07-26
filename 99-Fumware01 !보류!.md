---


---

<h1 id="iptime-a5004ns-공유기-펌웨어-11.004v-분석">IPTIME A5004NS 공유기 펌웨어 11.004v 분석</h1>
<h2 id="추출">2019-07-15 추출</h2>
<p><code>binwalk</code>결과</p>
<pre><code>147460        0x24004         LZMA compressed data, properties: 0x5D, dictionary size: 65536 bytes, uncompressed size: 269436 bytes
262144        0x40000         TRX firmware header, little endian, image size: 25534464 bytes, CRC32: 0x459BF462, flags: 0x0, version: 1, header size: 28 bytes, loader offset: 0x1C, linux kernel offset: 0x1DFD5C, rootfs offset: 0x0
262172        0x4001C         LZMA compressed data, properties: 0x5D, dictionary size: 65536 bytes, uncompressed size: 4796672 bytes
2227548       0x21FD5C        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 23567805 bytes, 1891 inodes, blocksize: 131072 bytes, created: 2019-04-24 01:12:05
</code></pre>
<p>2227548부터 Squash 파일 시스템이 감지 되었다. <code>dd</code>로 추출한 다음 <code>sasquatch</code>로 압축해제 한다.</p>
<pre><code>dd if=./a5004ns_kr_11_004.bin of=a5004ns bs=1 skip=2227548
sasquatch a5004ns
</code></pre>
<h2 id="펌웨어-흐름-파악">2019-07-16 펌웨어 흐름 파악</h2>
<p><code>find ./ -name "init*"</code> 와 <code>find ./ -name "rc*"</code>로 처음에 보면 도움이 될 듯한 파일을 찾아본다.</p>
<pre><code>./default/etc/init.d
./sbin/init
./sbin/inittime
./default/etc/init.d/rcS
./default/etc/strongswan.d/charon/rc2.conf
./sbin/rc
</code></pre>
<p>init.d</p>
<pre><code>#!/bin/sh
/sbin/inittime
</code></pre>

