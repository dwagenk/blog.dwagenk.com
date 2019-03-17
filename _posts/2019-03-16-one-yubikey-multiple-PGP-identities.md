---
layout: post
title: Using one Yubikey with two (not completely) separate PGP-Identities
---

## Prelude
I planned writing some posts about embedded systems development. Well, I've been too busy to actually get started with that and although I've got some half-baked posts somewhere on my hard drive, this blog has been empty up to now.
This post is a little out of the embedded systems programming scope I had in mind for this blog, but I've spent quite some time on getting it all together and felt the urge to properly document it. Leastwise to give some guidance to my future self about the setup of my PGP keys.

## The desired PGP setup
I'm not a heavy duty PGP user, but I've had keys setup for my e-mail addresses for about 5 years now. I'm using one e-mail address for personal, but more or less formal correspondence, and one, that is based on a pseudonym of mine and that I use for correspondence with just a few people.

Let's use an (partly realistic, partly fictional) example:

Suppose I'm a person called _Daniel Wagenknecht_ and mainly I'm using the e-mail address _dwagenk@mailprovider.org_. I'm also enthusiastic about comics and in those circles I'm known as _comicfreak_dan_, using the e-mail address _comicfreak_dan@comics.com_. Since I'm privacy aware I'm using PGP whenever it is not tooooo inconvenient. Putting both of my e-mail addresses into the same PGP identity results in something like this:
```bash
sec   rsa4096/0x73E8D88F 2019-03-12 [SCE]
uid           [ultimate] Daniel Wagenknecht <dwagenk@mailprovider.org>
uid           [ultimate] comicfreak_dan <comicfreak_dan@comics.com>
```
It looks wrong, using it is like introducing myself as _Daniel Wagenknecht a.k.a comicfreak_dan_ when I'm opening a bank account or meeting my doctor. I don't want to link my pseudonym to all my formal email correspondence.

That's why, when I first got started using PGP, I decided to use two identities instead. One is linked to my main, formal e-mail address and a second one goes with my pseudonymous email address.

The handling was easy, I just needed both private keys on my computer and for convenience set the same password for both of them. 

### In comes the Yubikey
Now after having had a Yubikey for a couple months and using it for 2-Factor-Authentication with some services, I also wanted to use it for PGP encryption and signing. But a single Yubikey has exactly one PGP subkey slot for each:
- encryption
- signing
- Authentication

So to keep the two PGP identities separate I could either use two Yubikeys, or transition only one PGP identity to the Yubikey and keep the other one in the _traditional_ way. Both solutions didn't really satisfy me. 

My idea now was to use two identities, meaning two separate PGP masterkeys, but both holding the same subkeys for encryption, signing and authentication. So the desired key structure looks similar to this:
```bash
sec   rsa4096/0x73E8D88F 2019-03-12 [C]
uid           [ultimate] Daniel Wagenknecht <dwagenk@mailprovider.org>
ssb   rsa4096/0x87D67E3D 2019-03-12 [S] [expires: 2020-03-11]
ssb   rsa4096/0xA4C5604D 2019-03-12 [A] [expires: 2020-03-11]
ssb   rsa4096/0xC26C348F 2019-03-12 [E] [expires: 2020-03-11]

sec   rsa4096/0xDD6AA744 2019-03-12 [C]
uid           [ultimate] comicfreak_dan <comicfreak_dan@comics.com>
ssb   rsa4096/0x87D67E3D 2019-03-12 [S] [expires: 2020-03-11]
ssb   rsa4096/0xA4C5604D 2019-03-12 [A] [expires: 2020-03-11]
ssb   rsa4096/0xC26C348F 2019-03-12 [E] [expires: 2020-03-11]
```

Of course there is a drawback. As you can see above, someone who has both my public keys will be able to reach the conclusion, that both keys belong to me. If both keys are present on the keyservers, then searching for one of the subkey ids there will also list both master keys and make it pretty obvious, that both belong to me.
I'm not living a dual life, so avoiding any link between the two of my identities by all means is not necessary and I'm fine with the solution. Of curse this could very well differ for you. Maybe you are a superhero and need to separate your identities? Maybe you're just living in a country where you're better off staying incognito as a political activist (always a good idea e.g. looking at Turkey right now or at the repression linked to the G20 summit in Hamburg, Germany about 1 <sup>1</sup>/<sub>2</sub> years ago).


Also I don't know, if people having more experience with PGP than I do, will disapprove with this concept, because different masterkeys sharing the same subkeys is something not intended by PGP.

## HowTo: use the same subkeys on a Yubikey with different master keys
There's a couple of tutorials online, on how to set up PGP / GPG / Yubikey to work together. I'd recommend reading https://github.com/drduh/YubiKey-Guide.  Here's just a quick summary:
1. Use an offline computer for the key generation and keeping your master key. If you want to play around and do a dry-run, use a temporary GNUPGHOME (`export GNUPGHOME=$(mktemp -d) ; echo $GNUPGHOME`)
2. Ensure you've got a secure GPG configuration following best practices, both on your regular and your computer for key generation 
3. create a new key with only the certification capability
4. add subkeys for signing, encryption and authentication
5. backup your keys to a secure location
6. configure your Yubikey
7. move the subkeys to your Yubikey
8. copy, import and trust your public key on your devices
9.  use it!

I'll not be going into detail about the common parts of this procedure, but focus on the _two masterkeys sharing identical subkeys_ part. GPG comes with a way to import existing keys, using something called the keygrip, that allows us to copy the subkeys from one master key to another. 
On my first try using the subkeys with the second master key I failed, because the imported subkeys had different fingerprints then the original ones despite being made up of the same private key material. The cause: fingerprints hash not only the private key, but also it's timestamp. So to have identical fingerprints for our subkeys we'll have to create a timewarp later...

### Now let's get started!
 
1. Teach your computer how to create a timewarp, install `faketime`. 
   -    Ubuntu: `sudo apt-get install faketime`
   -    Arch: `sudo pacman -Sy libfaketime`
   -    Mac: faketime should be available, too
   -    Windows: I've seen a tool called RunAsDate, that could work for our purposes

    Reach out to me, if you've got more information about how to make it work on Windows/Mac and I can include it here.

2. create a master key with only the certification capability 
    ```bash
    NAME="Daniel Wagenknecht"
    MAIL="dwagenk@mailprovider.org"
    COMMENT=""
    {
    echo 8          # RSA (set your own capabilities)
    echo S          # Toggle off signing
    echo E          # Toggle off encryption
    echo Q          # Finished with setting own capabilities
    echo 4096       # Key length
    echo 0          # Key does not expire
    echo y          # is this correct?
    echo $NAME      # Real Name
    echo $MAIL      # Email address
    echo $COMMENT   # Comment
    echo O          # Okay
    } | gpg2 --command-fd=0 --status-fd=1 --expert --full-generate-key

    # Save the key-id of the generated key in environment for later use
    KEYID=id_of_your_just_created_key
    ```
    Since I like to program, meaning I'd rather spent hours on automating a task, than repeatedly answering interactive prompts, I'm using bash syntax and some flags for GPG to pass in input to GPG. Check out [this](https://serverfault.com/a/858164) answer on StackExchange/ServerFault for details. 
    
    You should just need to adjust the variables at the top and copy the whole block into a terminal. The password prompts will still be interactive. 
    
    If you want to know what's actually happening, do the steps interactively by running `gpg2 --expert --full-generate-key` and using the echo statements above as guidance on what to select.
3. Create the second master key
    As I mentioned above we need to bend time a little and GPG will complain, if it notices what we're doing. Creating the second master key before the subkeys will prevent that.
    ```bash
    NAME="comicfreak_dan"
    MAIL="comicfreak_dan@comics.com"
    COMMENT=""
    {
    echo 8          # RSA (set your own capabilities)
    echo S          # Toggle off signing
    echo E          # Toggle off encryption
    echo Q          # Finished with setting own capabilities
    echo 4096       # Key length
    echo 0          # key does not expire
    echo y          # is this correct?
    echo $NAME      # Real Name
    echo $MAIL      # Email address
    echo $COMMENT   # Comment
    echo O          # Okay
    } | gpg2 --command-fd=0 --status-fd=1 --expert --full-generate-key

    # Save the key-id of the generated key in environment for later use
    KEYID2=id_of_your_just_created_key
    ```
4. add subkeys to the first master key
    ```bash
    # Signing
    {
    echo addkey
    echo 8          # RSA (set your own capabilities)
    echo E          # Toggle off encryption
    echo Q          # Finished with setting own capabilities
    echo 4096       # Key length
    echo 1y         # Key is valid for 1 year
    echo y          # is this correct?
    echo y          # really create?
    echo save   
    } | gpg2 --command-fd=0 --status-fd=1 --expert --edit-key $KEYID

    # Encryption
    {
    echo addkey
    echo 8          # RSA (set your own capabilities)
    echo S          # Toggle off signing
    echo Q          # Finished with setting own capabilities
    echo 4096       # Key length
    echo 1y         # Key is valid for 1 year
    echo y          # is this correct?
    echo y          # really create?
    echo save   
    } | gpg2 --command-fd=0 --status-fd=1 --expert --edit-key $KEYID

    # Authentication
    {
    echo addkey
    echo 8          # RSA (set your own capabilities)
    echo S          # Toggle off signing
    echo E          # Toggle off encryption
    echo A          # Toggle on Authentication
    echo Q          # Finished with setting own capabilities
    echo 4096       # Key length
    echo 1y         # Key is valid for 1 year
    echo y          # is this correct?
    echo y          # really create?
    echo save   
    } | gpg2 --command-fd=0 --status-fd=1 --expert --edit-key $KEYID
    ```
5. collect all the necessary data about the subkeys
    
    We need each subkeys so called _keygrip_ and it's _creation timestamp_. 
    
    Enter
    `gpg2 --list-secret-keys --with-keygrip --with-colons` and extract the needed information. See the highlighted parts in the output.
    <div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[...]
    sec:u:4096:1:AE299F4373E8D88F:1552427224:::u:::cC:::+:::23::0:
    fpr:::::::::4C5A569630B7FFEEF29A811BAE299F4373E8D88F:
    grp:::::::::C020F1AC4B24B51C71E2481547DA6B926E8E7001:
    uid:u::::1552427224::B2FC8BCA72AC2CD7656861523F4B4F6752E42F3C::Daniel Wagenknecht &lt;dwagenk@mailprovider.org&gt;::::::::::0:                    
    ssb:u:4096:1:074D014F87D67E3D:<b><i>1552427749</i></b>:1583963749:::::<b><i>s</i></b>:::+:::23:
    fpr:::::::::814F2B6C98BCE6F780187695074D014F87D67E3D:
    grp:::::::::<b><i>0F3765677327A472B3FE52F01B93A22A5E202D13</i></b>::
    ssb:u:4096:1:571C4F86C26C348F:<b><i>1552427813</i></b>:1583963813:::::<b><i>e</i></b>:::+:::23:
    fpr:::::::::AB9F307FA0D9A648ED165193571C4F86C26C348F:
    grp:::::::::<b><i>008FF4813D7897EE8DD33A27FDA533F9AF71C7DC</i></b>: 
    ssb:u:4096:1:1428372BA4C5604D:<b><i>1552427780</i></b>:1583963780:::::<b><i>a</i></b>:::+:::23:
    fpr:::::::::3B1D7D423AABBDF6EB62B38D1428372BA4C5604D:
    grp:::::::::<b><i>E703AA6F7A9FCA9F91E1FFC4F38B30081021CFA4:</i></b>
    [...]</code></pre></div></div>

    So in my case the values are

    | Subkey   | Timestamp   | Keygrip |
    |---|---|--- |
    | s = sign  |  1552427749 |   0F3765677327A472B3FE52F01B93A22A5E202D13 |
    | e = encrypt  |   1552427813 |  008FF4813D7897EE8DD33A27FDA533F9AF71C7DC |
    | a = authenticate  |   1552427780 |   E703AA6F7A9FCA9F91E1FFC4F38B30081021CFA4 |

6. import the subkeys to the second master key

    I haven't found a way to automatically feed the keygrips to gpg, you'll need to copy them into the interactive prompts.
    ```bash
    # Signing
    TIMESTAMP_S=1552427749
    {
    echo addkey
    echo 13         # Existing key 
    echo E          # Toggle off encryption
    echo Q          # Finished with setting own capabilities
    echo 1y         # key is valid for 1 year
    echo y          # is this correct?
    echo y          # really create?
    echo save   
    } | faketime "$(date -d@$TIMESTAMP_S)" gpg2 --command-fd=0 --status-fd=1 --expert --edit-key $KEYID2

    ```
    ```bash
    # Encryption
    TIMESTAMP_E=1552427813
    {
    echo addkey
    echo 13         # RSA (set your own capabilities)
    echo S          # Toggle off signing
    echo Q          # Finished with setting own capabilities
    echo 1y         # key is valid for 1 year
    echo y          # is this correct?
    echo y          # really create?
    echo save   
    } | faketime "$(date -d@$TIMESTAMP_E)" gpg2 --command-fd=0 --status-fd=1 --expert --edit-key $KEYID2

    ```
    ```bash
    # Authentication
    TIMESTAMP_A=1552427780
    {
    echo addkey
    echo 13         # RSA (set your own capabilities)
    echo S          # Toggle off signing
    echo E          # Toggle off encryption
    echo A          # Toggle on Authentication
    echo Q          # Finished with setting own capabilities
    echo 1y         # key is valid for 1 year
    echo y          # is this correct?
    echo y          # really create?
    echo save   
    } | faketime "$(date -d@$TIMESTAMP_A)" gpg2 --command-fd=0 --status-fd=1 --expert --edit-key $KEYID2
    ```
Now we're done with the _two masterkeys sharing identical subkeys_ part. You can go on with the regular procedure already outlined.

7. backup your keys to a secure location
8. configure your Yubikey. You can set a pub key url, and the corresponding key will work out of the box on any machine that has a working PGP / GPG setup and can reach that url.
9. move the subkeys to the Yubikey (only once, although they are associated with 2 master keys)
10. copy, import and trust your public key on your devices
11. use it!

### A note on Openkeychain for Android
On my smartphone I'm using K9Mail and Openkeychain for my encrypted mails. Openkeychain has support for the Yubikey via NFC (or USB-C, depending on which device you got), so there's no worries about jeopardizing my private keys when reading and writing encrypted mail on there. When configuring Openkeychain with a Yubikey, it will read the subkeys ids from the Yubikey and search for a matching master key in your keychain and online. It works fine for the first masterkey, but will not import the second one without using a trick.
1. open Openkeychain and go to `manage private keys` (name could be slightly off, I've got it installed in german)
2. select `use security-token`
3. if it finds one of your public keys (through keyserver or the url saved on the Yubikey), this key should be ready to use. If not provide it as a file.
4. test the first key on the device
5. backup the first key
6. delete the first key
7. repeat steps 1.-3. but make sure it doesn't find the first public key, but the second (e.g. set your phone to airplane mode and provide the second public key via file)
8. test the second key on the device
9. restore the first key
10. it should work with both keys now

With this setup I've experienced both, GPG / Enigmail and Openkeychain mentioning the wrong user id / masterkey when trying to decrypt something, but decryption works anyway. The same could happen on any device, that has both of my public keys in the keychain, so someone who's got both my public keys imported might get the notice, that an email is signed by _dwagenk@mailprovider.org_, whilst it was sent by _comicfreak_dan@comics.com_. This is unfortunate, but I guess with a setup like this it's better to really separate the use of the keys: each of your correspondents should only have one of your public keys.

Thanks for reading. Send me an e-mail or open an issue in the [repo](https://github.com/DWagenk/blog.dwagenk.me) if you've got any comments on this post. I'm interested in some discussion about my PGP setup!


[CC-BY-SA-4.0](http://creativecommons.org/licenses/by-sa/4.0/)
