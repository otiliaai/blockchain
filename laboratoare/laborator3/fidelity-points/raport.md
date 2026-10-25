# Laborator 3: Contracte Solidity (platformă de puncte de fidelitate)

**Nume:** ____________________  **Grupa:** ____________  **Data:** 25.10.2026

**Unelte:** Foundry (`forge`, `cast`, `anvil`), Solc 0.8.37, rețea locală anvil, rețeaua de test Sepolia (RPC Infura), MetaMask.

---

## 1. Pregătirea proiectului și rularea pe anvil (Step 1 și 2)

Proiectul a fost creat cu `forge init fidelity-points`, fișierele-exemplu `Counter` au fost șterse, iar `FidelityPoints.sol` și `Catalog.sol` au fost puse direct în `src/`. Compilarea (`forge build`) a reușit.

Prima greșeală întâlnită: fișierele erau copiate într-un subfolder `src/lab3_contracts/`, iar importul `../src/FidelityPoints.sol` din test nu mai găsea fișierul. Soluția a fost mutarea contractelor direct în `src/` și a testului în `test/`.

### Versiunea inițială (starter) pe anvil

- Deploy cu `--constructor-args 10`, deployer = contul 0 (`0xf39F...2266`), devenit `admin`.
- Contract: `0x5FbDB2315678afecb367f032d93F642f64180aa3`.
- Apelurile `cast call` pe getterele generate automat (`pointValue()`, `admin()`, `points(address)`) funcționează fără ca funcțiile să fie scrise în cod: sunt generate de compilator pentru variabilele `public`.
- `addPoints` apelat de admin: `status 1 (success)`. Clientul (contul 1, `0x7099...79C8`) a primit puncte.
- **Vulnerabilitate:** `setPointValue(1000)` apelat cu cheia **clientului** a reușit (`status 1`, `from 0x7099...79C8`, `gasUsed 23809`, hash `0xe7ced40c30521f8e2ce1ddce31a87056112d1687c14e7d6cb276081bbcf1700e`). Contractul reține `admin`, dar nu verifică niciodată cine apelează funcția.
- Valoarea unui punct a devenit 1000, iar `getTotalValue(CLIENT)` = 300 × 1000 = 300000.

### Citirea directă din storage (Step 2.4)

| Slot | Variabila | Valoare citită |
|---|---|---|
| 0 | `admin` | `0x...f39fd6e51aad88f6f4ce6ab8827279cfffb92266` (contul 0 anvil) |
| 1 | `pointValue` | `0x...03e8` = **1000** (`cast --to-dec`) |
| 2 | `points` (mapping) | gol; valorile stau la `keccak256(cheie, slot)`, calculat cu `cast index address $CLIENT 2` |

Concluzie: orice variabilă, chiar `internal` sau `private`, poate fi citită direct din storage. Vizibilitatea controlează doar cine poate apela din cod, nu face datele confidențiale.

### Comparație cu cei 21.000 gas ai unui transfer simplu

Un transfer simplu de ETH costă 21.000 gas. Apelurile de contract costă mai mult pentru că rulează cod și scriu în storage. Pe anvil, `addPoints` a consumat **66.997** gas (versiunea cu `totalPoints`, prima adăugare, două sloturi 0 → nenul, cca. 20.000 gas fiecare), iar `spendPoints` **32.187** gas.

Atunci când `$FP` arăta spre o adresă fără cod (anvil repornit, contractul dispăruse), tranzacțiile au avut `gasUsed` ≈ 21.000 (22.430 și 21.510) și eroarea `does not have any code`: au fost simple transferuri de ETH către o adresă goală, nu apeluri de contract.

---

## 2. Teste cu Forge (Step 3)

Testele inițiale (`test_AddPoints`, `test_SetPointValue`) au trecut (2 passed). `setUp` face `new FidelityPoints(10)`, deci contractul de test este deployer-ul și admin-ul. Comenzi rulate: `forge test`, `forge test -vvvv` (trace-ul apelurilor), `forge test --match-test test_AddPoints`, `forge test --gas-report`.

---

## 3. Exercițiul 1: extinderea contractului

### Cod final `src/FidelityPoints.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.36;

contract FidelityPoints {

    address public admin;                    // slot 0
    uint public pointValue;                  // slot 1

    mapping(address => uint) public points;  // slot 2

    uint totalPoints;                        // slot 3

    error NotAdmin(address caller);

    modifier onlyAdmin() {
        if (msg.sender != admin) revert NotAdmin(msg.sender);
        _;
    }

    constructor(uint _pointValue) {
        admin = msg.sender;
        pointValue = _pointValue;
    }

    function addPoints(address client, uint _points) public onlyAdmin {
        points[client] += _points;
        _increaseTotal(_points);
    }

    function setPointValue(uint _pointValue) public onlyAdmin {
        pointValue = _pointValue;
    }

    function getTotalValue(address client) public view returns (uint totalValue) {
        totalValue = points[client] * pointValue;
    }

    function spendPoints(uint _points) public {
        points[msg.sender] -= _points;
        _decreaseTotal(_points);
    }

    function _increaseTotal(uint s) internal {
        totalPoints += s;
    }

    function _decreaseTotal(uint s) internal {
        totalPoints -= s;
    }

    function getTotalPoints() external view returns (uint) {
        return totalPoints;
    }
}
```

### Explicații

1. **`spendPoints(uint _points)`**: scade din `points[msg.sender]`. `msg.sender` este adresa care a semnat tranzacția, deci un client își poate cheltui doar propriile puncte (rândul lui din mapping). Din Solidity 0.8, `-=` este checked: dacă `_points` depășește soldul, tranzacția dă revert cu `Panic(0x11)`.
2. **`totalPoints`**: variabilă de stare fără specificator, deci `internal` (fără getter automat), de aceea există `getTotalPoints()`. Este actualizată prin funcțiile `internal` `_increaseTotal` și `_decreaseTotal`, apelate din `addPoints` și `spendPoints`. Un mapping nu se poate parcurge, de aceea suma este ținută separat.
3. **`onlyAdmin` și eroarea `NotAdmin(address caller)`**: aplicate pe `addPoints` și `setPointValue`, nu pe `spendPoints`.

### Repetarea tranzacției din Step 2.3 pe versiunea nouă (anvil)

Contract nou pe anvil: `0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9` (hash deploy `0x229a0e3c65b4569441c548253c648558f65658bcfeb943e983a461a18a32e2f1`).

- `addPoints(CLIENT, 100)` cu adminul: `status 1`, `gasUsed 69165`, hash `0x339099e9b275fdbed5131bc408576bbf6ee5e09a229891970ccf7c1f10dc6f6c`.
- `setPointValue(1000)` cu cheia clientului: **respinsă**:

```
execution reverted: custom error 0x17a84242 ...
NotAdmin(0x70997970C51812dc3A010C7d01b50e0d17dc79C8)
```

`0x17a84242` este selectorul erorii `NotAdmin(address)`, iar argumentul este adresa clientului (`msg.sender`). Spre deosebire de Step 2.3, vulnerabilitatea este corectată. `cast` a dat „Failed to estimate gas” pentru că simularea dinaintea trimiterii a dat revert, deci tranzacția nu a mai fost trimisă.

Înainte de `onlyAdmin` (contract `0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512`), `getTotalPoints()` a dat 100 după `addPoints(CLIENT, 100)` și 70 după `spendPoints(30)` semnat de client (`from 0x7099...79C8`, `gasUsed 32187`), deci `totalPoints` se actualizează corect la adăugare și la cheltuire.

### Storage după modificări

| Slot | Variabila | Valoare |
|---|---|---|
| 2 | `points` (mapping) | `0x00...00` (gol) |
| 3 | `totalPoints` | `0x...64` = **100** |

Valoarea din mapping pentru client, citită cu `SLOT=$(cast index address $CLIENT 2)` și `cast --to-dec $(cast storage $FP $SLOT)`: **100** *(de completat cu rezultatul rulat)*.

### Răspuns: s-a schimbat slotul lui `points`?

**Nu.** `points` rămâne în slotul 2. Solidity atribuie sloturile variabilelor de stare în ordinea declarării, începând de la 0: `admin` (0), `pointValue` (1), `points` (2). `totalPoints` a fost declarat **după** `points`, deci primește slotul 3. Dacă ar fi fost declarat înaintea lui `points`, mappingul s-ar fi mutat în slotul 3. Erorile, modifierele și funcțiile nu ocupă sloturi de storage.

---

## 4. Exercițiul 2: teste

### Cod `test/FidelityPoints.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import {FidelityPoints} from "../src/FidelityPoints.sol";
import {Test, stdError} from "forge-std/Test.sol";

contract FidelityPointsTest is Test {
    FidelityPoints internal fp;
    address internal client = makeAddr("client");

    function setUp() public {
        fp = new FidelityPoints(10);   // contractul de test e deployer, deci admin
    }

    function test_AddPoints() public {
        fp.addPoints(client, 100);
        assertEq(fp.points(client), 100);
        assertEq(fp.getTotalValue(client), 1000);
    }

    function test_SetPointValue() public {
        fp.setPointValue(20);
        assertEq(fp.pointValue(), 20);
    }

    function test_SpendPoints() public {
        fp.addPoints(client, 100);
        vm.prank(client);
        fp.spendPoints(30);
        assertEq(fp.points(client), 70);
        assertEq(fp.getTotalPoints(), 70);
    }

    function test_SpendTooMuch() public {
        fp.addPoints(client, 100);
        vm.expectRevert(stdError.arithmeticError);
        vm.prank(client);
        fp.spendPoints(101);
    }

    function test_AddPointsNotAdminReverts() public {
        vm.prank(client);
        vm.expectRevert(abi.encodeWithSelector(FidelityPoints.NotAdmin.selector, client));
        fp.addPoints(client, 20);
    }
}
```

`vm.prank(client)` face ca **următorul apel** să aibă `msg.sender = client` (echivalentul semnării cu `--private-key $CLIENT_KEY` pe anvil). Fără el, `spendPoints` s-ar executa ca admin (contractul de test), care are 0 puncte.

### Rezultat

```
[PASS] test_AddPoints() (gas: 93575)
[PASS] test_AddPointsNotAdminReverts() (gas: 36741)
[PASS] test_SetPointValue() (gas: 37476)
[PASS] test_SpendPoints() (gas: 124707)
[PASS] test_SpendTooMuch() (gas: 106085)
Suite result: ok. 5 passed; 0 failed; 0 skipped
```

### Gaz (`forge test --gas-report`)

| Funcție | Min | Avg | Median | Max | # Calls |
|---|---|---|---|---|---|
| `addPoints` | 24.644 | 58.025 | 69.153 | 69.153 | 4 |
| `spendPoints` | 24.038 | 28.112 | 28.112 | 32.187 | 2 |

Interpretare: valorile maxime corespund apelurilor reușite. `addPoints` costă 69.153 pentru că prima adăugare scrie două sloturi din 0 în nenul (`points[client]` și `totalPoints`). `spendPoints` costă 32.187 pentru că modifică sloturi deja nenule. Valorile minime (24.644 și 24.038) corespund apelurilor care au dat revert (`NotAdmin`, respectiv underflow), oprite înainte de orice scriere în storage; un revert tot consumă gaz.

---

## 5. Deploy pe Sepolia (Step 4 și Exercițiul 3)

### Configurare

- `.env` cu `SEPOLIA_RPC_URL` (cheia Infura), confirmat în `.gitignore` (`grep -x ".env" .gitignore` afișează `.env`).
- `foundry.toml`: `[rpc_endpoints]` cu `sepolia = "${SEPOLIA_RPC_URL}"`.
- `unset ETH_RPC_URL`, apoi `cast chain-id --rpc-url sepolia` → **11155111** (corect).
- Semnarea se face cu un keystore criptat (`--account`), cheia privată nu este în `.env`. Contul folosit are adresa `0xd8728f3C7995C97A4fe3a4fCD477E96079F09763`; a fost alimentat cu 0,1 SepoliaETH din MetaMask (sold verificat: `0.100000000000000000`).

### Step 4: deploy inițial (versiunea starter)

- Contract: `0xFCb85c9E6d42a831637487d71CE57c15E3Cf050D`
- Hash deploy: `0xc2d60a68e9f63f58f82f7d18cf4da453d3a4179b2d2567dba3c781ba67dcb84c`
- `addPoints(deployer, 50)`: hash `0x26a916d286ceec8a5ea8ea65eb85964673e7cdcb60d21e79e1c2e85a1b94ce04`, bloc `11990525`, `gasUsed 126632`, `status 1`.
- `cast call ... points(address)` → `50`.

### Exercițiul 3: versiunea finală

| Element | Valoare |
|---|---|
| Deployer (adresa mea) | `0xd8728f3C7995C97A4fe3a4fCD477E96079F09763` |
| Adresa contractului | `0xE261DB1AfeDCC4AfeBc3e008220A64525957d311` |
| Hash deploy | `0x9ff9f7e0ac10fc3b639bb16eaa6e718a7043e44df65afdb66c0e7c906e4f8685` |
| Adresa colegei (Andra) | `0x54d29E555D699110993eDD8871fe969565591fFf` |
| Hash `addPoints(Andra, 50)` | `0xb0a9b2d2ca293c5e71f3204521f173509636a03bb29f87949c0bff50f2583ca2` |
| Bloc `addPoints` | 11990988 |
| `from` / `to` | `0xd872...9763` (eu) / `0xE261...d311` (contract) |
| `gasUsed` / preț gaz | 239.205 / 50.000.016 wei (≈ 0,05 gwei) |
| Taxă `addPoints` | ≈ 239.205 × 50.000.016 wei ≈ 0,000012 SepoliaETH |
| `cast call ... points(Andra)` | **50** |

Observații:

- `cast code $FP` fără `--rpc-url sepolia` a returnat `0x`, pentru că după `unset ETH_RPC_URL` comanda s-a uitat pe anvil-ul local. Cu `--rpc-url sepolia` a afișat bytecode-ul contractului, deci deploy-ul este confirmat.
- Tranzacția `addPoints` a reușit pentru că expeditorul este admin (deployer-ul), deci trece de `onlyAdmin`.
- Citirea punctelor cu `cast call` este gratuită (`eth_call`, execuție locală pe nod, fără tranzacție) și nu necesită SepoliaETH.

### Etherscan

Adresa contractului: https://sepolia.etherscan.io/address/0xE261DB1AfeDCC4AfeBc3e008220A64525957d311

- Tranzacția de deploy (Contract Creation): `From` = `0xd872...9763`. Bloc: ________. Taxă plătită: ________.  *(de completat din Etherscan)*
- Tranzacția `addPoints`: `From` = `0xd872...9763`, `To` = contractul. Parametrii apar ca date brute (hex), deoarece Etherscan nu are codul sursă al contractului (nu a fost verificat).

---

## 6. Exercițiile 4–7 (`Catalog.sol`)

*Nerealizate la momentul redactării acestui raport.*

---

## 7. Concluzii

- Un contract nu se modifică după deploy: orice schimbare de cod implică un deploy nou, la o adresă nouă (variabila `FP` trebuie actualizată).
- `msg.sender` este identitatea care a semnat tranzacția; un modifier ca `onlyAdmin` o verifică și respinge apelurile neautorizate cu o eroare custom.
- Ordinea declarării variabilelor de stare determină layout-ul din storage.
- Un `call` (citire) este gratuit, o tranzacție costă gaz, iar un revert tot consumă gaz dacă tranzacția ajunge în bloc.
- Tot conținutul storage-ului este public, indiferent de vizibilitate.