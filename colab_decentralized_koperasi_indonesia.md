# Step-by-Step Code (Google Colab): Build **Decentralized Koperasi Indonesia**

Dokumen ini berisi alur end-to-end untuk membuat MVP **Koperasi Desentralisasi** di Google Colab:
1. Menulis smart contract koperasi (Solidity).
2. Compile contract di Colab.
3. Deploy ke testnet (Polygon Amoy) dari Colab.
4. Simulasi anggota: daftar, setor simpanan, ajukan pinjaman, voting, cairkan pinjaman.

> ⚠️ Untuk produksi, wajib audit keamanan smart contract, KYC/AML, dan kepatuhan regulasi Indonesia.

---

## 0) Arsitektur MVP

Konsep sederhana koperasi:
- **Admin/Pengurus** membuat proposal pinjaman.
- **Anggota** daftar lalu menyetor simpanan (pool kas koperasi).
- Proposal pinjaman disetujui via **voting 1 anggota = 1 suara**.
- Jika suara setuju mayoritas, peminjam bisa mencairkan pinjaman dari pool.

---

## 1) Buka Google Colab dan siapkan environment

### Cell 1 — Install dependensi
```python
!pip -q install web3 py-solc-x eth-account python-dotenv
```

### Cell 2 — Konfigurasi compiler Solidity
```python
from solcx import install_solc, set_solc_version

install_solc("0.8.20")
set_solc_version("0.8.20")
print("Solidity compiler ready")
```

---

## 2) Tulis smart contract Koperasi

### Cell 3 — Kode Solidity
```python
koperasi_contract = r'''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract KoperasiDAO {
    struct Member {
        bool isRegistered;
        uint256 savings;
    }

    struct LoanProposal {
        uint256 id;
        address borrower;
        uint256 amount;
        string reason;
        uint256 yesVotes;
        uint256 noVotes;
        uint256 deadline;
        bool executed;
        bool approved;
    }

    address public admin;
    uint256 public totalPool;
    uint256 public proposalCount;

    mapping(address => Member) public members;
    mapping(uint256 => LoanProposal) public proposals;
    mapping(uint256 => mapping(address => bool)) public hasVoted;

    event MemberRegistered(address member);
    event Deposit(address member, uint256 amount);
    event LoanProposed(uint256 proposalId, address borrower, uint256 amount, string reason, uint256 deadline);
    event Voted(uint256 proposalId, address voter, bool support);
    event ProposalFinalized(uint256 proposalId, bool approved);
    event LoanDisbursed(uint256 proposalId, address borrower, uint256 amount);

    modifier onlyAdmin() {
        require(msg.sender == admin, "Only admin");
        _;
    }

    modifier onlyMember() {
        require(members[msg.sender].isRegistered, "Not a member");
        _;
    }

    constructor() {
        admin = msg.sender;
    }

    function registerMember(address _member) external onlyAdmin {
        require(!members[_member].isRegistered, "Already registered");
        members[_member].isRegistered = true;
        emit MemberRegistered(_member);
    }

    function depositSavings() external payable onlyMember {
        require(msg.value > 0, "Deposit must be > 0");
        members[msg.sender].savings += msg.value;
        totalPool += msg.value;
        emit Deposit(msg.sender, msg.value);
    }

    function createLoanProposal(address _borrower, uint256 _amount, string calldata _reason, uint256 _votingDurationSeconds)
        external
        onlyAdmin
    {
        require(members[_borrower].isRegistered, "Borrower must be member");
        require(_amount > 0, "Amount must be > 0");
        require(_amount <= totalPool, "Insufficient pool");
        require(_votingDurationSeconds >= 60, "Voting too short");

        proposalCount += 1;
        proposals[proposalCount] = LoanProposal({
            id: proposalCount,
            borrower: _borrower,
            amount: _amount,
            reason: _reason,
            yesVotes: 0,
            noVotes: 0,
            deadline: block.timestamp + _votingDurationSeconds,
            executed: false,
            approved: false
        });

        emit LoanProposed(proposalCount, _borrower, _amount, _reason, block.timestamp + _votingDurationSeconds);
    }

    function vote(uint256 _proposalId, bool _support) external onlyMember {
        LoanProposal storage p = proposals[_proposalId];
        require(p.id != 0, "Proposal not found");
        require(block.timestamp < p.deadline, "Voting closed");
        require(!hasVoted[_proposalId][msg.sender], "Already voted");

        hasVoted[_proposalId][msg.sender] = true;

        if (_support) {
            p.yesVotes += 1;
        } else {
            p.noVotes += 1;
        }

        emit Voted(_proposalId, msg.sender, _support);
    }

    function finalizeProposal(uint256 _proposalId) external onlyAdmin {
        LoanProposal storage p = proposals[_proposalId];
        require(p.id != 0, "Proposal not found");
        require(block.timestamp >= p.deadline, "Voting not ended");
        require(!p.executed, "Already finalized");

        p.executed = true;
        p.approved = p.yesVotes > p.noVotes;

        emit ProposalFinalized(_proposalId, p.approved);
    }

    function disburseLoan(uint256 _proposalId) external {
        LoanProposal storage p = proposals[_proposalId];
        require(p.id != 0, "Proposal not found");
        require(p.executed, "Proposal not finalized");
        require(p.approved, "Proposal not approved");
        require(msg.sender == p.borrower, "Only borrower can disburse");
        require(p.amount <= address(this).balance, "Insufficient contract balance");
        require(p.amount <= totalPool, "Insufficient recorded pool");

        uint256 amount = p.amount;
        p.amount = 0; // prevent re-entrancy style repeated disburse
        totalPool -= amount;

        (bool ok, ) = payable(msg.sender).call{value: amount}("");
        require(ok, "Transfer failed");

        emit LoanDisbursed(_proposalId, msg.sender, amount);
    }

    function getProposal(uint256 _proposalId) external view returns (LoanProposal memory) {
        return proposals[_proposalId];
    }
}
'''

print("Contract source ready")
```

---

## 3) Compile contract di Colab

### Cell 4 — Compile ABI + Bytecode
```python
from solcx import compile_source

compiled = compile_source(
    koperasi_contract,
    output_values=["abi", "bin"]
)

contract_id, contract_interface = compiled.popitem()
abi = contract_interface["abi"]
bytecode = contract_interface["bin"]

print("Compiled:", contract_id)
print("ABI methods:", len(abi))
```

---

## 4) Deploy ke Polygon Amoy (testnet)

### Cell 5 — Set RPC dan private key
> Simpan private key khusus testnet. Jangan gunakan wallet utama.

```python
from google.colab import userdata
from web3 import Web3

# Di Colab: buka panel Secrets lalu isi KEY ini
PRIVATE_KEY = userdata.get("PRIVATE_KEY")
RPC_URL = "https://rpc-amoy.polygon.technology"

w3 = Web3(Web3.HTTPProvider(RPC_URL))
assert w3.is_connected(), "Gagal konek RPC"

account = w3.eth.account.from_key(PRIVATE_KEY)
DEPLOYER = account.address
print("Connected. Deployer:", DEPLOYER)
```

### Cell 6 — Deploy transaction
```python
Koperasi = w3.eth.contract(abi=abi, bytecode=bytecode)
nonce = w3.eth.get_transaction_count(DEPLOYER)

tx = Koperasi.constructor().build_transaction({
    "from": DEPLOYER,
    "nonce": nonce,
    "gas": 3_000_000,
    "gasPrice": w3.to_wei("40", "gwei"),
    "chainId": 80002,  # Polygon Amoy
})

signed = w3.eth.account.sign_transaction(tx, PRIVATE_KEY)
tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
print("Deploy tx:", tx_hash.hex())

receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
contract_address = receipt.contractAddress
print("Contract deployed at:", contract_address)

koperasi = w3.eth.contract(address=contract_address, abi=abi)
```

---

## 5) Simulasi operasi koperasi

Agar bisa simulasi multi-user, siapkan 2 wallet anggota (private key testnet) di Colab Secrets:
- `MEMBER1_KEY`
- `MEMBER2_KEY`

### Cell 7 — Load akun anggota
```python
MEMBER1_KEY = userdata.get("MEMBER1_KEY")
MEMBER2_KEY = userdata.get("MEMBER2_KEY")

member1 = w3.eth.account.from_key(MEMBER1_KEY)
member2 = w3.eth.account.from_key(MEMBER2_KEY)

print("Member1:", member1.address)
print("Member2:", member2.address)
```

### Helper Cell — Fungsi kirim tx
```python
def send_tx(fn, sender_addr, sender_key, value=0, gas=300000):
    nonce = w3.eth.get_transaction_count(sender_addr)
    tx = fn.build_transaction({
        "from": sender_addr,
        "nonce": nonce,
        "value": value,
        "gas": gas,
        "gasPrice": w3.to_wei("40", "gwei"),
        "chainId": 80002,
    })
    signed = w3.eth.account.sign_transaction(tx, sender_key)
    tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
    rcpt = w3.eth.wait_for_transaction_receipt(tx_hash)
    return tx_hash.hex(), rcpt
```

### Cell 8 — Register anggota (oleh admin)
```python
print(send_tx(koperasi.functions.registerMember(member1.address), DEPLOYER, PRIVATE_KEY)[0])
print(send_tx(koperasi.functions.registerMember(member2.address), DEPLOYER, PRIVATE_KEY)[0])
```

### Cell 9 — Anggota setor simpanan
```python
# member1 setor 0.02 MATIC
print(send_tx(koperasi.functions.depositSavings(), member1.address, MEMBER1_KEY, value=w3.to_wei(0.02, "ether"))[0])
# member2 setor 0.03 MATIC
print(send_tx(koperasi.functions.depositSavings(), member2.address, MEMBER2_KEY, value=w3.to_wei(0.03, "ether"))[0])

print("Total pool:", w3.from_wei(koperasi.functions.totalPool().call(), "ether"), "MATIC")
```

### Cell 10 — Buat proposal pinjaman
```python
# Admin ajukan proposal pinjaman untuk member1 sebesar 0.01 MATIC, voting 5 menit
print(send_tx(
    koperasi.functions.createLoanProposal(member1.address, w3.to_wei(0.01, "ether"), "Modal usaha warung anggota", 300),
    DEPLOYER,
    PRIVATE_KEY
)[0])

proposal = koperasi.functions.getProposal(1).call()
print("Proposal #1:", proposal)
```

### Cell 11 — Voting anggota
```python
print("Member1 vote YES:", send_tx(koperasi.functions.vote(1, True), member1.address, MEMBER1_KEY)[0])
print("Member2 vote YES:", send_tx(koperasi.functions.vote(1, True), member2.address, MEMBER2_KEY)[0])
```

### Cell 12 — Finalize proposal (setelah deadline)
```python
import time
print("Tunggu deadline voting selesai...")
time.sleep(310)  # untuk demo sederhana di Colab

print("Finalize:", send_tx(koperasi.functions.finalizeProposal(1), DEPLOYER, PRIVATE_KEY)[0])
print("Proposal final:", koperasi.functions.getProposal(1).call())
```

### Cell 13 — Cairkan pinjaman oleh borrower
```python
balance_before = w3.eth.get_balance(member1.address)
print("Borrower balance before:", w3.from_wei(balance_before, "ether"), "MATIC")

print("Disburse:", send_tx(koperasi.functions.disburseLoan(1), member1.address, MEMBER1_KEY)[0])

balance_after = w3.eth.get_balance(member1.address)
print("Borrower balance after:", w3.from_wei(balance_after, "ether"), "MATIC")
print("Pool sisa:", w3.from_wei(koperasi.functions.totalPool().call(), "ether"), "MATIC")
```

---

## 6) Checklist agar benar-benar “decentralized”

Untuk naik level dari MVP ke produksi:
- Ganti `onlyAdmin` pada `registerMember` & `createLoanProposal` ke governance proposal (semua anggota bisa usul).
- Tambah **token membership** atau reputasi anggota.
- Tambah mekanisme cicilan + bunga syariah/konvensional sesuai model koperasi.
- Integrasi identitas (KYC) dan legal docs off-chain (IPFS + hash on-chain).
- Gunakan multisig treasury (misalnya Safe) agar dana tidak bergantung 1 admin key.
- Buat front-end (Next.js + wagmi + viem) untuk UX anggota.

---

## 7) Masalah umum & solusi cepat

- **`insufficient funds for gas * price + value`**
  - Isi faucet Amoy ke semua akun (deployer + member).
- **`replacement transaction underpriced`**
  - Naikkan `gasPrice` atau refresh nonce.
- **`Voting not ended`**
  - Tunggu deadline, atau saat development kurangi durasi voting.
- **RPC timeout**
  - Ulangi request, atau gunakan RPC provider alternatif.

---

## 8) Versi cepat: jalankan semua cell berurutan

Urutan ideal:
1. Install dependency.
2. Set compiler.
3. Paste Solidity contract.
4. Compile.
5. Set secrets (`PRIVATE_KEY`, `MEMBER1_KEY`, `MEMBER2_KEY`).
6. Deploy.
7. Register member.
8. Deposit.
9. Create proposal.
10. Vote.
11. Finalize.
12. Disburse.

Selesai — Anda sudah punya prototype **Decentralized Koperasi Indonesia** yang berjalan di testnet dari Google Colab.
