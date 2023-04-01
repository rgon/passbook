# `passbook`: Apple Wallet passes in python

Python library to create Apple Wallet (.pkpass) files (previously known as Passbook).

See the [Wallet Topic Page](https://developer.apple.com/wallet/) and the
[Wallet Developer Guide](https://developer.apple.com/documentation/walletpasses/building_a_pass) for more information about Apple Wallet.

This library is a fork of devartis/passbook with support for modern python 3, and depends only on the `cryptography` library.

## Requirements:
* An Apple Developer account https://developer.apple.com/

## Setup - Credential Ceremony
Passbook generation requires a certificate generation ceremony, as well as the Apple WWDRCA. These instructions have been tested on Linux.

1. Create private key
    openssl genrsa -out pass.yourcompany.com.key 2048

2. Create CSR: for submission to CAs
    openssl req -new -key pass.yourcompany.com.key -out CertificateSigningRequest.csr -subj "/emailAddress=youremail@company.com, CN=YourCompany Pass, C=COUNTRYCODE"

3. Download your apple-signed certificate
https://developer.apple.com/account/resources/certificates/yourAppleID/add
will download `pass.yourcompany.com.cer`

4. Download the Apple `AppleWWDRCAG4.cer' (*Apple Worldwide Developer Relations Intermediate Certificate Expiration*) from https://www.apple.com/certificateauthority/

5. Convert WWDR Cert to PEM
    openssl x509 -in AppleWWDRCAG4.cer -out AppleWWDRCAG4.pem

6. Create pem file with cert
    openssl x509 -in pass.yourcompany.com.cer -inform DER -out pass.pem -outform PEM

In production environments, you should store these credentials in a `.env` file and then import each variable's content on `passfile.create()`.
```shell
$ cat certificate.pem
```
```shell
### Apple Wallet Config
APPLE_WALLET_PASS_TYPE_ID=pass.com.yourcompany.com
APPLE_WALLET_TEAM_ID=K2K2K2K2K2
APPLE_WALLET_ORGANIZATION_NAME=
# Credentials
APPLE_WALLET_CERTIFICATE="-----BEGIN CERTIFICATE-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
_contents of pass.pem
-----END CERTIFICATE-----"
APPLE_WALLET_KEY="-----BEGIN PRIVATE KEY-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
_contents of pass.yourcompany.com.key
-----END PRIVATE KEY-----"
```

## Setup

1. Certificate is available @ http://developer.apple.com/certificationauthority/AppleWWDRCA.cer


## Install
You can install this fork like this:
```
pip install git+https://github.com/rgon/passbook.git
```

## Usage

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

## Resulting pass validation:
You may test the passes validity in
https://pkpassvalidator.azurewebsites.net/ (not affiliated).