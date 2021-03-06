- Feature Name: ps-signatures
- Start Date: 2019-08-08
- RFC PR: (leave this empty)
- Ursa Issue: (leave this empty)
- Version: 0.1

# Summary
[summary]: #summary

Pointcheval-Sanders signature scheme from the paper [Short Randomizable signatures](https://eprint.iacr.org/2015/525) with protocol for proof of knowledge of signature. A user can get signature on known or committed messages from the signer and then prove knowledge of signature and optionally reveal some messages under the signature. Implementation in this [PR](https://github.com/hyperledger/ursa/pull/46). This feature depends on the [PR](https://github.com/hyperledger/ursa-rfcs/pull/10) for RFC for proof of knowledge of committed messages in a commitment. 

# Motivation
[motivation]: #motivation

Anonymous credentials consists of issuer's signature over credential attributes with the property that it is efficient to prove knowledge of signature without revealing the signature itself. Additionally, it should be possible to reveal some of the attributes of the credential to the verifier. Anonymous credentials are needed in Hyperledger Aries.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are 3 mainly entities in anonymous credentials, the *issuer* which signs a credential (a vector of attributes) and gives the signature to the *holder* which then uses the signature to prove its knowledge to a *verifier* without revealing the signature. The issuer might not know all the attributes it is signing and the holder might have only given a commitment to those attributes. In this case, the issuer gives a multi-message blind signature to the holder which the holder can unblind. The issuer owns a signing key and a verification key. The verification key is published so that it is accessible by the holder and verifier.  The verification key is meant to support a certain number of messages only meaning that a verification key intended to be used for verifying signature over a vector of 5 messages cannot be used to verify signature over 4 or 6 messages.  
- `SignatureGroup` and `OtherGroup` are 2 type aliases that can refer to groups G1 and G2 or groups G2 and G1 depending on whether feature `PS_Signature_G2` or `PS_Signature_G1` is enabled. These features are mutually exclusive. The `SignatureGroup` is the group for the elements of `Signature` or `Sigkey`. `Verkey` contains elements of both groups.
- All participants in the system have access to parameters from random oracle which are present in `Params`. The method accepts a label that is hashed to create parameters.
    ```rust
    let params: Params = Params::new("some label".as_bytes());
    ```
- The signer calls the `keygen` method to create a 2-tuple of signing key and verification key (`Verkey`, `Sigkey`). The method takes the number of messages to support and system parameters. The size of `Sigkey` and `Verkey` grows linearly with the number of messages that have to be supported.
    ```rust
    pub fn keygen(count_messages: usize, params: &Params) -> (Verkey, Sigkey)
    ```
- To create blind signatures, signer and user both need to have access to a blinding key `BlindingKey`. It is created by the signer using the signing key. The signer can omit creation (and publishing) of this key if he never intends to sign blind signatures.
    ```rust
    let (verkey, sigkey) = keygen(count_msgs, &params);
    let blinding_key:BlindingKey = BlindingKey::new(&sigkey, &params);
    ```
- The signer can create a new signature on a collection of known messages using `Signature::new`. The number of messages should be compatible with the provided `Sigkey`. Regardless of the number of messages being signed, the signature always consists of 2 group elements.
    ```rust
    impl Signature {
        pub fn new(
            messages: &[FieldElement],
            sigkey: &Sigkey,
            params: &Params,
        ) -> Result<Self, PSError>
    }
    ```
    Refer test [test_signature_all_known_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-6cb154fe72e95313cfe9016f86d71e73R122)
- The signer can create a new blind signature on a collection of known and blinded messages using `BlindSignature::new`. This method accepts a Pedersen commitment (`commitment` below) to the committed messages created by the holder and the collection of known messages. Note that the commitment could be for several messages. This method **must** only be used if there is one or more blinded messages.
    ```rust
    impl BlindSignature {
        pub fn new(
            commitment: &SignatureGroup,
            messages: &[FieldElement],
            sigkey: &Sigkey,
            blinding_key: &BlindingKey,
            params: &Params,
        ) -> Result<Signature, PSError>
    }
    ```
    The signer should verify that the commitment is well formed by executing a proof of knowledge protocol as described in the test [test_signature_blinded_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-d55856d1eb6560e1b74d9b6f2a45acabR200).
- Once a holder has received a blind signature created with the `BlindSignature::new` method, he unblinds the signature with `Signature::unblind`. For this the holder must have the randomness he used in the commitment he sent to the issuer.
    ```rust
    impl BlindSignature {
        pub fn unblind(sig: &Signature, blinding: &FieldElement) -> Signature
    }
    ```
- To verify a signature, `Signature::verify` must be used. It takes the messages over which the signature is created.
     ```rust
    impl Signature {
        pub fn verify(&self, messages: &[FieldElement], verkey: &Verkey, params: &Params) -> Result<bool, PSError>
    }
    ```
    Refer tests [test_signature_many_blinded_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-6cb154fe72e95313cfe9016f86d71e73R151) and [test_signature_known_and_blinded_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-6cb154fe72e95313cfe9016f86d71e73R171) for the unblinding an verification of signature.  
- To prove knowledge of the signature, `PoKOfSignature` is used. The protocol has 2 phases, in the pre-challenge phase, the signature is transformed to an aggregate signature and a PoK of the underlying multi-message is created using `PoKOfSignature::init`. `revealed_msg_indices` specifies the indices of messages that are revealed to the verifier and hence do not need to be included in the proof of knowledge.  
`blindings` specifies randomness to be used while proving knowledge of the unrevealed messages in the signature. If `blindings` is `None`, a random field element is generated corresponding to each unrevealed message. If `blindings` is to be provided, it must be provided for all unrevealed messages. This parameter is useful when the knowledge of same message is being proved in another relation as well and it needs to be proved that message is same in both relations, eg. If there are 2 signatures whose knowledge is being proved and it also needs to be proven that a specific message under both signatures is equal, then in both proof of knowledge protocols, the same blinding factor must be used for that message. The test `test_PoK_multiple_sigs_with_same_msg` demonstrates that. Another use case can be of proving that signature contains a message present in another Pedersen commitment.
    ```rust
    impl PoKOfSignature {
        pub fn init(
            sig: &Signature,
            vk: &Verkey,
            params: &Params,
            messages: &[FieldElement],
            blindings: Option<&[FieldElement]>,
            revealed_msg_indices: HashSet<usize>,
        ) -> Result<Self, PSError>
    }
    ```
- After the challenge is generated (or received in case of an interactive protocol), `PoKOfSignature::gen_proof` is used to create the proof `PoKOfSignatureProof`. 
    ```rust
    impl PoKOfSignature {
        pub fn gen_proof(self, challenge: &FieldElement) -> Result<PoKOfSignatureProof, PSError>
    }
    ```
- The verifier can then verify the proof using `PoKOfSignatureProof::verify`.
    ```rust
    impl PoKOfSignatureProof {
        pub fn verify(
            &self,
            vk: &Verkey,
            params: &Params,
            revealed_msgs: HashMap<usize, FieldElement>,
            challenge: &FieldElement,
        ) -> Result<bool, PSError>
    }
    ```
    Refer the tests [test_PoK_sig](https://github.com/hyperledger/ursa/pull/46/files#diff-d55856d1eb6560e1b74d9b6f2a45acabR252) and [test_PoK_sig_reveal_messages](https://github.com/hyperledger/ursa/pull/46/files#diff-d55856d1eb6560e1b74d9b6f2a45acabR276) for the proof of knowledge of signature and proof of knowledge of signature with revealing some messages respectively.
- Another test that shows a more elaborate scenario where a signature is requested on a mix of known and committed messages and then the knowledge of that signature is proved in addition to revealing some messages is shown in test [test_scenario_1](https://github.com/hyperledger/ursa/pull/46/files#diff-3b79c2d1f6accb7035421822bd62d6c6R10).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The default feature is `PS_Signature_G2` meaning signatures are in group G2. 
This makes signing expensive but proof of knowledge efficient since 
proof of knowledge of signatures will involve a multi-exponentiation in group G1. With feature `PS_Signature_G1`, signature is in group G1 but proof of knowledge of signatures will involve a multi-exponentiation in group G2 which is more expensive. The reason for making `PS_Signature_G2` default is that it is expected that signing is a less common occurence than proof of knowledge of signature. If signing was a equally common like one time use signature or it was critical to keep signatures short and signing more efficient, then feature `PS_Signature_G1` should be used. 
The implementation that the RFC is describing is the implementation of sections 4.2, 6.1 and 6.2 of the [PS signatures paper](https://eprint.iacr.org/2015/525) and an attempt has been made to use the same symbol names in the code as used in the paper.  
The signature scheme from section 4.2 does not allow blind signatures straightaway and the paper does not describe any technique to do so. But less efficient techniques from Coconut or others can be used. The scheme is implemented as described in the paper.  
The signature scheme from section 6.1 of the paper allows for signing blinded messages as well. 2 variation of scheme in section 6.1 are implemented, one of the variations follows the paper with the enhancement being the support of un-blinded (signer-known) messages and is seen in `BlindSignature::new_from_paper`. It works as:
- Say the verification key supports 5 messages, the relevant generators are Y<sub>1</sub>, Y<sub>2</sub>, Y<sub>3</sub>, Y<sub>4</sub>, Y<sub>5</sub>.
- The holder wants to reveal only last 3 message to the issuer but commits to the first 2 messages in a commitment C = g<sup>t</sup>Y<sub>1</sub><sup>m1</sup>Y<sub>2</sub><sup>m2</sup> (t is the randomness and should be preserved for unblinding the sig) and sends C to the issuer (along with a proof of knowledge).
- The issuer now creates D = C * Y<sub>3</sub><sup>m3</sup>Y<sub>4</sub><sup>m4</sup>Y<sub>5</sub><sup>m5</sup> where m3, m4 and m5 are known to him. The issuer will now choose a random u and create the signature (g<sup>u</sup>, (XD)<sup>u</sup>) as described in the paper.
- The holder will unblind the signature as described in the paper.

But another variations implemented with some modifications and is seen in ``BlindSignature::new`. The public key is split into 2 parts, the 
tilde elements (X_tilde and Y_tilde) and non-tilde elements (X, Y). Now the verifier only needs the former 
(tilde elements) and thus verifier's storage requirements go down. Keygen and signing are modified as:
- Keygen: Rather than only keeping X as the secret key, signer keeps x, y_1, y_2, ..y_r as secret key. 
The public key is unchanged, i.e. (g, Y_1, Y_2,..., Y_tilde_1, Y_tilde_2, ...)
- Sign: Lets say the signer wants to sign a multi-message of 10 messages where only 1 message is blinded. 
If we go by the paper where signer does not have y_1, y_2, .. y_10, signer will pick a random u and compute signature as 
(g^u, (XC)^u.Y_2^{m_2*u}.Y_3^{m_3*u}...Y_10^{m_10*u}), Y_1 is omitted as the first message was blinded. Of course the term 
(XC)^u.Y_2^{m_2*u}.Y_3^{m_3*u}...Y_10^{m_10*u} can be computed using efficient multi-exponentiation techniques but it would be more efficient 
if the signer could instead compute (g^u, C^u.g^{(x+y_2.m_2+y_3.m_3+...y_10.m_10).u}). The resulting signature will have the same form 
and can be unblinded in the same way as described in the paper.  
This will make signer's secret key storage a bit more but will make the signing becomes more efficient, especially in cases 
where the signature has only a few blinded messages but most messages are known to the signer which is usually the case with 
anonymous credentials where the user's secret key is blinded (its not known to signer) in the signature. This variation makes 
signing considerably faster unless the no of unblinded messages is very small compared to no of blinded messages. 
Run test `timing_comparison_for_both_blind_signature_schemes` to see the difference

Some implementation optimizations have been done as well like the use of multi-pairings or multi-exponentiations (both constant and variable time). The PoK protocol for Pedersen commitments used is described in this [RFC](https://github.com/hyperledger/ursa-rfcs/pull/10). PS signatures require type-3 pairings.

The `PoKOfSignature` is based on section 6.2 of the paper and the implementation follows the paper almost as-it-is. However, the product of multiple pairings is replaced with a multi-pairing for efficiency. The paper specifies the check as e(sigma<sub>1</sub>, X_tilde) * e(sigma<sub>1</sub>, Y_tilde<sub>1</sub>)<sup>m<sub>1</sub></sup>...e(sigma<sub>1</sub>, g_tilde)<sup>t</sup> == e(sigma<sub>2</sub>, g_tilde). This is transformed into the check e(sigma<sub>1</sub>, X_tilde * Y_tilde<sub>1</sub><sup>m<sub>1</sub></sup> * ...g_tilde<sup>t</sup>) == e(sigma<sub>2</sub>, g_tilde). This check is transformed to e(sigma<sub>1</sub>, X_tilde * Y_tilde<sub>1</sub><sup>m<sub>1</sub></sup> * ...g_tilde<sup>t</sup>) * e(sigma<sub>2</sub>, g_tilde)<sup>-1</sup> == 1. Now this can be combined into one multi-pairing. The proof of knowledge (symbol pie in the paper) is done using [this protocol](https://github.com/hyperledger/ursa/pull/46/files#diff-168ead191b84bf0856f3d153e9da3ba5R14). 


# Drawbacks
[drawbacks]: #drawbacks

This scheme is based on the LRSW Assumption which was introduced in [1] and proven secure in the generic group model in [2] but this assumption is not popular.

# Rationale and alternatives
[alternatives]: #alternatives

- This is meant to be one of the supported signature schemes for anonymous credentials. 
- BBS+ is another signature scheme to be supported but PS signatures can be used to build threshold signatures using Coconut or others.


# Prior art
[prior-art]: #prior-art

The RFC author is not aware of any implementation (open or closed source) of PS signatures. 

# Unresolved questions
[unresolved]: #unresolved-questions

- It is expected that proof of knowledge of signature API will be refined further through broader discussion.
- Proving properties about signed messages (range proof) or relationships among signed messages (equality, inequality) has been considered out of scope.
- Is the LRSW Assumption acceptable?

# Changelog
[changelog]: #changelog

- 


# References
[1] Anna Lysyanskaya, Ronald L. Rivest, Amit Sahai, and Stefan Wolf. Pseudonym systems. In Howard M.
Heys and Carlisle M. Adams, editors, SAC 1999, volume 1758 of LNCS, pages 184–199. Springer,
August 2000.  
[2] Victor Shoup. Lower bounds for discrete logarithms and related problems. In Walter Fumy, editor,
EUROCRYPT’97, volume 1233 of LNCS, pages 256–266. Springer, May 1997.
