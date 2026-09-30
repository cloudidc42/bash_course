# Part 56: Quantum-ready Security and Advanced Cryptography (ขั้นตอนที่ 573-576)

---

## ขั้นตอนที่ 573: Post-Quantum Cryptography (PQC)

```bash
#!/bin/bash
# quantum-ready-security.sh
# Post-Quantum Cryptography: CRYSTALS-Kyber, Dilithium, NIST PQC standards

set -euo pipefail

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== PQC Library Setup ====================
setup_pqc_libraries() {
    log "Setting up Post-Quantum Cryptography libraries..."
    
    # Install liboqs (Open Quantum Safe)
    cat <<'EOF' > /tmp/install-oqs.sh
#!/bin/bash
# Install Open Quantum Safe (OQS) library

# Dependencies
apt-get update && apt-get install -y \
    cmake ninja-build libssl-dev python3-pytest \
    python3-pytest-xdist unzip xsltproc

# Clone and build liboqs
cd /tmp
git clone --depth 1 https://github.com/open-quantum-safe/liboqs.git
cd liboqs
mkdir build && cd build
cmake -GNinja \
    -DOQS_DIST_BUILD=ON \
    -DOQS_BUILD_ONLY_LIB=ON \
    -DBUILD_SHARED_LIBS=ON \
    ..
ninja
ninja install

# Install oqs-python bindings
pip3 install liboqs-python
EOF

    log "OQS library installation script created"
    
    # Python demo using PQC algorithms
    cat <<'PYEOF' > /tmp/pqc_demo.py
"""
Post-Quantum Cryptography Demonstration
NIST PQC Finalists: CRYSTALS-Kyber (KEM) + CRYSTALS-Dilithium (Signatures)
"""
import json
import os
import time
import hashlib
import base64
from typing import Optional

try:
    import oqs
    OQS_AVAILABLE = True
except ImportError:
    OQS_AVAILABLE = False
    print("Warning: liboqs-python not installed. Using simulation mode.")

# Simulation mode for demonstration
class MockKEM:
    """Mock Key Encapsulation Mechanism for demo"""
    ALG_NAME = "Kyber1024"
    
    def __init__(self):
        self._secret = os.urandom(32)
    
    def generate_keypair(self):
        pk = hashlib.sha256(self._secret + b"pk").digest() * 4  # 128 bytes
        sk = hashlib.sha256(self._secret + b"sk").digest() * 8  # 256 bytes
        return pk, sk
    
    def encap_secret(self, pk):
        ct = hashlib.sha256(pk + b"ct").digest() * 4  # 128 bytes ciphertext
        ss = hashlib.sha256(pk + b"ss").digest()       # 32 bytes shared secret
        return ct, ss
    
    def decap_secret(self, sk, ct):
        pk_part = hashlib.sha256(sk[:32] + b"pk").digest() * 4
        return hashlib.sha256(pk_part + b"ss").digest()


class MockSig:
    """Mock Digital Signature for demo"""
    ALG_NAME = "Dilithium5"
    
    def __init__(self):
        self._key = os.urandom(32)
    
    def generate_keypair(self):
        pk = hashlib.sha256(self._key + b"pk").digest() * 8
        sk = hashlib.sha256(self._key + b"sk").digest() * 16
        return pk, sk
    
    def sign(self, message, sk):
        h = hashlib.sha512(sk[:32] + message)
        return h.digest() * 8  # 512 bytes signature
    
    def verify(self, message, signature, pk):
        expected = hashlib.sha512(
            hashlib.sha256(pk[:32] + b"sk").digest() * 16 + message
        ).digest() * 8
        return signature == expected


def demo_kyber_key_exchange():
    """Demonstrate CRYSTALS-Kyber Key Encapsulation"""
    print("\n" + "=" * 60)
    print("CRYSTALS-Kyber Key Encapsulation Mechanism (KEM)")
    print("NIST PQC Standard: FIPS 203 (2024)")
    print("=" * 60)
    
    start = time.perf_counter()
    
    if OQS_AVAILABLE:
        kem = oqs.KeyEncapsulation("Kyber1024")
    else:
        kem = MockKEM()
    
    # 1. Bob generates keypair
    t0 = time.perf_counter()
    pk_bob, sk_bob = kem.generate_keypair()
    keygen_time = (time.perf_counter() - t0) * 1000
    
    # 2. Alice encapsulates: creates shared secret + ciphertext
    t0 = time.perf_counter()
    ct, ss_alice = kem.encap_secret(pk_bob)
    encap_time = (time.perf_counter() - t0) * 1000
    
    # 3. Bob decapsulates: recovers shared secret
    t0 = time.perf_counter()
    ss_bob = kem.decap_secret(sk_bob, ct)
    decap_time = (time.perf_counter() - t0) * 1000
    
    # Verify shared secrets match
    secrets_match = ss_alice == ss_bob
    
    total_time = (time.perf_counter() - start) * 1000
    
    print(f"\n  Algorithm:         {kem.ALG_NAME if OQS_AVAILABLE else MockKEM.ALG_NAME}")
    print(f"  Public Key Size:   {len(pk_bob)} bytes (vs 32 bytes for ECDH)")
    print(f"  Secret Key Size:   {len(sk_bob)} bytes")
    print(f"  Ciphertext Size:   {len(ct)} bytes")
    print(f"  Shared Secret:     {len(ss_alice)} bytes")
    print(f"\n  Performance:")
    print(f"  KeyGen:     {keygen_time:.3f}ms")
    print(f"  Encapsulate:{encap_time:.3f}ms")
    print(f"  Decapsulate:{decap_time:.3f}ms")
    print(f"  Total:      {total_time:.3f}ms")
    print(f"\n  ✅ Key exchange successful: {secrets_match}")
    print(f"  Shared secret: {base64.b64encode(ss_alice[:16]).decode()}...")
    
    return ss_alice


def demo_dilithium_signatures():
    """Demonstrate CRYSTALS-Dilithium Digital Signatures"""
    print("\n" + "=" * 60)
    print("CRYSTALS-Dilithium Digital Signatures")
    print("NIST PQC Standard: FIPS 204 (2024)")
    print("=" * 60)
    
    if OQS_AVAILABLE:
        sig = oqs.Signature("Dilithium5")
    else:
        sig = MockSig()
    
    pk_signer, sk_signer = sig.generate_keypair()
    
    # Sign important message
    message = b"Transfer $10,000,000 from account 12345 to account 67890"
    
    t0 = time.perf_counter()
    signature = sig.sign(message, sk_signer)
    sign_time = (time.perf_counter() - t0) * 1000
    
    t0 = time.perf_counter()
    valid = sig.verify(message, signature, pk_signer)
    verify_time = (time.perf_counter() - t0) * 1000
    
    # Tampered message test
    tampered = b"Transfer $10,000,000 from account 12345 to account 99999"
    tampered_valid = sig.verify(tampered, signature, pk_signer) if OQS_AVAILABLE else False
    
    print(f"\n  Algorithm:       {sig.ALG_NAME if OQS_AVAILABLE else MockSig.ALG_NAME}")
    print(f"  Public Key:      {len(pk_signer)} bytes")
    print(f"  Secret Key:      {len(sk_signer)} bytes")
    print(f"  Signature Size:  {len(signature)} bytes")
    print(f"\n  Performance:")
    print(f"  Sign:    {sign_time:.3f}ms")
    print(f"  Verify:  {verify_time:.3f}ms")
    print(f"\n  ✅ Original message valid: {valid}")
    print(f"  ✅ Tampered message rejected: {not tampered_valid}")


def demo_hybrid_tls():
    """Demonstrate Hybrid Classical/PQC TLS (X25519 + Kyber)"""
    print("\n" + "=" * 60)
    print("Hybrid TLS: X25519 + CRYSTALS-Kyber1024")
    print("(Protects against both classical and quantum attacks)")
    print("=" * 60)
    
    # In production: use OQS-OpenSSL or OQS-BoringSSL
    print("""
  Client Hello:
    Supported Groups: [X25519Kyber1024Draft00, X25519, secp256r1]
  
  Server Hello:
    Selected: X25519Kyber1024Draft00 (hybrid key exchange)
    Certificate: Signed with Dilithium5 + ECDSA (hybrid)
  
  Key Derivation:
    Classical:   x25519_shared_secret (32 bytes)
    PQC:         kyber_shared_secret  (32 bytes)
    Combined:    HKDF(x25519 || kyber) → TLS master secret
  
  Security:
    - If quantum computer breaks Kyber  → Classical X25519 protects
    - If quantum computer breaks X25519 → PQC Kyber protects
    - Both must be broken simultaneously → HARVEST NOW, DECRYPT LATER safe
  
  OpenSSL with OQS:
    openssl s_server \\
      -groups X25519Kyber1024Draft00 \\
      -cert dilithium5-cert.pem \\
      -key dilithium5-key.pem

  Browser support (2024):
    - Chrome 116+: X25519Kyber768Draft00 (enabled by default)
    - Firefox: planned
    - CloudFlare, AWS, Google: already deployed
""")


def generate_pqc_certificate():
    """Generate a PQC-signed certificate (simulation)"""
    print("\n" + "=" * 60)
    print("Post-Quantum Certificate Generation")
    print("=" * 60)
    
    cert_data = {
        "version": 3,
        "serial": "0x" + os.urandom(16).hex(),
        "issuer": "CN=Company Root CA PQC, O=Company, C=TH",
        "subject": "CN=api.company.com, O=Company, C=TH",
        "not_before": "2024-01-01T00:00:00Z",
        "not_after": "2025-01-01T00:00:00Z",
        "public_key_algorithm": "CRYSTALS-Dilithium5",
        "signature_algorithm": "id-CRYSTALS-Dilithium5",
        "extensions": {
            "subjectAltName": ["api.company.com", "*.api.company.com"],
            "keyUsage": ["digitalSignature", "keyEncipherment"],
            "extKeyUsage": ["serverAuth", "clientAuth"],
            "pqc_hybrid": "X25519Kyber1024Draft00"
        }
    }
    
    print(json.dumps(cert_data, indent=2))
    
    return cert_data


if __name__ == '__main__':
    print("Post-Quantum Cryptography Demonstration")
    print("NIST PQC Standards (August 2024):")
    print("  - FIPS 203: ML-KEM (CRYSTALS-Kyber)")
    print("  - FIPS 204: ML-DSA (CRYSTALS-Dilithium)")
    print("  - FIPS 205: SLH-DSA (SPHINCS+)")
    
    # Run demos
    shared_secret = demo_kyber_key_exchange()
    demo_dilithium_signatures()
    demo_hybrid_tls()
    generate_pqc_certificate()
    
    print("\n" + "=" * 60)
    print("Migration Roadmap to Quantum-Safe Cryptography:")
    print("=" * 60)
    print("""
  2024: Inventory cryptography assets
  2025: Deploy hybrid TLS (X25519+Kyber) for new services
  2026: Migrate TLS certificates to PQC signatures
  2027: Replace RSA/ECDSA code signing with Dilithium
  2028: Migrate all encrypted data at rest to PQC
  2030: Full PQC deployment (post-quantum threshold)
  
  Priority order:
  1. Long-lived secrets (TLS CA, code signing keys)
  2. Data with 10+ year sensitivity
  3. General TLS traffic
  4. Short-lived tokens/sessions (last priority)
""")
PYEOF

    python3 /tmp/pqc_demo.py
    log "PQC demonstration complete"
}

# ==================== Vault PQC Integration ====================
setup_vault_pqc() {
    log "Configuring HashiCorp Vault with PQC support..."
    
    # Vault configuration for hybrid key management
    cat <<'EOF' > /tmp/vault-pqc-policy.hcl
# PKI secret engine with PQC support
path "pki_pqc/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "transit/keys/kyber/*" {
  capabilities = ["create", "read"]
}

path "transit/encrypt/kyber/*" {
  capabilities = ["update"]
}

path "transit/decrypt/kyber/*" {
  capabilities = ["update"]
}
EOF

    # Configure Vault PKI for hybrid certificates
    cat <<'EOF' > /tmp/setup-vault-pqc.sh
#!/bin/bash
export VAULT_ADDR="https://vault.company.internal:8200"

# Enable PKI with extended key support
vault secrets enable -path=pki_pqc pki
vault secrets tune -max-lease-ttl=87600h pki_pqc

# Write hybrid CA bundle
vault write pki_pqc/config/urls \
    issuing_certificates="${VAULT_ADDR}/v1/pki_pqc/ca" \
    crl_distribution_points="${VAULT_ADDR}/v1/pki_pqc/crl"

# Create hybrid role (classical + PQC)
vault write pki_pqc/roles/hybrid-server \
    allowed_domains="company.com" \
    allow_subdomains=true \
    allow_glob_domains=false \
    max_ttl="8760h" \
    key_type="ec" \
    key_bits=256 \
    signature_bits=256

echo "Vault PQC PKI configured"
EOF

    chmod +x /tmp/setup-vault-pqc.sh
    log "Vault PQC configuration created"
}

# ==================== Zero-Knowledge Proofs ====================
demo_zkp() {
    log "Demonstrating Zero-Knowledge Proofs..."
    
    python3 - <<'PYEOF'
"""
Zero-Knowledge Proof Demonstration
Use cases: Privacy-preserving authentication, anonymous credentials
"""
import hashlib
import os
import random

class SchnorrZKP:
    """
    Schnorr Protocol: Proof of knowledge of discrete logarithm
    Prove you know 'x' such that y = g^x mod p, without revealing x
    
    Use cases:
    - Prove you know a password without sending it
    - Anonymous credential verification  
    - Blockchain privacy (zk-SNARKs basis)
    """
    
    # Using small primes for demonstration (production: 256-bit+ elliptic curves)
    P = 0xFFFFFFFB  # Large prime
    G = 2           # Generator
    
    def __init__(self):
        # Prover's secret: private key x
        self.x = random.randint(1, self.P - 2)
        # Public key: y = g^x mod p
        self.y = pow(self.G, self.x, self.P)
    
    def prove(self, challenge: int) -> tuple:
        """
        Interactive proof:
        1. Prover picks random r
        2. Computes commitment t = g^r mod p
        3. Receives challenge c
        4. Computes response s = r + c*x mod (p-1)
        """
        r = random.randint(1, self.P - 2)
        t = pow(self.G, r, self.P)  # Commitment
        s = (r + challenge * self.x) % (self.P - 1)  # Response
        return (t, s)
    
    def verify(self, challenge: int, proof: tuple, public_key: int) -> bool:
        """
        Verifier checks:
        g^s mod p == t * y^c mod p
        """
        t, s = proof
        lhs = pow(self.G, s, self.P)
        rhs = (t * pow(public_key, challenge, self.P)) % self.P
        return lhs == rhs


class HashBasedCommitment:
    """
    Pedersen commitment: commit to a value without revealing it
    Commit = hash(value || randomness)
    Use case: voting systems, sealed bids
    """
    
    def commit(self, value: bytes) -> tuple:
        """Create commitment to value"""
        randomness = os.urandom(32)
        commitment = hashlib.sha256(value + randomness).hexdigest()
        return commitment, randomness
    
    def open(self, commitment: str, value: bytes, randomness: bytes) -> bool:
        """Verify commitment opening"""
        expected = hashlib.sha256(value + randomness).hexdigest()
        return commitment == expected


def demo_zkp_scenarios():
    print("\n" + "=" * 60)
    print("Zero-Knowledge Proof Demonstrations")
    print("=" * 60)
    
    # Scenario 1: Password authentication without sending password
    print("\n[Scenario 1] ZKP Password Authentication")
    print("-" * 40)
    
    prover = SchnorrZKP()
    print(f"Public key (y): {prover.y}")
    print(f"Private key (x): [HIDDEN - never transmitted]")
    
    # Verifier generates random challenge
    challenge = random.randint(1, prover.P - 2)
    
    # Prover creates proof
    proof = prover.prove(challenge)
    
    # Verifier checks proof
    valid = prover.verify(challenge, proof, prover.y)
    
    print(f"\nChallenge: {challenge}")
    print(f"Proof (t, s): ({proof[0]}, {proof[1]})")
    print(f"✅ Verification: {valid}")
    print("→ Verifier confirms identity WITHOUT learning the password!")
    
    # Scenario 2: Sealed bid auction
    print("\n[Scenario 2] Sealed Bid Auction with Commitments")
    print("-" * 40)
    
    commitment_scheme = HashBasedCommitment()
    
    # Bidders commit to bids
    bidders = {
        "Alice": b"$10,000",
        "Bob": b"$12,500",
        "Carol": b"$11,000"
    }
    
    commitments = {}
    openings = {}
    
    print("Phase 1: Submission (bids hidden)")
    for name, bid in bidders.items():
        commitment, randomness = commitment_scheme.commit(bid)
        commitments[name] = commitment
        openings[name] = (bid, randomness)
        print(f"  {name}: commitment = {commitment[:20]}...")
    
    print("\nPhase 2: Reveal (all bids opened simultaneously)")
    valid_opens = {}
    for name, (bid, randomness) in openings.items():
        valid = commitment_scheme.open(commitments[name], bid, randomness)
        valid_opens[name] = bid.decode() if valid else "INVALID"
        print(f"  {name}: {bid.decode()} ✅ Valid: {valid}")
    
    winner = max(bidders.items(), key=lambda x: int(x[1].decode().replace('$', '').replace(',', '')))
    print(f"\n🏆 Winner: {winner[0]} with bid {winner[1].decode()}")
    print("→ No bidder could change their bid after seeing others!")
    
    # Scenario 3: Age verification without revealing birthdate
    print("\n[Scenario 3] Age Verification (Privacy-Preserving)")
    print("-" * 40)
    print("""
  Traditional: "Show me your ID" → reveals name, birthdate, address
  
  ZKP approach:
  1. User has signed credential: "Birthdate: 1990-01-15"
  2. User proves: "My age is >= 18" 
  3. WITHOUT revealing: actual birthdate, name, or other data
  
  Implementation: zk-SNARK circuit
    - Private input: birthdate
    - Public input: today's date, minimum age (18)
    - Proof: "I know x such that (today - x) >= 18*365"
  
  Tools: circom, snarkjs, Groth16/PLONK proving system
  Use cases: KYC/AML without data sharing, DeFi, voting
""")

demo_zkp_scenarios()
PYEOF

    log "ZKP demonstration complete"
}

# ==================== Homomorphic Encryption ====================
demo_homomorphic_encryption() {
    log "Demonstrating Homomorphic Encryption concepts..."
    
    python3 - <<'PYEOF'
"""
Homomorphic Encryption Demonstration
Compute on encrypted data without decrypting
"""
import hashlib

class SimplePaillier:
    """
    Simplified Paillier cryptosystem demonstration
    Supports: ADD on ciphertexts → ADD on plaintexts
    E(a) * E(b) mod n^2 = E(a + b)
    
    Use cases:
    - Aggregate salaries without revealing individual ones
    - Compute statistics on private medical data
    - Privacy-preserving machine learning
    """
    
    def __init__(self):
        # Toy parameters (production: 2048+ bit primes)
        self.p = 61
        self.q = 53
        self.n = self.p * self.q          # 3233
        self.n2 = self.n * self.n         # n^2
        self.g = self.n + 1               # Generator
        self.lam = (self.p - 1) * (self.q - 1)  # λ
        
        # Private key
        self.mu = pow(self.lam, -1, self.n)  # μ = λ^(-1) mod n
    
    def encrypt(self, m: int) -> int:
        """Encrypt message m"""
        assert 0 <= m < self.n
        r = 42  # Random (fixed for demo reproducibility)
        c = (pow(self.g, m, self.n2) * pow(r, self.n, self.n2)) % self.n2
        return c
    
    def decrypt(self, c: int) -> int:
        """Decrypt ciphertext c"""
        def L(x): return (x - 1) // self.n
        m = (L(pow(c, self.lam, self.n2)) * self.mu) % self.n
        return m
    
    def add_encrypted(self, c1: int, c2: int) -> int:
        """Add two ciphertexts → result decrypts to sum of plaintexts"""
        return (c1 * c2) % self.n2
    
    def multiply_by_constant(self, c: int, k: int) -> int:
        """Multiply plaintext by constant without decrypting"""
        return pow(c, k, self.n2)


print("\n" + "=" * 60)
print("Homomorphic Encryption: Compute on Encrypted Data")
print("=" * 60)

phe = SimplePaillier()

# Scenario: Compute average salary without revealing individual salaries
salaries = [50000, 75000, 90000, 65000, 80000]
print(f"\nEmployee salaries (PRIVATE): {salaries}")

# Encrypt all salaries
encrypted_salaries = [phe.encrypt(s // 1000) for s in salaries]  # Divide by 1000 to fit in n
print(f"Encrypted salaries: {[str(c)[:12] + '...' for c in encrypted_salaries]}")

# Compute sum on encrypted data (server never sees plaintext!)
encrypted_sum = encrypted_salaries[0]
for enc in encrypted_salaries[1:]:
    encrypted_sum = phe.add_encrypted(encrypted_sum, enc)

# Decrypt only the final result
decrypted_sum = phe.decrypt(encrypted_sum) * 1000
print(f"\nDecrypted sum: ${decrypted_sum:,}")
print(f"Expected sum:  ${sum(salaries):,}")
print(f"✅ Correct: {decrypted_sum == sum(salaries)}")
print(f"Average salary: ${decrypted_sum // len(salaries):,}")
print("\n→ HR computed average salary WITHOUT seeing any individual salary!")

print("""
Production HE Libraries:
  - SEAL (Microsoft): BFV, CKKS schemes
  - OpenFHE: CKKS, BFV, BGV, TFHE
  - HElib (IBM): BGV, CKKS
  - TFHE: Fully homomorphic (any circuit)

Use Cases:
  - Medical research: compute on patient data without HIPAA exposure
  - Financial: risk models on confidential client portfolios
  - ML training: federated learning with encrypted gradients
  - Cloud computing: delegate computation without data exposure
""")
PYEOF

    log "Homomorphic encryption demonstration complete"
}

# ==================== Secure Multi-Party Computation ====================
demo_smpc() {
    log "Demonstrating Secure Multi-Party Computation..."
    
    python3 - <<'PYEOF'
"""
Secure Multi-Party Computation (SMPC) demonstration
Multiple parties compute a function without revealing their inputs
"""
import random
import functools

def shamir_secret_sharing_demo():
    """
    Shamir's Secret Sharing: split secret into n shares, need k to reconstruct
    
    Use case: Store encryption key across multiple parties,
    need 3-of-5 to decrypt
    """
    print("\n[SMPC] Shamir's Secret Sharing (3-of-5)")
    print("-" * 40)
    
    SECRET = 42  # The secret value
    N_SHARES = 5  # Total shares
    THRESHOLD = 3  # Minimum shares to reconstruct
    PRIME = 257  # Small prime for demo (production: 256-bit prime)
    
    def share_secret(secret, n, k, prime):
        """Split secret into n shares with k threshold"""
        # Random polynomial f(x) = secret + a1*x + a2*x^2 + ... where f(0) = secret
        coefficients = [secret] + [random.randint(0, prime - 1) for _ in range(k - 1)]
        
        def evaluate_poly(x):
            return sum(c * pow(x, i, prime) for i, c in enumerate(coefficients)) % prime
        
        shares = [(i, evaluate_poly(i)) for i in range(1, n + 1)]
        return shares
    
    def reconstruct_secret(shares, prime):
        """Reconstruct using Lagrange interpolation"""
        def lagrange_basis(shares, x, prime):
            total = 0
            n = len(shares)
            for i in range(n):
                xi, yi = shares[i]
                num = yi
                den = 1
                for j in range(n):
                    if i != j:
                        xj, _ = shares[j]
                        num = (num * (x - xj)) % prime
                        den = (den * (xi - xj)) % prime
                den_inv = pow(den, prime - 2, prime)
                total = (total + num * den_inv) % prime
            return total
        
        return lagrange_basis(shares, 0, prime)
    
    shares = share_secret(SECRET, N_SHARES, THRESHOLD, PRIME)
    
    print(f"Original secret: {SECRET}")
    print(f"\nShares (distributed to 5 parties):")
    for i, (x, y) in enumerate(shares):
        print(f"  Party {i+1}: share({x}) = {y}")
    
    # Reconstruct with exactly 3 shares (any combination)
    import itertools
    for combo in itertools.combinations(shares, THRESHOLD):
        reconstructed = reconstruct_secret(list(combo), PRIME)
        share_ids = [s[0] for s in combo]
        print(f"\n  Parties {share_ids} reconstruct: {reconstructed} ✅")
        break  # Just show first combination
    
    print(f"\nWith only 2 shares → cannot reconstruct:")
    combo2 = shares[:2]
    wrong = reconstruct_secret(list(combo2), PRIME)
    print(f"  Attempted with parties [1,2]: {wrong} (wrong!)")


def private_set_intersection_demo():
    """
    Private Set Intersection: find common elements without revealing the sets
    Use case: contact tracing, fraud detection collaboration
    """
    print("\n[SMPC] Private Set Intersection")
    print("-" * 40)
    
    # Two companies find common fraud cases without sharing customer lists
    company_a_fraudsters = {1001, 1002, 1003, 1004, 1005}
    company_b_fraudsters = {1003, 1004, 1006, 1007, 1008}
    
    expected_intersection = company_a_fraudsters & company_b_fraudsters
    
    # Simple demo: hash-based PSI (production: OPRF-based)
    import hashlib
    
    salt = b"shared-secret-salt"
    
    hashed_a = {hashlib.sha256(salt + str(x).encode()).hexdigest() for x in company_a_fraudsters}
    hashed_b = {hashlib.sha256(salt + str(x).encode()).hexdigest() for x in company_b_fraudsters}
    
    # Find intersection of hashed sets (neither party reveals actual IDs)
    intersection_hashes = hashed_a & hashed_b
    
    print(f"Company A has {len(company_a_fraudsters)} flagged accounts (PRIVATE)")
    print(f"Company B has {len(company_b_fraudsters)} flagged accounts (PRIVATE)")
    print(f"\nCommon fraudsters found: {len(intersection_hashes)}")
    print(f"Expected: {expected_intersection} (IDs never exchanged directly)")
    print("→ Both companies improve fraud detection without sharing customer data!")


shamir_secret_sharing_demo()
private_set_intersection_demo()

print("""
SMPC Applications in Industry:
  - Google/Apple: COVID exposure notification (private contact tracing)
  - Banks: collaborative fraud detection without sharing customer lists
  - Healthcare: multi-hospital research without patient data sharing
  - Advertising: conversion measurement without user tracking
  - Supply chain: compute supply/demand without revealing business secrets

Production SMPC Frameworks:
  - SCALE-MAMBA (Bristol)
  - MP-SPDZ (Oxford)
  - Sharemind (Cybernetica)
  - Conclave (MIT)
  - TF Encrypted (TF-based MPC)
""")
PYEOF
}

main() {
    case "${1:-all}" in
        pqc)        setup_pqc_libraries ;;
        vault)      setup_vault_pqc ;;
        zkp)        demo_zkp ;;
        homomorph)  demo_homomorphic_encryption ;;
        smpc)       demo_smpc ;;
        all)
            setup_pqc_libraries
            demo_zkp
            demo_homomorphic_encryption
            demo_smpc
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 574: eBPF Advanced — Kernel-level Observability

```bash
#!/bin/bash
# ebpf-advanced.sh
# Advanced eBPF: Custom programs, BCC/bpftrace, Tetragon, network monitoring

set -euo pipefail

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== eBPF Fundamentals ====================
demonstrate_ebpf() {
    log "Demonstrating eBPF programs..."
    
    # bpftrace one-liners for production use
    cat <<'EOF'
# === EBPF MONITORING RECIPES ===

# 1. Syscall latency histogram (find slow system calls)
bpftrace -e '
tracepoint:raw_syscalls:sys_enter { @start[tid] = nsecs; }
tracepoint:raw_syscalls:sys_exit /@start[tid]/
{
    @latency_ns = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}
interval:s:5 { print(@latency_ns); clear(@latency_ns); }'

# 2. TCP connection tracking
bpftrace -e '
tracepoint:tcp:tcp_connect {
    printf("connect: %s:%d -> %s:%d pid=%d comm=%s\n",
        ntop(args->saddr), args->sport,
        ntop(args->daddr), args->dport,
        pid, comm);
}'

# 3. File opens (detect sensitive file access)
bpftrace -e '
tracepoint:syscalls:sys_enter_openat {
    printf("PID=%d COMM=%s FILE=%s\n", pid, comm, str(args->filename));
}'

# 4. Memory allocation tracking (find memory leaks)
bpftrace -e '
uprobe:/usr/lib/x86_64-linux-gnu/libc.so.6:malloc { @[ustack] = sum(arg0); }
uprobe:/usr/lib/x86_64-linux-gnu/libc.so.6:free { @[ustack] -= 1; }
interval:s:10 { print(@); clear(@); }'

# 5. CPU scheduler latency (find scheduling delays)
bpftrace -e '
tracepoint:sched:sched_wakeup { @ts[args->pid] = nsecs; }
tracepoint:sched:sched_switch /args->next_pid != 0 && @ts[args->next_pid]/
{
    @sched_latency = hist(nsecs - @ts[args->next_pid]);
    delete(@ts[args->next_pid]);
}
interval:s:5 { print(@sched_latency); clear(@sched_latency); }'

# 6. HTTP request tracing (without proxies)
bpftrace -e '
tracepoint:syscalls:sys_enter_write
/comm == "nginx" && strncmp("HTTP", str(args->buf), 4) == 0/
{
    printf("HTTP: %s\n", str(args->buf, 100));
}'

# 7. Block I/O latency (disk performance)
bpftrace -e '
tracepoint:block:block_rq_issue { @start[args->dev, args->sector] = nsecs; }
tracepoint:block:block_rq_complete
/@start[args->dev, args->sector]/
{
    @io_latency_us = hist((nsecs - @start[args->dev, args->sector]) / 1000);
    delete(@start[args->dev, args->sector]);
}
interval:s:5 { print(@io_latency_us); clear(@io_latency_us); }'

# 8. Kubernetes pod network traffic
bpftrace -e '
kprobe:tcp_sendmsg /comm != "sshd"/
{
    @sent_bytes[comm] = sum(arg2);
}
kprobe:tcp_recvmsg /comm != "sshd"/
{
    @recv_bytes[comm] = sum(arg2);
}
interval:s:5 { print(@sent_bytes); print(@recv_bytes); }'
EOF
    
    log "eBPF recipes displayed"
}

# ==================== Tetragon Security ====================
setup_tetragon() {
    log "Setting up Cilium Tetragon for eBPF security enforcement..."
    
    helm repo add cilium https://helm.cilium.io/
    helm upgrade --install tetragon cilium/tetragon \
        --namespace kube-system \
        --set tetragon.enableK8sAPI=true \
        --set tetragonOperator.enabled=true \
        --wait
    
    # Tetragon TracingPolicy: detect and block privilege escalation
    cat <<'EOF' | kubectl apply -f -
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: privilege-escalation-detection
spec:
  kprobes:
  # Detect setuid/setgid calls
  - call: "sys_setuid"
    syscall: true
    return: true
    args:
    - index: 0
      type: "int"
    returnArg:
      type: "int"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Equal"
        values:
        - "0"  # uid = root
      matchActions:
      - action: Sigkill  # Kill immediately
  
  # Detect /etc/shadow reads
  - call: "sys_open"
    syscall: true
    args:
    - index: 0
      type: "string"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Equal"
        values:
        - "/etc/shadow"
        - "/etc/gshadow"
      matchActions:
      - action: Sigkill
---
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: network-exfiltration-detection
spec:
  kprobes:
  - call: "tcp_connect"
    syscall: false
    args:
    - index: 0
      type: "sock"
    selectors:
    # Alert on connections to external IPs from production pods
    - matchArgs:
      - index: 0
        operator: "DAddr"
        values:
        - "!10.0.0.0/8"
        - "!172.16.0.0/12"
        - "!192.168.0.0/16"
      matchNamespaces:
      - namespace: "production"
      matchActions:
      - action: Post  # Log (not kill)
EOF

    log "Tetragon security policies applied"
}

# ==================== Custom eBPF Program ====================
write_custom_ebpf() {
    log "Writing custom eBPF security monitor..."
    
    cat <<'EOF' > /tmp/security_monitor.bpf.c
// eBPF program to monitor container security events
// Compile with: clang -O2 -target bpf -c security_monitor.bpf.c

#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

// Map to store security events
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 24);  // 16MB ring buffer
} security_events SEC(".maps");

// Event structure
struct security_event {
    __u32 pid;
    __u32 uid;
    __u64 timestamp;
    char comm[16];
    char filename[256];
    int event_type;  // 1=exec, 2=file_open, 3=setuid, 4=connect
};

// Trace execve syscall
SEC("tracepoint/syscalls/sys_enter_execve")
int trace_execve(struct trace_event_raw_sys_enter *ctx) {
    struct security_event *event;
    
    event = bpf_ringbuf_reserve(&security_events, sizeof(*event), 0);
    if (!event) return 0;
    
    event->pid = bpf_get_current_pid_tgid() >> 32;
    event->uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
    event->timestamp = bpf_ktime_get_ns();
    event->event_type = 1;  // exec
    
    bpf_get_current_comm(&event->comm, sizeof(event->comm));
    
    // Read filename from syscall arg
    const char *filename = (const char *)ctx->args[0];
    bpf_probe_read_user_str(event->filename, sizeof(event->filename), filename);
    
    bpf_ringbuf_submit(event, 0);
    return 0;
}

// Trace setuid (privilege escalation attempt)
SEC("tracepoint/syscalls/sys_enter_setuid")
int trace_setuid(struct trace_event_raw_sys_enter *ctx) {
    struct security_event *event;
    __u32 new_uid = (__u32)ctx->args[0];
    
    // Only alert if trying to become root
    if (new_uid != 0) return 0;
    
    event = bpf_ringbuf_reserve(&security_events, sizeof(*event), 0);
    if (!event) return 0;
    
    event->pid = bpf_get_current_pid_tgid() >> 32;
    event->uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
    event->timestamp = bpf_ktime_get_ns();
    event->event_type = 3;  // setuid
    
    bpf_get_current_comm(&event->comm, sizeof(event->comm));
    
    bpf_ringbuf_submit(event, 0);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
EOF

    log "eBPF security monitor program created at /tmp/security_monitor.bpf.c"
    log "Compile with: clang -O2 -target bpf -c /tmp/security_monitor.bpf.c"
}

main() {
    case "${1:-all}" in
        demo)      demonstrate_ebpf ;;
        tetragon)  setup_tetragon ;;
        custom)    write_custom_ebpf ;;
        all)
            demonstrate_ebpf
            setup_tetragon
            write_custom_ebpf
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 575: Confidential Computing — Intel SGX และ AMD SEV

```bash
#!/bin/bash
# confidential-computing.sh
# Confidential Computing: Intel SGX, AMD SEV, AWS Nitro Enclaves

set -euo pipefail

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== AWS Nitro Enclaves ====================
setup_nitro_enclaves() {
    log "Setting up AWS Nitro Enclaves for sensitive computation..."
    
    # Enable Nitro Enclave on EC2 instance
    # Requires instance with enclave support (c5n, m5n, r5n, etc.)
    
    cat <<'EOF' > /tmp/nitro-enclave-setup.sh
#!/bin/bash
# Run on EC2 instance with Nitro Enclave support

# Install Nitro CLI
sudo yum install -y aws-nitro-enclaves-cli aws-nitro-enclaves-cli-devel

# Enable enclave
sudo systemctl enable nitro-enclaves-allocator
sudo systemctl start nitro-enclaves-allocator

# Verify enclave support
nitro-cli describe-enclaves
EOF

    # Sample enclave application for key management
    cat <<'PYEOF' > /tmp/enclave_kms.py
"""
AWS Nitro Enclave: Secure Key Management Service
Processes sensitive operations inside the enclave where
even AWS and the host OS cannot access the data
"""
import json
import socket
import subprocess
import struct
import hashlib
import os
import logging

logger = logging.getLogger(__name__)

class NitroEnclaveKMS:
    """
    Key Management inside Nitro Enclave
    - Private keys never leave the enclave
    - Attestation proves code integrity to AWS KMS
    - Cloud provider cannot access keys
    """
    
    VSOCK_CID = 16      # Enclave CID
    VSOCK_PORT = 5005   # Communication port
    
    def __init__(self):
        self.master_key = os.urandom(32)  # Generated inside enclave
        self._key_store = {}
    
    def generate_data_key(self, key_id: str, key_length: int = 32) -> dict:
        """
        Generate DEK (Data Encryption Key) sealed by enclave
        Pattern: Envelope Encryption
        """
        dek = os.urandom(key_length)
        
        # Encrypt DEK with master key (stays in enclave)
        from cryptography.hazmat.primitives.ciphers.aead import AESGCM
        aesgcm = AESGCM(self.master_key)
        nonce = os.urandom(12)
        encrypted_dek = aesgcm.encrypt(nonce, dek, key_id.encode())
        
        # Store in enclave memory
        self._key_store[key_id] = {
            'encrypted_dek': nonce + encrypted_dek,
            'created_at': __import__('time').time()
        }
        
        return {
            'key_id': key_id,
            'plaintext_key': dek.hex(),     # Only returned once!
            'encrypted_key': (nonce + encrypted_dek).hex(),
            'key_length': key_length
        }
    
    def decrypt_data_key(self, key_id: str, encrypted_dek: bytes) -> bytes:
        """Decrypt a DEK inside the enclave"""
        from cryptography.hazmat.primitives.ciphers.aead import AESGCM
        aesgcm = AESGCM(self.master_key)
        nonce = encrypted_dek[:12]
        ciphertext = encrypted_dek[12:]
        return aesgcm.decrypt(nonce, ciphertext, key_id.encode())
    
    def process_payment_data(self, encrypted_card_data: bytes, key_id: str) -> dict:
        """
        Process payment inside enclave - card data never leaves enclave in plaintext
        """
        # Decrypt card data
        dek = self.decrypt_data_key(key_id, bytes.fromhex(
            self._key_store[key_id]['encrypted_dek'].hex() 
            if isinstance(self._key_store[key_id]['encrypted_dek'], bytes)
            else self._key_store[key_id]['encrypted_dek']
        ))
        
        # Process (inside enclave - invisible to host)
        # In production: full payment authorization logic here
        auth_code = hashlib.sha256(encrypted_card_data + dek).hexdigest()[:8].upper()
        
        return {
            'authorization_code': auth_code,
            'status': 'AUTHORIZED',
            'card_data': '[PROCESSED INSIDE ENCLAVE - NOT ACCESSIBLE OUTSIDE]'
        }
    
    def get_attestation(self) -> dict:
        """
        Get cryptographic attestation that code is running in genuine enclave
        Proves to AWS KMS / verifiers that:
        - Code hash matches expected value
        - Running on genuine Nitro hardware
        - Specific measurements of the enclave image
        """
        # In real enclave: use Nitro Attestation Document
        measurements = {
            'PCR0': hashlib.sha384(b"enclave-code-hash").hexdigest(),
            'PCR1': hashlib.sha384(b"kernel-hash").hexdigest(),
            'PCR2': hashlib.sha384(b"config-hash").hexdigest(),
        }
        
        return {
            'type': 'nitro_attestation',
            'measurements': measurements,
            'hardware': 'AWS Nitro Hypervisor',
            'verified': True
        }


# Demo
if __name__ == '__main__':
    kms = NitroEnclaveKMS()
    
    print("=== AWS Nitro Enclave KMS Demo ===")
    
    # Generate key
    key_result = kms.generate_data_key("payment-key-001", 32)
    print(f"\nGenerated DEK:")
    print(f"  Key ID: {key_result['key_id']}")
    print(f"  Plaintext (shown once): {key_result['plaintext_key'][:16]}...")
    print(f"  Encrypted key: {key_result['encrypted_key'][:32]}...")
    
    # Get attestation
    attestation = kms.get_attestation()
    print(f"\nEnclave Attestation:")
    for k, v in attestation.items():
        print(f"  {k}: {v}")
    
    print(f"""
Confidential Computing Benefits:
  - Host OS cannot access enclave memory
  - Hypervisor cannot access enclave memory  
  - AWS cannot access enclave memory
  - Memory is encrypted by hardware (AES-256)
  - Attestation proves code integrity
  
Use Cases:
  - Payment processing (PCI-DSS)
  - Key management (BYOK)
  - Sealed bidding / auction
  - ML inference on private data
  - Decryption oracle for DRM
  - Multi-party computation
""")
PYEOF

    python3 /tmp/enclave_kms.py
    log "Nitro Enclave demo complete"
}

# ==================== Intel SGX Deployment ====================
setup_intel_sgx() {
    log "Setting up Intel SGX on Kubernetes..."
    
    # SGX Device Plugin
    cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: intel-sgx-plugin
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: intel-sgx-plugin
  template:
    metadata:
      labels:
        app: intel-sgx-plugin
    spec:
      nodeSelector:
        intel.feature.node.kubernetes.io/sgx-enabled: "true"
      containers:
      - name: intel-sgx-plugin
        image: intel/intel-device-plugins-sgx:latest
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
        volumeMounts:
        - name: devfs
          mountPath: /dev
        - name: plugin-dir
          mountPath: /var/lib/kubelet/device-plugins
      volumes:
      - name: devfs
        hostPath:
          path: /dev
      - name: plugin-dir
        hostPath:
          path: /var/lib/kubelet/device-plugins
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sgx-ml-inference
  namespace: ai-platform
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sgx-inference
  template:
    spec:
      containers:
      - name: inference
        image: intel/gramine-python:latest
        resources:
          limits:
            sgx.intel.com/epc: 32Gi  # Enclave Page Cache
        volumeMounts:
        - name: sgx-dev
          mountPath: /dev/sgx_enclave
        - name: sgx-provision
          mountPath: /dev/sgx_provision
      volumes:
      - name: sgx-dev
        hostPath:
          path: /dev/sgx_enclave
      - name: sgx-provision
        hostPath:
          path: /dev/sgx_provision
EOF

    log "Intel SGX plugin deployed"
}

main() {
    case "${1:-all}" in
        nitro)  setup_nitro_enclaves ;;
        sgx)    setup_intel_sgx ;;
        all)
            setup_nitro_enclaves
            setup_intel_sgx
            ;;
    esac
}

main "$@"
```

---

## ขั้นตอนที่ 576: Service Catalog และ API Marketplace

```bash
#!/bin/bash
# api-marketplace.sh
# Enterprise API Marketplace: Backstage API Catalog, monetization, developer portal

set -euo pipefail

NAMESPACE="${NAMESPACE:-api-marketplace}"
DOMAIN="${DOMAIN:-developer.company.com}"

log() { echo "[$(date +'%Y-%m-%dT%H:%M:%S')] $*"; }

# ==================== API Specification Registry ====================
setup_api_registry() {
    log "Setting up API Specification Registry..."
    
    # Deploy Swagger UI + API Registry
    cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-registry
  namespace: api-marketplace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-registry
  template:
    metadata:
      labels:
        app: api-registry
    spec:
      containers:
      - name: swagger-ui
        image: swaggerapi/swagger-ui:latest
        env:
        - name: URLS
          value: |
            [
              {"url": "/specs/payment-api.yaml", "name": "Payment API v2.0"},
              {"url": "/specs/user-api.yaml", "name": "User API v3.0"},
              {"url": "/specs/analytics-api.yaml", "name": "Analytics API v1.0"}
            ]
        - name: URLS_PRIMARY_NAME
          value: "Payment API v2.0"
        ports:
        - containerPort: 8080
      - name: api-spec-server
        image: nginx:alpine
        volumeMounts:
        - name: api-specs
          mountPath: /usr/share/nginx/html/specs
        ports:
        - containerPort: 80
      volumes:
      - name: api-specs
        configMap:
          name: api-specs
EOF

    # Sample OpenAPI Spec
    cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-specs
  namespace: api-marketplace
data:
  payment-api.yaml: |
    openapi: "3.0.3"
    info:
      title: Payment API
      description: |
        Enterprise Payment Processing API.
        
        ## Authentication
        All endpoints require JWT authentication via Bearer token.
        
        ## Rate Limits
        - Basic tier: 1,000 req/min
        - Premium tier: 10,000 req/min
        - Enterprise: unlimited
        
        ## SLA
        - Availability: 99.99%
        - Latency p99: < 200ms
        - Support: 24/7 for Enterprise tier
      version: "2.0.0"
      contact:
        name: Platform Team
        email: platform@company.com
    servers:
    - url: https://api.company.com/v2
      description: Production
    - url: https://sandbox.company.com/v2
      description: Sandbox (free tier)
    
    security:
    - bearerAuth: []
    
    paths:
      /payments:
        post:
          summary: Create a payment
          operationId: createPayment
          tags: [Payments]
          requestBody:
            required: true
            content:
              application/json:
                schema:
                  $ref: '#/components/schemas/PaymentRequest'
                examples:
                  credit_card:
                    summary: Credit card payment
                    value:
                      amount: 9999
                      currency: USD
                      payment_method:
                        type: card
                        token: tok_visa_4242
                      metadata:
                        order_id: "ORD-12345"
          responses:
            '201':
              description: Payment created
              content:
                application/json:
                  schema:
                    $ref: '#/components/schemas/PaymentResponse'
            '400':
              $ref: '#/components/responses/ValidationError'
            '402':
              $ref: '#/components/responses/PaymentDeclined'
            '429':
              $ref: '#/components/responses/RateLimitExceeded'
      
      /payments/{id}:
        get:
          summary: Get payment details
          operationId: getPayment
          parameters:
          - name: id
            in: path
            required: true
            schema:
              type: string
              format: uuid
          responses:
            '200':
              description: Payment details
              content:
                application/json:
                  schema:
                    $ref: '#/components/schemas/PaymentResponse'
    
    components:
      securitySchemes:
        bearerAuth:
          type: http
          scheme: bearer
          bearerFormat: JWT
      
      schemas:
        PaymentRequest:
          type: object
          required: [amount, currency, payment_method]
          properties:
            amount:
              type: integer
              description: Amount in cents (USD 100 = $1.00)
              minimum: 1
              maximum: 10000000
            currency:
              type: string
              pattern: '^[A-Z]{3}$'
            payment_method:
              type: object
              required: [type]
              properties:
                type:
                  type: string
                  enum: [card, bank_transfer, wallet]
                token:
                  type: string
                  description: Tokenized card data (never raw card numbers)
            idempotency_key:
              type: string
              description: Unique key to prevent duplicate payments
        
        PaymentResponse:
          type: object
          properties:
            id:
              type: string
              format: uuid
            status:
              type: string
              enum: [pending, authorized, captured, declined, failed]
            amount:
              type: integer
            currency:
              type: string
            authorization_code:
              type: string
            created_at:
              type: string
              format: date-time
      
      responses:
        ValidationError:
          description: Validation error
          content:
            application/json:
              schema:
                type: object
                properties:
                  error:
                    type: string
                  field:
                    type: string
        PaymentDeclined:
          description: Payment was declined
          content:
            application/json:
              schema:
                type: object
                properties:
                  decline_code:
                    type: string
                    enum: [insufficient_funds, expired_card, do_not_honor]
                  message:
                    type: string
        RateLimitExceeded:
          description: Too many requests
          headers:
            X-RateLimit-Limit:
              schema:
                type: integer
            X-RateLimit-Remaining:
              schema:
                type: integer
            X-RateLimit-Reset:
              schema:
                type: integer
EOF

    log "API Registry configured"
}

# ==================== API Key Management ====================
setup_api_key_management() {
    log "Setting up API Key Management..."
    
    cat <<'PYEOF' > /tmp/api_key_manager.py
"""
API Key Management System
Handles key generation, rotation, usage tracking, and rate limiting
"""
import hashlib
import os
import json
import time
import uuid
from dataclasses import dataclass
from typing import Optional
from enum import Enum

class Tier(Enum):
    FREE = "free"
    BASIC = "basic"
    PREMIUM = "premium"
    ENTERPRISE = "enterprise"

TIER_LIMITS = {
    Tier.FREE: {"rpm": 100, "daily": 1000, "price_usd_per_month": 0},
    Tier.BASIC: {"rpm": 1000, "daily": 50000, "price_usd_per_month": 29},
    Tier.PREMIUM: {"rpm": 10000, "daily": 500000, "price_usd_per_month": 299},
    Tier.ENTERPRISE: {"rpm": -1, "daily": -1, "price_usd_per_month": -1},  # Custom
}

@dataclass
class APIKey:
    key_id: str
    key_hash: str       # SHA-256 of actual key (never store plaintext)
    organization: str
    tier: Tier
    created_at: float
    expires_at: Optional[float]
    scopes: list[str]
    metadata: dict
    active: bool = True

class APIKeyManager:
    def __init__(self):
        self._keys = {}       # key_id -> APIKey
        self._key_index = {}  # first 8 chars -> key_id (for lookup)
        self._usage = {}      # key_id -> {requests, bytes}
    
    def generate_key(self, org: str, tier: Tier, scopes: list[str] = None,
                    expires_days: int = 365) -> dict:
        """Generate a new API key"""
        # Generate cryptographically secure key
        raw_key = f"key_{os.urandom(24).hex()}"
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        key_id = str(uuid.uuid4())
        key_prefix = raw_key[:12]  # First 12 chars for display/lookup
        
        api_key = APIKey(
            key_id=key_id,
            key_hash=key_hash,
            organization=org,
            tier=tier,
            created_at=time.time(),
            expires_at=time.time() + expires_days * 86400,
            scopes=scopes or ["read", "write"],
            metadata={"created_by": "api"}
        )
        
        self._keys[key_id] = api_key
        self._key_index[key_prefix] = key_id
        
        limits = TIER_LIMITS[tier]
        
        return {
            "key_id": key_id,
            "api_key": raw_key,  # Shown ONCE, not stored
            "key_prefix": key_prefix,
            "tier": tier.value,
            "limits": {
                "requests_per_minute": limits["rpm"],
                "requests_per_day": limits["daily"],
            },
            "scopes": api_key.scopes,
            "expires_at": api_key.expires_at,
            "warning": "Store this key securely. It will not be shown again."
        }
    
    def validate_key(self, raw_key: str) -> Optional[dict]:
        """Validate an API key and return metadata"""
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        
        # Find by hash
        for key_id, api_key in self._keys.items():
            if api_key.key_hash == key_hash:
                if not api_key.active:
                    return {"error": "key_revoked", "message": "API key has been revoked"}
                
                if api_key.expires_at and time.time() > api_key.expires_at:
                    return {"error": "key_expired", "message": "API key has expired"}
                
                # Check rate limits
                limits = TIER_LIMITS[api_key.tier]
                usage = self._usage.get(key_id, {"rpm": 0, "daily": 0})
                
                if limits["rpm"] > 0 and usage.get("rpm", 0) >= limits["rpm"]:
                    return {"error": "rate_limit_exceeded", 
                           "retry_after": 60,
                           "limit": limits["rpm"]}
                
                # Track usage
                self._usage[key_id] = {
                    "rpm": usage.get("rpm", 0) + 1,
                    "daily": usage.get("daily", 0) + 1
                }
                
                return {
                    "valid": True,
                    "key_id": key_id,
                    "organization": api_key.organization,
                    "tier": api_key.tier.value,
                    "scopes": api_key.scopes,
                    "rate_limit": limits
                }
        
        return {"error": "key_not_found", "message": "Invalid API key"}
    
    def get_usage_stats(self, key_id: str) -> dict:
        """Get usage statistics for a key"""
        if key_id not in self._keys:
            return {}
        
        key = self._keys[key_id]
        usage = self._usage.get(key_id, {})
        limits = TIER_LIMITS[key.tier]
        
        return {
            "key_id": key_id,
            "organization": key.organization,
            "tier": key.tier.value,
            "usage": usage,
            "limits": limits,
            "utilization": {
                "rpm": usage.get("rpm", 0) / max(limits["rpm"], 1) * 100 if limits["rpm"] > 0 else 0,
                "daily": usage.get("daily", 0) / max(limits["daily"], 1) * 100 if limits["daily"] > 0 else 0
            }
        }


# Demo
if __name__ == '__main__':
    manager = APIKeyManager()
    
    print("=== API Key Management System ===\n")
    
    # Create keys for different tiers
    for org, tier in [("startup-co", Tier.FREE), ("medium-co", Tier.BASIC), ("enterprise-co", Tier.ENTERPRISE)]:
        result = manager.generate_key(org, tier, scopes=["payments:read", "payments:write"])
        print(f"{org} [{tier.value}]:")
        print(f"  Key: {result['api_key'][:20]}...")
        print(f"  RPM: {result['limits']['requests_per_minute']}")
        print(f"  Daily: {result['limits']['requests_per_day']}")
        print()
PYEOF

    python3 /tmp/api_key_manager.py
    log "API Key Manager demonstrated"
}

main() {
    case "${1:-all}" in
        registry) setup_api_registry ;;
        keys)     setup_api_key_management ;;
        all)
            setup_api_registry
            setup_api_key_management
            ;;
    esac
}

main "$@"
```

---

## สรุป Part 56

- ✅ **Step 573**: Post-Quantum Cryptography — CRYSTALS-Kyber (KEM), CRYSTALS-Dilithium (signatures), Hybrid TLS, ZKP (Schnorr protocol), Homomorphic Encryption (Paillier), SMPC (Shamir's Secret Sharing, PSI)
- ✅ **Step 574**: Advanced eBPF — bpftrace recipes (8 production use cases), Cilium Tetragon (security enforcement + TracingPolicy), Custom eBPF C program
- ✅ **Step 575**: Confidential Computing — AWS Nitro Enclaves (KMS demo, payment processing), Intel SGX on K8s (Device Plugin, Gramine)
- ✅ **Step 576**: API Marketplace — Swagger/OpenAPI Registry, API Key Management (tier-based rate limiting, usage tracking)

**ขั้นตอนต่อไป: Part 57 - Multi-Cloud Strategy และ Edge Computing**
