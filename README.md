# FORK: Use Cryptography instead of M2Crypto
This fork is updated to use the cryptography library for easy install on windows. In addition, m2crypto seems to be unmaintained.

## Installing this fork
You can install this fork like this:
```
pip install git+https://github.com/shivaRamdeen/passbook.git
```

# Passbook

[![Build Status](https://travis-ci.org/devartis/passbook.svg?branch=master)](https://travis-ci.org/devartis/passbook)

Python library to create Apple Wallet (.pkpass) files (Apple Wallet 
has previously been known as Passbook in iOS 6 to iOS 8).

See the [Wallet Topic Page](https://developer.apple.com/wallet/) and the
[Wallet Developer Guide](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/PassKit_PG/index.html#//apple_ref/doc/uid/TP40012195) for more information about Apple Wallet.

> If you need the server side implementation (API / WebServices) in django you should check http://github.com/devartis/django-passbook.


## Getting Started

1) Get a Pass Type Id

* Visit the iOS Provisioning Portal -> Pass Type IDs -> New Pass Type ID
* Select pass type id -> Configure (Follow steps and download generated pass.cer file)
* Use Keychain tool to export a Certificates.p12 file (need Apple Root Certificate installed)

2) Generate the necessary certificate

```shell
    $ openssl pkcs12 -in "Certificates.p12" -clcerts -nokeys -out certificate.pem   
```
3) Generate the key.pem

```shell
    $ openssl pkcs12 -in "Certificates.p12" -nocerts -out private.key
```

You will be asked for an export password (or export phrase). In this example it will be `123456`, the script will use this as an argument to output the desired `.pkpass`


## Note: Getting WWDR Certificate

Certificate is available @ http://developer.apple.com/certificationauthority/AppleWWDRCA.cer

It can be exported from KeyChain into a .pem (e.g. wwdr.pem).

# Typical Usage

## Create passbook file

```python
from django.conf import settings

from passbook.models import Barcode, BarcodeFormat, EventTicket, Location, Pass


def generate_pass(ticket):
    eventInfo = EventTicket()
    eventInfo.addPrimaryField("name", "event name", "EVENT")
    eventInfo.addSecondaryField("where", "event location", "WHERE")

    # QR CODE
    barcode = Barcode(message="test", format=BarcodeFormat.QR)

    # EVENT LOCATION
    location = Location(latitude=-44.1, longitude=-22.01)
    
    passfile = Pass(
        passInformation=eventInfo,
        passTypeIdentifier=settings.APPLE_WALLET_PASS_TYPE_ID,
        teamIdentifier=settings.APPLE_WALLET_TEAM_ID,
        organizationName="Org Name",
        backgroundColor="rgb(239,124,78)",
        foregroundColor="rgb(255,255,255)",
        labelColor="rgb(0,0,0)",
        serialNumber="1234121222",
        description="event description",
        barcode=barcode,
        locations=[location] if location else None,
    )

    # Including the icon and logo is necessary for the passbook to be valid.
    passfile.addFile("icon.png", open("socialpass-white.png", "rb"))
    passfile.addFile("logo.png", open("socialpass-white.png", "rb"))

    # GENERATE IN-MEMORY PASSBOOK
    passfile.create(
        settings.APPLE_WALLET_CERTIFICATE,
        settings.APPLE_WALLET_KEY,
        settings.APPLE_WALLET_WWDR_CERTIFICATE,
        settings.APPLE_WALLET_PASSWORD,
    )
    return passfile


```
It is possible generate the .pkpass file with the command

```python
passbook = generate_pass(ticket=ticket)
# _pass.read() # read bytes
passbook.writetofile("Event.pkpass")
```

To send pkpass file via email 
```python
import smtplib
from email import encoders
from email.mime.base import MIMEBase
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.utils import formatdate

def sendGmail(
    user,
    pwd,
    FROM,
    TO,
    SUBJECT,
    textMessage="new test",
):

    msg = MIMEMultipart()
    msg["From"] = FROM
    msg["To"] = TO
    msg["Date"] = formatdate(localtime=True)
    msg["Subject"] = SUBJECT
    msg.attach(MIMEText(textMessage))

    part = MIMEBase("application", "octet-stream")

    passbook = generate_pass(ticket=ticket)
    part.set_payload(passbook.read())
    del passbook # just to make sure it was deleted from memory

    encoders.encode_base64(part)
    part.add_header("Content-Disposition", 'attachment; filename="Event.pkpass"')
    msg.attach(part)

    try:
        server = smtplib.SMTP_SSL("smtp.gmail.com", 465)
        server.ehlo()
        server.login(user, pwd)
        server.sendmail(FROM, TO, str(msg))
        server.close()
        print("successfully sent the mail")
    except Exception as e:
        print(e)


if __name__ == "__main__":
    sendGmail(
        "sender@email",
        "sender_password",
        "sender@email",
        "receiver@email",
        "subjective text",
        "email body text"
    )

```
