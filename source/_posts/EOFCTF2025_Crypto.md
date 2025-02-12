---
title: EIP 4337
date: {{ date }}
cover   : "/img/2025_EOF.jpg"
---
EOF CTF 2025
===

跟學弟一起去參加人生首次Attack & Defense，意外拿到第四名。
以此記錄一下crypto菜雞在別人解完兩題開打AD時我還在解半天。

### PRSA
```python
from Crypto.Util.number import getPrime, getRandomRange, bytes_to_long
import os


def keygen(sz):
    p = getPrime(sz // 2)
    q = getPrime(sz // 2)
    n = p * q
    phi = (p - 1) * (q - 1)
    e = 0x10001
    d = pow(e, -1, phi)
    g = 1 + n
    mu = pow(phi, -1, n)
    pk = (n, e, g)
    sk = (n, d, phi, mu)
    return pk, sk


def rsa_encrypt(pk, m):
    n, e, g = pk
    return pow(m, e, n)


def rsa_decrypt(sk, c):
    n, d, phi, mu = sk
    return pow(c, d, n)


def paillier_encrypt(pk, m):
    n, e, g = pk
    r = getRandomRange(1, n)
    n2 = n * n
    # pow(g, m, n2) == (m * n + 1) % (n * n)
    return (pow(g, m, n2) * pow(r, n, n2)) % n2


def paillier_decrypt(sk, c):
    n, d, phi, mu = sk
    cl = pow(c, phi, n * n)
    return ((cl - 1) // n) * mu % n


def rsa_to_paillier(pk, sk, c):
    return paillier_encrypt(pk, rsa_decrypt(sk, c))


def paillier_to_rsa(pk, sk, c):
    return rsa_encrypt(pk, paillier_decrypt(sk, c))


def pad(m, ln):
    pad_ln = ln - len(m)
    pre = pad_ln // 2
    post = pad_ln - pre
    return os.urandom(pre) + m + os.urandom(post)


def main():
    pk, sk = keygen(1024)

    flag = os.environ.get("FLAG", "flag{fake_flag}")
    m = bytes_to_long(pad(flag.encode(), 1024 // 8 - 1))
    c = rsa_encrypt(pk, m)

    print("RSA Encrypted Flag:")
    print(f"{c = }")

    for _ in range(48763):
        print("1. RSA to Paillier")
        print("2. Paillier to RSA")
        print("3. Exit")
        choice = int(input("> "))
        if choice == 1:
            c = int(input("c = "))
            c = rsa_to_paillier(pk, sk, c)
            print(f"{c = }")
        elif choice == 2:
            c = int(input("c = "))
            c = paillier_to_rsa(pk, sk, c)
            print(f"{c = }")
        elif choice == 3:
            break
        else:
            print("Invalid choice")


if __name__ == "__main__":
    main()

```

### 解題思路
看程式可以知道程式會先給 flag 的密文，接著可以透過選擇1,2來轉換RSA 與 Paillier 加密。

這題主要分成兩個部分，第一個是得先還原 N。
第二部分就是要利用兩組RSA密文用Franklin-Reiter解出密文

#### 還原 N
在分析 Paillier 與 RSA 的轉換的功能的時候，可以發現：
$m = 1 -> rsa_to_paillier(1) = g^1*r^n \mod n^2 = p(1)$
而 Paillier 可以有同態運算的特性
例如：
$p(1) * p(1) = p(1+1) = p(2)$
因此我麼可以構造一個
p(1) = rsa_to_paillier(1)
p(2) = p(1) * p(1)
p(4) = p(2) * p(2)
rsa(2) = paillier_to_rsa(p(1)*p(1))
rsa(4) = paillier_to_rsa(p(2)*p(2))

則我們可以得：
2^e - rsa(2) = 2^e - 2^e mod n = kn
4^e - rsa(4) = 2^e - 2^e mod n = mn

因此我們求 gcd(kn, mn) 即可求得 n

```python
from pwn import *
import libnum
import sys
import gmpy2

sys.set_int_max_str_digits(65537) 

def chose(num, c):
    io.sendline(str(num).encode())
    io.sendlineafter(b"c = ", str(c).encode())
    a = io.recvuntil(b'c = ')
    ret = int(io.recvline().strip())
    return ret

def obtain_n(io):
    e = 0x10001
  
    p1 = chose(1,1)
    p2 = p1*p1
    p4 = p2*p2
    n2 = chose(2, p2)
    n4 = chose(2, p4)

    pairings = [(2,n2), (4, n4)]
    pt1, ct1 = pairings[0]
    
    N = ct1 - pow(pt1, e)
    
    # loop through and find common divisors
    for pt,ct in pairings:
        val = gmpy2.mpz(ct - pow(pt, e))
        N = gmpy2.gcd(val, N)

    return N


def exploit(io):
    io.recvuntil(b"N:")
    nn = io.recvline().strip()
    print("N: {}".format(nn))
    a = io.recvuntil(b'RSA Encrypted Flag:')
    a = io.recvuntil(b'c = ')
    a = io.recvline().strip()
    c = int(a)
    
    n = obtain_n(io)

    print("Recovered N: {}".format(n))

    e = 0x10001

  
    p_c = chose(1, c)
    

    p_one = chose(1,1)
    n_two = chose(2, p_one*p_one)
    p_two = chose(1, n_two)

    c_rsa_plus_2 = chose(2, (p_c * p_two % (n**2)) )
    print("--------------------------------------")
    print(c)
    print("--------------------------------------")
    print(c_rsa_plus_2)

if __name__ == "__main__":
    io = process(["python3", "prsa.py"])
    exploit(io)
    io.close()


```

#### Franklin-Reiter
接著我們可以使用Franklin-Reiter 去求 Flag。
首先，我們先把 rsa(flag) 轉換成 paillier(flag)
接著利用同態加密計算 paillier(flag + 2)
(不知道為何測試的時候+1會壞掉，所以這邊就+2了)
再把paillier加密後的結果拿去還原回去rsa(flag+2)
用 Franklin-Reiter即可求解
因為我沒有成功裝好 sage 的 ptyhon lib
所以我就把腳本分開寫了

```python
import libnum

def franklinReiter(e, n,c1,c2,a,b):
    R = Zmod(n)['X']
    (X,) = R._first_ngens(1)
    g1 = (X) ** e - c1
    g2 = (a*X+b) ** e - c2

    def gcd(g1, g2):
        while g2:
            g1, g2 = g2, g1 % g2
        return g1.monic()
    return -gcd(g1, g2)[0]

e = 0x10001
n = 99474132322130844782349278805823705012301718857161537034406676416110599731279127217551303964728460375402814646223999964345232799803243517294463485762250216919370337017668433950293989915204490806013595784783311989867363290954142935445244195865112667180122964157510510548047199181062107318585647423599814630033


c1 = 81910414879189258211058635220199311006585472685703199423738659485502903360726147115590832059786466669583750884502472678300590869233291068953396236844712449441033410372851203185296236297655058591980700612482932543635211562248601071550602076171271667952009145320227066562163562340630532876372606953076861730259

c2 = 76401503220611018809986029513996394520935295068296270000489260881522407088689599696303303577930970621866822739977708269798041317115128048825047138640832309850887340321428244496033983435818647665687829424214659076232905066749128531220407314506281696448278931733999484144880770602382866181257775993780482034804

ret = franklinReiter(e, n, c1, c2,1,2)
print(ret)
```

### 後記
第二題要破 2049 bits 的 Python random number 實在是寫不出來。
博士生涯也沒打過幾場CTF，謝謝學弟們在我僅存不到一年(大概吧?)的學生生涯找我一起玩
希望大家都玩得愉快。

![image](/img/eof.jpg)