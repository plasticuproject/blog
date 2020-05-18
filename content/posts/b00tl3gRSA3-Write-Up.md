---
layout: post
title: "b00tl3gRSA3 Write-Up"
categories: Write-Up
date: 2020-05-15
image: /blog/images/b00tl3gRSA3/picoctf_logo.png
description: b00tl3gRSA3 is a picoCTF 2019 cryptography challenge where we have to decrypt a message given just the RSA Public Key constituents.
tags: [picoCTF, Write-Up, Python, Cryptography]
katex: true
markup: "mmark"
---

![](/blog/images/b00tl3gRSA3/picoctf_logo.png#center)

---

## The Challenge

![Challenge](/blog/images/b00tl3gRSA3/challenge.png#center)


Let's connect with netcat like it tells us and see what we are given to work with.

![Challenge](/blog/images/b00tl3gRSA3/values.png#center)


It has returned to us numerical values for $$c$$, $$n$$ and $$e$$. Experience tells us that this is an **encrypted ciphertext**, **RSA Key Modulus** value, and **RSA Key Exponent** value, respectively. Values $$n$$ and $$e$$ make up the **RSA Public Key** that was used to encrypt a plaintext message $$m$$, and produce ciphertext $$c$$. We need to break this encryption by calculating a **RSA Private Key** from these values, then use it to decrypt $$c$$. Let's first look at how RSA Keys are generated before trying to crack this.


---

## RSA Key Generation

"RSA key generation is fairly simple. It's mostly multiplication and exponentiation. The whole basis of RSA keys is that choosing two numbers, $$p$$ and $$q$$, and multiplying them together is *very easy*, while trying to figure out what two numbers multiply together to form $$n$$ is *very hard*." <sup id="cite_note-1">[[1]](#cite_ref-1)</sup>


### The Math

- $$n = pq$$

To calculate $$n$$ we need to find two random prime numbers, $$p$$ and $$q$$, of similar length and multiply them. The longer the numbers the stronger the encryption, but the trade-off is that it will take more time and/or power to encrypt or decrypt messages.


- $$\lambda(n) = (p - 1)(q - 1)$$

Compute $$\lambda(n)$$, where $$\lambda$$ is [Carmichael's totient function](https://en.wikipedia.org/wiki/Carmichael%27s_totient_function). Since $$n = pq$$, $$\lambda(n) = lcm(\lambda(p),\lambda(q))$$, and since $$p$$ and $$q$$ are prime, $$\lambda(p) = \phi(p) = p − 1$$ and likewise $$\lambda(q) = q − 1$$. Hence $$\lambda(n) = lcm(p − 1, q − 1)$$. <sup id="cite_note-2">[[2]](#cite_ref-2)</sup>


- $$e = 65537$$

We next choose an exponent, $$e$$. This number should be greater than $$1$$, less than and [coprime](https://en.wikipedia.org/wiki/Coprime_integers) with $$\lambda(n)$$. The value used for this is usually $$65537$$.


- $$d \equiv e^{-1} \pmod{\lambda(n)}$$

$$d$$ is the modular [multiplicative inverse](https://en.wikipedia.org/wiki/Multiplicative_inverse) of $$e$$ modulus $$ \lambda(n)$$. $$d$$ is the **Private Key Exponent** and should never be shared.


- $$c \equiv m^e \pmod{n}$$

This will encrypt your integer converted message $$m$$ into an RSA encrypted ciphertext $$c$$.


- $$c^d \equiv (m^e)^d \equiv m \pmod n$$

This will decrypt the encrypted ciphertext $$c$$ back into it's integer message form $$m$$.


$$n$$ and $$e$$ are used to construct the Public Key, while $$n$$ and $$d$$ are used to create the Private Key.


### Constructing Values and Keys in Python

Calculating our values in Python code will look something like this:
```python
# modular multiplicative inverse
def mod_inv(a, n):
    t, r = 0, n
    new_t, new_r = 1, a
    while new_r != 0:
        quotient = r // new_r
        t, new_t = new_t, t - quotient * new_t
        r, new_r = new_r, r - quotient * new_r
    if r > 1:
        raise Exception("a is not invertible")
    if t < 0:
        t = t + n
    return t

# random prime numbers
p = 1973
q = 1511

# calculate public key modulus
n = p * q

# public exponent
e = 65537

# Carmichaels's Totient of n
lambda_n = ((p - 1) * (q - 1))

# calculate private key exponent
d = mod_inv(e, lambda_n)

```

We can also use Python libraries to construct Public and Private Keys. Here we will use the [pycrypto](https://pypi.org/project/pycrypto/) library.
> **Note**: *This library can be insecure and is not recommended for production use unless you know exactly what you are doing.*

```python
from Crypto.PublicKey import RSA

# modular multiplicative inverse
def mod_inv(a, n):
    t, r = 0, n
    new_t, new_r = 1, a
    while new_r != 0:
        quotient = r // new_r
        t, new_t = new_t, t - quotient * new_t
        r, new_r = new_r, r - quotient * new_r
    if r > 1:
        raise Exception("a is not invertible")
    if t < 0:
        t = t + n
    return t

# random prime numbers
p = 1973 
q = 1511

# calculate public key modulus
n = p * q

# public exponent
e = 65537

# Carmichaels's Totient of n
lambda_n = ((p - 1) * (q - 1))

# calculate private key exponent
d = mod_inv(e, lambda_n)

# construct keys
key_params = (n, e, d, p, q)
key = RSA.construct(key_params)

# print Public and Private Key
print(key.exportKey().decode('utf-8') + '\n')
print(key.publickey().exportKey().decode('utf-8') + '\n')
```
This will produce the output:
```
-----BEGIN RSA PRIVATE KEY-----
MCUCAQACAy19UwIDAQABAgMBmJECAge1AgIF5wIBTQICAZMCAgZK
-----END RSA PRIVATE KEY-----

-----BEGIN PUBLIC KEY-----
MB4wDQYJKoZIhvcNAQEBBQADDQAwCgIDLX1TAgMBAAE=
-----END PUBLIC KEY-----
```

We can now write these to files for storage, distribution, and use to encrypt or decrypt messages.

---


## Back To The Challenge

Now that we know how RSA Keys are generated, we can try to crack the Public Key values that we obtained from the server and construct the Private key. The first thing we need to do is try to factor $$n$$ into it's primes $$p$$ and $$q$$. With strong enough prime values this would be impossible, but since this is a CTF challenge we know there will be a vulnerability somewhere in this scheme. To factor $$n$$, I will use a web tool called [Integer Factorization Calculator](https://www.alpertron.com.ar/ECM.HTM). Using the factoring tool on this site tells us that $$n$$ can be made from the following multiples:

```python
# factors of n
p1 = 8694307819
p2 = 8775407717
p3 = 9026241761
p4 = 9057149533
p5 = 9585778841
p6 = 9733417549
p7 = 9855694997
p8 = 9903110561
p9 = 9960439201
p10 = 10016164247
p11 = 10213913951
p12 = 10375133233
p13 = 10888745003
p14 = 10928267249
p15 = 11093110097
p16 = 11969573459
p17 = 12351917503
p18 = 12471968101
p19 = 13871073973
p20 = 14396917097
p21 = 14539225093
p22 = 14717833373
p23 = 14837579117
p24 = 15209272151
p25 = 15280005421
p26 = 15484737221
p27 = 15554133887
p28 = 16042139507
p29 = 16278648473
p30 = 16335802403
p31 = 16382994611
p32 = 16728205063
p33 = 16809927443
p34 = 16861924201
```

That's way more than two numbers! What is going on here? Let's look again at what the challenge description says:
> *Why use p and q when I can use more?*

Now this makes sense. They insecurely built their Keys by using _many_ numbers to compute $$n$$, and because of this we were able to factor $$n$$ into it's constituents. We now have the information we need to build the Private Key values and decrypt $$c$$. We can do this in Python, but will have to make some changes to our code. We will first write some test code to assert that all these integers multiplied will equal our $$n$$ value. Then we will place all the values into a list, and create some logic to derive our $$\lambda(n)$$ from the entire list of integers. Then we can calculate $$d$$, our **Private Key Exponent**. That code will look like this:
```python
from math import gcd

# modular multiplicative inverse
def mod_inv(a, n):
    t, r = 0, n
    new_t, new_r = 1, a
    while new_r != 0:
        quotient = r // new_r
        t, new_t = new_t, t - quotient * new_t
        r, new_r = new_r, r - quotient * new_r
    if r > 1:
        raise Exception("a is not invertible")
    if t < 0:
        t = t + n
    return t

# Carmichael's totient
def calc_lambda_n(ps):
    lcm = ps[0] - 1
    for i in ps[1:]:
        j = i - 1
        lcm = lcm*j//gcd(lcm, j)
    return lcm

# test factorization
def test_factor(ps, n):
    q = 1
    for i in ps:
        q *= i
    if q == n:
        b = True
    elif q != n:
        b = False
    return b

# factors of n
p1 = 8694307819
p2 = 8775407717
p3 = 9026241761
p4 = 9057149533
p5 = 9585778841
p6 = 9733417549
p7 = 9855694997
p8 = 9903110561
p9 = 9960439201
p10 = 10016164247
p11 = 10213913951
p12 = 10375133233
p13 = 10888745003
p14 = 10928267249
p15 = 11093110097
p16 = 11969573459
p17 = 12351917503
p18 = 12471968101
p19 = 13871073973
p20 = 14396917097
p21 = 14539225093
p22 = 14717833373
p23 = 14837579117
p24 = 15209272151
p25 = 15280005421
p26 = 15484737221
p27 = 15554133887
p28 = 16042139507
p29 = 16278648473
p30 = 16335802403
p31 = 16382994611
p32 = 16728205063
p33 = 16809927443
p34 = 16861924201

# list of factors
ps = [p1, p2, p3, p4, p5, p6, p7, p8, p9, p10, p11, p12, p13, p14, p15, p16, p17, p18, p19, p20, p21, p22, p23, p24, p25, p26, p27, p28, p29, p30, p31, p32, p33, p34]

# verify p1*p2*...p34 = n
assert(test_factor(ps, n))

# Carmichaels's Totient of n
lambda_n = (calc_lambda_n(ps))

# calculate private key exponent
d = mod_inv(e, lambda_n)
```

Now we can just add our $$c$$, $$n$$, and $$e$$ values given, and decrypt $$c$$ by adding the following:
```python
message = pow(c, d, n)
print(bytes.fromhex(hex(message)[2:]).decode('utf-8'))
```

The final Python script will look as follows:
```python
from math import gcd

# cipher text and public key values
c = 14233215171999431620670742801992653887466654474564774567565482115708994971815998692805257101560625714845217078751863031453530789139353946667912836198369580283828831512520753040759574661583289733067194603349961285220341733595404870252973575776007626137207539918299185290666626201682475925787500697075273517928342450904756899125972158778923306466

n = 17190751765871320929594511788715040293755178677970494040999935752481499432324081420804535569049828358872534054083287610758506232432273611461779363808286315449034509728493030687881521253447668499524454596411931335376554928809260061677941571447357490867956927337940685369932675855590823386769800936470170609619423644569934236531433566627693968207

e = 65537

# modular multiplicative inverse
def mod_inv(a, n):
    t, r = 0, n
    new_t, new_r = 1, a
    while new_r != 0:
        quotient = r // new_r
        t, new_t = new_t, t - quotient * new_t
        r, new_r = new_r, r - quotient * new_r
    if r > 1:
        raise Exception("a is not invertible")
    if t < 0:
        t = t + n
    return t

# Carmichael's totient
def calc_lambda_n(ps):
    lcm = ps[0] - 1
    for i in ps[1:]:
        j = i - 1
        lcm = lcm*j//gcd(lcm, j)
    return lcm

# test factorization
def test_factor(ps, n):
    q = 1
    for i in ps:
        q *= i
    if q == n:
        b = True
    elif q != n:
        b = False
    return b

# factors of n
p1 = 8694307819
p2 = 8775407717
p3 = 9026241761
p4 = 9057149533
p5 = 9585778841
p6 = 9733417549
p7 = 9855694997
p8 = 9903110561
p9 = 9960439201
p10 = 10016164247
p11 = 10213913951
p12 = 10375133233
p13 = 10888745003
p14 = 10928267249
p15 = 11093110097
p16 = 11969573459
p17 = 12351917503
p18 = 12471968101
p19 = 13871073973
p20 = 14396917097
p21 = 14539225093
p22 = 14717833373
p23 = 14837579117
p24 = 15209272151
p25 = 15280005421
p26 = 15484737221
p27 = 15554133887
p28 = 16042139507
p29 = 16278648473
p30 = 16335802403
p31 = 16382994611
p32 = 16728205063
p33 = 16809927443
p34 = 16861924201

# list of primes
ps = [p1, p2, p3, p4, p5, p6, p7, p8, p9, p10, p11, p12, p13, p14, p15, p16, p17, p18, p19, p20, p21, p22, p23, p24, p25, p26, p27, p28, p29, p30, p31, p32, p33, p34]

# verify p1*p2*...p34 = n
assert(test_factor(ps, n))

# Carmichaels's Totient of n
lambda_n = (calc_lambda_n(ps))

# calculate private key exponent
d = mod_inv(e, lambda_n)

# decrypt our ciphertext
message = pow(c, d, n)

# print utf-8 encoded results
print(bytes.fromhex(hex(message)[2:]).decode('utf-8'))
```

And this will print out the resulting decrypted message:
```
picoCTF{too_many_fact0rs_0744041}
```

We enter the flag and are notified that our solution is correct.
![solved](/blog/images/b00tl3gRSA3/solved.png#center)


This was a fun challenge demonstrating how to use a basic RSA encryption method for un-padded messages. This is not a very real world example, as it is unlikely you will ever see $$n$$ computed with more than two values, and real messages are rarely un-padded, but this was a great way to get your feet wet with basic principles of RSA Encryption.

---


#### References:

1. <sup id="cite_ref-1">[\^](#cite_note-1)</sup> Bulischeck, N. ["An Introduction to RSA"](https://nbulischeck.io/posts/introduction-to-rsa/)
-  <sup id="cite_ref-2">[\^](#cite_note-2)</sup> Wikipedia. ["RSA (cryptosystem)"](https://en.wikipedia.org/wiki/RSA_(cryptosystem)#Key_generation)

---




