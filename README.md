# Write-up NDH2K16

## RSA

### Given files and hint
	
We were only given a single zip archive containing the following files:
		
* rsa_pub
* aes_key_cipher
* ciphermessage
	
We know from the topic that:
	
* RSA cipher was used to encrypt aes_key_cipher
* aes_key_cipher was used to encrypt ciphermessage
* aes was used without salt
		
### Solution
	
First we want to gather information about the given RSA public key, since we need to decode the AES key.

```
openssl rsa -pubin -in rsa_pub -text
```

```
Public-Key: (2072 bit)
Modulus:
    00:8d:a5:69:19:b5:26:d4:52:25:ac:ed:4b:e6:45:
    22:ce:f0:4a:63:91:0b:9f:6f:fe:a6:b1:12:55:41:
    01:3b:e4:5d:48:b6:fb:26:71:b7:54:0e:6a:4e:0b:
    55:e3:a9:e4:c4:5a:8d:5f:54:a0:69:9c:65:32:d4:
    a1:28:7f:ac:b0:08:b1:c5:6e:35:d6:01:dc:2a:9e:
    2e:66:51:89:ea:a3:5d:22:d7:be:a2:52:c1:ec:f2:
    70:31:ab:65:7d:5b:35:e8:2c:de:70:f8:25:9d:2e:
    14:e9:86:f3:62:e3:e8:6e:7b:d8:e4:81:2a:52:f2:
    e8:cc:2f:69:b8:b0:c9:59:77:8f:db:24:0a:8e:17:
    cb:95:72:45:70:12:d8:3b:6c:72:72:90:e5:0b:8e:
    7d:a2:8f:eb:df:ab:f5:23:da:03:b9:4b:94:32:2a:
    f4:21:f9:ca:02:ad:e6:04:da:ab:92:cd:9c:28:24:
    44:36:f1:15:fd:be:d7:6d:2c:85:00:7a:ca:7f:ce:
    49:89:3a:f8:0e:79:55:63:2e:c7:b8:9f:56:fa:b0:
    18:76:e0:fe:88:29:9a:37:34:0d:43:9c:b2:e0:1b:
    3c:07:e6:0c:88:47:42:1b:fe:04:9d:59:95:40:6b:
    26:f4:a0:8b:20:d1:49:b3:c6:1a:ae:a3:53:1d:62:
    f8:f8:d1:e2:87
Exponent: 65537 (0x10001)
writing RSA key
-----BEGIN PUBLIC KEY-----
MIIBJTANBgkqhkiG9w0BAQEFAAOCARIAMIIBDQKCAQQAjaVpGbUm1FIlrO1L5kUi
zvBKY5ELn2/+prESVUEBO+RdSLb7JnG3VA5qTgtV46nkxFqNX1SgaZxlMtShKH+s
sAixxW411gHcKp4uZlGJ6qNdIte+olLB7PJwMatlfVs16CzecPglnS4U6YbzYuPo
bnvY5IEqUvLozC9puLDJWXeP2yQKjhfLlXJFcBLYO2xycpDlC459oo/r36v1I9oD
uUuUMir0IfnKAq3mBNqrks2cKCRENvEV/b7XbSyFAHrKf85JiTr4DnlVYy7HuJ9W
+rAYduD+iCmaNzQNQ5yy4Bs8B+YMiEdCG/4EnVmVQGsm9KCLINFJs8YarqNTHWL4
+NHihwIDAQAB
-----END PUBLIC KEY-----
```
So we are facing RSA with a 2048bits modulo and a classical value of 65537 for the public exponent. The first thing to do is to check whether or not a factorization for the modulus as already been done.

If you wonder why this factorization would help us, I suggest you to take a look at this wiki to understand how RSA works : https://en.wikipedia.org/wiki/RSA_(cryptosystem)#Operation

Print the modulus in decimal:
```
HEXA=$(openssl rsa -pubin -in rsa/pub_key -modulus | grep Modulus | cut -d'=' -f2) && echo "ibase=16;$HEXA" | bc | tr -d '\n' | tr -d '\\'
```
```
299996217561787292756826251240073744022587364427659002955601969311597453693948323421942282716737653493469667806494795328718748694431287426493332498123774403296361258944222401796946976412532226598881087042326060698386611304550152758781853605660146138394024484376984580234460609993575374222942038026173435262460884234328411077658271473762471945787635582916630508147146325427058379173689622281755189370552117476758492729644576568772220182957835384550972772092654842082706142246481708409910183742375894996805693099913395071166112170527842473265346582564838421321907545834628201837626578791668861148755559537560386588395858682503
```

Ask factordb.com:
```
2999962175...03<624> = 10038779 · 2988373561...57<617>
```

Nice ! We already have a smooth factorization for this long-ass modulus. We know that the private exponent is the modular inverse of the public exponent modulo phi(modulus). Let's recover the private exponent and get that private key using python:

```python
from Crypto.PublicKey import RSA
import gmpy

# Init known RSA paramaters
n = long(2999962175...) # modulus
e = long(65537)         # pub exponent
p = long(10038779)      # first modulus prime factor 
q = long(2988373561...) # second modulus prime factor

# Get priv exponent
d = long(gmpy.invert(e,(p-1)*(q-1)))

# Get and print priv key in pem format
key = RSA.construct((n,e,d))
print RSA.exportKey(key)
```
Aaaaaaand we get :

```
-----BEGIN RSA PRIVATE KEY-----
MIIFMQIBAAKCAQQAjaVpGbUm1FIlrO1L5kUizvBKY5ELn2/+prESVUEBO+RdSLb7
JnG3VA5qTgtV46nkxFqNX1SgaZxlMtShKH+ssAixxW411gHcKp4uZlGJ6qNdIte+
olLB7PJwMatlfVs16CzecPglnS4U6YbzYuPobnvY5IEqUvLozC9puLDJWXeP2yQK
jhfLlXJFcBLYO2xycpDlC459oo/r36v1I9oDuUuUMir0IfnKAq3mBNqrks2cKCRE
NvEV/b7XbSyFAHrKf85JiTr4DnlVYy7HuJ9W+rAYduD+iCmaNzQNQ5yy4Bs8B+YM
iEdCG/4EnVmVQGsm9KCLINFJs8YarqNTHWL4+NHihwIDAQABAoIBBACDIRCrLBah
FuGWW1qHgAuAxzdLzR3eObIv/qEdR6MeAmY2QmMga2lvm21I54rKStCNAspH1CG+
ix/LJ1boMjTjsUPxowKJyUFP4yr3EaCyPvqiRCZdciDUm/0230fUfxpl8133u16P
CucGeuevMfs4ffgz4OtmpCU6JoxdtbN1J8vRWz3kkDNCVco7WeDdPgS1vcR6KZeB
HwYb9Xk+VjAHVN5YM8IPPSbkYlkNJ0c8/PEKIL4ynLGXOJvCR2zi1HZIhl/f9mA0
zUf5DD2DMQhByzOOtqWOV3/3Yk2tdlVZZ3v2OGiO3/NXEAEL6ZyG41WkG82blg2C
sxPCDaHe5HPzdkdZAoIBAQDsuYuEnwgauOUvcjEJhNMJY+6gBpyLm41DggIo1a8T
a5PMkBSBy8kyAJo5dtWZRXqbEor92ruC075A4we/pkw2hoBH9ZoeTa2t5BpzfFNh
L7wpxUtKtGEAEUTBgnTWBRReuiLgxiwqBa2ISwr7ns187Jz5LET/Bb5SJxmZsM7G
t9+z7bpQvp7NMVz9NCR9TF+oxisOimfSK8+nrdF0eMRDk7FSTgSNsGLbywS9BK64
4zjJabhOdG9wGOPLrPXLnUn6STeqGD+eJRPEW9SW8zrvA9iU+hGUuhiQmtn1E/ES
tAHCQY13B8B8v1nMDNYNuyx9+O8H6XhP1lnkrf7bLHPlAgQAmS37AoIBAGzBjcyf
N5z3Ryv2HXtPD5mn1LCmePNWwp66Mv3JtkaIzP1VUGaVVljnl/NAmj9xgTOPYFXi
UPV5DFZJN30gDLGcN4FX37d+XoWeX1yhSLlEsgDKyJ2Io2vhgyIYKk9NRB+FCpMT
2KRxuVj9iQ0y1xtGpZOAeC5l2BtsJUHLziPTxC2o0UlZWiHRRPR7KSx7kxM5//wN
MEeJozxZCfqlpR2a1AOJHmRuHez2p7WjWhZNJgC61lcM/UmV1cn0K3ShTaR0UOOP
gmLIi/1RZyj2lCPNM4q2HK+kk7aTvgDvaXj7RDeX4ENIR3HIg5vQZeMnE/jIDiEn
bQPY6bG7EnRlRtUCAz6h8wKCAQEAqXRCDolOPWZhhMRqUB0sekouUCBD7ICwDzlg
RJ7sT1sGCEtRc6c+clnn5p2qppcgIv/GbJDPKGsYeCakFY4Kf4sfff4NaPGL2BVs
MCNqXyl1WkrD58QtWBQgzjFcvLiZTnFScKrwi/5ohK0mjUFjkdXV2WLOp1T00+PC
tnyTF6APzrqjp7HUL2Ec3oVr3ErCxoq8ZgvJYsNdzUfkEkNHDRxHnV/7h4otEZvB
rgBYI5+yUYkiOnQiPpffPQQVWmIKN6wrGtCQWAnkS7alh9x35uopBLA8Nq+O/7dZ
DRbu8Ihi09wRu1ngJk2EeBlOg9jX//x10r2nq3EG+tDYkxSjoA==
-----END RSA PRIVATE KEY-----
```

We just copy-paste this nice private key to priv_key.pem and decrypt aes_key_cipher with it:

```
openssl rsault -decrypt -in aes_key_cipher -inkey priv_key.pem
```

Which gives us:

```
rsa_hackerzvoice_for_win
```

Noice. Seems like we got the hardest part. Let's decrypt the ciphertext with this password for AES, assuming they used aes-256-cbc:

```
openssl enc -aes-256-cbc -d -in ciphermessage -out ciphermessage.dec -k "rsa_hackerzvoice_for_win" -nosalt
```
```
cat ciphermessage.dec
```

Which smoothly gives us:

```
hey dude!

keep it up! https://www.youtube.com/watch?v=1F81S50xL8I

btw, i think the flag for this chall should be: ndh2k16_cac4015707

freeman
```	
