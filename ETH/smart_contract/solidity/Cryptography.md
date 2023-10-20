# Cryptography

## Ecrecover

### Convert signature bytes to (r, s, v)

```solidity
// Input: 0x4fde044566e1288a60cebbc94458b2da86940d78ea2a92a993232726fb2ce78f797aef5eb003a2ca56ffec9e9cefe28f5666c352da36306c8ab807ac02d2f2441c
// Output:
// r: 0x4fde044566e1288a60cebbc94458b2da86940d78ea2a92a993232726fb2ce78f
// s: 0x797aef5eb003a2ca56ffec9e9cefe28f5666c352da36306c8ab807ac02d2f244
// v: 28
function convertSigBytes2RSV(bytes memory sig) public pure returns(bytes32 r, bytes32 s, uint8 v) {
    assembly {
        r := mload(add(sig, 32))
        s := mload(add(sig, 64))
        v := and(mload(add(sig, 65)), 255)
    }
    if (v < 27) v += 27;
}

// Output: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8, true
function validation() public pure returns(address, bool) {
    bytes32 hash = 0xefcd4275fee6aa3a134bcb710d7906e5cb525d6b111877ff7fb620e00110ba48;
    bytes memory sig = hex"4fde044566e1288a60cebbc94458b2da86940d78ea2a92a993232726fb2ce78f797aef5eb003a2ca56ffec9e9cefe28f5666c352da36306c8ab807ac02d2f2441c";
    (bytes32 r, bytes32 s, uint8 v) = convertSigBytes2RSV(sig);
    address addr = ecrecover(hash, v, r, s);
    return (addr, addr == address(0x70997970C51812dc3A010C7d01b50e0d17dc79C8));
}
```
