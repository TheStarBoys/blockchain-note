# How to sign data

## Personal Sign

```typescript
import {
    personalSign,
    recoverPersonalSignature,
    MsgParams,
    SignedMsgParams,
    TypedData
} from 'eth-sig-util'

const ethUtil = require('ethereumjs-util')

let privateKey = ethUtil.toBuffer('0x9a01f5c57e377e0239e6036b7b2d700454b760b2dab51390f1eeb2f64fe98b68')

let data = 'hello'
let msg: MsgParams<TypedData> = {
    data: data
}

let signature = personalSign(privateKey, msg)
console.log('signature: ', signature)

let signedMsg:SignedMsgParams<TypedData> = {
    data: msg.data,
    sig: signature
}

let address = recoverPersonalSignature(signedMsg)
console.log('address: ', address)
// Output: 
// signature:  0xa6fb700f874d5a293ce7ca1f7850cc2fcff5a5e79e65ee901428802c1bf9762e01395ce53844bd9b9dfb79d96bb9a558d257d21171f958a00d573d970a2387391c
// address:  0x11a449ed8eadacda0a290ad0ffee174fdf0b3f7a
```

