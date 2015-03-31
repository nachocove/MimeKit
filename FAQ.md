# Frequently Asked Questions

## Question Index

* [How do I get the message body text?](#MessageBody)
* [How do I get the email addresses in the From, To, and Cc headers?](#AddressHeaders)
* [How do I decrypt PGP messages that are embedded in the main message text?](#DecryptInlinePGP)
* [How would I parse multipart/form-data from an HTTP web request?](#ParseWebRequestFormData)

### <a name="MessageBody">How do I get the message body text?</a>

(Note: for the TL;DR version, skip to [the end](#MessageBodyTLDR))

MIME is a tree structure of parts. There are multiparts which contain other parts (even other multiparts).
There are message parts which contain messages. And finally, there are leaf-node parts which contain content.

There are a few common message structures:

1. The message contains only a `text/plain` or `text/html` part (easy, just use that).

2. The message contains a `multipart/alternative` which will typically look a bit like this:

    ```
    multipart/alternative
       text/plain
       text/html
    ```

3. Same as above, but the html part is inside a `multipart/related` so that it can embed images:

    ```
    multipart/alternative
       text/plain
       multipart/related
          text/html
          image/jpeg
          image/png
    ```

4. The message contains a textual body part as well as some attachments:

    ```
    multipart/mixed
       text/plain or text/html
       application/octet-stream
       application/zip
    ```

5. the same as above, but with the first part replaced with either #2 or #3. To illustrate:

    ```
    multipart/mixed
       multipart/alternative
          text/plain
          text/html
       application/octet-stream
       application/zip
    ```

    or...

    ```
    multipart/mixed
       multipart/alternative
          text/plain
          multipart/related
             text/html
             image/jpeg
             image/png
       application/octet-stream
       application/zip
    ```

<a name="MessageBodyTLDR"></a>Now, if you don't care about any of that and just want to get the text of
the first `text/plain` or `text/html` part you can find, that's easy.

`MimeMessage` has two convenience properties for this: `TextBody` and `HtmlBody`.

`MimeMessage.HtmlBody`, as the name implies, will traverse the MIME structure for you and find the most
appropriate body part with a `Content-Type` of `text/html` that can be interpreted as the message body.
Likewise, the `TextBody` property can be used to get the `text/plain` version of the message body.

### <a name="AddressHeaders">How do I get the email addresses in the From, To, and Cc headers?</a>

The `From`, `To`, and `Cc` properties of a `MimeMessage` are all of type `InternetAddressList`. An
`InternetAddressList` is a list of `InternetAddress` items. This is where most people start to get
lost because an `InternetAddress` is an abstract class that only really has a `Name` property.

As you've probably already discovered, the `Name` property contains the name of the person
(if available), but what you want is his or her email address, not their name.

To get the email address, you'll need to figure out what subclass of address each `InternetAddress`
really is. There are 2 subclasses of `InternetAddress`: `GroupAddress` and `MailboxAddress`.

A `GroupAddress` is a named group of more `InternetAddress` items that are contained within the
`Members` property. To get an idea of what a group address represents, consider the following
examples:

```
To: My Friends: Joey <joey@friends.com>, Monica <monica@friends.com>, "Mrs. Chanandler Bong"
    <chandler@friends.com>, Ross <ross@friends.com>, Rachel <rachel@friends.com>;
```

In the above example, the `To` header's `InternetAddressList` will contain only 1 item which will be a
`GroupAddress` with a `Name` value of `My Friends`. The `Members` property of the `GroupAddress` will
contain 5 more `InternetAddress` items (which will all be instances of `MailboxAddress`).

The above example, however, is not very likely to ever be seen in messages you deal with. A far more
common example would be the one below:

```
To: undisclosed-recipients:;
```

Most of the time, the `From`, `To`, and `Cc` headers will only contain mailbox addresses. As you will
notice, a `MailboxAddress` has an `Address` property which will contain the email address of the
mailbox. In the following example, the `Address` property will contain the value `john@smith.com`:

```
To: John Smith <john@smith.com>
```

If you only care about getting a flattened list of the mailbox addresses in a `From`, `To`, or `Cc`
header, you can simply do something like this:

```csharp
foreach (var mailbox in message.To.Mailboxes)
    Console.WriteLine ("{0}'s email address is {1}", mailbox.Name, mailbox.Address);
```

### <a name="DecryptInlinePGP">How do I decrypt PGP messages that are embedded in the main message text?</a>

Some PGP-enabled mail clients, such as Thunderbird, embed encrypted PGP blurbs within the text/plain body
of the message rather than using the PGP/MIME format that MimeKit prefers.

These messages often look something like this:

    Return-Path: <pgp-enthusiast@example.com>
    Received: from [127.0.0.1] (hostname.example.com. [201.95.8.17])
        by mx.google.com with ESMTPSA id l67sm26628445yha.8.2014.04.27.13.49.44
        for <pgp-enthusiast@example.com>
        (version=TLSv1 cipher=ECDHE-RSA-RC4-SHA bits=128/128);
        Sun, 27 Apr 2014 13:49:44 -0700 (PDT)
    Message-ID: <535D6D67.8020803@example.com>
    Date: Sun, 27 Apr 2014 17:49:43 -0300
    From: Die-Hard PGP Fan <pgp-enthusiast@example.com>
    User-Agent: Mozilla/5.0 (Windows NT 6.3; WOW64; rv:24.0) Gecko/20100101 Thunderbird/24.4.0
    MIME-Version: 1.0
    To: undisclosed-recipients:;
    Subject: Test of inline encrypted PGP blocks
    X-Enigmail-Version: 1.6
    Content-Type: text/plain; charset=ISO-8859-1
    Content-Transfer-Encoding: 8bit
    X-Antivirus: avast! (VPS 140427-1, 27/04/2014), Outbound message
    X-Antivirus-Status: Clean
    
    -----BEGIN PGP MESSAGE-----
    Charset: ISO-8859-1
    Version: GnuPG v2.0.22 (MingW32)
    Comment: Using GnuPG with Thunderbird - http://www.enigmail.net/
    
    SGFoISBJIGZvb2xlZCB5b3UsIHRoaXMgdGV4dCBpc24ndCBhY3R1YWxseSBlbmNy
    eXB0ZWQgd2l0aCBQR1AsCml0J3MgYWN0dWFsbHkgb25seSBiYXNlNjQgZW5jb2Rl
    ZCEKCkknbSBqdXN0IHVzaW5nIHRoaXMgYXMgYW4gZXhhbXBsZSwgdGhvdWdoLCBz
    byBpdCBkb2Vzbid0IHJlYWxseSBtYXR0ZXIuCgpGb3IgdGhlIHNha2Ugb2YgYXJn
    dW1lbnQsIHdlJ2xsIHByZXRlbmQgdGhhdCB0aGlzIGlzIGFjdHVhbGx5IGFuIGVu
    Y3J5cHRlZApibHVyYi4gTW1ta2F5PyBUaGFua3MuCg==
    -----END PGP MESSAGE-----

To deal with these kinds of messages, I've added a method to OpenPgpContext called `GetDecryptedStream` which
can be used to get the raw decrypted stream.

There are actually 2 variants of this methods:

```csharp
public Stream GetDecryptedStream (Stream encryptedData, out DigitalSignatureCollection signatures)
```

and

```csharp
public Stream GetDecryptedStream (Stream encryptedData)
```

The first variant is useful in cases where the encrypted PGP blurb is also digitally signed, allowing you to get
your hands on the list of digitial signatures in order for you to verify each of them.

To decrypt the content of the message, you'll want to locate the `TextPart` (in this case, it'll just be 
`message.Body`)
and then do this:

```
static Stream DecryptEmbeddedPgp (TextPart text)
{
    using (var memory = new MemoryStream ()) {
        text.ContentObject.DecodeTo (memory);
        memory.Position = 0;

        using (var ctx = new MyGnuPGContext ()) {
            return ctx.GetDecryptedStream (memory);
        }
    }
}
```

What you do with that decrypted stream is up to you. It's up to you to figure out what the decrypted content is
(is it text? a jpeg image? a video?) and how to display it to the user.

### <a name="ParseWebRequestFormData">How would I parse multipart/form-data from an HTTP web request?</a>

Since classes like `HttpWebResponse` take care of parsing the HTTP headers (which includes the `Content-Type`
header) and only offer a content stream to consume, MimeKit provides a way to deal with this using the following
two static methods on `MimeEntity`:

```csharp
public static MimeEntity Load (ParserOptions options, ContentType contentType, Stream content, CancellationToken cancellationToken = default (CancellationToken));

public static MimeEntity Load (ContentType contentType, Stream content, CancellationToken cancellationToken = default (CancellationToken));
```

Here's how you might use these methods:

```csharp
MimeEntity ParseMultipartFormData (HttpWebResponse response)
{
    var contentType = ContentType.Parse (response.ContentType);

    return MimeEntity.Parse (contentType, response.GetResponseStream ());
}
```

If the `multipart/form-data` HTTP response is expected to be large and you do not wish for the content to be
read into memory, you can use the following approach:

```csharp
MimeEntity ParseMultipartFormData (HttpWebResponse response)
{
    // create a temporary file to store our large HTTP data stream
    var tmp = Path.GetTempFileName ();

    using (var stream = File.Open (tmp, FileMode.Open, FileAccess.ReadWrite)) {
        // create a header for the multipart/form-data MIME entity based on the Content-Type value of the HTTP
        // response
        var header = Encoding.UTF8.GetBytes (string.Format ("Content-Type: {0}\r\n\r\n", response.ContentType));

        // write the header to the stream
        stream.Write (header, 0, header.Length);

        // copy the content of the HTTP response to our temporary stream
        response.GetResponseStream ().CopyTo (stream);

        // reset the stream back to the beginning
        stream.Position = 0;

        // parse the MIME entity with persistent = true, telling the parser not to load the content into memory
        return MimeEntity.Load (stream, persistent: true);
    }
}
```
