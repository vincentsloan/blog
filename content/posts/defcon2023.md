+++
title = '2023 DEF CON Payment Village'
date = 2023-11-04
draft = false
+++

August 2023, I had the privilege of attending DEF CON 31, the renowned security and hacking conference.
This was my first time, so I spent some time preparing and scoping out talks and workshops. Some grizzled veterans
suggested I look at villages, dedicated spaces around certain topics. One village stood out to me immediately,
the Payment Village! 

So, on Friday August 11th 2023, I spent almost the entire day at the Payment Village for the morning
workshop and hacking challenges throughout the afternoon. I was joined by the infamous GoFundMe alumnus Serguey, 
and my new friend [Saleem](https://saleemrashid.com/), 9-year-old wunderkind and offensive security engineer.

The day started out with a workshop/lecture on the history of credit cards - from the "charge tokens" of the early 1900s,
to the well known magnetic stripe cards, and up to the modern EMV/chip cards we use today. EMV cards were meant
to address all the shortcomings of the magnetic stripe cards, which were easy to spoof/copy. The EMV spec was very
large though, and contained provisions for "backwards compatibility" - like emulating magnetic track data
on the chip.

Armed with this knowledge, we were given the hacking challenges. We were given some Payment Village branded EMV cards,
and an Android APK for a Software based Point of Sale (POS) system, which used the Android phone's NFC to read the 
chip. For those without an Android phone, NFC readers and a suite of Windows software was provided.
The challenges were as follows:

1. Charge $100,000 to the card.
2. Commit Other Fraudulent Operations.
3. Steal 15 cards from the SoftPOS. Provide PAN and Expiry for each card.

## Charging $100,000 to the card
We focused almost entirely on the first challenge - charging $100,000 to the card. The card started out with $100,000 available,
so we just had to get to a $0 balance. 

We had an Android phone ready to go, so we went the Android APK route rather than reader+Windows - mostly because we 
were familiar with tampering with Android. After installing the SoftPOS APK, the first hurdle to charging $100,000 was clear: 
there was a $10 transaction limit.

### Hacking the APK
So, the first thought was to decompile the APK and see what we could modify or intercept. One common approach is to
decompile Android APKs with Apktool. Apktool decompiles the bytecode in the APK and produces an intermediate language
called Smali. It's more readable than bytecode, but certainly not as nice as Java or Kotlin. The nice thing about it
though, is that the modified Smali is easily converted back into bytecode. This means you can easily make a change,
and redeploy the modified APK.

We loaded up Apktool with a very nice VSCode extension called APKLab, and decompiled the SoftPOS APK.
We enabled debugging, ran a transaction on the app, and monitored the logs with `adb logcat`. One thing stood out immediately -
the API calls were being made over plain HTTP. This would make a man-in-the-middle attack trivial.
So we got excited and hatched a plan - we'll find the string for the API host in the decompiled APK, and point it 
to our own man-in-the-middle server. From there, we could rapidly inspect and manipulate the requests. 
So we set up `mitmproxy` on our Thinkpad, modified a few lines of the decompiled APK to point the requests there, 
and were able to intercept, modify and replay requests.

### EMV tags
Next, we started looking at the EMV tags. The workshop spent a good amount of time discussing these, so it seemed
like an obvious place to put some effort. 

Some EMV tags were apparent as part of the API request:
```
amount     = 1000
trans_type = EMV
9f15       = 5000
9f34       = 1f0302
9f36       = 000F
9f26       = DFB901237F1E5E1A
82         = 0040
5a         = 1016435898071137
5f36       = 1
9f37       = A9E06117
5f2a       = 0840
9a         = 230811
9f02       = 000000001000
type       = purchase
9c         = 20
```
We went through some of the tags we were sending and looked them up on emvlab.org. While we were there, we explored other possible tags.

One EMV tag in particular looked quite interesting and promising, [Transaction Currency Exponent](https://emvlab.org/emvtags/show/t5F36/)
With this, we could in theory change a $10 transaction into a $1,000 transaction, without altering any
other part of the payload! It's not part of the cryptogram, so it could be altered without breaking things.

We changed the value to "0", indicating that 10000 should be interpreted as $1,000 rather than $10. We made a few test requests,
and were met with the error "Not enough money!". It appeared we had successfully drained the account! 

After retracing our steps and attempting reproduce this success however, we discovered the empty account was not due to our
hacking skills, but rather the fact that we switched up the currency while we were at it (USD to EUR), and the EUR account
was low or empty already. 

Disappointed but undeterred, we stepped back and tried a more systematic approach.

### Making Smali changes
Back to the application, we tried a more straight forward method of increasing the transaction amount, which as previously mentioned,
was limited to $10. If we could raise this limit, we could make quick work of charging the $100,000. 

We looked for the validation error in the Smali code ("Price cannot be more than $10 as we do not accept PIN (yet)."), and found the
condition that compared the input to a maximum:

```
NfcHome.Smali

  const/16 v1, 0x3e8
  
  const v2, 0x7f0900b0

  if-le v0, v1, :cond_2
```
The "const/16 v1, 0x3e8" translated to a constant value of 1000 - the limit in pennies. We changed this to "const v1, 0x989680",
which translated to 10000000, or $100,000 in pennies. We made the change, re-deployed the APK to our Android phone, and gave it a try.

Our input validation change was successful! The application accepted the large input. Progress. Of course, it failed on the server side.
The API response came back: "Authorization above 10.00 requires a secure cardholder verification, e.g. online/offline PIN".

So - back to the EMV tags. Rather than looking at the full catalog of EMV tags, we focused on what we were already sending in the request.
```
9f15 
MCC - Merchant Category Code https://emvlab.org/emvtags/?number=9f15

9f34
CMV Results https://emvlab.org/emvtags/show/9f34/

9f36
Application Transaction Counter (ATC) https://emvlab.org/emvtags/show/t9F36/

9f26
Application Cryptogram  https://emvlab.org/emvtags/?number=9f26

82
Application Interchange Profile https://emvlab.org/emvtags/?number=82

5a
PAN (card number) https://emvlab.org/emvtags/?number=5a

9f37
Unpredictable Number https://emvlab.org/emvtags/?number=9f37

5f2a
Transaction Currency Code https://emvlab.org/emvtags/?number=5f2a

9a
Transaction Date https://emvlab.org/emvtags/?number=9a

9f02
Amount Authorized https://emvlab.org/emvtags/?number=9f02
```

The API error and input validation both mentioned a PIN code. Looking through the EMV tags cataloged previously,
it looked like what we needed was in [9F34, Cardholder Verification Method (CVM) Results](https://emvlab.org/emvtags/?number=9f34). 
The application was sending `1f0302`. We looked this up in an online [CVM Results Decoder](https://paymentcardtools.com/emv-tag-decoders/cvm-results)
which identified this as "No CVM Required" - a code used for low-value transactions. We looked up other possible values, and
tried `410000` which indicates "Plaintext PIN verification performed by ICC". 

### Success
We tried again ...
It worked! After altering the application-side validation, and changing the CVMR tag, we were able to make a $100 transaction. 
We then tried successively larger transactions: $1,000, $10,000, $25,000. They all went through successfully. 
Finally, we reached an API response with "Not enough money!". 

## Other Fraud (Free Money!)
There were a few hints flying around that there was a way to get free money *onto* the card. Negative transaction values were one thought,
but we couldn't get away with that. After some trial and error, we found that we could do some shady things with refunds. The ability
to issue a refund was revealed with an API error we encountered earlier: "Transaction Type should be purchase or refund". 

For any transaction made with "type=purchase", you could subsequently make a call with "type=refund", keeping the rest of the payload the same.
Cool, so we could get some money back. It became much more interesting however, when we realized you could change the "amount". Maybe this existed
for partial refunds? Either way, you could make a refund for an amount much larger than the previous "purchase" amount. For example, a purchase for $1.00:

```
GET http://www.paymentvillageprocessing.com/auth_host.php?amount=100&type=purchase&....
----
Card xxxx
Auth Success.
Receipt No.353640
$ 1.00
```

We were able to modify that original request, and get $100 refunded.
```
GET http://www.paymentvillageprocessing.com/auth_host.php?amount=10000&type=refund&....
----
Refund complete
```

Success - free money ($99)!

Unfortunately, trying to try anything higher than $100 resulted in an error:
```
The amount is above automatic clearing threshold. Call 8-800-PAYMENTS and we will initiate it for you
```

So, refunding large amounts of money would certainly be possible, but would take some extra time. 
There may have been another EMV tag or the like we could have tweaked to get through the automatic clearing threshold, 
but we couldn't get past it in the time given.

## Stealing Cards
We didn't get a chance to try this one. Getting the PAN is trivial - it was already being read and transmitted in the SoftPOS application.
This was initially surprising to me - I had always thought that part of the whole point of EMV was to keep such things private.

The expiration date was not being read, and really shouldn't be part of the chip (at least in plaintext). However, we know that this is one
of those "backwards compatibility" concessions, in that "track 2" equivalent data is available, which includes expiration date. I'm not sure
how fun this would been to wire up with Smali modifications but it was certainly possible. There may have been a more obvious answer as well.

## Closing thoughts
This was a lot of fun! I learned a lot, not just about hacking android APKs and EMV tags...but also about myself.

In hindsight, hacking the APK for every change was an unnecessarily long feedback loop in the long run. If I were to do this again,
I'd have spent more time trying to generate the proper payload without the Android application.