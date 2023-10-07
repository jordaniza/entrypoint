## Entrypoint

Biconomy errors often throw based on the Entrypoint contract, by default this is at 0x5ff137d4b0fdcd49dca30c7cf57e578a026d2789

## Error tracing for AA25:

To understand error AA25 we need to trace the call flow:

1: Call `handleOps`:

```js
function handleOps(
    UserOperation[] calldata ops,
    address payable beneficiary
) public nonReentrant {
    uint256 opslen = ops.length;
    UserOpInfo[] memory opInfos = new UserOpInfo[](opslen);

    unchecked {
        for (uint256 i = 0; i < opslen; i++) {
            UserOpInfo memory opInfo = opInfos[i];
            (
                uint256 validationData,
                uint256 pmValidationData
            ) = _validatePrepayment(i, ops[i], opInfo);
```

2: Call `_validatePrepayment`:
(Our error is thrown here)

```js
function _validatePrepayment(
    uint256 opIndex,
    UserOperation calldata userOp,
    UserOpInfo memory outOpInfo
)
    private
    returns (uint256 validationData, uint256 paymasterValidationData)
{
    uint256 preGas = gasleft();
    MemoryUserOp memory mUserOp = outOpInfo.mUserOp;
    _copyUserOpToMemory(userOp, mUserOp);
    outOpInfo.userOpHash = getUserOpHash(userOp);

    // validate all numeric values in userOp are well below 128 bit, so they can safely be added
    // and multiplied without causing overflow
    uint256 maxGasValues = mUserOp.preVerificationGas |
        mUserOp.verificationGasLimit |
        mUserOp.callGasLimit |
        userOp.maxFeePerGas |
        userOp.maxPriorityFeePerGas;
    require(maxGasValues <= type(uint120).max, "AA94 gas values overflow");

    uint256 gasUsedByValidateAccountPrepayment;
    uint256 requiredPreFund = _getRequiredPrefund(mUserOp);
    (
        gasUsedByValidateAccountPrepayment,
        validationData
    ) = _validateAccountPrepayment(
        opIndex,
        userOp,
        outOpInfo,
        requiredPreFund
    );

    if (!_validateAndUpdateNonce(mUserOp.sender, mUserOp.nonce)) {
        revert FailedOp(opIndex, "AA25 invalid account nonce");
    }
```

3: Inspecting `_validateAndUpdateNonce`:

```js
function _validateAndUpdateNonce(
    address sender,
    uint256 nonce
) internal returns (bool) {
    uint192 key = uint192(nonce >> 64);
    uint64 seq = uint64(nonce);
    return nonceSequenceNumber[sender][key]++ == seq;
}
```

The error therefore is a result of the nonce passed not being equal to the nonce stored in the nonceSequenceNumber mapping, for a given sender (plus 1).

Relevant params:

- `sender` comes from `mUserOp.sender`
- `nonce` comes from `mUserOp.nonce`
- `mUserOp` is passed to \_validatePrepayment as `UserOpInfo memory outOpInfo`
- `opInfo` is a bit strange, we create a memory array of `UserOpInfo` called `opInfos`. This is of length opsLen based on the number of ops, however it is initialized so will contrain default struct values. What I am guessing is that the is opInfo is populated during runtime.

Looks like here:

```js
_copyUserOpToMemory(userOp, mUserOp);
```

We take the actual fetched userOp data and copy it into mUserOp.

Therefore, what we really need is to check the nonce and the sender of what we are calling in ops.
We can then check that versus the public nonceSequenceNumber mapping.

Weirdly, this issue seems to resolve itself over time.

Going to update and contact the team.
