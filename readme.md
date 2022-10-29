# HTTP CID Data

A static website that acts as a provider of content addressed data.

How to replicate this:

1. Get blocks representing some data:

```
car create -f my.car files
car ls lw.car
car gb lw.car bafybeifgz2z4kghtvo2iy6xxo3uil2yu5iils5ffvjzgn4znpbjpas7lw4 > bafybeifgz2z4kghtvo2iy6xxo3uil2yu5iils5ffvjzgn4znpbjpas7lw4
```

Extract each block from `ls` into it's own block named by it's CID.

2. make the http index provider records.

For this example the command used is:
```
go run ./cmd/provider e --ctxid helloworld --metadata 4AMA --cid bafkreiefztf2jizqd7qsgu4xhdjqbi3tkcrputiqbrjrtu7ywfgnsirgsu --addr /dns4/willscott.github.io/tcp/443/https/httpath/L2h0dHAtY2lkLWRhdGEv --outDir to/ -i cid.contact
```

from the modified version of the `index-provider` in the feat/http-provider branch:
https://github.com/filecoin-project/index-provider/tree/feat/http-provider

3. take the resulting files (an advertisement, and entries chunk, and a `head`, and put them with the blocks at the hosted location.)

4. Announce to an indexer.

```js
import { Duplex } from 'stream'
import fetch from 'node-fetch'
import { BufferList } from 'bl'
import { encode, Token, Type } from 'cborg'
import { CID } from 'multiformats/cid'
import { sha256 } from 'multiformats/hashes/sha2'

function _base64ToArrayBuffer(base64) {
    var binary_string = atob(base64);
    var len = binary_string.length;
    var bytes = Buffer.alloc(len);
    for (var i = 0; i < len; i++) {
        bytes[i] = binary_string.charCodeAt(i);
    }
    return bytes;
}

const ma64 = "NhN3aWxsc2NvdHQuZ2l0aHViLmlvBgG7uwOAhMABDWh0dHAtY2lkLWRhdGGlAyYAJAgBEiChXJ2VTZmsm3DKn1oxKYkuKL9vpC/gE9N5huKrM7BVZw=="
const mab = _base64ToArrayBuffer(ma64)

const none = Buffer.alloc(0)
const advertisementCid = CID.parse("baguqeeradhg7qdq6dpizzd3e552m5dsbdz5jmeu7fgapduz5ly3sk5b4wseq")
//const advertisementCid = CID.create(1, 0x0129, await sha256.digest(none))
console.log("advert is ", advertisementCid)
const cbor = encode([advertisementCid, [mab], Buffer.alloc(0)],
{
 typeEncoders: {
   Object: function (cid) {
     // CID must be prepended with 0 for historical reason - See: https://github.com/ipld/cid-cbor
     const bytes = new BufferList(Buffer.alloc(1))
     bytes.append(cid.bytes)

     return [new Token(Type.tag, 42), new Token(Type.bytes, bytes.slice())]
   }
 }
})

const url = 'https://dido.prod.cid.contact/ingest/announce';
                    let stream = new Duplex();
                    stream.push(cbor);
                    stream.push(null);

                    const response = await fetch(url, {
                        method: 'PUT',
                        mode: 'same-origin',
                        headers: {
                            'Content-Type': 'application/cbor'
                        },
                        body: stream
                    })
```

notes:
* the cid is the cid of your advertisement
* the multiaddr needs to be generated in golang, becuase the js multiaddr library can't support multiaddr extensions. the extend command in the index-provider library has an example of how to generate these bytes.

5. You can confirm the records become available, at `cid.contact/providers`, and for the cid you used in the extend command, e.g. https://dido.prod.cid.contact/cid/bafkreiefztf2jizqd7qsgu4xhdjqbi3tkcrputiqbrjrtu7ywfgnsirgsu

6. The [w3rc library](https://github.com/ipfs-shipyard/w3rc/tree/feat/http) is an experimental retrieval client with support for these HTTP providers, and can `w3rc get -r bafkreiefztf2jizqd7qsgu4xhdjqbi3tkcrputiqbrjrtu7ywfgnsirgsu`