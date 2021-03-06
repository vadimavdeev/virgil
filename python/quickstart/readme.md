- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
    - [Obtaining an Access Token](#obtaining-an-access-token)
    - [Install](#install)
- [Use case](#use-case)
    - [Step 0. Initialization](#step-0-initialization) 
    - [Step 1. Generate and Publish the Keys](#step-1-generate-and-publish-the-keys)
    - [Step 2. Sign then Encrypt](#step-2-sign-then-encrypt)
    - [Step 3. Send a Message](#step-3-send-a-message)
    - [Step 4. Receive a Message](#step-4-receive-a-message)
    - [Step 5. Decrypt then Verify](#step-5-decrypt-then-verify)
- [Source code](#source-code)

## Introduction

In this guide we will get you up and running quickly with a simple IP messaging chat application you can build as you learn more about Virgil Crypto Library and Virgil Keys Services. Sounds like a plan? Then let's get cracking!

On the diagram below you can see a full picture of how these things interact with each other.
![Use case messaging](https://raw.githubusercontent.com/VirgilSecurity/virgil/master/images/IPMessaging.jpg)

## Prerequisites

### Obtaining an Access Token

First you must create a free Virgil Security developer's account by signing up [here](https://developer.virgilsecurity.com/account/signup). Once you have your account you can [sign in](https://developer.virgilsecurity.com/account/signin), create an application and generate an access token for your application.

The access token provides authenticated secure access to Virgil Keys Services and is passed with each API call. The access token also allows the API to associate your app's requests with your Virgil Security developer's account.

Use this token to initialize the SDK client [here](#step-0-initialization).


### Install

To install [package](https://cdn.virgilsecurity.com/virgil-crypto/python/) use the command below:

```
python setup.py install
```

or you can use pip to download and install package automatically:

```
python -m pip install virgil_sdk
```

You can easily add an SDK dependency to your project, just add the following code:

```python
from VirgilSDK import virgilhub
import VirgilSDK.virgil_crypto.cryptolib as cryptolib
```

## Use Case
**Secure any data end to end**: users need to securely exchange information (text messages, files, audio, video etc) while enabling both in transit and at rest protection.

- Application generates public and private key pairs using Virgil Crypto library and uses Virgil Keys service to enable secure end to end communications:
	- public key on Virgil Public Keys Service;
	- private key on Virgil Private Keys Service or locally.
- Sender’s information is encrypted in Virgil Crypto Library with the recipient’s public key.
- Sender’s encrypted information is signed with his private key in Virgil Crypto Library.
- Application securely transfers the encrypted data, sender’s digital signature and UDID to the recipient without any risk to be revealed.
- Application on the recipient’s side verifies that the signature of transferred data is valid using the signature and sender’s public key in Virgil Crypto Library.
- The received information is decrypted with the recipient’s private key using Virgil Crypto Library.
- Decrypted data is provided to the recipient.

## Step 0. Initialization

Initialize variables of [configuration file](https://github.com/VirgilSecurity/virgil-sdk-python/blob/master/Examples/VirgilIPChat/config.py) with your values.

```python
VIRGIL_ACCESS_TOKEN = ''
USER_PRIVATE_KEY_PASSWORD = 'MY_PASSWORD'
```

> **VIRGIL_ACCESS_TOKEN** - is received for your application in [Developers portal](https://developer.virgilsecurity.com/dashboard/), described in [this step](/api-docs/python/quickstart#obtaining-an-access-token).

> **USER_PRIVATE_KEY_PASSWORD** - is a password of sender's private key.

Initialize global variables:
```python
virgil_hub = virgilhub.VirgilHub(VIRGIL_ACCESS_TOKEN,
                                 VIRGIL_IDENTITY_SERVER_URL,
                                 VIRGIL_KEYS_SERVICE_URL,
                                 VIRGIL_PRIVATE_KEY_SERVICE_URL)

chat_channel = Chat(CHAT_API_URL, 'DEMO', user_pass['identity'])
```

Read more about `Chat` class implementation [here](https://github.com/VirgilSecurity/virgil-sdk-python/blob/master/Examples/VirgilIPChat/chat.py).

## Step 1. Generate and Publish the Keys
First a simple IP messaging chat is generating the keys and publishing them to the Public Keys Service where they are available in an open access for other users (e.g. recipient) to verify and encrypt the data for the key owner.

The following code example generates a new public/private key pair.

```python
user_key_pair = cryptolib.CryptoWrapper.generate_keys(
            cryptolib.crypto_helper.VirgilKeyPair.Type_EC_Curve25519, USER_PRIVATE_KEY_PASSWORD)
```

The app is registering a Virgil Card which includes a public key and an email address identifier. The card will be used for the public key identification and searching for it in the Public Keys Service. You can create a Virgil Card with or without identity verification, see both examples [here](/api-docs/python/keys-sdk#publish-a-virgil-card).

```python
user_card = virgil_hub.virgilcard.create_card('email',
                                              USER_IDENTITY,
                                              None,
                                              None,
                                              user_key_pair['private_key'],
                                              USER_PRIVATE_KEY_PASSWORD,
                                              user_key_pair['public_key'])
```

## Step 2. Sign then Encrypt
The app is searching for all channel members' public keys on the Keys Service to encrypt a message for them. The app is signing the original message with sender’s private key so that the recipient can make sure the message had been sent by the declared sender.

```python
def sign_then_encrypt_message(text, recipients, private_key, private_key_password):

    # sign the original message with user's private key
    message_signature = cryptolib.CryptoWrapper.sign(text, private_key, private_key_password)

    # encrypt the original message for channel's members using
    # theirs public keys
    cipher = cryptolib.crypto_helper.VirgilCipher()
    for recipient in recipients:

        recipient_id = cryptolib.CryptoWrapper.strtobytes(recipient['id'])
        recipient_pubkey = cryptolib.CryptoWrapper.strtobytes(
            cryptolib.base64.b64decode(recipient['public_key']['public_key']).decode())

        cipher.addKeyRecipient(recipient_id, recipient_pubkey)

    encrypted_message = cipher.encrypt(cryptolib.CryptoWrapper.strtobytes(text), True)

    # return a model with encrypted message and the signature
    chat_message_model = {
       'message': helper.base64.b64encode(bytearray(encrypted_message)),
       'sign': helper.base64.b64encode(bytearray(message_signature))
    }

    return chat_message_model
```

## Step 3. Send a Message
The app merges the message text and the signature into one structure and sends the message to the channel using a simple IP messaging client.

```python
chat_channel.post_message(encrypted_message_model)
```

## Step 4. Receive a Message
An encrypted message is received on the recipient’s side using an IP messaging client.
In order to decrypt and verify the received data, the app on recipient’s side needs to get sender’s Virgil Card from the Keys Service.

```python
messages = chat_channel.get_messages(None)
```

## Step 5. Decrypt then Verify
The application is making sure the message came from the declared sender by getting his card on Virgil Public Keys Service. In case of success, the message is decrypted using the recipient's private key.

```python
def decrypt_then_verify_message(chat_message_model, card_id, private_key, private_key_password):

    # extract message & message signature from received message model.
    encrypted_message = bytearray(helper.base64.b64decode(chat_message_model['message']))

    try:
        # decrypt the message with user's private key
        message_data = cryptolib.CryptoWrapper.decrypt(encrypted_message, card_id, private_key, private_key_password)
        message = str(bytearray(message_data))

        # get a sender's Virgil Card to be used for signature validation
        sender_card = get_card_by_identity(chat_message_model['sender_identifier'])
        sender_public_key = sender_card['public_key']['public_key']

        # validate the signature of original message using sender's public key
        is_valid = cryptolib.CryptoWrapper.verify(message, chat_message_model['sign'], sender_public_key)
        if not is_valid:
            return '{} (signature is not valid)'.format(message)

        return message

    except Exception:
        return 'Message cannot be decrypted.'
```

## Source code

* [Use Case Example](https://github.com/VirgilSecurity/virgil-sdk-python/tree/master/Examples/VirgilIPChat)
* [IP-Messaging Simple Server](https://github.com/VirgilSecurity/virgil-sdk-javascript/tree/master/examples/ip-messaging/server)

> Note: Run scripts as root. 
