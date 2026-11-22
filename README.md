# ASK, the AI Signature Key

Version 1. Alethieum, 2026-11-22. In use at Ando Infer since 2026-11-11.

An ASK is an API key that a wallet makes for itself. The wallet signs one fixed sentence, and the signature is the key. The seller checks the signature, and the wallet that signed it is the account. There is no sign-up, no dashboard, and no secret issued by the seller.

ASK puts together parts that already exist: EIP-191 signed messages, ERC-1271 and ERC-6492 for smart accounts, and an ordinary bearer key. What it adds is the rule that one signature, of a sentence that says what it allows, is the whole credential.

## 1. The sentence

A seller fixes one sentence, with three parts of its own: its name, what the key spends, and its domain. At Ando Infer it is:

```
Ando Infer API key for wallet {W}, made {T}. Anyone with this signature can spend the Ando Infer balance of this wallet. Send it only to infer.andoai.xyz.
```

- **{W}** is the wallet's address, `0x` and 40 hex digits, all lowercase or in EIP-55 mixed case. The sentence uses W exactly as it is written in the key.
- **{T}** is when the key was made, in UTC, as `YYYYMMDDThhmmssZ`, for example `20261122T090000Z`. It must be a real instant.
- The sentence is one line, with no trailing newline.

The wallet signs the sentence as an EIP-191 personal message (`personal_sign`, `cast wallet sign`, viem's `signMessage`). Any wallet that signs messages can make an ASK, with no new code.

## 2. The key

```
ando_{W}_{T}_{S}
```

- **ando** is the seller's prefix.
- **{S}** is the signature, hex, `0x` optional: 65 bytes for a plain key pair, longer for some smart accounts.

The sentence is not sent. The seller rebuilds it from W and T.

## 3. Checking a key

A seller accepts a key only if all of these hold:

1. **Shape.** The key has the four fields above, and W, T and S are well formed.
2. **Time.** T is no more than 60 seconds after the seller's clock.
3. **Signature.** The seller rebuilds the sentence from W and T and takes its EIP-191 digest. Then:
   - **A plain key pair:** S is 65 bytes, and `ecrecover(digest, S)` is W. A high-s signature is first normalised to its low-s form.
   - **A deployed smart account:** W has code, and `W.isValidSignature(digest, S)` returns the ERC-1271 magic value `0x1626ba7e`, as its exact first 4 bytes, followed by 28 zero bytes.
   - **A smart account not yet deployed:** S ends with the ERC-6492 suffix (`0x6492…6492`, 32 bytes), and the ERC-6492 validator, simulated with `eth_call`, accepts it.
4. **Newest wins.** The seller keeps, for each account, the newest T it has accepted. A key with an older T is refused. A key with a newer T replaces every older one the first time it is used.
5. **An account.** W has an account with the seller. How a wallet funds its account is the seller's choice. At Ando Infer, it is a plain USDC transfer on Arc from W, credited to W. The address to send to is in `GET /v1`.

A seller may cache a passed check. A plain key pair's answer never changes. A smart account's answer can change with chain state, so cache it for a bounded time.

## 4. Ending a key

There is no revoke endpoint. To end a key, the wallet signs the sentence again with the current time and uses the new key once. Every older key is then refused. Do this when a key may have leaked, or when a smart account changes its owner or signer.

## 5. Moving an account

A wallet W can give its account to a new wallet W2. W2 signs its own key, and W signs a move sentence. At Ando Infer it is:

```
Ando Infer move of wallet {W} to wallet {W2}, made {T}. This gives wallet {W2} the Ando Infer balance and history of wallet {W}, for good. Send it only to infer.andoai.xyz.
```

The move key is W2's key with the move appended:

```
ando_{W2}_{T}_{S}_{W}_{M}
```

- **S** is W2's signature of its own key sentence at T.
- **M** is W's signature of the move sentence at the same T.

The seller checks both signatures as in section 3. It refuses the move if:
- W2 already has an account;
- W2 is a smart account that is not deployed yet, since only a deployed wallet's code can prove who controls it;
- the seller's own conditions are not met. At Ando Infer, those are a balance above zero and no request running.

After a move, W's keys are refused, and W2's key is the account's key.

## 6. Security

- **An ASK is a bearer key.** The sentence says so: anyone with the signature can spend the balance. Keep it as secret as any API key. A seller should never store or log the key itself, only a hash of it.
- **The domain is in the sentence.** A key for one seller is a signature of a different sentence from a key for another seller, so it is useless anywhere else. Wallet software does not check that domain, as it does for Sign-In with Ethereum. The signer must read the sentence, and should sign it only for the seller it names.
- **Checking a smart account shows the key to whoever runs the check.** ERC-1271 and ERC-6492 checks send the digest and the signature to a node. If that node is run by someone else, it sees enough to rebuild the key. A seller should run these checks on a node it controls, or in its own EVM.
- **Keys don't expire on their own.** A key lasts until a newer one is used. Section 4 is how it ends.
- **Clock.** T is the signer's clock. The 60-second allowance covers small drift. A key made far ahead of time is refused until its time comes.

## 7. Related work

- **Sign-In with Ethereum** (EIP-4361) proves control of a wallet to open a session. It needs a server nonce and defines no key or balance.
- **ERC-8128** (draft) signs each HTTP request with an Ethereum account. ASK signs once, and that signature is a lasting key.
- **Bearer tokens** (RFC 6750) are what API keys are today. They are issued by the provider and given to the customer. An ASK is the same kind of key, made by the customer's own wallet.

## 8. Test vectors

`vectors.json` has signed examples: a key with an EIP-55 wallet, the same key with a lowercase wallet, a move, a wrong signature, and a key made in the future. They use Foundry's published development accounts, which are public test values: never fund them. All of them were checked against Ando Infer's live verifier on 2026-11-22.

## Licence

This specification is released under CC0 1.0. Anyone may use, copy and build on it.
