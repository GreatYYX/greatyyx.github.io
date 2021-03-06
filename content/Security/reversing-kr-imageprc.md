Title: reversing.kr ImagePrc Walkthrough
Date: 2014-09-23 18:05:00
Tags: reversing.kr, wargame, hacking, reversing
Slug: reversing-kr-imageprc
Summary: 本文为逆向工程练习网站reversing.kr中ImagePrc解题攻略。
Summary_en: The write-up for ImagePrc in reversing.kr.

# 试玩 #

看起来很有意思的一道题目。拿到后打开一个窗口，可以用鼠标画一些东西，之后点Check，弹出“Wrong”。直觉认为应该是要画出一个图像，正确的话会得到flag。可惜这个图能画正确的概率真的很低。

# 逻辑分析 #

OD打开之后下MessageBoxA的API断点或者WM_LBUTTONUP的消息断点，点击Check后断下并跟踪到核心逻辑代码的位置。

```x86asm
004012D6   > \83BC24 940000>cmp dword ptr ss:[esp+0x94],0x64         ;  Case 111 (WM_COMMAND) of switch 0040113A
004012DE   .  0F85 01020000 jnz ImagePrc.004014E5
...
004013CD   >  8B8424 900000>mov eax,dword ptr ss:[esp+0x90]
004013D4   .  6A 30         push 0x30                                ; /Style = MB_OK|MB_ICONEXCLAMATION|MB_APPLMODAL
004013D6   .  68 34604000   push ImagePrc.00406034                   ; |ImagePrc
004013DB   .  68 40604000   push ImagePrc.00406040                   ; |Wrong
004013E0   .  50            push eax                                 ; |hOwner
004013E1   .  FF15 F8504000 call dword ptr ds:[<&USER32.MessageBoxA>>; \MessageBoxA
004013E7   .  56            push esi
004013E8   .  E8 13010000   call ImagePrc.00401500
004013ED   .  83C4 04       add esp,0x4
004013F0   .  33C0          xor eax,eax
004013F2   .  5B            pop ebx
004013F3   .  5F            pop edi
004013F4   .  5E            pop esi
004013F5   .  81C4 80000000 add esp,0x80
004013FB   .  C2 1000       retn 0x10
```

在按钮消息路由到弹出消息框之间就是我们要找的核心逻辑。下面分解一下逻辑：

```x86asm
...
004012E4   .  A1 F4844000   mov eax,dword ptr ds:[0x4084F4]
004012E9   .  8D5424 08     lea edx,dword ptr ss:[esp+0x8]
004012ED   .  53            push ebx
004012EE   .  52            push edx                                 ; /Buffer = 0018FB04
004012EF   .  6A 18         push 0x18                                ; |BufSize = 18 (24.)
004012F1   .  50            push eax                                 ; |hObject => B9051E9B
004012F2   .  FF15 1C504000 call dword ptr ds:[<&GDI32.GetObjectA>]  ; \GetObjectA
004012F8   .  B9 0A000000   mov ecx,0xA
004012FD   .  33C0          xor eax,eax
004012FF   .  8D7C24 24     lea edi,dword ptr ss:[esp+0x24]
00401303   .  8D5424 24     lea edx,dword ptr ss:[esp+0x24]
00401307   .  F3:AB         rep stos dword ptr es:[edi]
00401309   .  8B4424 14     mov eax,dword ptr ss:[esp+0x14]
0040130D   .  8B4C24 10     mov ecx,dword ptr ss:[esp+0x10]
00401311   .  8B3D 20504000 mov edi,dword ptr ds:[<&GDI32.GetDIBits>>;  GDI32.GetDIBits
00401317   .  6A 00         push 0x0                                 ; /Usage = DIB_RGB_COLORS
00401319   .  52            push edx                                 ; |pBitmapinfo
0040131A   .  6A 00         push 0x0                                 ; |Buffer = NULL
0040131C   .  894424 38     mov dword ptr ss:[esp+0x38],eax          ; |
00401320   .  50            push eax                                 ; |nLines
00401321   .  A1 F4844000   mov eax,dword ptr ds:[0x4084F4]          ; |
00401326   .  894C24 38     mov dword ptr ss:[esp+0x38],ecx          ; |
0040132A   .  8B0D DC844000 mov ecx,dword ptr ds:[0x4084DC]          ; |
00401330   .  6A 00         push 0x0                                 ; |StartLine = 0
00401332   .  50            push eax                                 ; |hBitmap => B9051E9B
00401333   .  51            push ecx                                 ; |hDC => 4C01191B
00401334   .  C74424 40 280>mov dword ptr ss:[esp+0x40],0x28         ; |
0040133C   .  66:C74424 4C >mov word ptr ss:[esp+0x4C],0x1           ; |
00401343   .  66:C74424 4E >mov word ptr ss:[esp+0x4E],0x18          ; |
0040134A   .  C74424 50 000>mov dword ptr ss:[esp+0x50],0x0          ; |
00401352   .  FFD7          call edi                                 ; \GetDIBits
...
```

程序首先获取绘制的对象，并提取绘制对象的像素内容。在`GetDIBits()`的时候栈情况如下：

```x86asm
...
0018FADC   4C01191B  |hDC = 4C01191B
0018FAE0   B9051E9B  |hBitmap = B9051E9B
0018FAE4   00000000  |StartLine = 0
0018FAE8   00000096  |nLines = 96 (150.)
0018FAEC   00000000  |Buffer = NULL
0018FAF0   0018FB1C  |pBitmapinfo = 0018FB1C
0018FAF4   00000000  \Usage = DIB_RGB_COLORS
...
```

可以看到图像为RGB图像，第一个扫描线是150（其实就是150的高度）。查看地址0018FB1C处数据的LPBITMAPINFO结构体指针的内容：

```ascii
...
0018FB1C  28 00 00 00 C8 00 00 00 96 00 00 00 01 00 18 00  (.È..
...
```

根据BITMAPINFO结构体格式，发现bitmap的长度是0xC8，高度0x96，即200x150大小。

接着往下看：

```x86asm
...
00401381   .  6A 18         push 0x18                                ; /ResourceType = 18
00401383   .  6A 65         push 0x65                                ; |ResourceName = 65
00401385   .  6A 00         push 0x0                                 ; |hModule = NULL
00401387   .  FF15 7C504000 call dword ptr ds:[<&KERNEL32.FindResour>; \FindResourceA
0040138D   .  50            push eax                                 ; /hResource
0040138E   .  6A 00         push 0x0                                 ; |hModule = NULL
00401390   .  FF15 80504000 call dword ptr ds:[<&KERNEL32.LoadResour>; \LoadResource
00401396   .  50            push eax                                 ; /hResource
00401397   .  FF15 88504000 call dword ptr ds:[<&KERNEL32.LockResour>; \LockResource
...
```

之后程序从自身获取资源编号为0x65、类型是0x18的资源并加载。这里提一句，`FindResource()`返回的eax指向的数据区构造如下：

```ascii
0047E048  60 E0 07 00 90 5F 01 00 00 00 00 00 00 00 00 00  徐....
0047E058  00 00 00 00 00 00 00 00 FF FF FF FF FF FF FF FF  ........
0047E068  FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF  ........
...
```

可以看到第0-4个字节0x0007E060是资源真正的起始地址的偏移，5-8个字节0x00105F90是资源大小，0047E060才是资源数据内容的起始。这就是PE文件对于资源的布局（当然不知道也能解题）。

`LoadResource()`并Lock之后，进入如下逻辑：

```x86asm
...
0040139D   .  33FF          xor edi,edi
0040139F   .  8BCE          mov ecx,esi
004013A1   .  2BC6          sub eax,esi
004013A3   >  8A11          mov dl,byte ptr ds:[ecx]
004013A5   .  8A1C08        mov bl,byte ptr ds:[eax+ecx]
004013A8   .  3AD3          cmp dl,bl
004013AA      75 21         jnz XImagePrc.004013CD
004013AC   .  47            inc edi
004013AD   .  41            inc ecx
004013AE      81FF 905F0100 cmp edi,0x15F90
004013B4   .^ 7C ED         jl XImagePrc.004013A3
004013B6   .  56            push esi
004013B7   .  E8 44010000   call ImagePrc.00401500
004013BC   .  83C4 04       add esp,0x4
004013BF   .  33C0          xor eax,eax
004013C1   .  5B            pop ebx
004013C2   .  5F            pop edi
004013C3   .  5E            pop esi
004013C4   .  81C4 80000000 add esp,0x80
004013CA > .  C2 1000       retn 0x10
...
```

这段逻辑很简单，一个循环，一共循环0x15F90次（刚好是上面分析的资源的大小）。之后把绘制的图像数据和资源图像数据分别放入dl和bl进行每个byte的逐一比较。一开始我以为只要比较结果正确就能拿到flag，所以直接修改比较次数为0，可惜程序正常执行，却没有flag的踪迹。所以思路转为从资源入手。

# 资源操作 #

打开exeScope之类的资源提取工具，提取编号0x65的资源，发现大小是0X15F80，比我们需要的数据小0x10个字节。于是重新跟程序到`LoadResource()`执行后，eax为47e060，提取47e060-493ff0一共0X15F90字节所有数据到文件。发现其实最后0x10个字节直接被填充为全0。

二进制文件提取出来后，需要让它作为图像显示出来。文件大小为0X15F90，十进制是90000，相当于200x150x3，也就是200x150的RGB三通道图像。我这边直接利用Python的[Pillow图像处理库](https://github.com/python-pillow/Pillow/)做还原（Python3的PIL为Pillow）：

```python
from PIL import Image

width = 200
height = 150

fp = open('rgb.mem', 'rb')
data = fp.read()
im = Image.frombytes('RGB', (width, height), data)
im = im.transpose(Image.FLIP_TOP_BOTTOM)
im.show()
im.save('result.bmp')
```

提取后发现图片是上下颠倒的，不知道是由于作者故意做成颠倒的还是WinAPI中GetDIBits是倒着读的。所以使用`transpose()`做翻转。之后可以看到作者绘制的内容为GOT三个字符，即为flag。